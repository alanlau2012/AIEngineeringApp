## AI 工程技能学习系统 · v1 开发方案（微信小程序 + 微信云开发）

### 背景与目标
- 痛点：AI 工程技能栈宽泛、学习路径不清晰、强实践导向但缺少体系化“技能树”。
- 目标：交付一个微信小程序，提供技能树-理论-实践任务-测试认证-成长体系的完整闭环，支持后续内容扩展与企业化演进。
- 范围定位：本方案仅覆盖 v1（微信云开发 TCB），优先交付速度与可用性；企业化后端（NestJS+PG）留作 v2。

### 方案确认（v1）
- 运行平台：微信小程序。
- 数据与后端：微信云开发（TCB）云数据库 + 云函数。
- 登录与鉴权：wx.login → openid 建档，按用户维度存储学习进度与成就。
- 资产托管：云存储（题图、图标等）或直接使用在线图源。

### 功能范围（v1 最小可用集合）
- 技能树：三大分支（基础/框架/平台），节点状态（locked/available/learning/completed），整体/分支进度。
- 技能详情：理论知识要点、实践任务清单（minitask）、测试解锁门槛、难度/标签/预计时长、进度百分比。
- 测试与认证：题目渲染、作答交互、解析、计时、评分；通过阈值（默认 80%）→ 颁发 EXP/技能点、标记技能完成。
- 个人中心：等级/EXP/成就、学习统计（最近 7 天）、最近活动、功能菜单。
- 首页：用户信息、学习进度概览、推荐技能、快捷入口。
- 成长体系：等级-经验分段、成就授予、技能解锁规则（依赖/百分比）。
- 埋点与统计：PV/点击/完成/答题正确率等基础埋点（云函数上报 + 聚合）。

### 页面映射与原型对照
- pages/home：用户卡片、进度条、推荐技能、快捷入口（参考 `home.html`）。
- pages/skill-tree：整体进度、筛选 Tab、分支与节点网格（参考 `skill-tree.html`）。
- pages/skill-detail：理论要点、实践任务、测试入口、收藏/继续学习（参考 `skill-detail.html`）。
- pages/quiz：题干/选项/解析/进度点/计时/结果页奖励（参考 `quiz.html`）。
- pages/profile：成就网格、学习统计、最近活动、设置等（参考 `profile.html`）。
- 注：现有高保真 HTML 原型将迁移至 `docs/prototypes/` 以便对照与演示。

### 技术架构
- 前端：微信小程序（TypeScript）+ WXML/WXSS，抽取主题变量与通用组件以复用。
- 后端：TCB 云函数（Node.js 运行时）+ 云数据库（文档型）。
- 网络：小程序 → 云函数（通过 wx.cloud.callFunction）→ 云数据库；或小程序直连云数据库（受控读写，推荐经云函数聚合）。
- 登录：wx.login 获取 code，exchange 为 openid（使用云函数自动获取 openid 并保存）。

### 数据模型（云数据库集合）
- users：{ _id, openid, name, level, exp, createdAt, lastActiveAt }
- branches：{ _id, name, order, color }
- skills：{ _id, branchId, title, icon, exp, tags[], difficulty:1-5, deps: string[], statusDefault:"locked", theory: { minutes:int, points:[{idx:int, text:string}] } }
- tasks：{ _id, skillId, title, description, xp:int, order:int }
- questions：{ _id, skillId, type:"single|multi|judge", stem, options:[string], answer:[int], explain, imageUrl? }
- progress：{ _id, userId, skillId, status:"locked|available|learning|completed", percent:int(0-100), updatedAt }
- achievements：{ _id, code, name, icon, condition:{ type, value }, reward:{ exp:int, point?:int } }
- quizSessions：{ _id, userId, skillId, answers:[{qId, choice}], score:int, correct:int, total:int, durationSec, createdAt, detail:{} }
- events（埋点）：{ _id, userId, type, payload, ts }
- 索引建议：users.openid 唯一；progress.userId+skillId 组合唯一；skills.branchId；questions.skillId；events.userId+ts。

### 解锁/成长规则（默认）
- 解锁：技能 deps 全部 completed，或其前置分支百分比 ≥ 阈值（如 60%）→ available。
- 状态推进：学习中→ 完成：当 percent=100% 且测试通过。
- 经验与等级：level by exp 阶梯（示例：每级 1000 EXP 递增）；
  - 完成任务授予任务 xp 累计；
  - 测试通过奖励：150 EXP + 1 技能点（可配置）。
- 成就：按条件触发（首次通过、连续学习 N 天、完成框架层等）。

### 云函数 API 设计（示例）
- user.init
  - 入参：无（从上下文取 openid）；
  - 出参：{ user, level, exp, stats }；首次调用初始化用户。
- skills.tree
  - 入参：无；
  - 出参：{ branches:[{id,name,progressText,skills:[{id,title,status,icon,exp,tags,percent}]}], overall:{completed,learning,locked,percent} }
- skills.detail
  - 入参：{ skillId }；
  - 出参：{ skill, tasks:[…], progress:{status,percent}, quiz:{count,locked:boolean} }
- progress.mark
  - 入参：{ skillId, deltaPercent?:int, setStatus?:string }；
  - 出参：{ progress, user:{exp,level}, unlocks:[skillId…], achievements:[…] }
- quiz.get
  - 入参：{ skillId }；
  - 出参：{ questions:[{id,stem,options,type,imageUrl?}], limit:int, durationSec }
- quiz.submit
  - 入参：{ skillId, answers:[{qId, choiceIdxs:int[]}], durationSec }；
  - 出参：{ score:int, correct:int, total:int, pass:boolean, reward:{exp:int, point?:int}, detail:[{qId, correctAnswer, userAnswer, explain}] }
- track.event
  - 入参：{ type, payload }；出参：{ ok:true }

### 目录结构（v1）
```
AIEngineeringApp/
  ├─ miniprogram/
  │  ├─ app.json / app.ts / app.wxss
  │  ├─ pages/
  │  │  ├─ home/ index.{wxml,wxss,ts,json}
  │  │  ├─ skill-tree/ index.{wxml,wxss,ts,json}
  │  │  ├─ skill-detail/ index.{wxml,wxss,ts,json}
  │  │  ├─ quiz/ index.{wxml,wxss,ts,json}
  │  │  └─ profile/ index.{wxml,wxss,ts,json}
  │  ├─ components/ skill-node/ progress-bar/ question-option/ stat-card/
  │  ├─ utils/ api.ts types.ts store.ts format.ts
  │  ├─ assets/ icons/
  │  └─ seed/ skills.json questions.json
  ├─ cloudfunctions/
  │  ├─ user.init/
  │  ├─ skills.tree/
  │  ├─ skills.detail/
  │  ├─ progress.mark/
  │  ├─ quiz.get/
  │  ├─ quiz.submit/
  │  └─ track.event/
  ├─ docs/
  │  └─ prototypes/  ← 迁移现有 *.html 原型
  └─ README.md （更新运行说明）
```

### UI/组件复用
- 从 `wechat-common-styles.html` 抽取主题变量/通用样式至 app.wxss。
- 组件：skill-node、progress-bar、question-option、stat-card、achievement-item、progress-donut。
- 动效：fade-in、pulse、选项 hover/selected/correct/incorrect，与原型一致。

### 开发步骤（执行清单）
1) 环境准备
- 安装微信开发者工具、Node.js 18+；开通云开发环境（创建 envId）。
- 在项目中启用云开发（控制台），记录 envId，开通云数据库/存储/云函数。

2) 项目初始化（前端）
- 新建 `miniprogram/` 与 `app.json/ts/wxss`；配置 TabBar（首页/技能树/学习/我的）。
- pages 与 components 骨架创建；抽取公共样式。

3) UI 静态还原（数据假值）
- 完成 5 个页面的静态布局与交互态（状态样式、按钮态、导航）。
- 组件联动：skill-node 点击跳转到 skill-detail；quiz 按钮切换题目。

4) 种子数据与本地渲染
- 在 `miniprogram/seed` 放置 `skills.json`、`questions.json`；
- `utils/api.ts` 先接本地 mock，打通页面数据渲染与状态切换。

5) 云开发初始化与数据导入
- 创建集合：users, branches, skills, tasks, questions, progress, achievements, quizSessions, events；
- 导入种子数据（基础/框架/平台分支+若干技能与题目）。

6) 云函数开发与接入
- 按“云函数 API 设计”实现函数，封装 `utils/api.ts` → `wx.cloud.callFunction`；
- `user.init` 首次启动即调用；其余按页面生命周期请求。

7) 核心业务逻辑
- 解锁规则/状态推进；经验/等级计算；成就授予；测试评分与奖励；
- 题目随机化与乱序；计时/中断恢复；弱网重试与幂等（以 quizSessions 保障）。

8) QA 与优化
- 覆盖用例：技能树链路、任务完成、测试通过/失败、断点续学、错题解析；
- 性能：列表懒渲染、图片压缩、云函数冷启动优化；
- 稳定性：错误态/空态/弱网提示、重试策略、日志留存。

9) 上线流程
- 配置合法域名/云开发资源；
- 提交体验版 → 内测反馈 → 审核发布；
- 监控：云函数告警、错误日志、关键指标看板（完成率、通过率）。

### 里程碑与工期（参考 4 周）
- M1（第1周）：项目初始化 + UI 静态还原（5 页 + 核心组件）
- M2（第2周）：云数据库建模 + 云函数（skills/progress/quiz）+ 前端数据接入
- M3（第3周）：核心逻辑（解锁/等级/成就/计时/幂等）+ QA
- M4（第4周）：性能与稳定性优化 + 上线与埋点看板

### 测试与验收标准
- 学习闭环：技能树 → 详情完成≥2任务 → 解锁测试 → 得分≥80% → 标记 completed → EXP/等级增长 → 成就可见。
- 一致性：刷新后状态一致；中断恢复；重复提交不重复计分。
- 交互/视觉：与高保真一致（导航/动效/状态色）；
- 性能：P50 首屏 < 800ms，云函数 P95 < 600ms（冷启动除外）。

### 安全与合规
- 最小权限：云数据库安全规则按用户隔离，写入仅允许本人文档；
- 鉴权：所有敏感写入经云函数校验 openid；
- 隐私：脱敏存储日志；错误日志不含个人身份信息；
- 备份：云数据库按日备份快照。

### 风险与对策
- 内容供给不足 → 提前定义技能/题库 CSV 模板，批量导入；
- 评分/作弊风险 → 抽题/乱序/时间阈值、结果抽检；
- 冷启动延迟 → 预热关键云函数，或在首页延迟加载非关键数据；
- 迁移到企业后端 → 以云函数契约为 API 边界，v2 平滑替换网关。

### 交付物
- 微信小程序源码（miniprogram/），云函数源码（cloudfunctions/）；
- 云数据库集合结构与初始化脚本（导入 JSON）；
- 埋点事件字典与看板指标说明；
- 使用说明（README 更新）。

### 立即下一步（待执行）
- 在微信开发者工具中创建项目并启用云开发，获得 envId；
- 初始化 `miniprogram/` 与 TabBar；
- 迁移 `*.html` 原型到 `docs/prototypes/`；
- 创建集合与导入基础种子数据；
- 开发 `user.init`、`skills.tree`、`skills.detail` 三个云函数打通首页/技能树/详情链路。

### 附录：示例种子数据（节选）
```json
{
  "branches": [
    {"_id": "base", "name": "基础层"},
    {"_id": "framework", "name": "框架层"},
    {"_id": "platform", "name": "平台层"}
  ],
  "skills": [
    {"_id": "ml-basic", "branchId": "base", "title": "机器学习基础", "exp": 100, "difficulty": 2, "deps": [], "tags": ["ML"], "statusDefault": "available"},
    {"_id": "python", "branchId": "base", "title": "Python编程", "exp": 100, "difficulty": 1, "deps": [], "tags": ["Python"]},
    {"_id": "data-prep", "branchId": "base", "title": "数据预处理", "exp": 120, "difficulty": 2, "deps": ["python"], "tags": ["Data"]},
    {"_id": "pytorch", "branchId": "framework", "title": "PyTorch", "exp": 150, "difficulty": 3, "deps": ["ml-basic"], "tags": ["DL"]}
  ],
  "tasks": [
    {"_id": "t1", "skillId": "pytorch", "title": "环境配置与张量基础", "description": "安装 PyTorch，完成 5 个张量操作", "xp": 50, "order": 1},
    {"_id": "t2", "skillId": "pytorch", "title": "第一个神经网络", "description": "实现 MNIST 全连接网络", "xp": 80, "order": 2}
  ],
  "questions": [
    {"_id": "q1", "skillId": "pytorch", "type": "single", "stem": "以下哪个更适合研究与原型？", "options": ["TensorFlow", "PyTorch", "Keras", "Scikit-learn"], "answer": [1], "explain": "PyTorch 动态计算图更灵活"}
  ]
}
```
