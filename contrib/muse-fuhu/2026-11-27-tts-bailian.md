# 百炼 CosyVoice API 备线深学（课程台账专题 2 · 第二轮）

- 日期：2026-11-27 凌晨
- 前置：`2026-11-26-tts-voice.md`（推荐已定：CosyVoice3-0.5B 本地线＋百炼 API 备线）、`2026-11-26-cosyvoice-test.md`（本地线卡点：执行克隆仓库代码须用户明确同意；本机无 GPU、可用内存不足 1Gi）
- 本轮任务：本地线被用户同意卡住，推进备线——阿里云百炼 CosyVoice API 路线，做到决策就绪。
- 口径：只读官方文档与公开网页、只读已连服务连接器文档；**未创建账号、未填凭证、未调用任何 API、未花一分钱、未克隆/运行任何第三方代码**。查不到的写「未确认」，不编。没实跑过的不许写已验证。
- **状态：已学未落地**——备线决策就绪、样音未出。卡点见第五节。

## 一、结论先行

1. **备线可行、成本极低、费用不是门槛**：百炼 `cosyvoice-v3.5-flash` 合成 0.8 元/万字符、`cosyvoice-v3.5-plus` 1.5 元/万字符（北京地域官方模型页原价）；一条 60 秒口播约 150–200 字，约 0.012–0.03 元；日更一条月成本约 0.5–1 元。
2. **重要纠正（以前一笔记为准绳）**：前一笔记写「声音复刻 9.9 元/个」，该价引自第三方网关文档、**不是阿里官方价**。阿里云百炼官方《Voice cloning》指南原文：Qwen-Audio-TTS / Qwen-Audio-Realtime / CosyVoice 的「Voice creation is free」（创建音色免费，合成按目标模型计费）。另有第三方整理的官方定价口径称 `qwen-voice-enrollment` 按 0.01 元/音色、免费 1000 个/账号——两处口径有出入，**以百炼控制台实际账单为准**，预算时按「创建免费或近乎免费」处理，不再按 9.9 元/个报价。
3. **3.5 代无系统音色是备线第一道工序**：`cosyvoice-v3.5-plus` / `cosyvoice-v3.5-flash` 不提供预置音色，合成前必须先声音复刻（或声音设计）拿到专属音色 ID；音色绑定 `target_model`，plus 建的音色不能用于 flash 合成，反之亦然。
4. **本地线 CPU 判定已可先下结论**：本机可用内存仅 0.3–0.7Gi，加载 2.5GB 权重＋torch 运行时在内存这一项上**直接判定跑不动**，不必等用户同意后再试内存；用户同意后也应改在免费 GPU 或内存更大的机器上跑（判定标准见第四节）。
5. **已连服务里没有现成可用的中文自然口语 TTS**：唯一文档确认的语音生成入口是 Runway 的 `generate_speech`（须逐次用户批准＋耗 Runway 积分，音色不可控），只能作对照样音备选，不作主线。

## 二、百炼 CosyVoice 路线要点（附官方来源）

### 2.1 模型与价格（北京地域，官方模型页原价，2026-11-27 核对）

| 模型 | 定位 | 合成单价（元/万字符） | 限流 |
|---|---|---|---|
| `cosyvoice-v3.5-flash` | 轻量档，低延迟、自然度够用 | 0.8 | 180 RPM |
| `cosyvoice-v3.5-plus` | 旗舰音质档 | 1.5 | 180 RPM |
| `cosyvoice-v3-flash` | 上一代轻量（有系统音色） | 1.0 | — |
| `cosyvoice-v3-plus` | 上一代旗舰（有系统音色） | 2.0 | — |

- 官方模型页：https://help.aliyun.com/en/model-studio/cosyvoice-v3-5-flash ；https://help.aliyun.com/en/model-studio/cosyvoice-v3-5-plus
- 计费口径：按输入文本字符数计费（响应 `usage.characters` 为准），输出音频不收费；SSML 标签是否计入字符数，官方文档未明言。
- **免费额度：口径不一致，未确认**。模型页未列免费额度行；第三方评测称新加坡地域 1 万字符/90 天；另一来源称免费额度仅限北京地域实时推理且须实名认证。**不要按「免费 1 万字符」做预算**，以百炼控制台实际展示为准。

### 2.2 调用方式（官方 HTTP API 文档）

- 来源：https://www.alibabacloud.com/help/en/model-studio/cosyvoice-tts-http-api
- Endpoint（北京）：`POST https://{WorkspaceId}.cn-beijing.maas.aliyuncs.com/api/v1/services/audio/tts/SpeechSynthesizer`
- 鉴权：`Authorization: Bearer <DASHSCOPE_API_KEY>`；API key 与 workspace ID、地域必须匹配（北京与新加坡的 key 不通用）。
- 另有三种接入：DashScope Python SDK（≥2.0.1）、WebSocket 流式、AOQ 接入。非流式返回音频 URL（有效期 24 小时，过期重合成）；流式用 `X-DashScope-SSE: enable`，Base64 音频块按序拼接。
- 对管线友好：纯 HTTP POST、可直接写进无人值守脚本；无需 GPU、无需维护服务。

### 2.3 合成参数（与自然口语相关，官方 HTTP API 文档）

| 参数 | 作用 | 取值 |
|---|---|---|
| `input.voice` | 音色 ID（必填；3.5 只能填复刻/设计音色 ID） | 如 `cosyvoice-v3.5-flash-<prefix>-<id>` |
| `input.instruction` | 自然语言指令，控方言、情感、角色 | 字符串；第三方口径称 ≤100 字符（官方文档未标上限，以官方为准） |
| `input.rate` | 语速 | 0.5–2.0，默认 1.0 |
| `input.pitch` | 语调 | 0.5–2.0，默认 1.0 |
| `input.volume` | 音量 | 0–100，默认 50 |
| `input.enable_ssml` | SSML：控语速/语调/停顿/发音、加背景音乐音效 | true/false；3.5 全系支持 |
| `input.seed` | 随机种子；同模型同文本同参数同种子可复现同一结果 | 0–65535 |
| `input.language_hints` | 目标合成语言（数字、缩写、符号读法纠偏用） | 如 `["zh"]` |
| `input.word_timestamp_enabled` | 逐词时间戳（仅流式＋复刻音色） | true/false；**对管线逐字字幕对齐有用** |
| `input.enable_markdown_filter` | 把 Markdown 转为适合朗读的文本再合成 | true/false |
| `input.format` / `sample_rate` | 输出编码/采样率 | mp3/pcm/wav/opus；8000–48000 Hz |

- 指令控制写法（官方指南＋第三方教程一致）：方言用固定句式「请用X话表达。」（如「请用四川话表达。」），情感如「用温柔的语气，语速稍慢。」；不要写成「用四川话说这句话」等非固定写法，可能被判无效指令。
- 注意：`enable_aigc_tag`（隐形 AIGC 标识嵌入）仅 `cosyvoice-v3-flash` / `v3-plus` / `v2` 支持，**3.5 系列不支持该参数**；发布标注仍按现行管线口径在平台侧标「含 AI 生成内容」。
- 来源：https://www.alibabacloud.com/help/en/model-studio/cosyvoice-tts-http-api ；https://help.aliyun.cn/en/model-studio/ssml-latex-user-guide

### 2.4 声音复刻（官方《Voice cloning》指南，2026-11-27 核对）

- 来源：https://docs.bailian.console.aliyun.com/en/model-studio/voice-cloning-user-guide ；HTTP API 参考：https://www.alibabacloud.com/help/en/model-studio/voice-clone-design-http-api
- 流程两步：① 创建音色（`model=voice-enrollment`，`action=create_voice`，传 `target_model`＋`prefix`＋参考音频 URL 或 Base64 Data URI）→ 得 `voice_id`；② 合成时 `model` 与创建时同一模型、`voice` 填该 `voice_id`。
- 参考音频要求（官方表）：
  - 格式：WAV（16-bit）、MP3、M4A；大小 ≤10MB；采样率 ≥16kHz；单声道或双声道（只取第一声道）。
  - 时长：推荐 10–20 秒，不超过 60 秒；内容至少 5 秒连续清晰朗读、其余停顿 ≤2 秒。
  - 内容：正常语速的自然说话，**不要唱歌、不要背景音乐、不要环境噪声、不要他人声音**；录音脚本用完整句子、语义连贯、带适当情感表达（温暖/友好/严肃），不要机械朗读；不要含敏感内容（政治/色情/暴力），否则复刻失败。
  - 录音环境建议：10 平米以下小房间、关门窗、关空调风扇、距设备约 10cm。
- 语言：`cosyvoice-v3.5-plus/flash` 参考音频支持普通话及广东话、东北话、甘肃、贵州、河南、湖北、江西、闽南语、宁夏、山西、陕西、山东、上海话、四川话、天津话、云南等方言＋英/法/德/日/韩/俄/葡/泰/印尼/越南语。
- 管理：账号自定义音色上限 1000 个（复刻与设计共用）；音色可查询/更新参考音频（保留同一 voice_id）/删除；**复刻音色 30 天未使用会被系统自动清理**（第三方教程口径，官方指南未明言——管线须定期调用保活或接受重建）。
- 计费：官方指南原文「Voice creation is free」；合成按目标模型单价计费。MiniMax 线另有首次使用 9.9 元解锁费，与 CosyVoice 无关，不要混淆。
- 合规红线（与前一笔记一致）：只许用本人或有明确授权的声音做复刻；用未授权真人声音克隆发布属禁区。

### 2.5 声音设计（无录音备选）

- 不用参考音频，用自然语言描述生成音色（描述 ≤500 字符，仅中/英文；预览文本 15–200 字符）。
- 适用：用户不便录音、或想先快速出一个固定口播音色试听时；缺点是音色为平台生成、不是本人声，长期品牌声线仍应走复刻。
- 来源：同上克隆指南（Voice design）。

### 2.6 费用预估（按官方原价算，免费额度未计入）

- 样音阶段：复刻创建官方口径免费；同稿测试稿约 40 字，flash 合成约 0.003 元/次——样音成本可忽略。
- 生产阶段：一条 60 秒口播约 150–200 字 → flash 约 0.012–0.016 元/条，plus 约 0.024–0.03 元/条；日更 1 条、月 30 条 → flash 约 0.5 元/月，plus 约 1 元/月。
- 结论：**费用不是决策门槛**；决策门槛是账号开通＋实名＋参考音录制＋用户点头。

## 三、已连服务 TTS 盘点（只读连接器文档，未调用、未花钱）

| 服务 | 连接状态 | 文档中的语音生成能力 | 能否出自然口语中文配音 | 备注 |
|---|---|---|---|---|
| Runway（runway_mcp） | 已连接 | 文档明确列出 `generate_speech` 音频工具 | 未确认 | 每次调用须用户批准、消耗 Runway 积分；音色为 Runway 自有语音生成，不可克隆指定音色、中文自然度未确认；只可作对照样音备选，不作主线。未查余额（只读文档）。 |
| Descript（descript） | 已连接 | 文档只列项目查找/编辑/发布、媒体导入、转写与时间线导出，**未列出 TTS / Overdub 语音合成工具名** | 未确认，不计入可用 | 连接器层面无语音合成入口；Descript 产品本身的 Overdub 不在本连接器文档能力内，不作数。 |
| Groq（custom.groq） | 已连接 | Whisper 转写模型 | 否 | 只有语音转文字，无 TTS。 |
| 其余已连服务 | — | 文档中无语音合成功能 | 否 | Canva、Gamma、Notion、Todoist 等均与 TTS 无关。 |

- 结论：**已连服务里没有现成可直接出自然口语中文配音的 TTS**；配音线仍须走百炼 API（备线）或本地 CosyVoice（本地线）。

## 四、对比结论：备线出「同稿样音」下一步清单

### 4.1 备线（百炼）下一步——决策就绪，待三项本人步

| # | 步骤 | 谁做 | 状态 |
|---|---|---|---|
| 1 | 开通阿里云账号＋百炼（华北2北京地域）＋实名认证 | 用户本人（账号类本人步，助手不代注册） | 待用户 |
| 2 | 百炼控制台创建 API Key（DASHSCOPE_API_KEY）＋取 Workspace ID，经安全录入卡交助手 | 用户本人创建、助手只接收凭证 | 待用户 |
| 3 | 录 10–20 秒参考音：本人自然口语（日常说话语气，不要朗读腔），WAV/MP3/M4A，≥16kHz，≤10MB，并明确授权仅用于本管线自用克隆 | 用户本人 | 待用户 |
| 4 | 用参考音调创建音色（`cosyvoice-v3.5-flash` 优先，成本低；同稿再做 `plus` 版对照） | 助手（有 key 后） | 待 1–3 |
| 5 | 同稿合成测试稿（沿用 2026-11-26 测试稿：「今天我们去茶园看看，新摘的茶叶已经送到合作社了，一共一百二十三斤。」＋数字/多音字/中英混读/长句四类句子），指令「用轻松聊天的语气，语速稍慢」，rate 约 0.95 | 助手（有 key 后） | 待 1–3 |
| 6 | 三方同稿盲听：现新闻腔 vs 本地线 vs 百炼线；逐项计数断句/多音字/数字/电音吞字；用户亲耳点头才批量 | 用户亲耳定版 | 待样音 |

- 备线不依赖本地线用户同意项：百炼线只需上述 1–3 三项本人步，不动克隆仓库代码。
- 若用户不想录音：可改走「声音设计」出平台生成音色先行试听，复刻本人声以后再补。

### 4.2 本地线 CPU 跑不动的判定标准（供用户同意后实测用）

- **硬门槛（内存，先下结论）**：权重 2.5GB＋torch 运行时，加载需数 GB 可用内存；本机勘察可用仅 0.3–0.7Gi → **本机现状直接判定跑不动**，不必等同意后再验证内存这一项。用户同意后也应改在免费 GPU（Kaggle T4 / 魔搭 A10）或内存更大的机器上跑，不要在本机硬试。
- **速度参考（第三方实测，非本机）**：CosyVoice 3 在无 GPU 笔记本 CPU 上约 8 秒音频需约 35 秒生成（RTF 约 4.4 倍）；本机为运行时共享 1.5 核，预期更慢。来源：https://terryl.in/ja/blog/cosyvoice-3-voice-cloning
- **实测判定线（同意后在有足够内存的机器上跑）**：
  1. 加载即 OOM 被杀 → 本机/该机 CPU 线死，转免费 GPU 或百炼；
  2. 单句约 15 字超过 3 分钟未出音 → CPU 线死；
  3. RTF > 10（60 秒配音超 10 分钟）→ 批量不可用，转免费 GPU 或百炼；
  4. RTF ≤ 5 且内存扛住 → 本机线可作离线批量备选，但仍以百炼为生产主线（免运维）。
- 前一笔记的卡点（执行克隆仓库代码须用户明确同意）未变，仍在等用户点选。

### 4.3 两线分工建议（待用户定版）

- 生产主线：百炼 API（免运维、按字计费极低、商用条款干净、指令控情感语速、逐词时间戳可接字幕对齐）。
- 离线批量备选：本地 CosyVoice 3 开源权重（零接口费），前提是有 GPU 或足够内存的机器；本机现状不满足。
- 对照样音备选：Runway `generate_speech`（已连服务，须审批＋花积分），仅在需要第三方音色对照时用一次。

## 五、仍卡住的点（已学未落地的原因）

1. **百炼账号与凭证**：无阿里云账号、无 API Key、无 Workspace ID——须用户本人开通＋实名＋创建 key（本人步，助手不代办）。
2. **参考音频**：无本人录音——须用户本人录 10–20 秒自然口语并明确授权克隆自用。
3. **免费额度口径未确认**：模型页未列免费额度；第三方口径互不一致（新加坡 1 万字符/90 天 vs 仅北京实时推理须实名）——以百炼控制台为准，预算未计入免费额度。
4. **复刻创建计费口径有出入**：官方克隆指南称创建免费，第三方定价整理称 `qwen-voice-enrollment` 0.01 元/音色（免费 1000 个/账号）——以控制台账单为准；无论哪种，金额都可忽略，不影响决策。
5. **本地线用户同意未到**：执行克隆仓库代码仍须用户明确同意（前一笔记卡点）；且本机内存已先判定跑不动，同意后也应换机器跑。
6. **自然度无实测数据**：百炼 3.5 相对开源 3.0 的自然度只有官方口径，无独立评测；最终以同稿盲听＋用户亲耳为准。

## 六、风险玩法不采纳声明

- **多号轮换、第三方 API 反代、绕封号、买号，一律不采纳**。原因：① 反代/多号违反阿里云服务条款，账号封禁风险直接断供生产管线；② 凭证经第三方中转有泄露风险，且无法保证计费与内容合规；③ 本任务已有官方直连的低价正路（0.8 元/万字符），无任何必要走灰色中转。笔记与派单中不出现此类路径。
- 同理：不走非官方的 CosyVoice 第三方网关（如简创 AIGC 等）调用百炼——其 9.9 元/个复刻加价与官方免费创建口径冲突，且多一层凭证托管；只走阿里云官方 endpoint。

## 七、来源汇总

1. 百炼《Voice cloning》官方指南（复刻流程/音频要求/免费创建/配额）：https://docs.bailian.console.aliyun.com/en/model-studio/voice-cloning-user-guide
2. CosyVoice HTTP API 官方文档（endpoint/参数/响应）：https://www.alibabacloud.com/help/en/model-studio/cosyvoice-tts-http-api
3. Qwen-Audio / CosyVoice 声音复刻 HTTP API 参考：https://www.alibabacloud.com/help/en/model-studio/voice-clone-design-http-api
4. `cosyvoice-v3.5-flash` 官方模型页（0.8 元/万字符/180 RPM）：https://help.aliyun.com/en/model-studio/cosyvoice-v3-5-flash
5. `cosyvoice-v3.5-plus` 官方模型页（1.5 元/万字符/180 RPM）：https://help.aliyun.com/en/model-studio/cosyvoice-v3-5-plus
6. SSML 与 LaTeX 官方指南（3.5 支持 SSML）：https://help.aliyun.cn/en/model-studio/ssml-latex-user-guide
7. 百炼计费口径第三方整理（qwen-voice-enrollment 0.01 元/个、免费 1000 个；免费额度 90 天、仅北京）：https://www.xxcmw.com/rgzn/202611/4485.html
8. CosyVoice 计费字符口径分析（按字符非 token、音频 URL 24 小时有效）：https://ponr.org/en/articles/speech-hanzi-counts-twice.html
9. CosyVoice 3 CPU 实测（无 GPU 笔记本约 8 秒音频 35 秒生成）：https://terryl.in/ja/blog/cosyvoice-3-voice-cloning
10. 第三方网关文档（仅作对照：其 9.9 元/个复刻价与官方口径冲突，不采纳）：https://docs.jcaigc.cn/voice/voice-clone.html ；https://docs.jcaigc.cn/voice/tts.html
11. 指令控制第三方教程（方言固定句式、情感标签、30 天未用清理口径）：https://www.yuntts.com/1134.html
12. 连接器文档（只读）：`/opt/hatch/connectors/runway_mcp/CONNECTOR.md`（`generate_speech` 在列）、`/opt/hatch/connectors/descript/CONNECTOR.md`（无 TTS 工具名）
13. 前置笔记：`~/workspace/learning/deep/2026-11-26-tts-voice.md`、`~/workspace/learning/deep/2026-11-26-cosyvoice-test.md`
