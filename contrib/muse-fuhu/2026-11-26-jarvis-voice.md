# 贾维斯语音入口深学报告（课程台账专题 9）

- 日期：2026-11-26 晚
- 执行：云端 Muse（深学任务，纯研究，未动任何设备与配置）
- 输入材料：`~/workspace/jarvis-build/` 任务书（2026-11-24 版）、isair/jarvis 核实记录、IndexTTS-2.5 试用记录、2026-11-21 云端试验台实测笔记；S0.5 决策书原件在 Mac 侧（dsh 交付），云端留存结论为「借鉴架构自研、不 fork isair/jarvis」，本报告按此结论做修订点分析
- 方法：官方文档与模型卡原文、开源仓库与实测博客交叉核对；厂商自报数字均已标注，未独立复核的不写成定论

## 结论速览

1. **KWS 选型不用改，调法要改**：官方未公布 zh-en 3M 模型的误唤醒率（未确认）；同架构微调模型的公开实测显示，默认 `keywords_score=1.0` 会把近音词误报成唤醒词，降到 0.2 后近音误报大降、2 小时朗读语音零误唤醒。S1 必须把「近音负样本＋score/threshold 调参矩阵」写进证据，而不是只用默认参数跑一遍。
2. **S7 路线要拆成两半看**：播报半链不用新鉴权——mijiaAPI 本身就有音箱 `play-text` 纯播报动作，我方连接器（mijiaAPI 5.0.0）的通用 `run_action` 即可调；轮询半链不在 mijiaAPI 里，走的是另一套 MiNA 服务（MiService），登录是小米账号密码/cookie，与 mijiaAPI 的扫码登录是两套鉴权——这是决策书与任务书都没写清的最大修订点。
3. **实时链 ASR 要换流式**：现用 SenseVoice 是离线模型，只适合文件转写（S8）；Mac 实时链应换 sherpa-onnx 流式 Zipformer/Paraformer。云端备链定火山豆包流式 ASR＋流式 TTS（WebSocket 首包 300–400ms，社区实测口径）。
4. **HUD 不用从零写**：MagicMirror²（MIT，现行 v2.38.0）有 server-only 模式，Mac 起服务、电视浏览器开页面，模块系统现成；自研只剩「贾维斯状态卡」一个自定义模块。小米电视自带浏览器对它的兼容性未确认，保留手写单页为退路。

---

## 一、sherpa-onnx KWS 唤醒词：公开实测口径

对象：我方现用 `sherpa-onnx-kws-zipformer-zh-en-3M-2025-12-20`（中英双语，3M 参数，流式 Zipformer transducer）。

### 1. 官方口径（k2-fsa 文档与模型页）

- 默认参数：`keywords_score=1.0`、`keywords_threshold=0.25`、`max_active_paths=4`、`num_trailing_blanks=1`（官方文档示例配置原文）。
- 量化编码器 4.4 MiB；模型块长 160/320 ms 两档（官方文档数据，转引自 Accessor 项目的文档梳理）。我方试验台实测：整包解压 38MB（含 fp32 与 int8 两套）。
- **官方未公布该模型的误唤醒率（次/小时）与召回率基准——此项「未确认」**，k2-fsa 只给用法不给评测数。下述公开实测均为第三方或同架构旁证，引用时已标明对象差异。

### 2. 资源占用（第三方实测）

| 来源 | 环境 | 数据 |
|---|---|---|
| Accessor 项目（用与我方完全相同的 2025-12-20 模型） | 2012 年双核 i7-3520M、单线程 | 3.11 秒音频解码约 0.16 秒，**RTF≈0.05**；合成噪声下约 15 dB SNR 仍可靠、10 dB 漏检 |
| MAGI Consensus 项目（同系列 gigaspeech 3.3M 模型） | 桌面机 | 常驻监听约占**单核 3%** |
| 我方试验台（2026-11-21） | 云端共享 vCPU | 阈值 0.25 一次命中「贾维斯」，无唤醒词样本零误报（文件级小样本，只作旁证） |

结论：KWS 常驻成本在 Mac（Apple Silicon）上可忽略量级，S1 的看门狗资源预算不用为它留大头；真正的变量是噪声环境下的漏检（SNR 10 dB 级别开始丢），远场测试必须带电视外放背景，与任务书 S1 证据②的设计一致。

### 3. 误唤醒率（最接近的公开实测：同架构微调模型）

Icerm/xiaoguang-kws：在官方 wenetspeech 3.3M KWS 上针对唤醒词「小光」及其近音词微调的模型卡（测试音频为合成语音＋2 小时 AISHELL-1 测试集朗读语音），sherpa-onnx 一侧参数为 threshold=0.25、**keywords_score=0.2**：

| 指标 | sherpa-onnx（0.25 / boost 0.2） | openWakeWord（对照） |
|---|---|---|
| 唤醒词命中（留出人声） | 98.7% | 95.5% |
| 近音「小关」误触 | 2.2% | 7.0% |
| 近音「小黄」误触 | 1.4% | 16.8% |
| 近音组「消光/小广/小逛」误触 | 22% | 80% |
| **误唤醒（2h 朗读语音）** | **0 次/小时** | 0.5 次/小时 |
| 另一组人声（edge-tts 11 音色）命中 | 87.9%（漏检全来自 2 个方言腔音色） | 96.0% |

两个必须写进 S1 的教训：

- **boost 是误唤醒的主开关**：模型卡明确标注 score=1.0（即我方现在的默认值）时近音词会被报成唤醒词；降到 0.2 才有上表数字。对「贾维斯」而言近音负样本就是「加维斯、贾维思、假维斯、贾维斯顿」这类词——云端 ASR 试验里已真实出现过把「贾维斯」听成「加维斯」，近音混淆不是假想敌。
- **该模型是针对单一唤醒词微调过的**，数字比我方开箱模型乐观，只能当上限参考；我方开箱模型的真实误唤醒率**未确认**，必须由 S1 在 Mac 现场实测得出，不许拿这张表充数。

反面参照（不同模型，仅作风险量级提醒）：一个日语 phoneme Zipformer KWS 实验模型在 10.64 小时负样本上测出 53.9 次/小时误报、召回仅 74%——未调参的小 KWS 模型误报可以差到这个量级，所以 S1 的 2 小时误唤醒 ≤1 次这条通过线必须实测，不许默认达标。

---

## 二、流式 ASR＋TTS 低延迟组合现状

### 1. 本地链（推荐，Mac 中枢）

**KWS（已有）→ Silero VAD / sherpa 端点检测 → sherpa-onnx 流式 ASR → 云端大脑 → Kokoro 按句流式 TTS（已有）**

- 现状修正：任务书 S2 定的 SenseVoice int8 是**离线**模型（整段喂入整段出），2026-11-21 实测文件级 RTF 0.15 很好，但它没有流式增量输出，做实时链会把「说完→出字」的等待拉长。sherpa-onnx 同栈内有流式 Zipformer（中英双语）与流式 Paraformer 可选，与 KWS/Kokoro 同进程同运维，是改动最小的换法；SenseVoice 保留给 S8 微信语音文件转写等离线场景。
- 延迟旁证：KWS 环 RTF≈0.05（见第一节）；Kokoro int8 量化包约 80MB、CPU 可跑（2026 行业延迟梳理数据）；级联链的通用经验值是调好的级联约 400–800ms 声到声（voice-agent-next 2026-09 综述，厂商与综述口径）。**我方 Mac 上端到端延迟未确认**——任务书 S1/S2/S5 的 Mac 实测（RTF、首句出声 ≤1.5s）就是补这一格，不许用综述数字替代。
- 开源 ASR 大势（只作背景）：2026 年开源流式 ASR 的领跑者是 NVIDIA Nemotron 系（cache-aware 分块 80–1120ms），但中文与 Mac 落地生态不如 sherpa-onnx 现成，不换栈。

### 2. 云端链（推荐，作备链与高质量档）

**火山引擎豆包流式 ASR（WebSocket）→ 云端大脑 → 豆包流式 TTS（WebSocket）**

- TTS 首包延迟：流式合成 <300ms（火山开发者社区 2026 年实测文口径，测试节点阿里云北京）；另一篇实测文给 300–400ms 区间。属社区实测非官方 SLA，量级可信、精确值未确认。
- ASR 侧：豆包有流式语音识别大模型 WebSocket 接口；**其端到端延迟的公开实测数字未查到，未确认**，接入时须自测。
- 选它的理由：国内网络直连、中文自然度与术语重音口碑好、与抖音生态同源；成本口径：公版音色资源包约 2.1–2.8 元/万字符（官方计费文档转引）。
- 全球云链参照（不作我方选型，仅量级对照）：第三方基准（dev.to 单人实测）Deepgram＋Claude＋Cartesia 级联 P50 410ms、Deepgram＋Claude＋ElevenLabs P50 480ms、OpenAI Realtime 端到端 P50 232ms；综述口径典型云级联 1.1–1.3s 声到声。我方大脑已定云端 Muse，广域网往返在国内走豆包语音前后端更稳。

### 3. 与既有结论的衔接

- Groq Whisper（我方已接连接器）：批量转写用，**非流式**，不进实时链；且既往实测有把「贾维斯」转成「加维斯」的记录，实时链更不能拿它当唯一 ASR。
- IndexTTS-2.5：维持任务书结论——不进实时主链，仅 S5 可插拔占位，本节无修订。

---

## 三、米家设备联动：公开方案现状与我方对照

### 1. 公开方案全景（四条线）

| 方案 | 做什么 | 鉴权 | 与我方关系 |
|---|---|---|---|
| **mijiaAPI（Do1e/mijia-api）** | 米家云设备控制：家庭/设备/属性/动作/场景；**含音箱 `play-text` 纯播报动作** | 扫码登录（3.x 起去掉密码登录），令牌约 30 天 | **我方连接器正是它**（5.0.0） |
| **MiService / MiNA（miservice、miservice_fork）** | 小爱服务：对话记录轮询、音箱 TTS（ubus `text_to_speech`）、播放状态/暂停 | 小米账号密码（serviceLoginAuth2）或 cookie；有风控拦截史 | 我方**没有**这一层，S7 轮询半链靠它 |
| **ha_xiaomi_home（小米官方 HA 集成）** | 米家设备全量接入 Home Assistant；云端 MQTT 订阅推送（非轮询）；有中枢网关时可局域网本地控制 | OAuth 2.0（不存密码） | 官方线，我方用户口径已定「米家控制走自建连接、不走 Home Assistant」，仅作对照 |
| **刷机线（Open-XiaoAI / xiaobot 等）** | 音箱本地跑自有程序、真拦截小爱回答 | 需刷机/补丁 | **禁区**：用户已定不刷机；同生态的 dsh-xiaoai-bridge 插件（open-xiaoai 桥＋本地 sherpa-onnx 唤醒）也依赖刷机，同样不采用，仅借鉴其播报闸门设计（播报期关麦克风通路） |

### 2. S7 两半链的真实接口（本次查清的关键事实）

**播报半链（现成，不用新鉴权）**：mijiaAPI 对音箱提供两个易混动作——`execute-text-directive` 是把文本当指令发给小爱解析（会触发它自己的 AI 回答），`play-text` 才是纯播报（参数须以 `_in=[text]` 传入，通用 run_action 塞 `value` 会被云端以参数个数不匹配拒绝，错误码 -704220025；此坑来自基于 mijiaAPI 3.x 的 miot-mcp 项目文档）。我方连接器的小米侧客户端在 S7 只需按此姿势调 `run_action('play-text')`，扫码登录与现有设备控制共用，无新增用户步骤。

**轮询半链（新增组件与新鉴权）**：对话记录接口为 MiNA 的 `https://userprofile.mina.mi.com/device_profile/v2/conversation`（xiaogpt 与 xiaoai-bridge 两条独立实现互证），记录含 query（小爱识别文本）、answers、毫秒时间戳；轮询间隔公开实现为 0.3–3 秒（xiaogpt 普通模式 3 秒、静默模式 0.3 秒；另一实测博客为约 1 秒）。接管动作链：发现新 query 且带约定前缀 → 先 `player_pause` 打断小爱原生回答（只能打断、拦不住第一声，与任务书第七节已知限制一致）→ 问大脑 → `text_to_speech` 播报。现成轮子：PyPI 的 **xiaoai-bridge**（轮询＋handler 回调＋TTS/音频播放，多音箱区分，灵感源 idootop/migpt-next）与 xiaogpt 本体；xiaoai-bridge 的许可证本次未查到明示，**未确认**，采用前须核对。

**MiService 登录的真实代价**：账号＋密码 MD5 走 serviceLoginAuth2，或 passToken/cookie 登录；公开实测有被小米风控拦下（海外登录大陆账号必挂、响应缺 userId 报错）的记录，xiaogpt 官方 FAQ 对策是改 cookie 登录。我方是国内网络、本人账号，风险量级低，但它与 mijiaAPI 扫码登录是**两套凭据**：密码须由用户经安全录入方式给一次并只存 Mac 本地受限目录——这一项要进任务书 S7 的「用户本人一步」清单，与扫码并列，不许后补惊喜。

### 3. 对照结论

- 我方 mijiaAPI 连接器覆盖：设备查询/控制/回读、场景、音箱播报（play-text）——S4 工具白名单②与 S7 播报半链全覆盖。
- 缺口只有一个：**对话记录轮询（MiNA）**。补法不是改 mijiaAPI，是在 Mac 侧并排加一个 MiService 客户端（或直接评估 xiaoai-bridge），与 mijiaAPI 各管一段、令牌分开存放。
- ha_xiaomi_home 的云端 MQTT 推送（属性变化主动推、不用轮询）是官方给 HA 的待遇，mijiaAPI 侧无对应推送接口；我方设备状态仍靠查询回读，与任务书 S4「回读验证」口径一致，不构成修订。

---

## 四、多终端 HUD：现成开源件

任务书 S9 现方案是 Mac 起局域网网页服务自研四态页面。现成件核查：

| 候选 | 形态 | 适配度 |
|---|---|---|
| **MagicMirror²**（MagicMirrorOrg，MIT，现行稳定版 v2.38.0，要求 Node ≥22.22.2 或 ≥24） | 模块化信息屏：时钟/日历/天气/新闻为内置模块，模块间有通知系统；**server-only 模式**可只跑服务端、任意浏览器打开 | **首选底座**：S9 要的时钟、天气、日程卡片全是现成模块；贾维斯四态与播报文本做一个自定义模块（订阅守护进程状态推送），不用自研框架、重连与布局 |
| Home Assistant Lovelace＋kiosk | 仪表盘强在设备面板 | 用户口径不用 HA，且 HUD 主体是语音状态而非设备墙，不选 |
| Dakboard | 托管服务、免费层有限 | 数据出局域网、要账号，不选 |
| 各类 GitHub「Jarvis HUD」单页模板 | 炫技型 Iron Man 风界面 | 只有皮没有状态机与数据接口，至多借视觉样式，不作底座 |

未确认项：小米电视（86″ Mini LED）自带浏览器对 MagicMirror² 客户端的渲染兼容性——MM2 前端是标准 Web 技术但带较多 JS，电视浏览器性能弱是已知风险类；S9 第一步先拿电视浏览器开 MM2 默认页实测，能开再定底座，开不动回退任务书的手写轻量单页（四态＋大字卡片，零框架）。

---

## 五、S1 推进清单（逐项第一步动作）

S1（常驻唤醒）开工前与开工首日动作，按序：

1. **外接麦克风到位确认**（S1/S2 的物理前置，Mac 无内置麦）：第一步——清点现有 USB 麦克风/声卡库存，无可用设备则把型号需求（USB、全向、3–5m 拾音）列入给用户的「本人一步」清单与 S7 登录清单合并递交，不单次打扰。
2. **Mac 侧 S0 基线单跑**：第一步——dsh 在 Mac 上按任务书模型清单下载三件套（KWS/ASR/TTS，官方源、文件名逐一对齐试验台笔记），各跑一条文件级样本，把 Mac 版 sherpa-onnx 版本号与运行日志落 `S0/env-report.md`。
3. **KWS 调参矩阵先行（本次新增）**：第一步——在 S0 通过后、S1 正式计数前，先录/合成一组标定音频：正样本「贾维斯」×10（不同语速音量）、近音负样本（加维斯/贾维思/假维斯）×10、电视外放录音 30 分钟；对 threshold {0.25, 0.35} × score {1.0, 0.5, 0.2} 六组参数各跑一遍，选定工作点再开始 S1 的 20 次计数，避免拿默认参数的工作点当结论。
4. **流式 ASR 候选冻结**：第一步——下载 sherpa-onnx 流式中文模型（Zipformer 中英双语流式版）一份，与 SenseVoice 用同一段 3 分钟中文录音对比「整段耗时＋首字延迟」，结果写进 S2 证据草稿，冻结实时链模型后再进 S2 正式测试集。
5. **回声门控设计落码前验证**：第一步——用 Mac 扬声器外放 Kokoro 播报样本、同时跑 KWS，记录播报期误触发次数，验证「播报期麦克风门控」策略的必要性与门控窗口长度，再写进守护进程状态机（待机/聆听/思考/播报四态与 S9 共用同一状态源）。
6. **MiService 可行性探针（为 S7 提前探路，不阻塞 S1）**：第一步——在 Mac 建独立 venv 装 `miservice_fork`，只做登录流程 dry-run 到「需要凭据」为止，把实际要用户给的东西（账号密码一次／或 cookie）写成清单项回审核层，与第 1 项合并成一张「本人一步」总清单。
7. **mijiaAPI 播报姿势核验（只读）**：第一步——用云端连接器对两台音箱跑 `info --model` 只读查询，确认 L16A 与 SM4 的 miot 规格里 `play-text` 动作的存在与参数形态，结论记入 S7 预研笔记；不发任何控制指令。
8. **HUD 底座探针**：第一步——查 Mac 的 Node 版本是否 ≥22.22.2；满足则在 Mac 本地以 server-only 模式起 MagicMirror² 默认页，再用小米电视浏览器打开默认页拍一张照片存证，兼容性结论回填 S9（电视操作等用户在家时段，与第 1、6 项清单合并安排）。

---

## 六、对 S0.5 决策书的修订点

决策书结论「借鉴自研、不 fork」**维持不变**，本次查证反而加固它（isair 的 STT/TTS 是 Whisper＋Piper，中文与延迟均不如我方已实测的 sherpa 栈；其架构五点借鉴已在任务书中落实）。需修订的是决策书/任务书的以下六处口径：

1. **S7 鉴权口径修订（最重要）**：任务书现写「Mac 侧米家客户端用 mijiaAPI 同款方案自建」——不准确。应拆为：设备控制＋音箱播报走 mijiaAPI（扫码登录，已有）；对话记录轮询走 MiNA/MiService（账号密码或 cookie 登录，新增凭据与风控面）。S7「用户本人一步」清单须新增「小米账号密码经安全录入给一次（仅存 Mac 本地受限目录）」一项，与扫码并列、开工前一次办齐。
2. **S7 播报实现修订**：播报不用等 MiNA 的 `text_to_speech`，mijiaAPI 的 `play-text` 动作即可（注意 `_in=[text]` 参数姿势与与 `execute-text-directive` 的区别）；MiNA TTS 降为备选。任务书 S7 证据项不变，但实现路径缩短一截。
3. **S2 模型口径修订**：实时链 ASR 由 SenseVoice（离线）改为 sherpa-onnx 流式模型，SenseVoice 转 S8 离线转写专用；S2 通过线新增「首字延迟」指标（建议 ≤600ms，Mac 实测为准），CER/RTF 口径不变。
4. **S1 证据口径修订**：新增近音负样本组与 score/threshold 调参矩阵为前置证据（见推进清单第 3 项）；2 小时误唤醒 ≤1 次的通过线维持，但报告中须注明公开旁证（微调模型 0 次/小时）只是上限参考、开箱模型以本机实测为准。
5. **S9 底座修订**：HUD 改为「MagicMirror² server-only＋一个自定义状态模块」为首选、手写单页为退路，选型以电视浏览器实测照片为据；任务书四态与卡片要求不变。
6. **异常分支补一条**：MiNA 轮询链新增「MiService 登录被风控/令牌失效」处置——识别登录失败后退避至 1 次/小时、写告警、音箱入口降级为「Mac 本机麦＋Mac 扬声器」继续可用，不许重试风暴（与现有米家令牌过期分支并列，触发条件不同、不要合并）。

## 来源清单（主要）

- sherpa-onnx 官方 KWS 模型文档：https://k2-fsa.github.io/sherpa/onnx/kws/pretrained_models/index.html
- Accessor 语音性能页（同款 KWS 模型实测）：https://t-lind.github.io/accessor/VOICE_PERFORMANCE.html
- xiaoguang-kws 模型卡（误唤醒/近音实测）：https://huggingface.co/Icerm/xiaoguang-kws
- 日语 phoneme KWS 模型卡（反面量级参照）：https://huggingface.co/mitsu-h/icefall-ja-phoneme-kws-cv25
- MAGI Consensus（KWS 资源占用）：https://www.tjabo.dev/projects/magi-consensus
- voice-agent-next 2026 语音链综述与延迟梳理：https://github.com/kadirnar/voice-agent-next/blob/HEAD/docs/research/REPORT.md 、 https://github.com/kadirnar/voice-agent-next/blob/HEAD/docs/research/05-latency-transports-production.md
- 五栈延迟第三方实测：https://dev.to/kenimo49/i-benchmarked-5-voice-ai-stacks-only-2-stayed-under-300ms-2bka
- 火山引擎 TTS 2026 实测文（首包延迟/计费）：https://developer.volcengine.com/articles/7667459245423788075 、 https://developer.volcengine.com/articles/7648595943507689526
- xiaogpt 轮询原理实测博客：https://meirong.dev/posts/xiaogpt-xiaoai-speaker-homelab/
- xiaogpt 管线与外部 API（DeepWiki 代码梳理）：https://deepwiki.com/y-hug/xiaogpt/4.5-query-processing-pipeline 、 https://deepwiki.com/y-hug/xiaogpt/7.4-external-apis
- xiaoai-bridge（MiNA 轮询现成件）：https://pypi.org/project/xiaoai-bridge/
- miot-mcp（mijiaAPI 的 play-text 姿势与坑）：https://unyly.org/mcp/miot-mcp
- ha_xiaomi_home 官方说明（OAuth/MQTT/本地控制）：https://raw.githubusercontent.com/XiaoMi/ha_xiaomi_home/main/doc/README_zh.md
- dsh-xiaoai-bridge 插件页（刷机线参照）：https://awesome-dsh-plugin.com/p/OMSociety/dsh-xiaoai-bridge/
- MagicMirror² 指南（版本与模块体系）：https://www.dotlinux.net/blog/magic-mirror-guide/
