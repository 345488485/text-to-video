# FrameFlow · AI 视频创作工作台

将一句话创意转化为可追踪、可预览、可下载的视频任务。

FrameFlow 面向个人创作者与营销团队，提供中文创作工作台、异步视频生成、多供应商配置与本地额度账本，便于开展产品验证和业务集成。

> **项目状态：MVP / 本地单用户版本。** 当前不是开箱即用的多租户收费 SaaS。正式对外运营前，需要补齐身份认证、权限隔离、支付、内容治理及生产基础设施，详见下方「商业部署边界」。

## 核心能力

| 模块 | 当前能力 |
| --- | --- |
| 创作工作台 | 中文提示词、提示词润色、画幅/时长/清晰度选择、额度预估 |
| 任务管理 | 异步创建、状态查询、进度展示；取消能力取决于供应商 |
| 视频资产 | 成功作品入库、视频预览与下载 |
| 额度账本 | SQLite 持久化账户、任务、额度预留/结算/释放及账单流水 |
| 供应商接入 | 火山方舟、通义万相、Runway、Luma；可在设置页配置与切换 |
| 本地演示 | Mock 供应商，不调用真实视频生成接口 |
| 界面体验 | Vben Admin 工作台、响应式布局、三套可切换主题 |

**能力说明：** Mock 仅用于流程演示和测试，不会根据提示词生成真实视频。套餐切换与额度账本属于本地业务能力，不代表已接入真实支付或订阅系统。

## 技术栈

- **前端：** Vue 3、TypeScript、Ant Design Vue、Vben Admin 5.7、Vite。
- **后端：** Python 3.11+、FastAPI、Uvicorn。
- **数据：** SQLite，默认位置为 `data/frameflow.db`。
- **工程：** uv、pnpm、Python unittest、Vitest 与浏览器验收脚本。

## 快速开始

### 1. 获取代码

```bash
git clone https://github.com/345488485/text-to-video.git
cd text-to-video
```

准备 Python 3.11+ 和 [uv](https://docs.astral.sh/uv/)。首次体验无需视频平台 API Key。

### 2. 启动本地演示

```bash
uv sync
uv run python run.py
```

打开 <http://127.0.0.1:4173>。未配置密钥和已保存供应商时，默认使用 Mock。

后端优先托管 `frontend/apps/web-antd/dist`；未构建 Vben 前端时，回退到 `public/` 中的原版界面。要使用完整 Vben 工作台及供应商设置页，请继续构建前端。

### 3. 构建完整前端

需要 Node.js `^22.18.0 || ^24.0.0`，项目固定使用 pnpm `10.33.4`。

```bash
cd frontend
pnpm install --frozen-lockfile
pnpm run build:antd
cd ..
uv run python run.py
```

开发模式：在项目根目录启动后端，再另开终端执行：

```bash
cd frontend
pnpm run dev:antd
```

页面地址以 Vite 输出为准；`/api` 请求代理到本机 `4173` 端口。

## 接入真实视频服务

### 方式一：配置火山方舟环境变量

将 `.env.example` 复制为 `.env`，填写服务端密钥后重启后端：

```dotenv
ARK_API_KEY=YOUR_ARK_API_KEY
VIDEO_PROVIDER=auto
```

`auto` 在存在方舟环境密钥时选择真实服务，否则使用 Mock；已保存的供应商激活配置可能覆盖这一选择。模型 ID 和服务地址的仓库默认值见 `.env.example`，请按账号权限核验，不保证所有账号均可调用。

### 方式二：通过工作台配置

在完整 Vben 前端访问 `/settings/video-providers`，选择平台、保存密钥并设为默认。

- 支持火山方舟（北京）、通义万相（北京）、Runway、Luma。
- 网页保存密钥使用 **Windows DPAPI**，绑定当前 Windows 用户；不可将这部分能力视为跨平台密钥管理方案。
- 连接测试只查询只读信息，不创建视频，也不保证模型权限或余额充足。
- 平台参数及取消能力存在差异，创作选项以当前集成为准。

> 真实生成可能产生供应商费用。界面额度是内部记账单位，不等于供应商报价；请先设置供应商预算与告警。不要将密钥提交到仓库、写入前端或截图公开。

详细说明：[API 配置](docs/API_CONFIGURATION.md) · [供应商能力与密钥管理](docs/VIDEO_PROVIDERS.md)。

## 常用配置

| 变量 | 默认值 / 用途 |
| --- | --- |
| `HOST` | `127.0.0.1`，默认仅本机访问 |
| `PORT` | `4173` |
| `DATABASE_PATH` | `data/frameflow.db` |
| `VIDEO_PROVIDER` | `auto`；也支持 `mock`、`volcengine` |
| `ARK_API_KEY` | 火山方舟服务端 API Key |
| `ARK_MODEL_ID` / `ARK_FAST_MODEL_ID` | 标准 / 快速模型 ID，参见 `.env.example` |
| `ARK_WATERMARK` | `true` |
| `FRAMEFLOW_FRONTEND_DIR` | 可选，指定前端构建目录 |
| `VIDEO_SETTINGS_PATH` | 可选，覆盖供应商加密配置文件位置 |

已保存密钥与环境变量是两套配置来源；删除网页配置不会删除环境变量。迁移前请阅读供应商文档中的 Windows 用户绑定与备份限制。

## API 与开发验证

服务启动后：

- 健康检查：<http://127.0.0.1:4173/api/health>
- 交互式 API 文档：<http://127.0.0.1:4173/api/docs>
- OpenAPI：<http://127.0.0.1:4173/api/openapi.json>

后端检查（项目根目录）：

```bash
uv sync --extra dev
uv run python -m compileall -q backend tests run.py
uv run python -m unittest discover -s tests -v
```

前端检查（`frontend/`）：

```bash
pnpm --filter @vben/web-antd typecheck
pnpm exec vitest run --config apps/web-antd/vitest.frameflow.config.ts
pnpm run build:antd
```

浏览器验收可使用 `uv run pythontests/run_mock_server.py` 启动隔离的 Mock 服务（端口 `4174`），避免调用付费生成接口。上述命令是开发入口，不代表每次提交都已完成全部验收。

## 项目结构

```text
backend/                  FastAPI 接口、数据存储、供应商路由与适配器
frontend/apps/web-antd/    Vben 创作工作台
public/                   未构建时使用的原版前端
tests/                   后端测试与浏览器验收脚本
docs/                    开发、配置及发布文档
run.py                    本地服务入口
.env.example              环境变量示例
```


