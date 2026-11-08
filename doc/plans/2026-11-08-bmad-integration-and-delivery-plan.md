# BMAD 接入与企业研发平台实施计划

> 日期：2026-11-08。当前目标已由用户确认：20 团队、50 项目、50 个实际同时执行的规划 Run；企业内网自建，复用现有 Kubernetes / 数据库运维。500 个本地开发工作区只保留为待确认的 Comet 压测假设。
>
> 本文是源码核实后的可行性判断与实施任务，不代表已接入 BMAD、已运行 PoC 或已完成生产验收。配套：[总结与风险](../enterprise_ai_rd_platform_summary_and_risks.md)、[主架构](2026-11-05-team-ai-development-architecture.md)、[产品形态](../团队级AI研发平台-产品形态.md)。

## 1. 明确结论

**可行：当前 Paperclip 适合作为该产品的基础，建议继续实施。不能直接上线：安装 BMAD skills 后，现有项目不会自动变成完整的企业研发平台。**

可行性的依据是已有 Company / Project / Issue、技能导入与投递、OpenCode adapter、任务问答、文档修订、人工确认、Case handoff 和持久回答投递。缺口集中在 BMAD 工作区与交互适配、受保护的研发阶段控制、Git 发布和版本交接、Comet 接入及选定部署组合的生产验证。这些缺口可以按现有模块边界补齐，没有必要先重写 Agent 运行时、通用流程引擎或另一套任务系统。

第一步确定为 **“把固定版本 BMAD 接入平台实际执行的 Runtime，完成一条 PRD 规划闭环”**。只把 BMAD 放到 Paperclip 源码仓库的 `.agents/skills`，最多帮助维护者开发本项目，不能证明平台里的 Agent 能加载它，更不能证明浏览器用户可以回答其问题。

对当前项目的判断分三项：

| 问题 | 答复 |
| --- | --- |
| 是否值得以现有 Paperclip 继续建设？ | 是，已有执行、任务、交互与版本化基础适合复用 |
| 是否仅通过配置 / 安装 skills 就完成全部产品？ | 否，阶段准入、方法恢复、Git 版本映射和 Comet 仍需产品集成与服务端开发 |
| 是否已经证明 20 团队 / 50 项目 / 50 Run 的生产能力？ | 否；配置规模更小不会自动消除隔离、持久性、双活和模型限额问题 |

## 2. 当前源码证据与边界

核验 HEAD：`b43ff274123714092cf4d92884a08007a2e85a9a`。相对此前 `3d3485478` 的提交差异只涉及四份设计文档，不能视为上述产品能力已实现。图服务本轮返回 `Transport closed`，已按规则降级到定向文件、调用处和测试源码读取；以下不是全仓安全或 HA 审计。

| 能力 | 当前证据 | 实施判断 |
| --- | --- | --- |
| Company 技能导入、版本与授权 | [技能服务](../../server/src/services/company-skills.ts)、[技能路由](../../server/src/routes/company-skills.ts) | 复用现有 Company skills；PoC 不必先修改全局 skills catalog |
| 运行使用的技能版本 | [执行配置](../../server/src/services/heartbeat/execution-configuration.ts) 会记录 `skillVersions` / `skillSelectionSnapshot` | 复用已有快照；跨多个 heartbeat 的业务规划还需固定同一方法版本，不能假定新 run 永远沿用旧技能 head |
| 外部技能命名 | `buildSkillRuntimeName` 对外部技能加 hash，`rewriteFrontmatterName` 只改顶层 name | 校验 BMAD 内部技能名、相对路径与实际 runtimeName，不以 UI 列出技能作为加载成功 |
| OpenCode 技能执行 | [技能投递](../../packages/adapters/opencode-local/src/server/skills.ts)、[执行路径](../../packages/adapters/opencode-local/src/server/execute.ts) | 当前路径使用 HOME 下 `.claude/skills`，远程目标会准备资产；按任务 / 执行环境隔离 HOME 与工作目录，防止共享目录相互覆盖 |
| 问题与回答 | [交互服务](../../server/src/services/issue-thread-interactions.ts)、[回答投递](../../server/src/services/question-response-delivery.ts) | 已有持久交互与继续执行基础；仍需把 BMAD 方法步骤、等待和恢复接上 |
| 人工确认 | [解析策略](../../server/src/services/issue-thread-interaction-resolution.ts)、交互服务的修订 target 校验 | 显式 `human_only`；按需要绑定真实接收人；卡片本身不授予后续操作权限 |
| 文档修订 | [文档服务](../../server/src/services/documents.ts) 要求更新携带当前 `baseRevisionId` | 复用冲突保护与原生文档，不再造 PRD 存储或审批表 |
| Case / Workflow | [Case handoff](../../server/src/services/case-handoffs.ts)、[产品定义](../PRODUCT.md) | 已有原子交接、幂等和执行约束；workflow 的阶段图是建议，BMAD 业务门禁仍由服务端扩展强制 |
| BMAD 产品集成 | 本次定向搜索 `server/src`、`packages/shared/src`、skills / teams catalog 和 DB schema 未检出 BMAD 专用实现；建议的 `services/engineering` 目录不存在 | 视为待实施；不能将架构文档中建议接口、表或模块写成当前 API |
| Kubernetes 与 HA | [插件说明](../../packages/plugins/sandbox-providers/kubernetes/README.md)、[旧规模部署分析](2026-11-08-on-prem-100-teams-deployment.md) | alpha 插件、默认临时工作区和受控接管均需按实际组合验证，50 Run 也不能绕过 |

已有技能和交互测试可以作为回归入口，不能替代真实 BMAD 执行。当前工作树未安装 `node_modules`，本轮未运行应用测试或模型调用。

## 3. BMAD 版本与交付方式

本轮读取的官方安装页将 v7 标为 preview alpha，并建议要求稳定的使用方选择 v6.12；`V6.12/package.json` 当前版本为 `6.12.4`。建议首个候选固定到已核实的 commit `bab9082c3e884e0d72c759bcd1a6d90bcc01be84`，制作内网可复现包，并记录全部文件摘要。此 commit 是本轮候选，不代表已认证其与当前 Runtime 兼容。[官方安装说明](https://docs.bmad-method.org/start/install-bmad/)、[固定版本元数据](https://github.com/bmad-code-org/BMAD-METHOD/blob/bab9082c3e884e0d72c759bcd1a6d90bcc01be84/package.json)

首个目标技能为该版本的 `bmad-prd`，PM persona 可用 `bmad-agent-pm`；以实际版本的依赖清单为准。上游 PRD 技能明确依赖 `_bmad/scripts`、配置解析、`customize.toml`、工作区 `prd.md`、`addendum.md` 和 `.memlog.md`。记忆文件由其脚本维护，不能用一段角色 prompt 替代完整方法包。[PRD 技能源码](https://github.com/bmad-code-org/BMAD-METHOD/blob/bab9082c3e884e0d72c759bcd1a6d90bcc01be84/src/bmm-skills/plan/bmad-prd/SKILL.md)

采用两个配套交付物：

1. **固定方法资产及执行镜像 / 工作区初始化。** 完整保留本条规划所需的 skills、references、模板、脚本与配置布局；Node、Python、uv、CLI 和依赖在内网准备，不在每个 run 临时拉取最新版。
2. **一个 Company 管理的 Paperclip 规划接入 skill。** 仅负责调用所选 BMAD 入口、解析实际 runtimeName、把交互映射到原生卡片、保存恢复依据及提交文档；方法逻辑继续来自固定上游包。具体路径和名称在步骤 1 确认，不另发明 runtime adapter。

技能内容和完整方法依赖都必须进入可校验的发布清单；只固定外层 wrapper 版本、内部读取可变 `_bmad` 目录仍然会漂移。关闭试点来源自动追踪最新版；生产方法的编辑 / 升级使用现有技能策略约束。当前普通 Company 技能操作默认开放，不能假定只有管理员能修改。

## 4. 六步实施及放行条件

| 步骤 | 交付内容 | 依赖与放行条件 | 主要责任 |
| --- | --- | --- | --- |
| 1. 接入 BMAD 最小闭环 | 一个 Company、一个项目、一个 PM Agent、一个 Runtime、固定 `bmad-prd`；问答后生成可读取 PRD | 真实运行证明依赖、技能名、路径、等待与回答后继续有效；使用测试资料，不将其当企业生产能力 | Runtime / 技能集成 |
| 2. 做可靠的规划状态与人工门禁 | 受保护控制记录、检查点、回答消费版本、PRD 修订确认、停止 / 撤权 / 过期处理 | 重启、重复答案、旧确认、Agent 自批、旧 API 绕过测试通过 | 后端与产品 UI |
| 3. 完成 Git 与本地交接 | 持久发布意图、确定候选 commit、远端核对、不可变交接；增加必要架构 / 候选任务 | 本地 IDE Skill 能据准确版本生成 Story；Git 成功而 DB 回写失败可恢复，未提交文件不被覆盖 | 后端 / Git / Harness |
| 4. 验证多团队与执行隔离 | 项目权限、独立 HOME / 工作区、凭据 / 网络 / 资源配额、公平准入 | 两个 Company 的越权和干扰测试通过后才开放多团队；资格验证所选 Kubernetes 执行组合 | 后端 / 平台运维 |
| 5. 接入 Comet 观察 | 纯只读采集、来源绑定、序号 / 代际、状态新鲜度、成果关联 | 至少一条真实 Harness → Comet → 项目页链路，文件前后不变、乱序 / 离线恢复正确 | Harness / 后端 / 前端 |
| 6. 完成生产验证和推广 | 单活接管、DB / 对象 / 密钥恢复、50 Run 混合负载、监控与发布 | 全部实际启用能力通过安全与正确性门槛；按 2–3 → 5–10 → 20 团队开放 | 平台运维 / 测试 / 产品 |

基础认证、对象授权和有限凭据从步骤 1 就使用现有强制边界，不能留到步骤 4 再补。步骤 4 是多团队范围的扩大与隔离验收。运维可在步骤 1 同期做 Kubernetes 插件和持久工作区的小规模验证，尽早暴露部署阻断；不先采购完整生产设备。

### 步骤 1 的可执行清单

1. 固定 Paperclip、OpenCode、模型、BMAD commit 与依赖版本，保存 manifest、摘要和授权方式。首个 PoC 使用现有 `opencode_local`；若其真实问答 / 停止能力不达标，先记录失败，再资格验证已有其他 Runtime 路径，不以换运行时掩盖失败。
2. 建一个专用测试 Company / Project，绑定测试仓库和独立工作目录；使用隔离的运行 HOME，限制 Git / 模型凭据。先单个活动 run，避免把共享 HOME 当隔离手段。
3. 在该执行工作区准备固定 BMAD 资产和配置，验证脚本、模板及内部引用可解析，禁止隐式安装未固定的额外技能。BMAD 的研究、子任务和外部交接也必须受当前工具权限与资源准入约束。
4. 经现有 Company skills 导入接入 skill 并绑定 PM Agent；分别验证“导入版本”“实际投递内容”“模型确实读取并执行方法”三份证据。技能同步测试或 Studio 的受限技能测试不能单独证明完整规划任务。
5. 从现有普通任务提交一条需求，显式调用 PRD Create 意图；由平台决定调用入口，不要求用户选 BMAD 菜单、Agent 或手工运行命令。此时不必先新增项目按钮或完整角色组织。
6. 将 BMAD 的问题转换为原生 `ask_user_questions`，显式 `human_only` 与 `wake_assignee`；保存成功后进入已有等待状态并结束本轮，避免 CLI 一直等待 stdin 或短 timer 反复唤醒。遇到无可用执行槽位时保持持久待派发状态。
7. 通过浏览器回答；新 run 读取持久交互结果、同一方法版本和同一工作区记忆，从对应步骤继续。若恢复失败，步骤 1 不通过；不能用自动填写假设跳过真实问答。
8. 产出 `prd.md` 并同步为任务文档 `prd`，记录输入、方法版本、来源 run、文件摘要和修订 ID。只做一个 PRD 不代表架构、开发交接或生产能力完成。

**步骤 1 的验收场景：** 提交“做一个内部报修系统” → 看到真实问题卡片 → 回答角色、处理流程和范围 → AI 继续产出 PRD → 用户能打开对应内容。等待时不占活动执行槽位；日志能定位所用方法资产和脚本；清空仅供执行的临时会话后仍能据持久资料重建继续所需上下文。不可恢复的工作区数据不能删除来模拟“恢复成功”。

### 步骤 2 的正确性契约

复用原生交互、回答投递、文档和 Case 历史；在现有服务中增加窄的研发控制能力。按主架构的逻辑数据设计实现当期必需字段，不为步骤 1 一次性创建所有规划、发布和 Comet 表。

- 控制记录绑定 Company、Project、Issue / Case、方法版本、当前步骤、输入修订、待处理 interaction、已消费回答版本及 `controlVersion`。权威字段只能由受授权的服务端动作推进；普通 Case fields、模型文本和可编辑文档不是批准状态。
- 问题、正式确认和普通评论分开处理。正式确认绑定当前 `issue_document` 修订，必要时指定真实接收人；新修订需要新确认。批准仍需后续操作的权限检查。
- 同一回答可被重复投递，但只产生一次有效控制状态转换。输入变化、停止、撤权、任务关闭或方法升级后，旧回答不能直接推进；保留可解释的暂停 / 冲突状态。
- 对 Case handoff、任务完成、文档确认后继续、通用 routes 和 Agent 工具逐项验证绕过路径。仅在新按钮上加校验不构成服务端门禁。
- 方法检查点支持恢复业务步骤，不承诺重放整个 CLI 会话。`.memlog.md` 是 BMAD 方法记忆，平台控制记录是授权和阶段依据，两者用途不同。
- 验证旧方法进行中的任务继续使用原版本；缺失版本应明确阻塞。普通新 run 的技能选择不能无声替换一个尚未完成的规划方法。

### 步骤 3～5 的交付边界

Git 发布沿用“数据库持久意图 → 固定候选 commit → 写专属 ref → 查询远端核对 → 落库 → 绑定确认 / 交接”。远端成功但响应丢失时核对同一个候选，不再生成不同提交。若先审阅平台草稿，最终交接仍需证明确认内容与 Git 内容完全对应，不能重新生成内容后沿用批准。

完整交接按工作规模包含必要 PRD、架构、候选任务、依赖和验收标准。平台不生成替代本地流程的开发 Story；本地自研 Skill 负责选任务与生成 Story，Workflow 负责开发及验证。小改动可走简化规划路径，但不绕过权限、版本和必要的人类确认。

Comet 排在规划与交接之后。它可以单独启用，未启用时以显式标记的手工成果关联完成试点；不能宣称已经具有自动进度。首个规划 PoC 无需等待 Comet 完成，全量产品验收仍需验证实际启用的采集链路。

## 5. 部署与采购调整

沿用单活控制与受控接管、HA PostgreSQL、隔离执行池和持久存储方向。Kubernetes 插件仍为 alpha，默认 `emptyDir` 不保证 Pod 丢失后的工作区恢复；继续使用既有实现做资格验证，不将“有插件”当“生产可用”。

先复用现有集群中的少量资源完成步骤 1～4，再按真实进程树和模型限额定容。总结文档的 6 / 9 Worker 是给定每节点 12 槽位时的参考布局，不能据此冻结采购：

| 示例布局 | 约定故障后的理论槽位 | 含义 |
| --- | ---: | --- |
| 6 节点，失去 1 节点 | 5 × 12 = 60 | 算术上覆盖 50 Run；不证明真实吞吐 |
| 8 节点，按 3 / 3 / 2 分域，失去最大域 | 5 × 12 = 60 | 说明 9 节点不是唯一可行的三域布局 |
| 9 节点，按 3 / 3 / 3 分域，失去 1 域 | 6 × 12 = 72 | 等规模故障域及更多余量，是否需要由实测决定 |

统一以 Kubernetes `Allocatable` CPU 和 GiB 计量；物理核心、逻辑 CPU / vCPU、GB / GiB 不直接混用。32 核、双 25GbE、1.92 TB NVMe、3–5 TB 对象存储都没有本产品的实测依据，不是软件要求。优先按实际可用资源、I/O、网络与保留周期确定预算。[Kubernetes 资源语义](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

正式容量测试为持续 50 个真实 Run，100 个同时到达请求验证超额持久排队；100 是突发试验值，不是承诺同时执行 100。500 Comet 来源保留为单独压力假设，不作为已确认用户规模。原有 100 团队 / 200 Run 文档只保留历史扩容参考。

## 6. 实施包、验证与验收证据

| 实施包 | 主要改动归属 | 必须保留的验证 |
| --- | --- | --- |
| BMAD 方法包与接入 skill | Company skills、受控方法资产、工作区初始化、Agent 配置 | 文件 / 依赖完整性、实际 runtimeName、版本快照、真实问答 / PRD 运行记录 |
| 研发控制与界面 | 建议的 `server/src/services/engineering/` 窄服务；必要 db/shared/routes；现有项目与任务 UI | 人工策略、修订冲突、幂等、暂停撤权、所有可推进入口、关闭功能后的兼容性 |
| Git 交接 | 现有 Git 授权 / 文档 / work products 与必要发布记录 | 同一候选重试、远端成功本地失败、并发分支、精确版本交接 |
| 多团队 / 运行保障 | 现有执行准入、工作区、Kubernetes 插件配置及必要修复 | 隔离、总量限制、停止证据、Pod 丢失恢复与接管 |
| Comet | 现有 Harness 采集适配、平台来源接收与投影 | 只读、目录边界、重复乱序、离线与过期、真实版本契约 |

现有回归入口包括 `server/src/__tests__/opencode-local-skill-sync.test.ts`、`server/src/__tests__/heartbeat-runtime-skills.test.ts`、`server/src/services/issue-thread-interaction-resolution.test.ts`、`server/src/__tests__/issue-thread-interactions-service.test.ts`。新增测试验证实际跨模块契约，不仅验证 mock 返回值；真实模型验收记录 Runtime / 模型 / 方法版本和费用。

每一步提交可复现证据：环境与版本、输入、操作、预期、实际、产物修订 / 哈希、失败及修复后复验。连续几个示例成功只能证明这些示例，不宣称模型永不出错。未完成的步骤标记“未验证”，不把本次文档检查变成运行成绩。

实施先后由上述放行条件决定；在步骤 1 暴露真实适配量前，不给没有依据的总工期或承诺人员天数。第一项工程任务可以直接建为：**“固定 BMAD 6.12.4 方法资产，经 Company skill + OpenCode 打通单项目 PRD 问答闭环，并提交恢复证据。”**
