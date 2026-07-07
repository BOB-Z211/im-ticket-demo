# IM 工单系统 Demo 后端对接审计

版本：2026-07-07 初稿  
范围：基于当前 `work/im-ticket-demo/index.html` 静态 demo 梳理。  
目标：把 demo 功能映射到已有后端能力；后端已有的先接入，后端没有的列为接口/逻辑缺口。

## 使用方式

这份文档先作为本地评审稿。你确认结构和覆盖范围后，再搬到飞书文档协作。

建议评审时逐项标记：

- `支持`：后端已有接口和逻辑，前端只需要 adapter。
- `部分支持`：后端有类似接口，但字段、状态、聚合或写操作不完整。
- `不支持`：后端没有对应业务事实或动作接口。
- `待确认`：需要后端同学确认接口位置、字段、权限或状态机。

## 结论先行

当前 demo 不应该直接把所有业务规则写进前端。重构接入时应优先做三件事：

1. 把 demo 功能拆成「可直接接入」「可拼接接入」「需要补接口」「继续 mock」四类。
2. 在前端加一层 API adapter，把当前静态数据和真实接口隔离。
3. 后端负责业务事实、权限、状态流转、审计日志和 AI/自动化结果落库；前端负责展示、输入体验、临时 UI 状态和乐观反馈。

## 功能模块总览

| 一级模块 | 当前 demo 覆盖内容 | 对接复杂度 | 接入建议 |
| --- | --- | --- | --- |
| 导航与工作台 | IM/EDM 分组、Sessions/User Chat、Dashboard 切换、侧栏折叠 | 低 | 多数为前端路由和权限菜单；后端提供菜单权限即可 |
| 会话列表 | Session/To-do 双 tab、状态筛选、搜索、会话卡片、未读、风险图标、品牌来源 | 中 | 先接现有会话列表接口；缺少字段用 adapter 补齐 |
| 会话主区 | 头部用户、响应时间、消息流、日期分隔、系统消息、订单号识别、交接卡片 | 高 | 消息流和系统事件必须以后端为准；交接卡可先 mock 后接 |
| Composer 输入区 | 文本输入、发送、Shift+Enter、双 Esc 清空、emoji、附件、产品卡、历史、AI 优化 | 中高 | 输入交互前端负责；发送、附件、产品卡、AI 需要后端能力 |
| 快捷回复 | `/` 触发、变量替换、键盘选择、抽屉管理、添加/编辑/删除、语言切换 | 中 | 模板配置后端化；联想交互前端化 |
| AI 辅助 | AI 生成回复胶囊、展开/收起、点赞/点踩、复制、使用建议 | 高 | AI 服务或后端 orchestration 返回建议；反馈需要埋点/保存 |
| 右侧 User Info | 基础资料、ID/系统版本复制、注册时间、最近在线、风险标识 | 中 | 优先接用户画像/客户详情接口 |
| User OA Orders | RDO/RSO 分组、订单卡、状态标签、筛选、绑定/解绑、详情抽屉 | 高 | 应接现有 OA/订单接口；绑定状态和订单状态以后端为准 |
| OA 创建/详情 | 创建 OA、订单号校验、截图上传、Review Info、Refund Info、提交、编辑 | 高 | 表单校验可前端做；订单校验、创建、编辑、上传、权限必须后端 |
| Tools 面板 | Search All、Quick Replies、Product Recommendations、To-do List 等入口 | 中 | 入口前端；各工具数据源按功能接入 |
| Conversation History | Chat Log、Key Events、点击关键节点滚动、高亮、时间轴 | 高 | Chat Log 和 Key Events 后端事件化；滚动高亮前端负责 |
| Product Recommendation | 产品任务列表、优先级/国家筛选、多选、发送产品卡 | 中高 | 产品任务和发送记录后端化 |
| Dashboard | 数据概览、工作量、质量、考勤、筛选、图表、排序 | 中高 | 目前为模拟数据；生产应接报表接口或 BI 数据源 |
| 国际化和本地状态 | 英/中/日文案、localStorage、多 tab 同步、toast、复制 | 低 | 前端为主；用户语言偏好可后端保存 |

## 功能清单与后端映射

### 1. 导航、权限与工作区

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| 左侧主导航 IM/EDM | 固定菜单，IM 展开，EDM 展开 | 渲染菜单、折叠状态 | 返回可见菜单、权限、默认入口 | 待确认 | 是否已有 RBAC 菜单接口 | P1 |
| Team/Agents/Sessions/User Chat | 点击 Team 进入 Dashboard；Sessions 当前页 | 路由切换、active 状态 | 菜单权限和页面权限 | 待确认 | 需要确认重构系统路由设计 | P1 |
| 侧栏折叠 | 前端视觉状态 | 保存 UI 状态 | 可选保存用户偏好 | 待确认 | 不阻塞后端对接 | P3 |
| 顶部语言/在线状态/用户名 | 静态展示 | 语言切换、在线状态 UI | 用户身份、在线状态、偏好 | 待确认 | 当前用户信息接口 | P1 |

### 2. 会话列表与 To-do

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| Session 列表 | 显示来源、姓名、ID、最近消息、时间、状态、未读 | 渲染、筛选、选中态 | 提供会话分页、排序、未读、状态、最后消息 | 待确认 | 是否已有 IM 会话列表接口 | P0 |
| 会话状态筛选 | All/Pending/Serving/Converting/Unsolved/New/Replace/Closed | 前端传筛选条件 | 按状态查询/聚合数量 | 待确认 | 状态枚举是否与后端一致 | P0 |
| 搜索类型 | User ID/Name/Keyword | 输入、关键词高亮 | 按用户 ID/姓名/消息关键词搜索 | 待确认 | Keyword 是否支持跨消息检索 | P1 |
| 未读红点 | 最后一条非客服消息时显示 | 展示未读数量 | 计算未读、已读回执 | 待确认 | 是否有 read cursor/read_at | P0 |
| 品牌/来源 | US-EINSEO、JP-Yavaota 等 | 展示品牌图标 | 返回渠道/店铺/品牌 | 待确认 | 来源字段标准化 | P0 |
| 风险图标 | Klaus 等显示风险标记 | 展示 | 返回风险等级/风险标签 | 待确认 | 风险来源、更新时间、可解释文案 | P1 |
| To-do tab | RSO/RDO 分组待办、折叠、点击进入会话 | 展示和切换 | 返回待办类型、数量、关联会话/订单 | 待确认 | To-do 是否来自 OA 工作流还是 IM 任务 | P1 |
| To-do 分组 | Pending Payment/Pending Review 等 | 分组展示 | 返回分组、状态和任务数 | 待确认 | 状态枚举和优先级规则 | P1 |
| 联系人/在线搜索 | Search All drawer 里展示在线/离线 | 搜索 UI | 用户/联系人搜索、在线状态 | 待确认 | 在线状态接口 | P2 |

### 3. 会话头部和状态动作

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| 会话头部用户信息 | 头像、姓名、风险、响应时间 | 展示 | 返回当前会话用户和 SLA 指标 | 待确认 | First Response/Last Response 指标来源 | P1 |
| Destination account ID | 展示 2016 并可复制 | 展示/复制 | 返回目标账号/店铺/客服账号 | 待确认 | 字段命名与业务含义 | P1 |
| 状态下拉 | Pending/Serving 等动作后更新状态 | 展示动作菜单、触发请求 | 校验权限、变更会话状态、写系统事件 | 待确认 | 状态机、允许动作、关闭/转接规则 | P0 |
| 设置菜单 | Tag/Block/Unsub/Add to Contacts | 展示入口 | 标签、拉黑、退订、联系人逻辑 | 待确认 | 是否已有用户运营接口 | P2 |
| 高风险弹窗 | 高风险会话首次提示，确认后显示 banner | 展示、确认状态 | 返回风险命中、详情、是否强提醒 | 待确认 | 风险提示是否需审计确认 | P1 |

### 4. 消息流

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| 文本消息 | 客户/客服气泡、头像、作者、时间 | 渲染、滚动、输入发送 | 返回消息列表、发送消息、消息 ID、发送状态 | 待确认 | 消息分页、游标、发送失败重试 | P0 |
| 系统消息 | 居中展示 `事件 · 时间` | 渲染 | 后端写入事件消息 | 待确认 | 系统事件类型枚举 | P0 |
| 日期分隔 | Jun 22 等 | 前端根据时间分组 | 返回准确时间戳 | 待确认 | 时区标准 | P1 |
| AI 生成消息标记 | 气泡旁 Generated 标签 | 展示 | 返回消息来源/生成标记 | 待确认 | AI 消息是否可追溯 prompt/model | P2 |
| Amazon Order ID 识别 | 自动识别订单号，3 秒后 ready/risk | 可做前端识别和 loading | 后端校验订单/风控结果 | 待确认 | 应提供 order risk check 接口 | P0 |
| 点击可用订单号创建 OA | ready 订单号可点开创建抽屉 | 触发创建表单 | 校验订单、返回商品/国家/SKU | 待确认 | 订单号校验和预填充接口 | P0 |
| 交接卡片 | Handover Card、风险标签、下一步、禁忌、确认接手 | 渲染卡片、按钮交互 | 生成卡片、保存接手人/时间、写系统事件 | 待确认 | 是否已有交接/转派实体 | P0 |
| 交接确认系统行 | 点击后显示 `Transferred from Noimz · 时间` | 渲染系统行 | 后端生成系统事件并返回 | 待确认 | 接手来源、接手人、时间戳以后端为准 | P0 |
| 消息右键菜单 | Copy/Translate/Mark as risk | 展示菜单 | 翻译、风险标记、消息操作审计 | 待确认 | Translate/Mark risk 是否真实支持 | P2 |
| 多 tab 同步 | localStorage storage 事件刷新消息 | demo 状态同步 | 真实环境由接口/推送同步 | 不适用 | 需实时方案替换 localStorage | P0 |

### 5. Composer 输入与发送

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| 文本输入 | contenteditable、多行、placeholder | 输入体验 | 无 | 前端支持 | 无后端依赖 | P0 |
| Enter 发送 / Shift+Enter 换行 | 发送文本消息 | 键盘处理、禁用态 | 创建消息、返回消息 ID/状态 | 待确认 | 发送接口和幂等 key | P0 |
| 双 Esc 清空 | 清空输入框 | 前端快捷键 | 无 | 前端支持 | 无后端依赖 | P3 |
| Emoji | emoji popover 插入 | 前端输入 | 无 | 前端支持 | 无后端依赖 | P3 |
| 附件 | 选择文件并显示附件 tray | 文件选择、预览 | 上传文件、返回 file_id/url、发送附件消息 | 待确认 | 文件上传接口、大小/类型限制 | P1 |
| 产品卡按钮 | 打开产品任务弹窗 | 展示选择 | 返回可推荐产品/任务，发送卡片消息 | 待确认 | 产品卡消息 schema | P1 |
| 历史按钮 | 打开 Conversation History | 弹窗交互 | 返回历史消息和关键事件 | 待确认 | 历史会话/事件接口 | P1 |
| AI 优化按钮 | 展示 AI 生成建议，可发送 | UI 触发和使用建议 | 调用 AI 生成、保存反馈 | 待确认 | AI 服务接口和埋点 | P1 |

### 6. 快捷回复

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| `/` 触发快捷回复 | Composer 内输入 `/hi` 弹出候选 | 联想、选中态、键盘上下/回车 | 返回模板列表 | 待确认 | 快捷回复模板接口 | P1 |
| 变量替换 | `{customer_name}`、`{company_name}` 替换为实际值 | 根据当前上下文替换展示/填入 | 提供模板变量和客户上下文 | 待确认 | 变量白名单和缺失回退 | P1 |
| Quick Replies 抽屉 | Team/My tabs、搜索、分组折叠 | 展示、搜索、折叠 | 返回团队/个人模板、权限 | 待确认 | 模板归属和权限 | P1 |
| 新增快捷回复 | Add Quick Replies modal | 表单、校验 | 创建模板/分类 | 待确认 | 创建接口、审核/共享范围 | P2 |
| 编辑/删除快捷回复 | Edit modal | 编辑 UI | 更新/删除模板 | 待确认 | 权限、审计、团队模板不可随意删 | P2 |
| 多语言模板 | en/zh/ja 切换 | 语言筛选和展示 | 返回语言版本或按语言查询 | 待确认 | 模板国际化策略 | P2 |

### 7. AI 辅助

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| AI 建议胶囊 | 生成中/可展开/可收起 | 状态展示 | 返回建议内容、来源消息、生成状态 | 待确认 | AI suggestion 接口 | P1 |
| 使用建议发送 | Send (Tab+Enter) | 把建议填入或发送 | 创建消息并标记 AI 来源 | 待确认 | AI 消息审计字段 | P1 |
| 点赞/点踩 | feedback 按钮 | 交互和选中 | 保存反馈、用于模型评估 | 待确认 | feedback 接口 | P2 |
| 复制建议 | Copy to input/clipboard | 前端复制 | 可选埋点 | 待确认 | 是否需要记录复制行为 | P3 |
| AI 交接总结 | 交接卡片 issue/action/don'ts/risk | 展示结果 | AI 总结、人工确认、落库 | 待确认 | Handover summary 生成/保存流程 | P0/P1 |

### 8. Conversation History

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| Chat Log | 弹窗左侧按用户/客服/AI/系统样式展示 | 渲染和滚动 | 返回历史消息完整列表 | 待确认 | 历史消息分页和类型 | P1 |
| Key Events | 弹窗右侧时间轴 | 展示、点击 | 返回事件列表或从事件日志聚合 | 待确认 | Key Event 生成规则 | P1 |
| 点击事件定位消息 | 右侧节点点击左侧滚动并闪烁 | 前端交互 | 返回事件与消息关联 id/time | 待确认 | event.message_id 或 event.anchor_time | P1 |
| 时间轴连线 | 圆点中心连线 | 前端样式 | 无 | 前端支持 | 无 | P3 |
| OK 关闭 | 关闭弹窗 | 前端 | 无 | 前端支持 | 无 | P3 |

### 9. 右侧 User Info

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| 基础资料 | 头像、姓名、ID、国家、性别、年龄 | 展示 | 返回客户资料 | 待确认 | 用户详情接口字段 | P0 |
| 等级/粉丝 | Lv/Fans | 展示 | 返回用户画像指标 | 待确认 | 这些指标是否仍保留 | P2 |
| 设备/版本 | iOS + app version | 展示/复制 | 返回设备环境 | 待确认 | 设备字段来源 | P1 |
| 注册/最近在线 | Signup Date/Last Seen | 展示 | 返回时间戳 | 待确认 | 时区和格式 | P1 |
| 风险标识 | 用户名旁风险 icon | 展示 | 返回风险标签/等级/原因 | 待确认 | 风险解释字段 | P1 |

### 10. User OA Orders

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| OA 订单分组 | RDO/RSO 分组、数量 | 展示和折叠 | 返回订单类型分组和 count | 待确认 | RDO/RSO 定义和接口 | P0 |
| 订单卡片 | 商品图、标题、型号、时间、品类、标签、订单号 | 展示 | 返回订单、商品、关联关系和状态 | 待确认 | 数据字段映射 | P0 |
| 订单关系标签 | Customer Orders/Pending Match/Linked Orders | 展示 | 判断当前会话/用户与订单关系 | 待确认 | 绑定态和用户历史态需分清 | P0 |
| 业务状态标签 | Pending Review/Pending Payment/Cancelled/Review Failed | 展示 | 返回 OA 订单真实状态 | 待确认 | 状态枚举和颜色映射 | P0 |
| Filter All/关系筛选 | 按关系筛选右侧订单 | 前端筛选或传参 | 可返回按关系过滤结果 | 待确认 | 大数据量时需后端过滤 | P2 |
| Bind/Unbind | 订单绑定当前会话或解绑 | 触发动作、刷新 UI | 校验权限、绑定/解绑、写日志 | 待确认 | 当前会话维度绑定接口 | P0 |
| View Detail | 打开 OA detail drawer | 打开抽屉 | 返回订单详情、Review/Refund 信息 | 待确认 | OA 详情接口 | P0 |
| Copy Order ID | 复制订单号 | 前端 | 无 | 前端支持 | 无 | P3 |

### 11. OA 创建和详情抽屉

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| 创建 OA 入口 | RDO/RSO + 按订单号预填 | 打开抽屉、预填 | 创建前校验和初始化 | 待确认 | create draft/init 接口 | P0 |
| 订单类型 | RDO/RSO 下拉 | 展示/选择 | 校验可创建类型 | 待确认 | 类型枚举 | P0 |
| Amazon Order No 校验 | loading/success/error | 输入、触发校验 | 校验格式、订单存在性、风控、重复绑定 | 待确认 | verify order 接口 | P0 |
| 商品信息预览 | 校验成功显示商品 | 展示 | 返回商品/ASIN/SKU/国家/店铺 | 待确认 | 预览字段 | P0 |
| Pre-submit Screenshot | 上传截图 | 文件选择/预览 | 上传附件、绑定 OA draft | 待确认 | 上传接口和附件归属 | P1 |
| Add Review Info | Review link + screenshot | 表单、校验 | 保存 review 信息 | 待确认 | Review 子表/状态 | P0 |
| Add Refund Info | 金额、币种、账号、收款码 | 表单、校验 | 保存 refund 信息、币种、金额合法性 | 待确认 | Refund 子表/状态 | P0 |
| Submit | 创建 OA 订单并插入右侧列表 | 提交、禁用态、成功反馈 | 创建订单、返回订单详情、写事件 | 待确认 | create order 接口 | P0 |
| Detail Drawer | 只读展示详情，Edit/Bind/Unbind | 展示和触发 | 返回详情、更新、绑定/解绑 | 待确认 | detail/update/bind 接口 | P0 |
| Review/Refund pending | 缺失信息展示 pending | 展示 | 返回各子流程状态 | 待确认 | 子流程状态机 | P1 |

### 12. Tools 面板和抽屉

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| Tools tab | 右侧切换 User Info/Tools | 切换显示 | 返回工具权限 | 待确认 | 工具权限配置 | P2 |
| Search All | 按 User ID/Name/Keyword 搜索，Online Now | 展示、搜索条件 | 全局用户/会话搜索、在线状态 | 待确认 | search all 接口 | P2 |
| Quick Replies tool | 打开快捷回复抽屉 | 入口 | 模板接口 | 待确认 | 同快捷回复模块 | P1 |
| Product Recommendations | 打开产品推荐抽屉 | 入口和选择 | 产品任务/推荐接口 | 待确认 | 推荐任务接口 | P1 |
| To-do List | 入口展示 | 入口 | 待办任务接口 | 待确认 | 当前 demo 已移除 WorkOrder Info，需确认工具列表最终项 | P2 |

### 13. Product Recommendation / Product Card

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| 产品推荐弹窗 | Product Recommendation Tasks | 展示 | 返回推荐任务 | 待确认 | 任务来源和排序规则 | P1 |
| 产品推荐抽屉 | 优先级 S/1/2/3、国家筛选、多选 | 前端筛选或传参 | 返回可推荐商品、优先级、国家 | 待确认 | 产品推荐接口 | P1 |
| 发送产品卡 | 发送到聊天流，作为 productCards 消息 | 选择和触发 | 创建产品卡消息并落库 | 待确认 | product card message schema | P1 |
| Cancel Send | 关闭抽屉，按钮吸底 | 前端 | 无 | 前端支持 | 无 | P3 |

### 14. Dashboard

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| 数据概览 | Sessions/Customers/Avg Msg/Duration/First Response | 展示 | 返回统计指标 | 待确认 | 报表指标口径 | P2 |
| Workload | 消息分布、每日会话、工作量表 | 图表和排序 | 返回聚合数据 | 待确认 | 工作量数据源 | P2 |
| Quality | 首响分布、FR 排名 | 图表 | 返回质量指标 | 待确认 | SLA/质量口径 | P2 |
| Attendance | 在线/离线/异常离线 | 图表 | 返回考勤/在线数据 | 待确认 | 是否属于本次重构范围 | P3 |
| 筛选 | 日期、团队、类型、Apply/Reset | 表单和刷新 | 按筛选返回数据 | 待确认 | 查询参数标准 | P2 |

### 15. 国际化、偏好和通用交互

| 功能点 | 当前 Demo 表现 | 前端职责 | 后端职责 | 现有后端支持 | 缺口/待确认 | 优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| 多语言 | en/zh/ja 文案切换 | i18n 渲染 | 可返回用户偏好 | 待确认 | 是否需要后端保存语言 | P3 |
| Toast | copied/saved/bound 等 | 展示反馈 | 返回动作结果 | 前端支持 | 无 | P3 |
| Copy | ID/订单号/链接复制 | 前端复制 | 无 | 前端支持 | 无 | P3 |
| Resize | 会话列表和右侧面板 resize | 前端布局 | 可选保存偏好 | 前端支持 | 无 | P3 |
| Scroll thumb | 自定义滚动条 | 前端样式 | 无 | 前端支持 | 无 | P3 |
| localStorage | demo 会话状态缓存 | demo 专用 | 生产不应依赖 | 不适用 | 替换为接口和实时推送 | P0 |

## 建议 API Adapter 分层

先在前端建立一层 adapter，避免 UI 直接绑后端接口。初期 adapter 可以读 mock，后端接口确认后逐项替换。

```js
api.sessions.list(params)
api.sessions.getMessages(sessionId, params)
api.sessions.sendMessage(sessionId, payload)
api.sessions.updateStatus(sessionId, action)
api.sessions.acceptHandover(sessionId, handoverId)
api.sessions.getHistory(sessionId)

api.customers.getProfile(customerId)
api.customers.search(params)
api.customers.getOaOrders(customerId, params)

api.oa.verifyAmazonOrder(payload)
api.oa.createOrder(payload)
api.oa.getOrderDetail(orderId)
api.oa.updateOrder(orderId, payload)
api.oa.bindOrderToSession(orderId, sessionId)
api.oa.unbindOrderFromSession(orderId, sessionId)

api.quickReplies.list(params)
api.quickReplies.create(payload)
api.quickReplies.update(id, payload)
api.quickReplies.delete(id)

api.ai.suggestReply(sessionId, context)
api.ai.submitFeedback(suggestionId, feedback)

api.products.listRecommendationTasks(params)
api.products.sendProductCards(sessionId, payload)

api.dashboard.getOverview(params)
api.dashboard.getWorkload(params)
api.dashboard.getQuality(params)
api.dashboard.getAttendance(params)
```

## 建议数据模型草案

### Session

```json
{
  "id": "im_session_20491862",
  "customer_id": "20491862",
  "customer_name": "Amelia Reed",
  "avatar_url": "...",
  "source": "US-EINSEO",
  "destination_account_id": "2016",
  "status": "pending",
  "unread_count": 1,
  "last_message": {
    "id": "msg_xxx",
    "type": "text",
    "direction": "inbound",
    "text": "Started about 3 days ago...",
    "created_at": "2026-06-22T18:25:00Z"
  },
  "risk": {
    "level": "high",
    "labels": ["malicious_complaint", "suspicious_order"]
  },
  "metrics": {
    "first_response_seconds": 83,
    "last_response_seconds": 23
  }
}
```

### Message

```json
{
  "id": "msg_xxx",
  "session_id": "im_session_20491862",
  "type": "text|system|product_cards|handover_card",
  "direction": "inbound|outbound|system",
  "author_id": "agent_noimz",
  "author_name": "Noimz",
  "text": "message body",
  "created_at": "2026-06-22T18:34:00Z",
  "metadata": {}
}
```

### Handover Card

```json
{
  "id": "handover_amelia_001",
  "session_id": "im_session_20491862",
  "status": "pending|accepted",
  "category": "After-sales Issue",
  "current_status": "Awaiting Tech Confirmation",
  "sentiment": "poor",
  "risk_labels": ["negative_review_risk"],
  "current_issue": "Unable to bind App, reinstalled but still failing",
  "next_action": {
    "text": "Follow up by night shift before",
    "deadline_at": "2026-06-22T18:00:00Z"
  },
  "commitments": [
    "Provide solution within 24h",
    "Replacement/refund if failed"
  ],
  "strict_donts": [
    "Do NOT ask for order screenshots again (already provided 2 times)"
  ],
  "transferred_from_agent_id": "agent_noimz",
  "transferred_from_agent_name": "Noimz",
  "accepted_by_agent_id": null,
  "accepted_at": null,
  "created_at": "2026-06-22T18:34:00Z"
}
```

### OA Order

```json
{
  "id": "oa_order_xxx",
  "order_type": "RDO|RSO",
  "amazon_order_no": "026-6986879-2656330",
  "relation_type": "customer_order|pending_match|linked_order",
  "status": "pending_review|pending_payment|cancelled|review_failed",
  "product": {
    "title": "贵妃多多款五代-玫红-新APP",
    "model": "Rosethorn 2",
    "category": "03玫瑰花品类",
    "image_url": "..."
  },
  "review": {
    "status": "pending",
    "link": null,
    "screenshot_file_id": null
  },
  "refund": {
    "status": "pending",
    "amount": null,
    "currency": "USD",
    "account": null
  },
  "bound_session_id": "im_session_20491862"
}
```

## P0 缺口清单

这些是 demo 进入真实对接时最先需要确认的项目。

| 缺口 | 页面场景 | 后端需要提供 | 备注 |
| --- | --- | --- | --- |
| 会话列表接口 | 左侧 Session 列表 | 分页、状态、来源、未读、最后消息、风险、客户基础信息 | 优先接现有 IM 会话接口 |
| 消息流接口 | 中间聊天区 | 消息分页、方向、作者、类型、系统事件、产品卡、交接卡 | 需要 message type schema |
| 发送消息接口 | Composer 发送 | 创建消息、发送状态、失败原因、幂等 key | IM 主链路 |
| 会话状态动作接口 | 状态下拉/关闭/转接 | 状态机、权限、系统事件 | 不能前端自行改状态 |
| 交接卡片接口 | Handover Card | 卡片详情、接受动作、接手人/时间、来源客服、系统事件 | 当前 demo 已模拟 |
| 客户详情接口 | 右侧 User Info | 客户画像、设备、注册/在线、风险 | IM 主链路 |
| 用户 OA 订单接口 | 右侧 User OA Orders | RDO/RSO、关系、状态、商品、绑定态 | 需要区分用户历史订单和当前会话绑定 |
| Amazon Order 校验接口 | 消息订单号/OA 创建 | 校验、风控、商品预填、重复绑定检查 | 当前 demo 3 秒 mock |
| OA 创建/详情/编辑接口 | OA 抽屉 | create/detail/update、Review/Refund 子信息、附件 | OA 主链路 |
| 绑定/解绑订单接口 | OA 卡片 Bind/Unbind | 绑定当前 IM 会话、解绑、审计 | 不能只改前端标签 |

## P1 缺口清单

| 缺口 | 页面场景 | 后端需要提供 | 备注 |
| --- | --- | --- | --- |
| Conversation History | 历史弹窗 | Chat Log、Key Events、事件与消息关联 | Key Events 规则需确认 |
| 快捷回复模板 | Composer `/` 和抽屉 | 团队/个人模板、变量、语言、权限 | 联想交互前端做 |
| AI 回复建议 | AI 胶囊 | 生成建议、来源、状态、反馈 ID | 可先 mock |
| 产品推荐任务 | Product Cards | 推荐任务、商品、优先级、国家、发送卡片 | 可先接只读 |
| 附件上传 | Composer/OA 截图 | 上传、file_id、访问 URL、类型限制 | OA 和聊天共用 |
| 风险详情 | 高风险弹窗/banner | 风险标签、命中原因、处置建议 | 可先只展示标签 |

## P2/P3 可后置项

| 功能 | 原因 |
| --- | --- |
| Dashboard 报表 | 与 IM 主链路相对独立，可后置 |
| 快捷回复管理新增/编辑/删除 | 先接只读模板，管理功能后置 |
| Search All 全局搜索 | 非主链路，后置 |
| Dashboard 考勤 | 可能属于另一个系统，需确认范围 |
| UI 偏好保存 | 可前端 localStorage 或后端用户偏好，非阻塞 |
| 菜单权限精细化 | 可先静态，后续接 RBAC |

## 需要后端同学确认的问题

1. 现有后端 IM 会话列表接口在哪里？是否包含未读、最后消息、来源、状态、客户基础信息？
2. 现有消息表是否支持系统消息、产品卡消息、交接卡片消息，还是需要新 message type？
3. 当前后端状态枚举和 demo 的 `Pending/Serving/Converting/Unsolved/New/Replace/Closed` 是否一致？
4. 转接/交接在后端已有实体吗？是否有 `transfer_from`、`transfer_to`、`accepted_by`、`accepted_at`？
5. 接手交接后，系统事件由后端自动生成，还是前端调用两个接口？
6. User Info 中的国家、性别、年龄、等级、粉丝、设备版本、注册时间、最近在线分别来自哪些表？
7. OA 的 RDO/RSO 是否已有统一接口？订单关系 `Customer Orders/Pending Match/Linked Orders` 如何判断？
8. OA 订单绑定是绑定到用户、IM 会话，还是两者都有？
9. Amazon Order ID 风控校验已有接口吗？是否能返回 risk reason？
10. 快捷回复是否已有团队/个人/语言/变量配置？
11. AI 回复建议和交接总结是否属于本次重构，还是先保留 mock？
12. 历史会话 Key Events 是否已有事件日志，还是需要从消息/订单/状态流转中聚合？
13. 实时更新采用轮询、SSE 还是 WebSocket？
14. 附件上传是否复用现有 OA/IM 文件服务？
15. Dashboard 是否属于本次后端对接范围？

## 推荐接入顺序

1. `P0-Read`：会话列表、会话消息、用户详情、OA 订单列表。
2. `P0-Write`：发送消息、状态动作、OA 订单绑定/解绑、OA 创建。
3. `P0-Handover`：交接卡片读取、接受交接、系统事件落库。
4. `P1-Assist`：历史会话、快捷回复只读、产品推荐只读、附件上传。
5. `P1/P2-AI`：AI 建议、交接总结、AI 反馈。
6. `P2-Manage`：快捷回复管理、Dashboard、全局搜索、菜单权限。

## 前端改造建议

1. 不要继续把新业务规则写进 `index.html` 的大函数里。
2. 先新增 `api` adapter，保持 UI 不变。
3. 把静态 `sessionData`、quick replies、OA mock 数据拆到 `mock/*.json` 或 `mock-data.js`。
4. 每个模块先走 adapter：
   - adapter mock 模式：返回当前 demo 数据。
   - adapter real 模式：调用后端接口并做字段映射。
5. 对后端暂不支持的能力，用 `mockOnly: true` 标记，避免误以为已接真实接口。
6. 对写操作加统一状态：`idle/loading/success/error`，不要只靠 toast。
7. 对所有业务动作保留后端错误展示位置：发送失败、绑定失败、OA 校验失败、接手失败。

## 当前 Demo 中应避免固化到前端的规则

| 规则 | 为什么不应固化在前端 |
| --- | --- |
| 订单号中包含 `000` 就是风险 | 真实风控应由后端判断 |
| 高风险用户名单写死 | 风险标签应来自风控/用户画像 |
| 交接卡片内容写死 | 应由 AI/客服交接流程生成并保存 |
| 接手时间由前端保存 | 业务审计时间必须以后端为准 |
| OA 订单状态前端切换 | 订单真实状态必须以后端为准 |
| Bind/Unbind 只改标签 | 绑定关系是核心业务事实 |
| Dashboard 随机指标 | 报表口径必须统一 |

## 搬到飞书文档后的协作建议

建议在飞书文档中把每个功能点表格增加这些列：

- `后端负责人`
- `接口地址`
- `Controller/Service`
- `支持状态`
- `缺口说明`
- `预计完成时间`
- `评审评论`

你可以逐条评论“漏了”“优先级错了”“这个后端已有”，我再按评论修订。
