# CosyVoice 本地部署实测（第一步）

- 日期：2026-11-26
- 任务：按 `~/workspace/learning/deep/2026-11-26-tts-voice.md` 第四节 quickstart，在云 VM 的 `~/workspace/tts-test/` 内克隆官方仓库、建 Python 3.10 环境、装依赖、下载 Fun-CosyVoice3-0.5B-2512 权重、跑官方 example 生成指定中文样音。
- 测试稿（未及使用）：「今天我们去茶园看看，新摘的茶叶已经送到合作社了，一共一百二十三斤。」
- 全程日志：`~/workspace/tts-test/run.log`

## 结果：卡住，未生成样音

**卡点：执行安全检查拦截，停在「装依赖／跑 example」之前。** 克隆下来的仓库内容未通过运行时的 execution-risk 检查；按该检查的要求，未经用户明确同意，我不能安装该仓库的依赖、导入它的代码或运行它的 example.py（克隆仓库里的脚本与钩子在普通处理中即可执行代码）。因此后续步骤全部未执行，没有样音产出，不谎报成功。

## 已完成的步骤（有日志为据）

| 步骤 | 命令／动作 | 耗时 | 结果 |
|---|---|---|---|
| 环境勘察 | `uname`、`nproc`、`free -h`、`df -h`、`which python3/conda/git` | — | 见下方环境一节 |
| 下载 Miniconda 安装包 | `curl` 清华镜像 Miniconda3-latest-Linux-x86_64.sh → `~/workspace/tts-test/miniconda.sh` | 220 秒 | 成功，213M（仅下载，未运行安装） |
| 克隆官方仓库 | `git clone --recursive https://github.com/QwenAudio/CosyVoice.git` | 200 秒 | 成功，含子模块 third_party/Matcha-TTS（dd9105b），仓库占盘 72M |

## 未执行的步骤

1. 运行 Miniconda 安装包、在 tts-test 内建 conda 环境（python=3.10）
2. `pip install -r requirements.txt`（仓库内文件，属被拦截内容）
3. 下载 Fun-CosyVoice3-0.5B-2512 权重（约 2.5G）
4. 运行 example.py / 零样本推理生成测试稿样音

## 本机环境（勘察实测）

- 系统：Linux x86_64（内核 7.0.0-44），4 vCPU（运行时共享额度 1.5 核）
- GPU：**无**（nvidia-smi 不存在）
- 内存：总 7.7Gi，勘察时可用仅约 0.3–0.7Gi，另有 1.0Gi swap——**内存非常紧**
- 磁盘：140G 可用，充足
- Python：系统自带 3.12.3；无 python3.10、无 conda（故先下了 Miniconda 安装包备用）

## 下一步需要的条件

1. **用户明确同意执行克隆仓库的代码**（装依赖＋跑 example）——这是当前硬卡点，同意后我可从第 1 步续跑。
2. 即使获准执行，还要预先说明两个独立风险：①无 GPU、纯 CPU 推理 0.5B 模型，单句生成可能很慢（分钟级以上，未实测）；②可用内存不足 1Gi，装依赖（torch 等大包）与加载模型都有 OOM 被杀的现实可能。若本机跑不动，备选是按原方案文档口径改走免费 GPU（Kaggle T4／魔搭）或百炼 API 出同稿样音——但那超出本次「本地部署实测」的范围，需另行派单。

## 约束遵守情况

- 只在 `~/workspace/tts-test/` 内操作，未动其他文件与管线（本报告按任务要求写在 learning/deep 下）。
- 未向任何平台上传任何样音（且并无样音产出）。
- 被拦截后未自行破例执行仓库代码。
