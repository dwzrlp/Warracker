# AGENTS.md

## 项目现状

- Warracker：自托管的保修管理应用。本仓库 `dwzrlp/Warracker` 是上游 `sassanix/Warracker` 的 fork，默认分支 `main`
- 后端 `backend/`：Flask（`create_app()` 工厂在 `backend/__init__.py`），Gunicorn + gevent 运行，PostgreSQL（psycopg2 连接池，`backend/db_handler.py`）
- 前端 `frontend/`：纯静态 HTML/CSS/JS，无构建步骤，由 Nginx 直接提供；第三方库直接内置在 `frontend/js/lib/` 与 `frontend/chart.js`
- 部署形态：单个 Docker 镜像（`Dockerfile`，Python 3.15 slim-trixie + Nginx + Supervisor），数据库为独立的 `postgres:15-alpine` 容器

## 命令

| 任务 | 命令 |
|------|------|
| 从源码构建并运行 | `cp env.example .env` 后 `docker compose up --build` → http://localhost:8005 |
| 拉取镜像运行 | 见 `Docker/docker-compose.yml` 与 `Docker/.env.example` |
| Python 语法检查 | `python -m compileall -q backend` |
| 安装后端依赖（临时 venv） | `pip install -r backend/requirements.txt`（编译 psycopg2 需要 libpq / `pg_config`） |
| 编译后端翻译 | `pybabel compile -d locales`（容器启动时 `Docker/entrypoint.sh` 会自动执行） |

仓库没有自动化测试、lint 或 CI 构建；`create_app()` 启动时必须连上 PostgreSQL。

## 参考文件

| 需要 | 文件 |
|------|------|
| 功能介绍、部署、目录、环境变量 | `README.md` |
| 环境变量全集 | `env.example`、`Docker/.env.example`、`backend/config.py` |
| 容器启动流程 | `Docker/entrypoint.sh`、`Docker/supervisord.conf`、`nginx.conf` |
| Gunicorn 内存模式 | `backend/gunicorn_config.py`（`WARRACKER_MEMORY_MODE`） |
| 依赖自动更新 | `renovate.json`、`.github/workflows/renovate.yml` |

## 关键约定

- 数据库结构变更只能新增 `backend/migrations/` 下的编号脚本，由 `apply_migrations.py` 在启动时执行；不要修改已发布的迁移
- `requirements.txt` 全部用 `==` 精确锁定；Gunicorn 需写成 `gunicorn[gevent]`，否则 gevent worker 缺 `packaging` 无法启动
- `Dockerfile` 中 Debian 包版本由 Renovate 注释维护，改版本时保持 `# renovate:` 注释与 trixie 实际版本一致
- 前端新增文案：在 `locales/en/translation.json` 加 key，并同步其他语言目录；语言代码使用 `zh_CN`、`zh_HK` 这种下划线形式（见 `frontend/js/i18n.js`）
- 更新 `frontend/js/lib/` 或 `frontend/chart.js` 时，同步修改 HTML 中的 `?v=` 版本号与 `frontend/sw.js` 的 `CACHE_NAME`，否则浏览器和 Service Worker 会继续使用旧文件
- Apprise 的 `notify()` 返回 `AppriseResult`，对外接口需要 `bool` 时显式转换

## 注意事项与禁区

- 不要提交 `.env`，不要打印其中的密钥；`SECRET_KEY` 至少 32 字节
- 不要随意升级 `postgres` 镜像大版本：已有数据卷需要 `pg_upgrade` 或导出导入，直接换镜像会导致数据库无法启动
- PR 只开在 `dwzrlp/Warracker`（`gh pr create --repo dwzrlp/Warracker`），不要开到上游 `sassanix/Warracker`
- 与上游同步时注意 `README.md`、`locales/` 等容易冲突的文件

## Git

- 所有 Git 相关文字一律使用中文：commit、分支说明、PR 标题与正文、issue、code review 评论
- commit 格式：`<type>(<scope>): <中文描述>`，如 `fix(backend): ...`
- 禁止在 commit、PR、代码注释中加入任何 AI 署名
