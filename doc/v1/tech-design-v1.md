# Todo List V1 技术设计说明

## 文档信息

| 项目     | 内容                                                               |
| -------- | ------------------------------------------------------------------ |
| 文档名称 | Todo List V1 技术设计说明                                          |
| 文档类型 | V1 实施版                                                          |
| 当前版本 | 0.3                                                                |
| 更新时间 | 2026年5月9日                                                       |
| 上游输入 | [prd-v1.md](./prd-v1.md)                                           |
| 说明     | 面向开发落地，覆盖模块划分、数据结构、拖拽实现、本地存储和时间规则 |

---

## 1. 文档目标

- 将 PRD 中的交互规则、数据规则和展示规则落到 Vue 3 Web 端的实现方案。
- 固定 V1 的模块边界、状态组织、本地存储结构与关键算法，减少开发期临时决策。
- 为 T2 至 T10 的工程开发提供直接可执行的实现依据。

## 2. 本期实现范围

### 2.1 本期功能范围

- 实现任务列表加载与本地回显。
- 实现搜索、筛选、统计与空状态展示。
- 实现新增、编辑、切换完成状态、删除。
- 实现拖拽排序，包含“全部任务视图”和“仅未完成视图”两种场景。
- 实现截止时间展示、相对日期文案与逾期判断。

### 2.2 明确不做的能力

- 不接入服务端，不做远端同步与多端一致性。
- 不实现移动端专项适配。
- 不接入第三方 UI 组件库，不单独创建独立 `utils` package。
- 不实现登录注册、子任务、标签、提醒通知、统计分析等 V1 范围外功能。

### 2.3 技术边界

- 仅支持桌面宽屏浏览器体验。
- 状态管理使用 Vue 3 Composition API + composables，不引入 Pinia。
- 持久化仅使用浏览器 `localStorage`。
- 拖拽能力使用 `vue.draggable.next`，不再评估其他拖拽库。

## 3. 技术栈与选型

| 项目       | 选型                        | 说明                                                 |
| ---------- | --------------------------- | ---------------------------------------------------- |
| 前端框架   | Vue 3                       | 使用 Composition API 组织状态与视图                  |
| 语言       | TypeScript                  | 约束任务模型、存储结构和派生逻辑                     |
| 构建工具   | Vite                        | 作为 Web 应用开发与构建工具                          |
| Monorepo   | pnpm workspace + Turborepo  | 统一管理前端应用与后续扩展项目                       |
| 状态组织   | composables                 | `useTasks` 维护源数据，`useTaskFilters` 维护派生状态 |
| 表单校验   | vee-validate + Zod          | 处理新增/编辑表单校验与回显                          |
| 时间处理   | date-fns                    | 处理格式化、周边界判断、逾期判断                     |
| 拖拽方案   | vue.draggable.next          | 用于列表拖拽交互与顺序调整                           |
| 界面样式   | 原生 HTML + `@picocss/pico` | 满足 V1 轻量 UI 需求                                 |
| 持久化方案 | localStorage                | 满足本地持久化与刷新恢复要求                         |

## 4. 系统结构

- App：负责应用初始化、读取存储、挂载 Toolbar、TaskList、统计区域与弹窗。
- Toolbar：负责“添加任务”入口、搜索输入和“显示已完成”开关。
- TaskList：负责渲染当前可见任务列表，并在非搜索状态下接入 `vue.draggable.next`。
- TaskItem：负责单条任务的状态切换、优先级展示、截止时间展示和悬浮操作按钮。
- TaskModal：负责新增/编辑任务弹窗，统一复用表单结构和校验逻辑。
- useTasks：负责维护任务源数组，封装新增、编辑、状态切换、删除、排序和落盘逻辑。
- useTaskFilters：负责搜索、筛选、统计和空状态等派生逻辑。
- storageService：负责读取、校验、序列化和写入 `localStorage`，屏蔽存储细节。
- timeUtils：负责相对日期文案、时间格式化和逾期判断。

## 5. 数据模型

V1 使用单一任务数组作为源数据，列表展示和统计都由该数组派生。

```ts
export type TaskStatus = 'todo' | 'done'

export type TaskPriority = 'high' | 'medium' | 'low' | null

export type Task = {
  id: string
  title: string
  status: TaskStatus
  priority: TaskPriority
  dueDate: string | null
  dueAt: number | null
  createdAt: number
  updateAt: number
  completedAt: number | null
  sortOrder: number
}

export type TaskStoragePayloadV1 = {
  schemaVersion: 1
  tasks: Task[]
}
```

### 5.1 存储结构

- 本地存储 key 固定为 `todolist:web:tasks`。
- 存储层使用稳定 key + 顶层 `schemaVersion`，不在单条任务内写版本字段。
- `priority`、`dueDate`、`dueAt`、`completedAt` 在存储层统一用 `null` 表示空值，不省略字段。
- V1 暂不实现自动迁移；当读取失败、`schemaVersion` 缺失或不是 `1` 时，按不支持的旧数据处理，返回空数组并忽略旧值。

### 5.2 表单模型

```ts
export type TaskFormValues = {
  title: string
  priority: TaskPriority
  dueDate: string | null
  dueAt: number | null
}
```

### 5.3 字段约束

- `title` 必填，最长 100 字符。
- `dueDate` 和 `dueAt` 互斥，不能同时存在。
- `dueDate` 仅在用户选择日期但未选择具体时间时写入。
- `dueAt` 仅在用户选择具体时间时写入。
- `updateAt` 在新增、编辑、切换状态时更新，排序时不更新。

## 6. 状态与数据流

1. App 启动时调用 `storageService.loadTasks()` 读取 `todolist:web:tasks`。
2. `storageService` 解析 JSON，校验 `schemaVersion === 1` 及任务结构是否合法。
3. 读取成功后，`useTasks` 持有完整任务数组作为单一数据源，并按 `sortOrder` 降序整理初始顺序。
4. `useTaskFilters` 基于源数据派生 `visibleTasks`、`todoCount`、`totalCount`、`isSearching` 和空状态文案。
5. 用户执行新增、编辑、切换状态、删除或拖拽排序时，统一通过 `useTasks` 修改源数据。
6. 每次有效变更后，`useTasks` 立即调用 `storageService.saveTasks({ schemaVersion: 1, tasks })` 落盘。
7. 视图层根据最新派生结果重新渲染列表、统计和空状态。

## 7. 核心规则落地

### 7.1 排序规则

- 列表按 `sortOrder` 降序展示，值越大越靠前。
- 新增任务时，`sortOrder = 当前最大 sortOrder + 1`，保证新任务插入最前。
- `TaskList` 使用 `vue.draggable.next` 包裹可见任务列表，`item-key` 使用 `id`。
- 搜索状态下禁用拖拽，直接将 `draggable` 组件设置为 `disabled`。
- 全部任务视图下，拖拽完成后以当前可见顺序作为完整任务顺序，统一重算 `sortOrder`。
- 默认视图下，拖拽只展示未完成任务，但排序结果必须回写到完整任务列表：
  - 先取当前完整任务列表和当前可见的未完成任务子序列。
  - 根据 `vue.draggable.next` 返回的新未完成任务顺序，替换完整列表中原未完成任务的相对位置。
  - 已完成任务在完整列表中的相对顺序保持不变。
  - 替换完成后按完整列表从前到后统一重算 `sortOrder`。
- 若拖拽目标无效、任务 id 缺失或排序结果和原顺序一致，则本次拖拽不落盘。

### 7.2 搜索与筛选规则

- 搜索输入为实时生效，不设置独立提交按钮。
- 搜索仅作用于 `title` 字段，匹配时先 `trim`，再执行大小写不敏感的模糊匹配。
- 默认仅展示 `status = todo` 的任务。
- 勾选“显示已完成”后展示全部任务。
- 空状态优先级固定为：搜索无结果 -> 所有任务已完成 -> 请添加任务。
- 搜索状态下不允许拖拽，但删除和状态切换仍可执行；编辑按钮仍仅对未完成任务展示。

### 7.3 时间字段规则

- `dueDate` 和 `dueAt` 互斥存储；若同时出现，视为非法数据。
- `dueDate` 只展示日期文案；`dueAt` 展示“日期文案 + 空格 + `HH:mm`”。
- 日期文案按以下优先级判断，越靠前优先级越高：
  - 昨天、今天、明天：直接显示“昨天”“今天”“明天”。
  - 本周内其他日期：显示“周一”到“周日”。
  - 今年内其他日期：显示 `M月d日`。
  - 跨年日期：显示 `yy年M月d日`。
- 本周边界按周一开始、周日结束处理；实现时使用 `date-fns` 配合 `weekStartsOn: 1`。
- 时间部分始终使用 24 小时制 `HH:mm`，例如 `06:05`。
- `dueDate` 的逾期判断以当天 `23:59:59.999` 为边界；当前日期晚于该日期时视为逾期。
- `dueAt` 的逾期判断以时间戳本身为边界；`dueAt < now` 时视为逾期。
- 未逾期且日期文案为“今天”的截止信息显示蓝色；逾期时显示红色。

### 7.4 状态切换规则

- `todo -> done`：写入当前时间到 `completedAt` 和 `updateAt`。
- `done -> todo`：将 `completedAt` 置空，并更新 `updateAt`。
- 状态切换不改变 `createdAt` 和 `sortOrder`。
- 已完成任务不展示编辑按钮，也不允许进入编辑流程。

## 8. 异常与边界处理

| 场景                 | 处理策略                                                               |
| -------------------- | ---------------------------------------------------------------------- |
| 本地存储为空         | 返回空数组并展示“请添加任务”                                           |
| 本地存储数据损坏     | JSON 解析失败、字段结构不合法或 `schemaVersion` 不支持时，按空数组处理 |
| 截止时间非法         | 表单保存前阻止提交；存量非法数据在读取时过滤或按无截止时间处理         |
| 选择了过去的 `dueAt` | 表单校验失败，不允许保存                                               |
| 拖拽目标无效         | 忽略本次排序变更，不修改源数据也不落盘                                 |
| 重复任务 id          | 视为数据损坏，读取时丢弃重复项并在开发期通过日志暴露                   |

## 9. 技术决策记录

| 决策项         | 当前选择                       | 原因                                                |
| -------------- | ------------------------------ | --------------------------------------------------- |
| 前端状态组织   | Composition API + composables  | 规模适中，满足 V1 且便于后续抽离                    |
| 拖拽库         | `vue.draggable.next`           | 与 Vue 3 适配成熟，能直接承载列表拖拽交互           |
| 排序方向       | `sortOrder` 降序               | 与 PRD 保持一致，值越大越靠前                       |
| 本地存储 key   | `todolist:web:tasks`           | 业务域、端类型与数据对象明确，后续可以保持 key 稳定 |
| 存储版本策略   | 顶层 `schemaVersion: 1`        | 便于后续做数据迁移，并避免和应用版本混淆            |
| 相对时间周边界 | 周一到周日                     | 与中文语境和常见工作周口径一致                      |
| 搜索方式       | 标题实时模糊匹配，大小写不敏感 | 与 PRD 规则一致，实现成本最低                       |

## 10. 技术风险与应对

- 默认视图拖拽要把“可见未完成任务顺序”映射回“完整任务数组”，实现复杂度最高。应对方式是先抽出纯函数并补单元测试，再接入 `vue.draggable.next`。
- `localStorage` 仅识别 `schemaVersion = 1`，后续改结构时必须补迁移函数。应对方式是在 storage 模块内预留版本分支入口。
- 不使用组件库会增加弹窗可用性和交互细节成本。应对方式是提前列出焦点管理、关闭方式和按钮态检查清单。
