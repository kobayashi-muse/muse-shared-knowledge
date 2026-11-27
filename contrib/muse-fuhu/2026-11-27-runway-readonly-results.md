# Runway 首测第 0 步「只读三连」实跑结果

时间：2026-11-27 凌晨（实跑）。执行范围：仅只读工具（status / list-tools / show_plans_and_credits / whoami）＋本地 ffprobe 只读测量。未调用任何媒体工具，未提交任何任务，未产生任何费用。

## 结论先行

- **连接正常**：`status` 返回 connected，`whoami` 认证通过。
- **余额**：Free 套餐，剩余 **70 积分**（套餐 50＋购买 20）。
- **单条成本估算：估不出来，因为根本跑不了**——`upscale_video` 在本工作区的可用模型列表为**空**，工具明文要求**付费套餐**，免费工作区提交会被 account_limitation 拒掉，积分多少与此无关（见下「假设推翻」第 1 条）。
- **方案假设被推翻 3 处**，详见文末。

---

## ① status（连接状态）

命令：`/opt/hatch/bin/runway-mcp status`
原始输出：

```json
{"ok":true,"status":"connected"}
```

## ② list-tools → upscale_video 真实 schema

命令：`/opt/hatch/bin/runway-mcp list-tools`（共返回 28 个工具，upscale_video 在列）。schema 全文如下（description 原文照录）：

### description

> Upscale or restore an existing video using Runway's AI video upscalers (4K with the default model, up to 8K with MediaKit, or a chosen scale factor with FLUX Video Upscale). Use this to sharpen, clean up, and increase the resolution of footage — it does NOT change the content, motion, style, or duration; for those edits use `generate_video` (video-to-video).
>
> CREDIT MODE: Runway account availability is checked against the currently connected workspace.
>
> PAID PLAN REQUIRED: Video upscaling needs a paid Runway plan unless the workspace holds a credit promotion that covers it. This tool always submits in Credit Mode, so an Explore Mode entitlement does not cover it. On a free workspace without one, the task is rejected with an account-limitation result: offer the upgrade card it points to and do not retry. There is no free upscaling alternative; do NOT try to fake an upscale with `generate_video`.
>
> WHEN TO USE:
> - The user wants a low-res / soft / blurry video made crisper or larger (e.g. "upscale this to 4K", "make this video higher resolution", "enhance the quality of this clip").
> - The user has a finished clip (their own upload, or an output from another tool that they re-uploaded) and wants a higher-fidelity version of the exact same video.
>
> WHEN NOT TO USE:
> - To restyle, edit, remove/replace backgrounds, remove objects, or otherwise change the content → use `generate_video` with `referenceVideo`.
> - To generate a brand-new video → use `generate_video`.
>
> INPUT:
> - Provide the source video as `video.url`. This must be a Runway-hosted `/datasets/<UUID>.<ext>` URL returned by `complete_upload` (pass its `assetId` as `video.assetId` too), OR a public HTTPS video URL (the tool imports it into Runway automatically).
> - Pass `video.durationSeconds` when known. With the default model, videos must be under 40 seconds.
>
> REUSING A VIDEO FROM THIS CHAT: if the source is a video a Runway tool generated earlier in this conversation, pass its task ID as `video.taskId` (no `url`, no `videoFile`). The server looks up the URL and duration. Do NOT ask the user to upload or download it.
>
> HANDLING USER ATTACHMENTS:
> - Claude/Cursor/local agents: if the user attached a local video, run `init_upload` -> curl -> `complete_upload` first to get a Runway-hosted asset URL and asset id, then pass them as `video.url` and `video.assetId`.
> - ChatGPT: if it supplies an uploaded video as `videoFile`, pass it directly. This tool will download the temporary ChatGPT file server-side, upload it to Runway, and use it as the source. NEVER pass local paths such as `/mnt/data/file.mp4` as `video.url`; use `videoFile` instead.
>
> REUSING ASSETS: `complete_upload` returns a stable, hosted asset URL and its asset id — pass both as `video.url` and `video.assetId`, never re-upload. Without the id, the tool imports a copy of the video into the user's Assets.
>
> USER-FACING REPLIES: Talk about the result ("upscaling your video to 4K"), not the underlying engine. Do not mention the model/provider unless the user explicitly asks.
>
> DEFAULT MODEL (`topaz-video-upscale`): upscales to 4K and keeps the look of the source. Clips must be under 40s. Prefer it unless the user wants restoration, a different output size, a longer clip, or names MediaKit.
>
> AI MEDIAKIT MODELS (optional `model` values):
> - `mediakit-enhance` (AI MediaKit Enhance): pick for restoration and sharpening that keeps the source look, optionally upscaled; good for compressed, noisy or old footage. Source: up to 300s, at most 1440px on the short side and 2560px on the long side. Output: the source resolution unless `resolution` is set. Cost: about 21 credits per minute of video at up to 720p and 30 fps output, x2 for 1080p output, x4 for 2K, x8 for 4K, x16 for 6K, x32 for 8K; x2 above 30 fps and x4 above 60 fps; each task rounds up to whole credits.
> - `mediakit-enhance-fast` (AI MediaKit Enhance Fast): pick for the cheapest quick clean-up or upscale of long clips. Source: up to 300s, at most 1440px on the short side and 2560px on the long side, at least 360px on each side. Output: the source resolution unless `resolution` is set. Cost: about 10.3 credits per minute at up to 720p and 30 fps output, x2 for 1080p output, x4 for 2K, x8 for 4K, x16 for 6K, x32 for 8K; x2 above 30 fps and x4 above 60 fps; each task rounds up to whole credits.
> MediaKit needs a paid plan and is not enabled on every workspace: only pass one of these models when `whoami` lists it in `availableVideoToolModels.upscale_video`. On a model_unavailable result, tell the user and offer the default model. Do not retry the MediaKit model.
>
> OTHER MODELS (optional `model` values):
> - `flux-video-upscale` (FLUX Video Upscale): pick for a chosen scale factor (`upscaleFactor` 1.5–3, default 2) instead of a fixed output size, or an upscale steered by a prompt. `creativity` `precise` keeps the source as it is; `creative` (default) regenerates fine detail and costs more. `promptText` optionally names details to preserve or enhance. Source: up to 20s, at most 2560x1440px and 50 MB. Output: the source size times `upscaleFactor`, at most 14.4 megapixels per frame. Cost: 3.5 credits (`precise`) or 5 credits (`creative`) per megapixel of an output frame, per second of video; a 10s 1280x720 clip at 2x is about 130 or 185 credits.
> These are not enabled on every workspace: only pass one when `whoami` lists it in `availableVideoToolModels.upscale_video`. On a model_unavailable result, tell the user and offer the default model. Do not retry that model.
>
> Parameters:
> - video: `{ url, assetId?, durationSeconds?}` — the source video to upscale. URL must be public HTTPS or a Runway-hosted `/datasets` URL; `assetId` is the id `complete_upload` or a previous call returned for that URL. Do NOT use ChatGPT local paths like `/mnt/data/...`; use `videoFile` for ChatGPT uploads.
> - videoFile: ChatGPT-only uploaded video file param `{download_url, file_id, mime_type, file_name}`. Use this instead of `video` only when ChatGPT provides a user-uploaded file.
> - model: Optional. `topaz-video-upscale` (default), a MediaKit model or another model above.
> - resolution: MediaKit models only. Output size set by the SHORT side: `1080p` is 1080px, `2k` 1440px, `4k` 2160px (3840x2160 for 16:9). `mediakit-enhance` takes 240p/360p/480p/540p/720p/1080p/2k/4k/6k/8k, `mediakit-enhance-fast` 240p/360p/480p/540p/720p/1080p/2k/4k. Set it when the user names a size ("upscale to 4K" → `4k`); omit it to keep the source size.
> - upscaleFactor: `flux-video-upscale` only. 1.5–3. Default: 2.
> - creativity: `flux-video-upscale` only. `precise` keeps the source as it is; `creative` (default) regenerates fine detail and costs more. `promptText` optionally names details to preserve or enhance.

### input_schema

```json
{
"type": "object",
"additionalProperties": false,
"properties": {
"video": {
"type": "object",
"additionalProperties": false,
"description": "Source video to upscale. URL must be a public HTTPS video URL or a Runway-hosted /datasets/<UUID>.<ext> URL returned by complete_upload. Do not pass ChatGPT local paths like /mnt/data/...; use videoFile instead.",
"properties": {
"url": { "type": "string", "description": "Public HTTPS or Runway-hosted video URL. Never use /mnt/data or other local file paths here."},
"assetId": { "type": "string", "description": "Runway dataset asset id for the source video. Do NOT invent this — only pass the id complete_upload or a previous call returned for the SAME video URL, so the tool reuses that asset instead of importing a copy."},
"durationSeconds": { "type": "number", "description": "Source video duration in seconds. Pass when known — videos must be under 40s."},
"taskId": { "type": "string", "description": "Runway task ID of a finished video generation from earlier in this conversation (e.g. a generate_video result). The server resolves the video URL and duration itself — pass this instead of url, and never ask the user to upload a video Runway just generated."}
}
},
"videoFile": {
"type": "object",
"additionalProperties": false,
"description": "ChatGPT-only file param for a user-uploaded source video. Prefer video.url when you already have a public or Runway-hosted URL.",
"properties": {
"download_url": { "type": "string", "description": "Temporary ChatGPT file download URL."},
"file_id": { "type": "string", "description": "ChatGPT file ID."},
"mime_type": { "type": "string", "description": "MIME type, e.g. video/mp4."},
"file_name": { "type": "string", "description": "Original file name."}
},
"required": ["download_url", "file_id"]
},
"model": {
"type": "string",
"enum": ["topaz-video-upscale", "mediakit-enhance", "mediakit-enhance-fast", "flux-video-upscale"],
"description": "Upscale model. Default: 'topaz-video-upscale'. Other models only when whoami lists them in availableVideoToolModels.upscale_video."
},
"resolution": {
"type": "string",
"enum": ["240p", "360p", "480p", "540p", "720p", "1080p", "2k", "4k", "6k", "8k"],
"description": "MediaKit models only: output short side (1080p = 1080px, 2k = 1440px, 4k = 2160px). Omit to keep the source size."
},
"upscaleFactor": { "type": "number", "minimum": 1.5, "maximum": 3, "description": "flux-video-upscale only. Default: 2."},
"creativity": { "type": "string", "enum": ["precise", "creative"], "description": "flux-video-upscale only. Default: 'creative'."},
"promptText": { "type": "string", "maxLength": 4000, "description": "flux-video-upscale only: optional details to preserve or enhance."},
"confirmRepeat": { "type": "boolean", "description": "Set true only when the user explicitly asked to run this same request again. Without it, an identical request from the last 10 minutes returns the earlier task instead of starting (and charging for) a new generation."},
"rationale": { "type": "string", "description": "Why are you calling this tool? What is the user actually trying to accomplish? Be specific."}
}
}
```

### output_schema（要点）

结果以 `kind` 区分：`task_pending` / `task_batch_pending`（已受理、后台运行）、`account_limitation`（工作区无权运行，带 `capability`/`upgradeUrl`/`nextTool`）、`model_deprecated`；成功类结果带 `taskId`、`creditsRemaining`、`lowCredits`、`isFreePlan`、`meta`（模型/画幅/分辨率/时长/音频标志）等字段。`required: ["kind"]`。

### schema 关键参数速览

| 项 | 值 |
|---|---|
| 默认模型 | `topaz-video-upscale` → 输出 4K；**片长必须 <40 秒** |
| MediaKit（enhance / enhance-fast） | 源片 ≤300 秒、短边 ≤1440px、长边 ≤2560px；输出档位按短边 240p–8k（fast 到 4k） |
| FLUX | 源片 ≤20 秒、≤2560×1440、≤50MB；倍数 1.5–3 |
| 输入 | Runway 托管 `/datasets/<UUID>.<ext>` URL（先 init_upload→complete_upload）或公开 HTTPS URL，或既有任务 taskId |
| 付费门槛 | **明文 PAID PLAN REQUIRED**：免费工作区（无积分促销覆盖时）提交即 account_limitation 拒绝，无免费替代 |
| 默认模型单价 | schema **未给** topaz 的积分单价（只有 MediaKit/FLUX 的费率），仍未确认 |

## ③ show_plans_and_credits（套餐与余额）

命令：`/opt/hatch/bin/runway-mcp call-tool --name show_plans_and_credits --arguments-json '{}'`
原始输出要点（structuredContent）：

- 当前套餐：**Free**（`isFreePlan: true`，`planName: "Free"`）
- 剩余积分：**70**（`credits: {plan: 50, purchased: 20, total: 70, unlimited: false}`）
- 工作区 ID：71948935
- 付费档（输出所载价格，仅作决策数据记录）：Standard $19/月（年付 $15/月）、625 积分/月、含「4K upscaling」；Pro $35/月（年付 $28/月）、2250 积分/月、含「Unlimited 4K video upscaling」；Max $95/月（年付 $76/月）、9500 积分/月。
- 积分包：$10/1000 积分起，但 Free 套餐下 `topUp.available: false`（免费档不能单独买积分包）。

## ④ 附加只读核验：whoami（模型可用性）

命令：`/opt/hatch/bin/runway-mcp call-tool --name whoami --arguments-json '{}'`
原始输出要点：

- 认证通过；工作区「桂东」（id 71948935，个人）；模型可用性数据为 fresh（2026-11-26T17:09:55Z 检查）。
- 原文首行：**"This workspace is on the Free plan — no video, speech, sound-effect, or music models are available."**
- `availableVideoToolModels`：`upscale_video: []`、`enhance_frame_rate: []`、`remove_video_background: []`、`edit_video: []` —— **全部为空**。
- `availableVideoModels: []`、`availableSpeechModels: []`、`availableMusicModels: []`、`availableSoundEffectModels: []`、`availableImageUpscaleModels: []`、`availableMeshModels: []`。
- 实际可用的只有：图片生成模型（nano-banana-pro / 2.1 / 2 / 2-lite、p-image-ideogram、gen-4、gen-4-image-turbo）与背景移除（sam3）。
- 积分与 ③ 一致：total 70。

## ⑤ 样本实测（本地 ffprobe，只读）

样本：`~/workspace/social-ops/week3/assets/W3D4-video-final.mp4`

| 项 | 实测值 |
|---|---|
| 时长 | **40.656 秒** |
| 分辨率 / 帧率 | 1080×1920 / 30fps（h264） |
| 音轨 | AAC（在位） |
| 文件大小 | 36,247,043 字节（约 34.6 MiB） |

## ⑥ 单条成本估算

- **现状下不可执行，成本无从谈起**：可用模型列表为空＋免费套餐硬门槛，提交必被拒，与 70 积分余额无关。
- 假设将来开通付费套餐后的参考估算（W3D4：40.656 秒 ≈ 0.678 分钟，源短边 1080px，30fps）：
- `mediakit-enhance-fast`、输出保持 1080p 档：10.3 × 2 × 0.678 ≈ **14 积分/条**；
- `mediakit-enhance`、输出 1080p 档：21 × 2 × 0.678 ≈ **29 积分/条**；
- `mediakit-enhance`、输出 4K 档：21 × 8 × 0.678 ≈ **114 积分/条**（已超当前 70 余额）；
- `topaz-video-upscale`（默认）：schema 未公布单价，**未确认**；且 W3D4 时长超其 <40 秒上限（见下），整片路线不可用；
- `flux-video-upscale`：源片 ≤20 秒，W3D4 不适用。
- 注：MediaKit 是否在付费后即对本工作区开放，schema 只说「not enabled on every workspace」，开通后仍须以 whoami 复核为准。

---

## 与方案假设不符之处（逐条对账）

1. **方案默认「已连接即可调用 upscale_video，区别只是花多少积分」——错。** 方案 §一.5 与 §五只把「单价、余额」列为未确认，没预见**套餐门槛**：当前 Free 套餐下 `availableVideoToolModels.upscale_video` 为空数组，工具明文 PAID PLAN REQUIRED。方案第 2 步（上传＋提交）在现状下必然吃 account_limitation，**首测链条在第 2 步断掉，前置条件变成「先有付费套餐」**，这是决策项（Standard $19/月起），不是执行项。
2. **方案称样本「整片约 50 秒」——实测 40.656 秒。** 且默认模型 topaz 要求 <40 秒，40.656 秒**仍然超限 0.656 秒**：即便开通付费，衔接点 A（整片＋默认模型）照样被拒，只能走 MediaKit（≤300 秒）或衔接点 B 分段（每段约 8 秒，天然合规）。方案 §四对 A 的风险评估只提了字幕重绘，没提时长上限。
3. **方案 §五.2 问「余额够不够」——70 积分确实在账上，但对升频毫无意义**：免费档连积分包都买不了（topUp.available=false），积分只能花在图片生成/去背景等已开放工具上。方案失败兜底里的「余额不足：停手回报差额」一条对升频不适用，应改为「套餐不达标：停手回报」。
4. **** 方案 §四衔接点 C 登记的 `generate_speech`、`remove_video_background`、`upscale_image` 三个备选，在 Free 套餐下可用模型同为**空**；当前真正能用的只有 `generate_image`（gen-4 / nano-banana 系）与图片/视频去背景的 sam3（`availableBackgroundRemovalModels: ["sam3"]`，其工具侧可用性以实际调用为准）。C 的登记内容须按此修正。
5. **** §五.1 部分销账：输出档位已明（topaz→4K；MediaKit 按短边 240p–8k；FLUX 倍数 1.5–3），时长/大小上限已明（topaz <40s；MediaKit ≤300s；FLUX ≤20s/≤50MB），MediaKit/FLUX 单价已明；**topaz 单价仍未确认**。§五.2 销账：Free 套餐、70 积分。§五.3（音轨/帧率）仍未确认——且在开通付费前无从实测。

## 下一步（供主聊天决策，本轮未执行）

- 若决定继续 Runway 升频线：先定套餐（Standard $19/月含 4K upscaling、625 积分/月；或 Pro 的 Unlimited 4K upscaling），开通后重跑 whoami 确认 `upscale_video` 模型列表非空，再按衔接点 B（分段）走首测，验收标准沿用方案 §三。
- 若不付费：Runway 升频线整体搁置，「高清修复收尾」另找开源/本地工具（此为另一专题，本轮不展开）。
