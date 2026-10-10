# OneBase 自定义目录

OneBase AI（[pirate-608/OneBase-AI](https://github.com/pirate-608/OneBase-AI)，MIT）是一个 RAG 底座：像配置静态网站一样一键构建与部署 AI 动态服务。CLI 命令叫 `onebase`，PyPI 包名 `onebase-ai`（当前 0.1.5，2026-03-07 发布），要求 Python ≥ 3.10、Docker ≥ 20.10。

`onebase init` 生成的项目骨架：

```
my-kb/
├── onebase.yml       # 核心配置
├── .env              # 数据库密码、API Key
├── requirements.txt  # onebase get-deps 生成
└── base/             # 文档目录
```

自定义目录有两条路：**方式一**改 `knowledge_base.struct`（控制前端导航树），**方式二**改 `knowledge_base.path`（控制文档物理位置）。两者可组合，改完重新 `onebase build` 生效。

> 以下标注「实测」的结论来自本机 v0.1.5 实跑（2026-10-10），与官方文档不一致处以实测为准。

## 方式一：struct 自定义导航树

`struct: default`（默认）自动扫描 `path` 目录：只认 `.md` / `.txt` / `.pdf`，忽略隐藏文件，空目录不显示，标题取文件名去掉扩展名。

改成映射即手动指定——key 是前端标题，value 是相对 `path` 的文件路径：

```yaml
knowledge_base:
  path: ./docs
  struct:
    概述: overview.md
    产品文档:
      快速入门: 产品文档/快速入门.md
    常见问题: faq.txt
```

规则（实测）：

- 手动 struct **完全覆盖**自动扫描：没列的文件不入库、不进前端导航。
- `struct` 只接受**映射（dict）**或字符串 `default`。官方文档写的列表写法（`- 概述: overview.md`）实测直接报校验错误 `Input should be a valid dictionary`，照抄会跑不起来。
- 指向不存在的文件不会中断构建：0.1.5 把缺失路径收集进 `missing_files` 却从不打印，构建日志毫无提示（官方文档称会列警告，与实测不符），路径要自己保证。
- value 支持 `../` 穿出 `path` 目录。

## 方式二：path 改物理位置

```yaml
knowledge_base:
  path: ./docs    # 默认 ./base
```

- 把 `base/` 改名为任意名称（如 `docs/`），把 `path` 指过去即可，实测自动扫描正常。
- 绝对路径同样生效（实测），文档可以放项目目录之外。
- `onebase build` 和 `onebase serve` 都没有目录参数，改目录只改 `onebase.yml` 的 `knowledge_base.path`。

## 本机实测的坑（v0.1.5）

1. **CLI 读不到项目 `.env`，数据库起不来**：`onebase build` 生成的 compose 里 `POSTGRES_PASSWORD` 是空串，postgres 容器无限重启，最终报 `Database not ready within 30s`。根因是 `db.py` 的 `load_dotenv()` 从包安装目录向上找 `.env`，永远找不到项目里的那份。绕过方式——把密码注入环境变量再构建：
   ```bash
   export POSTGRES_PASSWORD=$(grep '^POSTGRES_PASSWORD=' .env | cut -d= -f2)
   onebase build
   ```
2. **依赖要装进同一个解释器**：报 `No module named 'langchain_ollama'` 之类时，按官方流程 `onebase get-deps > requirements.txt` 后安装；若 onebase 是用 uv tool 装的，依赖也装进 tool 环境：
   ```bash
   uv pip install --python ~/.local/share/uv/tools/onebase-ai/bin/python3 -r requirements.txt
   ```
3. **PyPI 镜像缺包**：阿里云镜像缺 `pydantic_core` 的 metadata（404 卡死），清华 403、中科大 404，本机换腾讯镜像安装成功：
   ```bash
   uv tool install onebase-ai --default-index https://mirrors.cloud.tencent.com/pypi/simple
   ```

## build 需要一条向量化通道

`onebase build` = 扫描 → 切块 → 调 embedding 写进 pgvector，必须有向量模型，四选一：

| 通道 | 用法 | 代价 |
|---|---|---|
| Ollama 容器 | `onebase build --with-ollama` | 拉镜像 + 拉模型，本机 Docker Hub 镜像源很慢 |
| 云端 API | `engine.embedding.provider` 改 `openai` / `dashscope` / `zhipu` 等，`.env` 填 key | 最快，要 key |
| Xinference / vLLM | `--with-xinference` / `--with-vllm` | 比 ollama 更重 |
| Docker Model Runner | `--with-docker-model` | 要 Docker Desktop 4.40+，纯 Docker Engine 无此插件 |

本机现状：手上的 key 实测都没有向量端点（deepseek、xiaomi 的 `/embeddings` 均 404），不用 ollama 就需要一个带 embeddings 的云端 key。

## 命令速查

```bash
pip install onebase-ai                       # 官方安装方式
onebase init                                 # 生成 onebase.yml + .env + base/
onebase get-deps > requirements.txt          # 按引擎算依赖
onebase build                                # 构建向量库
onebase serve -d                             # 后台启动全栈服务（默认 :8000）
onebase stop                                 # 停止服务（-v 同时删数据卷）
```
