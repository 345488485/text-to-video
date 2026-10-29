# FrameFlow Vben 5 前端迁移设计

## 1. 目标

将现有原生 HTML、CSS、JavaScript 前端迁移为完整的 Vben Admin 5 前端工程，同时保持现有 Python FastAPI 后端、火山方舟视频生成流程、SQLite 数据和 API 契约不变。

迁移完成后，系统应具备可维护的 Vue 3 + TypeScript 工程结构、Vben 应用布局、路由、Pinia 状态管理、主题切换和 Ant Design Vue 业务组件，并保留当前所有已经可用的业务功能。

## 2. 技术选型

- 前端框架：Vben Admin 5.7 系列
- UI 方案：Vben Ant Design Vue 应用
- 语言与构建：Vue 3、TypeScript、Vite
- 包管理与工程：pnpm 11、Vben pnpm monorepo
- 状态管理：Pinia
- 路由：Vue Router
- 后端：现有 FastAPI，不改变业务实现
- 数据：现有 SQLite 数据库，不执行数据迁移

本机 Node 24 和 pnpm 11 满足当前 Vben 工程要求。

## 3. 总体架构

项目调整为前后端分离开发、同源生产部署：

```text
video_project/
├─ backend/                 # 保留现有 FastAPI 与供应商适配器
├─ frontend/                # Vben 5 前端工程
│  ├─ apps/web-antd/        # FrameFlow 主应用
│  └─ packages/             # Vben 必需共享包
├─ public/                  # 迁移期保留的旧版前端，不继续开发
├─ data/                    # 现有 SQLite 数据
├─ tests/                   # Python 后端测试
└─ backups/                 # 迁移前备份
```

开发环境中，Vite 将 `/api` 请求代理到 `http://127.0.0.1:4173`。生产构建后，FastAPI 从 Vben 构建目录提供静态文件，并对非 API 路由回退到 `index.html`，支持 Vue Router 刷新。

## 4. 页面与路由

### 4.1 创作工作台 `/studio`

- 一句话视频描述输入与字符计数
- 示例提示词和智能润色
- 比例、时长、分辨率、声音开关
- 成本与耗时预估
- 视频任务提交、进度轮询、取消和错误提示
- 生成结果预览、下载和再次生成
- 默认保持低成本配置：720p、Fast、5 秒、关闭声音

创作表单使用 Ant Design Vue 表单组件；视频预览保留沉浸式工作台布局，不套用普通后台表格页面。

### 4.2 首页 `/dashboard`

- 可用额度
- 本月已用额度与进度
- 当前套餐
- 最近创作
- 服务状态和快捷创作入口

### 4.3 作品 `/works`

- 作品卡片或表格视图
- 视频预览
- 下载
- 删除二次确认
- 空状态和加载状态

### 4.4 套餐账单 `/billing`

- 当前套餐与套餐列表
- 套餐切换
- 本月生成账单
- 套餐变更记录
- 供应商成本核对入口

### 4.5 灵感模板 `/templates`

- 展示预设模板
- 选择模板后跳转创作页并填入提示词

### 4.6 品牌素材 `/assets`

- 本地图片选择和预览
- 文件类型与 10 MB 大小校验
- 明确标识素材只在当前浏览器会话使用，不伪装成云端持久化

## 5. 前端模块边界

- `api/`：统一封装现有 `/api/*` 请求、错误解析和 TypeScript 类型
- `stores/usage.ts`：额度和套餐状态
- `stores/generation.ts`：当前生成任务、轮询生命周期和结果
- `stores/works.ts`：作品列表和删除操作
- `views/`：各业务页面，只负责组合组件和触发 store action
- `components/studio/`：提示词编辑器、生成参数、费用预估、视频预览
- `router/routes/modules/`：FrameFlow 业务路由和菜单元数据

业务页面不直接拼接 API URL，也不在组件内部复制任务轮询逻辑。

## 6. 数据流

1. 用户在创作页编辑参数。
2. 防抖调用 `/api/generations/estimate` 更新额度与耗时。
3. 提交时生成幂等键并调用 `/api/generations`。
4. generation store 保存任务并轮询 `/api/generations/{id}`。
5. 成功后刷新额度与作品列表，页面显示视频结果。
6. 失败或取消时停止轮询，展示后端返回的可操作错误信息。

刷新页面后，本阶段不自动恢复浏览器内未完成任务；后端仍会恢复数据库中的活动任务。任务恢复能力留待后续增加任务列表 API 后实现，避免前端推测服务端状态。

## 7. 主题与品牌

- 使用 Vben 偏好设置和主题能力作为基础，不再维护独立 DOM 主题菜单。
- 保留三个品牌预设：曜石黑、云境白、东方绛。
- 默认主题为曜石黑。
- 品牌颜色通过语义化 CSS 变量覆盖，避免直接修改 Vben 核心包。
- 主题选择写入 Vben 的本地偏好存储。

## 8. API 兼容性

前端继续使用以下现有接口：

- `GET /api/health`
- `GET /api/capabilities`
- `POST /api/intent/parse`
- `POST /api/generations/estimate`
- `POST /api/generations`
- `GET /api/generations/{task_id}`
- `POST /api/generations/{task_id}/cancel`
- `GET /api/account/usage`
- `GET /api/billing`
- `POST /api/billing/plan`
- `GET /api/works`
- `DELETE /api/works/{task_id}`

不修改 API 字段命名，TypeScript 类型以当前 Pydantic 模型和实际响应为准。

## 9. 错误处理

- API 客户端统一解析 `{ error: { code, message, retryable } }`。
- 表单校验错误显示在对应字段或提交区域。
- 页面数据加载失败使用 Vben/Ant Design Vue 的提示与重试按钮。
- 任务轮询的临时网络错误自动退避重试，不立即判定任务失败。
- 取消、删除、切换套餐等写操作提供明确反馈；删除必须二次确认。
- 不在前端日志、通知或构建产物中暴露 API Key。

## 10. 部署与运行

### 开发

- FastAPI：`python run.py`，监听 4173
- Vben：从 `frontend/` 启动 Ant Design Vue 应用
- Vite 将 `/api` 代理到 FastAPI

### 生产

- 构建 Vben Ant Design Vue 应用
- FastAPI 静态托管路径切换至 Vben `dist`
- Vue Router 使用 history 回退
- `/api` 和 `/api/docs` 不进入前端回退
- 保留当前 CSP，不引入外部脚本；根据 Vben 构建产物验证是否需要调整内联样式策略

## 11. 迁移策略

1. 保留已验证的迁移前 ZIP 备份。
2. 引入官方 Vben 5 工程并固定依赖版本。
3. 保留 Vben 核心与 Ant Design Vue 应用，移除无关演示业务页面。
4. 建立 FrameFlow API 类型和业务 stores。
5. 依次迁移创作、首页、作品、账单、模板和素材页面。
6. 切换 FastAPI 静态目录至 Vben 构建产物。
7. 全量验证后停止使用旧 `public/`；本次迁移不删除旧目录。

## 12. 验收标准

- Vben 应用可安装、启动、类型检查并完成生产构建。
- FastAPI 后端测试全部通过。
- 首页、创作、作品、账单、模板和素材菜单均可进入且无死按钮。
- 视频预估、润色、提交、轮询、取消、预览、下载完整可用。
- 可用额度、本月额度、套餐账单和最近创作读取真实后端数据。
- 三种品牌主题可切换并持久化。
- 桌面端和常见移动端宽度无横向溢出或内容遮挡。
- 生产模式下直接访问任意前端路由不会返回 404。
- 浏览器控制台无未处理异常，关键操作有加载、成功和失败反馈。

## 13. 非目标

- 本次不新增登录注册、组织权限或多租户后端。
- 本次不更换 SQLite、任务队列或视频供应商。
- 本次不实现品牌素材云端上传。
- 本次不删除迁移前前端源码和备份。
