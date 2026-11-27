# 专题 1 落地推进：Runway 付费首测最小方案＋能力缺口三项核实

时间：2026-11-27 凌晨。依据：只读三连实跑结果（`deep/2026-11-27-runway-readonly-results.md`）、首测方案（`deep/2026-11-27-runway-first-test.md`）、Runway 连接器文档（`/opt/hatch/connectors/runway_mcp/CONNECTOR.md`）、官方文档 `~/docs/scheduling-and-watching.md`、hooks/tracking/workflow 三个命名空间实查、Runway 定价公开检索。
边界：本轮只做文档阅读、公开检索与只读工具调用（`hooks.list`、`tracking.list`、`workflow.list`）；**未创建账号、未付费、未调用任何媒体工具、未跑第三方代码**。凡文档没写、实测没覆盖的参数一律标「未确认」。

---

## 一、Runway 付费首测最小可执行方案（只写方案，不执行付费）

### 1.1 现状定格（只读三连已通，结论复述）

- 连接正常：`status` → connected；`whoami` 认证通过；工作区「桂东」（id 71948935）。
- 套餐：Free，剩余 70 积分（套餐 50＋购买 20）。
- **硬门槛**：`availableVideoToolModels.upscale_video` 为空数组；工具 schema 明文 PAID PLAN REQUIRED。免费档下提交必被 `account_limitation` 拒，70 积分与升频无关；且 Free 档 `topUp.available: false`，连积分包都买不了。
- 样本：`~/workspace/social-ops/week3/assets/W3D4-video-final.mp4`，实测 40.656 秒、1080×1920/30fps、AAC 音轨、约 34.6 MiB。
- 关键上限：默认模型 topaz 要求片长 <40 秒，样本 40.656 秒**超限 0.656 秒**——即便开通付费，整片＋默认模型路线照样被拒；只能走 MediaKit（源片 ≤300 秒）或分段。

### 1.2 套餐与价格（两处来源对账，有一处出入，付款前必再核）

| 档 | 月付 | 年付折合 | 月积分 | 升频相关 |
|---|---|---|---|---|
| Standard | $15/月 | $12/月 | 625 | 含 4K upscaling |
| Pro | $35/月 | $28/月 | 2250 | Unlimited 4K video upscaling |
| Max | $95/月 | $76/月 | 9500 | 同 Pro，另有 HDR/ProRes 等 |

- 来源：Runway 定价页 2026 年 11 月多家第三方复核一致（lumalabs.ai、ai-alternative.com、theaihunter.com、growthtribune.com、aitoolsquare.com 均引 runway.com/pricing）。
- **出入**：本机连接器 `show_plans_and_credits` 在 2026-11-27 实跑输出 Standard 为 **$19/月**，与定价页 $15/月不符。付款前必须在 runway.com/pricing 当场再核一次，以付款页实价为准；**不许按任一单方数字直接付款**。
- 积分不结转：Standard 与 Pro 的月积分当月清零（Max 可结转一个月），此点多家来源一致。

### 1.3 单条成本估算（费率来源分级标注）

- MediaKit 费率来自工具 schema 原文（官方口径，可信）：`mediakit-enhance-fast` 约 10.3 积分/分钟（720p、30fps 输出），`mediakit-enhance` 约 21 积分/分钟；1080p 输出 ×2、2K ×4、4K ×8；>30fps ×2、>60fps ×4；每任务向上取整。
- topaz（默认模型）单价：schema **未公布**，仍未确认。第三方来源（growthtribune.com，引定价页）称「Upscale to 4K：2 积分/秒」，仅作参考估算，**不是官方口径**，首测实测扣减为准。
- 按样本 40.656 秒 ≈ 0.678 分钟、源短边 1080px、30fps 估算：
  - `mediakit-enhance-fast`、输出 1080p：10.3 × 2 × 0.678 ≈ **14 积分/条**；
  - `mediakit-enhance`、输出 1080p：21 × 2 × 0.678 ≈ **29 积分/条**；
  - `mediakit-enhance`、输出 4K：21 × 8 × 0.678 ≈ **114 积分/条**；
  - 分段单段（约 8 秒）走 enhance-fast 1080p：约 **3 积分/段**，整条 5 段约 14 积分；4K 输出整条约 55 积分。
- 档位够用性推演（按衔接点 B、分段、1080p 输出）：Standard 每月 625 积分 ≈ 44 条成片，覆盖每日 1 条有余；若要 4K 输出每日 1 条，Standard 仅够约 11 条/月，须 Pro 的 Unlimited 4K。**结论：首测与试生产走 Standard 月付即可；4K 日更才需 Pro。**

### 1.4 决策分叉（给主聊天/用户，二选一）

- **A. 开 Standard 月付（$15/月，按付款页实价）**：最小可执行 spend。开通后按 1.5 走首测；月内可覆盖约 44 条 1080p 分段升频，够验证「升频是否入管线」。
- **B. 不付费**：Runway 升频线整体搁置，「高清修复收尾」改找开源/本地工具（另立专题，本轮不展开）。
- **不选 Pro 起步**：Pro 的价值在 Unlimited 4K 与 2250 积分，首测用不上；等 1080p 路线验收通过、且确认要 4K 日更时再升级，不许先买高档再验证。

### 1.5 首测执行步骤（开通付费后，执行端按此跑；本轮未执行）

前置门：用户本人完成付款升级（付款确认属平台强制本人步，助手不代付）；媒体工具每次调用都要审批卡，执行时会有审批卡，属平台机制。

1. **复核门**：重跑 `whoami`，确认 `availableVideoToolModels.upscale_video` 非空；再跑 `show_plans_and_credits`，确认套餐已变、积分到账。任一不符即停手回报，不硬上。
2. **最小首测（1 段，约 3 积分）**：取样本 5 段中的 1 段（约 8 秒，本地副本），走衔接点 B：`init_upload` → `complete_upload` → `upscale_video`（`mediakit-enhance-fast`，输出 1080p）。只测 1 段，不整条押注。
3. **验收（沿用首测方案 §三标准，逐项可复核）**：ffprobe 对分辨率/帧率/时长（与源差 ≤0.1 秒）；音轨在位（AAC 保留）；同点位抽帧与基线并排，茶垄纹理、叶片边缘、远山轮廓锐度肉眼可辨；烧录字幕与「AI 生成示意图」角标逐字核对无变形；`show_plans_and_credits` 前后差记实际扣减，与估算对账。
4. **等终态纪律**：`get_task` 轮询至终态；失败/超时先查 `list_recent` 确认是否其实已产出，确认无产出才许第二次提交（防重复扣费）。
5. **产物处理**：产物另存 `~/workspace/social-ops/runway-test/`，**不动平台侧排期、不覆盖原文件**。
6. **分叉**：1 段通过 → 补测其余 4 段（约 11 积分），整条验收通过后谈入管线；1 段不通过（无肉眼提升/字幕变形/成本失控）→ 如实报不达标，Runway 升频降为备选，现行 ffmpeg 链照常跑。
7. **失败兜底**：产物无音轨 → ffmpeg `-c:v copy` 把源音轨回 mux 再复验；字幕/角标被 AI 重绘变形 → 整片升频路线判不通过，转衔接点 B（本方案已是 B，无需再转）；余额不足 → 停手回报差额，不许先斩后奏充值。

### 1.6 本节状态：已学未落地

- 卡点：**付费决策与付款在用户**（Standard $15/月起，按付款页实价），助手本轮只写了方案，未执行任何付费操作；开通后仍须重跑 `whoami` 确认 MediaKit 是否对本工作区开放（schema 只说「not enabled on every workspace」）。
- 未确认项（付款后首测才能销账）：topaz 单价；升频是否保留音轨、是否改帧率；MediaKit 在本工作区是否开放。

---

## 二、能力缺口清单三项核实（对照 ~/docs/ 与实查）

### 2.1 缺口 1：workflow 工作流引擎——成立（真实能力、零使用）

- 证据 1（工具面）：`workflow` 命名空间已加载，14 个函数在册：create / launch / launch_async / list / list_runs / view / view_run / update / delete / pause / resume / stop / stop_agent / save_run。用途原文：「VM-local JavaScript workflows: orchestration code executed by the daemon that coordinates child agents… repeatable multi-agent automation… where the plan should live in deterministic code」。
- 证据 2（实查）：`workflow.list` 返回「No saved workflows found」——**零个已存工作流**，缺口清单「未用」属实。
- 落地第一步（可执行，未执行）：把「两小时盯办巡查」的汇总逻辑写成一个最小 workflow 跑一次，与现行人工盯办结果对账。写脚本须守 workflow 合同（`export const meta = {...}` 开头、无 Node/文件系统/shell/浏览器 API、经 `agent()` 派活）。

### 2.2 缺口 2：hooks 事件驱动自动化——成立但表述误判（工具在、有 1 个钩子在跑；「事件驱动」不是真事件）

- 证据 1（工具面）：`hooks` 命名空间在册，add / list / dry_run / enable / disable / run / logs / view / update / remove 齐全。定义原文：「event-based automations where a Bash polling script decides when to wake a worker agent」——**钩子本体就是一个 Bash 轮询脚本**，每轮以 `silent` 或 `wake` 收尾；脚本在无凭据环境运行（"without secrets or connector credentials"），只能用公开端点或本地数据。
- 证据 2（实查）：`hooks.list`（含停用）返回**恰 1 个钩子**：`clawbot-inbox`，enabled，轮询 30 秒，脚本 `~/hooks/scripts/clawbot-inbox.sh`，投递主聊天。缺口清单所说的「桥消息、群消息现全是轮询」里，桥消息一侧其实已有钩子形态在跑。
- 证据 3（官方文档纠正）：`~/docs/scheduling-and-watching.md` 原文：「The same is true for event-based automations: they also work by polling, not live streams.」——**事件自动化同样是轮询，不是实时流**。缺口清单「不靠定时轮询，事件一到就触发」的写法是误判，改为：钩子＝高频轻量轮询＋条件触发唤醒，比 cron 的优势是按条件唤醒、不是按时间唤醒；延迟下限是轮询间隔（clawbot-inbox 为 30 秒）。
- 落地第一步（可执行，未执行）：`hooks.list` 已盘（本轮完成）；下一步挑一个事件源做单点验证——候选：新邮件到达（须先确认脚本能否读到邮箱，脚本无连接器凭据，**很可能走不通**，先 dry_run 证伪再立项）。记账「账单邮件到达即提取」在钩子无凭据的前提下不成立，仍须 cron 轮询。

### 2.3 缺口 7：tracking 承诺跟踪命名空间——已解决（在用，30 项）

- 证据 1（工具面）：`tracking` 命名空间在册，create / list / search / get / create_entry / update / set_status / delete 等齐全；与 Goals 分工原文：「Goals 是长期目标，这个管具体承诺」。
- 证据 2（实查）：`tracking.list`（limit 100）返回 **30 项**，含 active 与 completed：办文、直播、记账、小说、京东 cookie、医保名单导入、两癌参保、Postiz 迁移等均已建项。缺口清单「当前是否已有跟踪项在用——未确认」现已核实：**在用，30 项**。
- 修正：缺口清单第 7 条「落地第一步」里「为空就把『桥转发待确认』四件建为首批跟踪项」的前提已不存在——跟踪项早已存在，不许重复建项；办文逾期件、待转发件若需跟踪，先 `tracking.search` 查是否已有对应项，有则记进展，无则新建。

### 2.4 三项核实小结

| 缺口 | 结论 | 证据 |
|---|---|---|
| 1 workflow | 成立：真实能力、零使用 | workflow 命名空间 14 函数在册；`workflow.list` 零个已存工作流 |
| 2 hooks | 成立但表述误判：工具有、1 个钩子在跑；「事件驱动」实为轮询 | hooks 命名空间在册；`hooks.list` 仅 `clawbot-inbox`（30 秒轮询）；`~/docs/scheduling-and-watching.md` 明言事件自动化也是轮询 |
| 7 tracking | 已解决：在用，30 项 | tracking 命名空间在册；`tracking.list` 返回 30 项 |

---

## 三、下一步可执行动作（按优先级）

1. **Runway 决策**：主聊天把 1.4 的 A/B 分叉报用户定夺；若选 A，用户完成付款升级后，执行端按 1.5 跑最小首测（1 段约 3 积分），验收标准沿用首测方案 §三。
2. **workflow 首跑**：把「两小时盯办巡查」汇总逻辑写成最小 workflow 跑一次，与人工盯办结果对账（缺口 1 落地第一步）。
3. **hooks 单点验证**：先 dry_run 证伪「新邮件到达」事件源（脚本无凭据，大概率不通），再决定是否另选事件源；不许直接把记账账单提取写进钩子方案。
4. **tracking 使用纪律**：办文逾期件、待转发件先 `tracking.search` 查已有项，杜绝重复建项（缺口 7 修正）。

## 四、仍卡住的点

- Runway 付费决策与付款在用户；topaz 单价、MediaKit 本工作区开放性、升频音轨/帧率均须开通后首测才能销账。
- Standard $19/月（连接器输出）与 $15/月（定价页）出入未解，付款前必在 runway.com/pricing 当场再核。
- hooks 的事件源范围仍只验证了 clawbot-inbox 一例；新邮件等事件源待 dry_run 证伪/证实。

## 五、风险玩法不采纳声明

本轮不涉及多号轮换、第三方 API 反代、绕封号、买号等玩法；Runway 首测走官方连接器与官方付费套餐，付款走用户本人官方渠道。上述风险玩法与本任务无关，不采纳的原因：违反平台规则、有封号与账号安全风险，且本任务的官方通道已够用，无需绕路。

---

## 附：本轮核实证据路径

- `~/workspace/learning/deep/2026-11-27-runway-readonly-results.md`（只读三连实跑结果）
- `~/workspace/learning/deep/2026-11-27-runway-first-test.md`（首测方案 §三验收标准、§四衔接点）
- `/opt/hatch/connectors/runway_mcp/CONNECTOR.md`（媒体工具每次审批、异步任务制、不许盲目重跑）
- `~/docs/scheduling-and-watching.md`（事件自动化也是轮询，非实时流）
- 实查输出：`hooks.list`（1 个钩子 clawbot-inbox）、`tracking.list`（30 项）、`workflow.list`（0 个已存工作流）
- 定价检索：Runway 定价页 2026 年 11 月多家第三方复核（Standard $15/月、Pro $35/月、Max $95/月；年付 $12/$28/$76）
