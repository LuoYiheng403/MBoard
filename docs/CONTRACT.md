# MBoard 开发契约

> 这份文档是并行开发的唯一事实来源。任何类型、接口、文件归属的调整，必须先在团队频道同步。

## 1. 项目定位

MBoard = Trello 式看板 + AFFiNE 式无限画布。固定账号登录（`config/accounts.yml`），所有登录用户共享同一份数据。

技术栈：Next.js 15.5 App Router + React 19 + TypeScript + Tailwind 3.4 + Prisma 6 + SQLite + Excalidraw 0.18 + Yjs 实时协作 + dnd-kit。

## 2. 目录与文件所有权

| 范围 | 负责人 | 允许修改 |
| --- | --- | --- |
| 基础设施、认证、工作台外壳、UI 组件、画布编辑器、协作服务、部署 | Lead | `app/layout.tsx` `app/page.tsx` `app/login/**` `app/(workspace)/layout.tsx` `app/(workspace)/settings/**` `app/(workspace)/my-tasks/**` `app/canvas/**` `app/api/auth/**` `app/api/account/**` `app/api/members/**` `components/ui/**` `components/workspace/**` `components/account/**` `components/brand.tsx` `components/theme-toggle.tsx` `components/canvas/editor/**` `components/canvas/markdown-view.tsx` `lib/auth/**` `lib/members.ts` `lib/http.ts` `lib/prisma.ts` `lib/types.ts` `lib/utils.ts` `lib/collab/**` `middleware.ts` `server.mjs` `prisma/**` 根配置 |
| 看板模块 | kanban-dev | `app/(workspace)/boards/**` `app/api/boards/**` `app/api/lists/**` `app/api/cards/**` `components/kanban/**` `lib/kanban.ts` |
| 画板库与资源 | canvas-dev | `app/(workspace)/canvases/**` `app/api/canvases/**` `app/api/assets/**` `app/api/documents/**` `components/canvas/library/**` `lib/assets.ts` |

禁止修改：`package.json`、`tsconfig.json`、`next.config.mjs`、`tailwind.config.ts`、`app/globals.css`、`prisma/schema.prisma`、`middleware.ts`、`components/ui/**`、`components/workspace/**`、`lib/types.ts`（需要加类型先发消息给 Lead）。

## 3. 视觉与交互规范

- 只用现有 Tailwind token：`bg/surface/surface-2/elevated/line/ink/muted/faint/brand/danger/success/warning`，不要写死颜色（Excalidraw 内部除外）
- 复用 `components/ui`：`Button` `Input` `Textarea` `Modal` `Dropdown/MenuItem/MenuDivider` `Avatar` `EmptyState` `Spinner` `ConfirmDialog` `Tip`
- 文案全中文；提示统一 `toast.success / toast.error`（sonner）
- 所有异步区域要有 loading、空状态、失败提示；按钮提交时 loading 且防重复点击
- 支持深色模式；移动端可用（看板列横向滚动、抽屉可关闭）
- 圆角 12-16px，边框 `border-line`，卡片阴影 `shadow-card`，动效克制
- 单个文件控制在 400 行以内，复杂逻辑加简洁中文注释

## 4. 通用约定

- 服务端统一从 `@/lib/prisma` 取 `prisma`；从 `@/lib/auth/session` 取 `getSession()`
- API 中间件已做登录拦截，路由内仍需 `getSession()` 拿用户做 `lastEditor` 之类的记录
- 响应格式：成功直接返回业务 JSON；失败 `{ error: string }` + 对应状态码，统一用 `@/lib/http` 的 `fail/notFound/serverError`
- Next 15 动态路由参数是 Promise：`{ params }: { params: Promise<{ id: string }> }`
- 日期统一 ISO 字符串返回（Prisma Date 经 JSON 自动序列化）
- 排序字段 `sort` 升序；新建时取当前最大值 + 1（或用末尾序号）

## 5. 类型（已定义于 `lib/types.ts`）

`AppUser` `BoardSummary` `BoardDetail` `BoardPayload` `ListDTO` `CardDTO` `CardLabel` `MemberDTO` `AttachmentDTO` `CanvasSummary` `CanvasPayload` `SceneData` `DocumentDTO` `AssetDTO` `ApiError`

附件 URL 统一为 `/api/assets/{assetId}`。

## 6. 看板 API 契约（kanban-dev 实现）

| 方法 | 路径 | 请求 | 响应 |
| --- | --- | --- | --- |
| GET | `/api/boards` | - | `{ boards: BoardSummary[] }` |
| POST | `/api/boards` | `{ title, accent?, description? }` | `{ board: BoardSummary }` 201 |
| GET | `/api/boards/[id]` | - | `{ board: BoardDetail, lists: ListDTO[] }` |
| PATCH | `/api/boards/[id]` | `{ title?, description?, accent? }` | `{ board: BoardDetail }` |
| DELETE | `/api/boards/[id]` | - | `{ ok: true }` |
| POST | `/api/boards/[id]/lists` | `{ title }` | `{ list: ListDTO }` 201 |
| PATCH | `/api/lists/[id]` | `{ title? }` | `{ list: { id, title, sort } }` |
| DELETE | `/api/lists/[id]` | - | `{ ok: true }` |
| POST | `/api/lists/[id]/cards` | `{ title, descriptionMd? }` | `{ card: CardDTO }` 201 |
| PATCH | `/api/cards/[id]` | `{ title?, descriptionMd?, labels?, assignees?, dueDate?, archived?, coverAssetId? }` | `{ card: CardDTO }` |
| DELETE | `/api/cards/[id]` | - | `{ ok: true }` |
| POST | `/api/boards/[id]/reorder` | `{ lists?: {id,sort}[], cards?: {id,listId,sort}[] }` | `{ ok: true }` |
| POST | `/api/cards/[id]/attachments` | `{ assetId }` | `{ attachment: AttachmentDTO }` 201 |
| DELETE | `/api/cards/[id]/attachments/[attachmentId]` | - | `{ ok: true }` |

规则：
- 卡片列表过滤 `archived: false`，按 `sort` 升序
- `labels` 存 JSON 字符串，接口层负责 parse/stringify
- reorder 必须校验列表/卡片属于该看板，使用事务一次性更新
- 看板详情排序：`lists.sort asc`，`cards.sort asc`

## 7. 画板库与资源 API 契约（canvas-dev 实现）

| 方法 | 路径 | 请求 | 响应 |
| --- | --- | --- | --- |
| GET | `/api/canvases` | - | `{ canvases: CanvasSummary[] }` 按 `updatedAt desc` |
| POST | `/api/canvases` | `{ title?, description? }` | `{ canvas: CanvasSummary }` 201 |
| GET | `/api/canvases/[id]` | - | `{ canvas: CanvasSummary, scene: SceneData }` |
| PATCH | `/api/canvases/[id]` | `{ title?, description? }` | `{ canvas: CanvasSummary }` |
| DELETE | `/api/canvases/[id]` | - | `{ ok: true }` |
| POST | `/api/canvases/[id]/thumbnail` | multipart `file`（PNG，≤2MB） | `{ url: string }` |
| POST | `/api/assets` | multipart `file` + `kind?` | `{ asset: AssetDTO }` 201 |
| GET | `/api/assets/[id]` | - | 文件二进制流（带 Content-Type/Cache-Control） |
| GET | `/api/documents` | `?q=` 可选 | `{ documents: Omit<DocumentDTO,'contentMd'>[] }` |
| POST | `/api/documents` | `{ title, contentMd }` | `{ document: DocumentDTO }` 201 |
| GET | `/api/documents/[id]` | - | `{ document: DocumentDTO }` |
| PATCH | `/api/documents/[id]` | `{ title?, contentMd? }` | `{ document: DocumentDTO }` |
| DELETE | `/api/documents/[id]` | - | `{ ok: true }` |

资源实现要点（`lib/assets.ts`）：
- 上传目录 `process.env.UPLOAD_DIR`（相对项目根目录），不存在时自动创建
- 文件名用 `crypto.randomUUID()` + 原扩展名保存到磁盘，数据库存 `storageKey`
- 白名单：`image/*` `video/*` `text/markdown` `text/plain` `application/pdf`，其他 400
- 单文件上限 50MB（缩略图 2MB）
- `kind` 取值：`image` | `video` | `markdown` | `file` | `thumbnail`，缺省按 mime 推断
- 读取文件时做路径安全校验，禁止 `..` 穿越
- `elementCount`：解析 `sceneJson.elements.length`，解析失败按 0

## 7.5 账号设置 API（Lead 实现）

| 方法 | 路径 | 请求 | 响应 |
| --- | --- | --- | --- |
| PATCH | `/api/account/profile` | `{ displayName }` | `{ user }` 并刷新会话 Cookie |
| POST | `/api/account/password` | `{ currentPassword, newPassword, confirmPassword }` | `{ ok: true }` |

规则：
- 账号白名单仍在 `config/accounts.yml`，个人修改写入 `UserProfile` 表并覆盖配置初始值
- 显示名会进入 JWT，用于画布协作光标与在线成员展示
- 修改密码需要校验当前密码，bcrypt 存储，最少 6 位
- 修改显示名后需要刷新画布，其他成员的光标才会显示新名字

## 7.6 成员与负责人（Lead 实现）

| 方法 | 路径 | 请求 | 响应 |
| --- | --- | --- | --- |
| GET | `/api/members` | - | `{ members: MemberDTO[] }` |

规则：
- 成员列表 = `config/accounts.yml` 账号 + `UserProfile` 里的显示名/头像色覆盖
- 卡片 `assignees` 存用户名 JSON 数组（最多 10 人），接口会校验用户名必须在配置白名单里
- 看板页支持 `?card=<id>` 直接打开卡片详情，「我的任务」页用它跳转
- 侧边栏「我的任务」显示当前用户被分配的任务数

## 8. 画布编辑器（Lead 实现，供参考）

- 路由 `/canvas/[id]`，独立全屏页面，不在工作台侧边栏布局内
- Yjs 数据：`elements`（Y.Map id→元素 JSON）、`order`（Y.Array id 顺序）、`meta`（appState）
- 图片：元素 `customData.assetId` 指向 `/api/assets/{id}`，客户端按需拉取为 dataURL 注入 Excalidraw files
- 文档卡片：元素 `customData.mbDocId`，点击后右侧面板渲染 Markdown
- 视频卡片：元素 `customData.mbVideoAssetId`，点击弹出播放器

## 9. 本地运行

```bash
npm run dev             # 默认 http://localhost:3000
PORT=3100 npm run dev   # 端口冲突时
npm run typecheck       # tsc --noEmit
npm run build           # 生产构建
```

账号：`maxy / 123456`、`tannami / 123456`、`fashi / 123456`、`tomato / 123456`（默认账号，可在设置页修改显示名和密码）。

## 10. 交付要求

1. 自己范围内的页面/接口能通过 `npx tsc --noEmit`（允许其他模块未完成导致的报错，但自己文件不能有）
2. 不引入新依赖
3. 完成后在任务里说明：新增文件清单、已实现的交互、未覆盖的点
4. 不要执行 `git commit`、不要改 `package.json`、不要删别人的文件
