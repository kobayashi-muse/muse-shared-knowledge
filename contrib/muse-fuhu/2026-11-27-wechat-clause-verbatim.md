<!-- 分享版：已脱敏，去除账号、单位与内部排期等标识；方法与结论不变。 -->

# 微信 3.27 条官方页逐字直核（补核报告）

核查日期：2026-11-27（凌晨 01:58–02:05，Asia/Shanghai）。只做公开页面检索与原文核对，未登录任何平台、未发表、未排期、未做任何直播浏览器操作。
承接：`2026-11-27-wechat-automation-clause.md` 未确认清单第 1 项（官方页面逐字直核待补）；`2026-11-26-platform-mechanics.md` 未确认清单第 5 项。
用户口径（2026-11-27 01:08 已拍板）：全自动无人值守口径维持不变。本报告只做事实核查，不改口径。

---

## 结论先行

**官方页逐字直核：仍未核到。** 目标官方页 `mp.weixin.qq.com/mp/opshowpage?action=newoplaw#t3-3-27` 本轮 2026-11-27 01:58 再试直达，仍被本机抓取策略拦截（`Domain mp.weixin.qq.com is blocked by policy`），与上一轮失败原因完全相同。按指令未重复该请求、未换任何方式绕过。

**但本轮拿到三份可全文打开的独立来源**（见第二节），对已有笔记的转述做了逐项比对：**条款本体、编号、日期、三项构成、处罚梯度全部一致，无一处矛盾**；另补到两处新信息（「April 10 公告」日期口径、第三方排版工具导入亦可触发判定）与一处口径纠正（某篇拟定时文章 7 天窗口实际开启日为 目标定时点前 7 天，而非前 6 天）。

**定时提前期规则核实结果：确认 7 天。** 公众号后台定时发表为「提前 5 分钟至 7 天」，来源为引腾讯客服帮助页（kf.qq.com）的第三方实测文章（细节核于 2026-10-24），与另一独立工具方「七天以内」口径一致。以一篇拟定时发表的文章为例，其定时点落在 7 天窗口内即可设置；每次设置定时须管理员扫码确认（平台强制本人步），额度在设置时即扣、取消可返还。

---

## 一、官方页直核尝试记录（本轮实跑）

| 尝试 | 时间 | 结果 |
|---|---|---|
| `browser.open` 直达官方规范页 `https://mp.weixin.qq.com/mp/opshowpage?action=newoplaw` | 2026-11-27 01:58 | 失败：`Domain mp.weixin.qq.com is blocked by policy`，无 HTTP 状态码 |
| `browser.open` 腾讯新闻逐字引用文 `https://news.qq.com/rain/a/20260409A03TE700` | 01:59 | 失败：`Domain news.qq.com is blocked by policy` |
| `browser.open` 新浪转载文 `https://k.sina.com.cn/article_7879996023_1d5af327706801plgo.html` | 02:00 | 失败：`Domain k.sina.com.cn is blocked by policy` |
| `browser.open` Yicai Global 英文报道 | 02:01 | 失败：`Domain yicaiglobal.com is blocked by policy` |
| `browser.search` 两轮（官方页定位＋定时规则） | 01:58–01:59 | 成功，返回多源摘要（见第三节） |

**卡点如实记录**：本机抓取层按域名策略拦截腾讯系（mp.weixin.qq.com、news.qq.com）及部分新闻域名（k.sina.com.cn、yicaiglobal.com），与页面本身是否存在、是否登录无关。被拦截的请求按指令不再重复、不换端点绕过。**因此「官方页面逐字直核」在本环境内仍是不可能完成的动作，不是未尝试。**

**新发现的第二官方 URL 变体（未打开，留待可直达环境）**：第三方文章引用的官方规范页另有一地址 `https://mp.weixin.qq.com/publicpoc/opshowpage?action=newoplaw`（与上一轮的 `/mp/opshowpage` 路径不同，同属 mp.weixin.qq.com 域名）。因同域名已被策略拦截，本轮未尝试打开——打开即属重复被禁请求。可直达官方页的环境应优先试此变体与原锚点 `#t3-3-27`，一并核对页首文件全称。

---

## 二、本轮全文打开的三份独立来源（均实读全文）

### 来源 A — MarketingToChina《WeChat Bans AI-Generated Content: What Foreign Brands Must Do Now》
- URL：https://marketingtochina.com/wechat-bans-ai-generated-content-foreign-brands/
- 读取时间：2026-11-27 02:01（页面抓取时间 2026-11-26 17:59 UTC）；刊发 2026-08-26，更新 2026-11-08。
- 关键原文（英文，照录）：
  > "Tencent updated its Operating Standards for the Official Accounts Platform, adding Section 3.27: a direct ban on using AI, scripts, or automated programs to replace humans in content creation and publishing."
  > "Within days of the April 10 announcement, accounts began receiving batch removal notices citing 'non-human automated content generation.'"
  > 三类违规："Content generated entirely by AI with no genuine human creative input"；"Bulk publishing through scripted programs or APIs"；"Promoting or distributing tools that enable automated content creation"。
  > "Penalties escalate by severity: traffic throttling, content removal, account suspension, up to permanent ban. At least one account that went viral for claiming 2 million yuan ($276,000) in annual revenue from an AI-only Official Account operation was permanently banned as an example."
  > "Tencent's spokesperson was explicit… You can use AI to draft, research, translate, and edit. What you cannot do is remove the human from the loop entirely."
  > 视频号："Creators publishing AI-assisted video content must now declare it with an 'AI technology generation' tag visible at the bottom of the video. This is already enforced."

### 来源 B — AIBase《Rejecting AI Ghostwriting: WeChat Official Accounts Crack Down on Non-Real-User Automated Writing》
- URL：https://news.aibase.com/news/26975
- 读取时间：2026-11-27 02:04（页面抓取时间 2026-11-26 17:59 UTC）。
- 关键原文（英文，照录）：
  > "The new regulations explicitly prohibit using AI, scripts, or APIs to replace real people in creating and publishing content. Specific violation scenarios include: AI-generated or rewritten content completely detached from human expression, mass continuous publishing through program hosting, and spreading tutorials or services for such automated content creation."
  > "Even if some creators claim they only used third-party formatting tools for import or published AI technology tutorials, if the content characteristics meet the 'non-human automation' standard, it will still trigger the system's violation judgment. The platform emphasizes that the content must carry the genuine thoughts and expressions of the creators."
  > "Light violations may result in traffic restrictions or direct deletion of the relevant articles, while severe violations could lead to account function restrictions or permanent bans."

### 来源 C — OmniFire《How to Schedule WeChat Official Account Posts from Outside China (2026)》
- URL：https://www.omnifire.app/blog/schedule-wechat-official-account-posts/
- 读取时间：2026-11-27 01:59（页面抓取时间 2026-11-26 17:59 UTC）；更新 2026-11-25；文内声明细节核于 2026-10-24，来源标注为腾讯客服帮助页（kf.qq.com：publishing modes、mass-send rules、notification quotas、scheduled sends）。
- 关键原文（英文，照录）：
  > "In the WeChat backend you can schedule a mass send for any time from five minutes to seven days ahead. The administrator has to scan a QR code and confirm on their phone each time."
  > "A scheduled send uses its quota as soon as you set it, not when it goes out. … Cancel before the time and the count comes back."
  > "Seven days in the WeChat backend."
  > 额度："Subscription: 1 a day; service: 4 a month"；"Posts published without a notification have no count limit" 且同样可定时。
  > "WeChat merged its old mass send (群发) and publish (发布) actions into a single publish action (发表)."
- 注意：OmniFire 本身是第三方调度工具（持官方授权、代持排期再到时调用微信发布）。本报告只取其对微信后台规则的陈述部分，不采纳其工具路线（见第六节不采纳声明）。

---

## 三、与已有笔记 `2026-11-27-wechat-automation-clause.md` 的逐项比对

| 核对项 | 已有笔记的表述 | 本轮比对结果 |
|---|---|---|
| 条款编号与名称：3.27「非真人自动化创作行为」 | 属实、2026-03-27 新增 | **一致**。来源 A、B 均确认 Section 3.27 / Article 3.27，内容指向相同 |
| 归属文件全称 | 《微信公众平台运营规范》与《微信公众账号行为规范》两种称呼并存，未确认页首标题 | **仍未解决**。来源 A 用 "Operating Standards for the Official Accounts Platform"（支持《微信公众平台运营规范》一说）；腾讯新闻的《微信公众账号行为规范》一说本轮未能重读原文。**页首标题仍待官方页直核** |
| 入规日期 2026-03-27 | 多源一致 | **一致**。本轮检索结果（澎湃/新浪/红星/Yicai 转述）仍为 3 月 27 日；无来源改口 |
| 官方回应日期 | 2026-04-09 微信团队向澎湃、红星、第一财经回应 | **小出入待澄清**。来源 A 写 "April 10 announcement"。两种可能：4-09 为媒体回应日、4-10 为公告/集中执行日；或英文报道日期口径差一天。**不影响条款本体**，记为待澄清，不擅自统一 |
| 三项违规构成 3.27.1 / 3.27.2 / 3.27.3 | 脱离真人表达的 AI 生成改写拼接搬运；脚本程序托管批量连续发布；传播自动化创作教程方法服务 | **一致**。来源 A、B 的三类转述与中文转录逐项对应（生成/改写/拼接/搬运；批量连续发布；教程服务传播），无增删 |
| 处罚梯度：内容层流量限制→删除；账号层能力限制→封禁 | 条款明文梯度 | **一致，且顶端更明确**。来源 B 明确轻重两档（轻：限流或直接删文；重：功能限制或永久封禁）；来源 A 明确梯度顶端为 permanent ban。已有笔记的「封禁」表述与此相符 |
| 执行实况：2026 年 4 月批量删文、教 AI 写公众号的账号被封 | 多家媒体报道＋处罚提示截图口径一致 | **一致，并补一例**。来源 A 补：一个宣称 AI-only 公众号年入 200 万元的账号被作为典型永久封禁；来源 B 补：多名创作者后台文章被批量删除。执行实况的可信度上升 |
| 微信团队回应原话（辅助可用、替代不行） | 两段引文，多家媒体一致转述 | **未新增核实**。本轮打开的英文来源只转述大意（"remove the human from the loop entirely"），未提供中文原话新文本。已有笔记的两段引文维持原状，不升级核实等级 |
| 辅助与替代的边界（润色纠错配图检索可用；最终内容须体现真人风格立场判断） | 官方回应＋Foresight News 口径 | **一致**。来源 A、B 均重申同一边界（draft/research/translate/edit 可用；final content must reflect real creator's style, stance, judgment / genuine thoughts and expressions） |
| **新增风险点：第三方排版工具导入亦可触发判定** | 已有笔记未提及 | **新增**。来源 B 明确：即使只称「用了第三方排版工具导入」，只要内容特征符合「非真人自动化」标准，系统仍会判定违规。此点与我方管线直接相关（见第五节审核增补） |
| 视频号 6.4 标识义务 | 二手整理，未直核官方页 | **方向一致**。来源 A 确认视频号 AI 辅助视频须加 "AI technology generation" 标注且已在执行；与已有笔记的 6.4 方向相符，条文逐字原文仍未直核 |
| 2025-08-31 AI 标识公告与 3.27 正交叠加 | 已有笔记第三节 | **一致，无新出入**。本轮未发现反向口径 |

**比对总评**：已有笔记的转述经本轮三份独立全文来源交叉检验，**无一处被推翻**；其「多源交叉核实」的定级维持成立，唯一未完成项仍是「官方页面逐字直核」。

---

## 四、定时提前期规则核实（某篇拟定时文章 目标定时点）

只核规则本身，未执行任何发表/排期操作。

| 规则点 | 核实结果 | 来源 |
|---|---|---|
| 公众号后台定时最远提前期 | **7 天**（最早 5 分钟） | 来源 C 全文：`five minutes to seven days ahead`；细节核于 2026-10-24，引腾讯 kf.qq.com 帮助页 |
| 独立佐证 | 「一般情况下可以设置七天以内的时间」 | 微羽助手指南 https://www.wefeather.cn/guide/mp-platform-publish（检索摘要，2026-11-27 01:59） |
| 旧口径冲突 | 云梁网络称「只能定时今、明两天内」 | https://www.yunliangwang.com/news/gongzhonghao/616.html（检索摘要）。判定：**旧口径**——来源 C 明确微信已把旧群发（群发）与发布（发布）合并为单一发表（发表）动作，今明两天之说属合并前规则，不再适用 |
| 某篇拟定时文章 目标定时点 的 7 天窗口 | 窗口自 **目标定时点前 7 天** 起打开 | 目标定时点 减 7 天 = 目标定时点前 7 天 |
| 口径纠正 | 此前说法「7 天窗口 11-26 进入」**差半天**。11-26 当天窗口早已打开，11-27 设置亦在窗口内，不存在「须等 11-26 才能设」 | 由上条算术得出 |
| 设置定时须管理员扫码确认 | **每次都要**，属平台强制本人步 | 来源 C：`The administrator has to scan a QR code and confirm on their phone each time.` |
| 额度占用时点 | 设置时即扣，不是发出时扣；定时前取消可返还 | 来源 C：`A scheduled send uses its quota as soon as you set it… Cancel before the time and the count comes back.` |
| 额度 | 订阅号每天 1 次通知、服务号每月 4 次；无通知发表不限次数、同样可定时 | 来源 C 全文，与已有台账口径一致 |

**结论**：某篇拟定时文章 目标定时点 的定时设置，在 11-27 当天即可操作（窗口已开），但须管理员（用户本人）扫码确认一次，且设置即占用当日/当月通知额度——此为待人工项，不由助手代办。

---

## 五、落地：两条审核标准增补（写进专题 3 笔记，待并入派单/审核标准）

1. **公众号发表前检查新增一项「导入方式」**：成片/草稿是否经第三方排版工具导入发表。来源 B 明确此类导入亦可触发「非真人自动化」系统判定。凡经第三方工具导入的稿件，发表前须确认内容已由真人实质改写定稿并留痕；不能只凭「内容是真人写的」就放行——判定看的是内容特征与生产链路，不看单方说法。
2. **公众号定时设置检查新增两项**：①定时时间须落在「提前 5 分钟至 7 天」窗口内，超窗的排期须拆成「先存草稿、窗口开启日再设」两步，不许按旧「今明两天」口径排长线；②每次设置定时均须管理员扫码确认，列为待人工项随心跳汇总，不阻塞其他流程；额度在设置时即扣，排期前先核当日/当月剩余额度。

状态：专题 3 仍为「学习中」。本轮完成的是「3.27 条多源比对复核＋定时 7 天规则核实」；「官方页逐字直核」仍标未完成，不报学完。

---

## 六、风险玩法不采纳声明

多号轮换、第三方 API 反代、绕封号、买号等玩法**一律不采纳**，原因：
- 3.27.2 明文打击的正是「脚本、程序托管批量、连续发布」——绕封号与多号轮换属于被打击形态本身，不是合规空间；
- 第三方调度工具（如来源 C 的 OmniFire）即使持官方授权，仍受 7 天窗口、额度、每次管理员扫码三重约束，不能替代真人发布动作；且我方公众号为个人主体、未认证，API 直发本就受限（来源 C：未认证账号 2025 年 7 月起不许 API 直发），反代绕行只会把账号推向封禁；
- 本轮核查只用公开页面检索，未登录、未调用任何接口、未测试任何绕行手段。

---

## 七、仍待补的点

1. **官方页逐字直核（核心未完成项）**：mp.weixin.qq.com 在本环境被策略拦截。待有可直达官方页的环境时，用 `https://mp.weixin.qq.com/mp/opshowpage?action=newoplaw#t3-3-27` 与变体 `https://mp.weixin.qq.com/publicpoc/opshowpage?action=newoplaw` 两个地址补核：页首文件全称（两种称呼之争）、3.27 条逐字文本、生效日期表述。
2. 「April 9 回应」与「April 10 公告」的日期口径差一天，待官方时间线或原始报道澄清。
3. 微信团队回应原话的中文原文仍为多家媒体一致转述，未见官方页面原文。
4. 视频号运营规范 6.4、5.12、4.7 的逐字原文仍未直核（同为官方页拦截）。
5. 定时规则的最终依据仍是引腾讯客服帮助页的第三方文章，非 kf.qq.com 原文直读；kf.qq.com 帮助页原文待补读（本轮未单独打开核对）。
6. 处罚执行细节（删文比例、封禁申诉恢复与否）仍为媒体与当事人单方口径，未独立核实。

---

## 附：本轮读取来源清单

- https://marketingtochina.com/wechat-bans-ai-generated-content-foreign-brands/ （全文已读，2026-11-27 02:01）
- https://news.aibase.com/news/26975 （全文已读，2026-11-27 02:04）
- https://www.omnifire.app/blog/schedule-wechat-official-account-posts/ （全文已读，2026-11-27 01:59）
- 检索摘要：https://www.wefeather.cn/guide/mp-platform-publish、https://www.yunliangwang.com/news/gongzhonghao/616.html、https://www.coinlive.com/zh/news-flash/1089083、https://k.sina.com.cn/article_7879996023_1d5af327706801plgo.html（摘要已读，全文被拦截未读）
- 被拦截未读：mp.weixin.qq.com（官方规范页）、news.qq.com（腾讯新闻逐字引用文）、k.sina.com.cn、yicaiglobal.com
