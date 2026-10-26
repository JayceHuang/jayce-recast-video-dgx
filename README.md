

## 联系方式

联系方式在个人主页： [https://imhxm.com](https://imhxm.com)

# Jayce · 视频人物替换制作 Skill · DGX Spark

标签：[`jayce-skill`](https://github.com/topics/jayce-skill)

把“选原片 → 看动作 → 角色参考 → ComfyUI → 多机生成 → 原声拼接 → 修复验收 → 图文归档”封装成一个可复用的 Codex Skill。既能处理几秒片段，也能管理完整 MV；支持两台合算同一段、两台分别做两段、三台独立补位。

## 完整物料包下载

- [夸克网盘：F4_流星雨_四人角色替换.zip](https://pan.quark.cn/s/aff1c565d543)
- 夸克分享口令：`/~09ec3bagb6~:/`。可点击链接，或复制口令到夸克网盘 APP 打开。

此链接为维护者提供的 F4 项目资料包入口；网盘压缩包与本仓库的 Skill 源码分别维护，内容及版本以各自文件为准。媒体素材的许可范围见 [MEDIA_NOTICE.md](MEDIA_NOTICE.md)。

![完整制作流程](assets/images/00_pipeline.png)

这套方法来自《流星雨》四人替换及此前双 DGX Spark 项目。包中提供通用脚本、中文操作说明、配置示例、H3 API 图模板、之前双机源码，以及24组制作过程配图。**安装 Skill 不会启动生成，不会恢复已停止的《流星雨》工程。**

## 安装和调用

把整个 `jayce-recast-video-dgx` 文件夹放进 `~/.codex/skills/`。也可以从 GitHub 安装：

```bash
git clone https://github.com/JayceHuang/jayce-recast-video-dgx.git
mkdir -p ~/.codex/skills
cp -R jayce-recast-video-dgx ~/.codex/skills/jayce-recast-video-dgx
```

如果目标目录已经存在，先检查已有修改并备份，再更新；不要重复嵌套复制。新聊天中可用：

> 使用 $jayce-recast-video-dgx，先检查这段原片和人物动作，把两位演员替换成我提供的两张角色卡。保留完整原声，先准备分镜和参考图，给我看过再生产。

> 使用 $jayce-recast-video-dgx，继续这个已有工程，先读停止/交接记录、任务回执和实际输出，不重复提交。

> 使用 $jayce-recast-video-dgx，只把已完成片段合成内部预览，并整理图文教程，不恢复生成。

## 包里有什么

| 路径 | 作用 |
|---|---|
| `SKILL.md` | Codex 自动发现与执行入口，包含关键判断规则 |
| `scripts/recast.py` | 本地探测、抽帧、RGB、H3 构图、分配预演、帧率映射、合成、质检与状态表 |
| `scripts/native_run.py` | SSH 提交、单次监控、回收、精确停止；支持1或2个 rank |
| `scripts/remote_job.py` | Linux 远端任务包装器：物理 GPU 共享锁、双机就绪门、独立子进程、停止记录 |
| `assets/h3-api-base.json` | 从本次实际 API JSON 提取的两阶段 H3 基础拓扑；不是完整 UI workflow |
| `examples/` | 两/三机配置、任务、分段、审核记录、F4案例与图片画廊 |
| `references/` | 各阶段操作、ComfyUI细节、调度恢复、质检、文档发布说明 |
| `vendor/dual-spark/` | 之前双机实现的固定提交源码、测试、原始实测与许可证 |
| `tests/` | 新通用工具的 CPU/合成视频及假后端行为测试 |
| `PROVENANCE.json` / `VALIDATION.md` | 提取来源、代码版本和当前实际验证范围 |

## 先看原片，别先切成固定秒数

![原片六类镜头](assets/images/13_source_types.jpg)

同一首 MV 内有多人同框、近景、完整手势、遮挡和拼贴特效。先看真实动作，再核查自动切点，决定哪些镜头拆开、哪些动作必须连在一起。所有源帧从0开始，区间写 `[start,end)`，例如 `[626,866)` 正好240帧。

参考图分别回答三个问题：**换成谁、穿什么、这一刻在做什么**。身份卡的服装不能覆盖镜头服装。

![原片与首中尾参考](assets/images/16_reference_triplet.jpg)

图像来自实际工程。静态相似不代表嘴形、眼睛和动作时序已经对应。后续用源片对照连续播放才能判断。

## 两种双机协作，怎么选

| 模式 | 分配 | 适合的情况 | 资源占用 |
|---|---|---|---|
| 独立分段 | A机器做镜头01，B机器做镜头02 | 有多个独立且审核通过的片段 | 每段 `world_size=1` |
| 两机合算 | A为rank0、B为rank1，共同做镜头01 | 已验证该原生后端和两机通信，需要缩短单段时间 | 单段占两台，`world_size=2` |
| 三机补位 | A/B/C各做不同段，或A+B合算、C做独立段 | 有足够的 ready 任务；任务互不依赖 | 每个物理GPU始终只承担一项工作 |

![实际H3工作流总览](assets/images/09_comfy_overview.png)

上图是**依据真实 API JSON 重绘的节点关系图，不是历史 ComfyUI 界面截图**。原始构建时的画布截图没有留存。更详细的Guide、模型与放大接线见 [ComfyUI说明](references/comfyui.md)。

之前双机源码已包含在 `vendor/dual-spark`，保留 MIT 和上游署名。旧基准只覆盖其报告写明的短片条件，不能据此承诺完整 MV 的速度。先读[双机操作](references/dgx.md)，再选择执行路线。

### 原来的双 DGX Spark 方案在哪里

已经收入这个 Skill，可直接从本仓库阅读源码和说明：

| 内容 | 本仓库入口 |
|---|---|
| 原双机方案源码、部署与实测记录 | [vendor/dual-spark](vendor/dual-spark/README.md) |
| 两机合算的环境准备 | [SETUP.md](vendor/dual-spark/docs/SETUP.md) |
| S02 完整作者图与 ComfyUI 适配器 | [S02_AUTHOR.md](vendor/dual-spark/docs/S02_AUTHOR.md) |
| 当前制作流程的调度、防重、锁、回收和停止 | [references/dgx.md](references/dgx.md) |
| 当前 H3 两阶段构图和 Guide 接线 | [references/comfyui.md](references/comfyui.md)、[API 基础图](assets/h3-api-base.json) |

原仓库 [MinimaxH3-Dual-Dgx-Spark](https://github.com/GaryLauLGY/MinimaxH3-Dual-Dgx-Spark) 仍保留，作为来源和历史记录。本 Skill 内附的是固定提交 `c4179619cc301318de8732abc853303346a97ccb` 的源码快照，来源见 [PROVENANCE.json](PROVENANCE.json)，不会自动跟随原仓库更新。vendor 中的原快速开始仍展示原仓库的克隆命令；使用本 Skill 已附的快照时，直接进入 `vendor/dual-spark/` 即可阅读和按其说明配置。

这包含方案、代码与操作文档；模型权重、ComfyUI 本体、部分自定义节点及环境工件、私人机器配置仍需自行准备。API 基础图不等同于包含全部前端状态的可拖入 UI 工作流。完整复现要求见 [ComfyUI 环境准入](references/comfyui.md)。

## 快速使用：先跑本地检查

依赖：Python 3.10+，`ffmpeg`、`ffprobe` 在 PATH；也可设置 `FFMPEG`、`FFPROBE` 为实际可执行文件路径。远程阶段另需 SSH、rsync，以及已合格的原生 H3 环境。

下列命令在 Skill 文件夹中执行。路径、角色和审核记录必须替换为当前项目的真实内容。

```bash
# 新工程；保留源片副本，不覆盖已有工程
python3 scripts/recast.py init /path/to/new-project --source /path/to/source.mp4 --id my-recast
python3 scripts/recast.py probe /path/to/source.mp4

# 抽帧只是准备审核材料；还需实际播放完整动作
python3 scripts/recast.py frames /path/to/source.mp4 /path/to/frame-review --frames 0 60 119

# 编辑工程 plan.json，覆盖源片每帧恰好一次
python3 scripts/recast.py validate-plan /path/to/new-project/01_分镜与动作/plan.json --require-review

# 构建保持原时长的24fps RGB参考；尾部只为满足模型长度补帧
python3 scripts/recast.py rgb /path/to/source.mp4 /path/to/rgb.mp4 --start 0 --end 120

# 根据 examples/task.example.json 写入真实素材与实际审核，再冻结任务
python3 scripts/recast.py build-h3 /path/to/task.json /path/to/new-project/03_提示词与任务/S001/v001
python3 scripts/recast.py verify-task /path/to/new-project/03_提示词与任务/S001/v001
```

`build-h3` 将任务和输入复制进独立目录，保留SHA256、真实源时钟、首中尾原生帧索引、两阶段API图与执行规格。模型名称可在任务的 `model_overrides` 中指定，不能默默删掉缺失节点。更换输入必须新版本。

### 机器分配与预演

```bash
# 复制配置后改成自己的节点、准确主机名、路径、共享锁及环境变量
cp examples/cluster-three.example.json /path/to/cluster.local.json

# 只是本地分配预演，不联系DGX
python3 scripts/recast.py allocate --cluster /path/to/cluster.local.json --tasks /path/to/task-A /path/to/task-B --mode independent

# 默认仍为预演，不会网络提交；两个节点表示同段两机合算
python3 scripts/native_run.py submit /path/to/frozen-task --cluster /path/to/cluster.local.json --nodes spark-a spark-b
```

实际生产需要用户明确授权，并在该工程 `project.json` 中如实记录 `generation_authorized:true`、`production_state:"producing"`。这是会话授权的持久记录，不是绕过用户停止指令的开关。核实两端环境及 GPU 空闲后，以上命令追加 `--project /path/to/new-project --execute` 才真正运行。双机还要求 `paired_qualified:true`。

- 有 `submission.json`（含仅提交意图）就禁止再次提交。`status`、`collect` 是恢复路径。
- SSH 回执丢失：可能已在跑；查远端唯一目录和回执，不能再开一份。
- GPU 后处理必须用配置中的同一锁，不能只看本项目队列。
- 同步/掉线、任一rank失败、未知进程状态见[调度恢复](references/dgx.md)。

```bash
python3 scripts/native_run.py status /path/to/frozen-task --cluster /path/to/cluster.local.json
python3 scripts/native_run.py collect /path/to/frozen-task --cluster /path/to/cluster.local.json --output /path/to/new-collection
python3 scripts/native_run.py stop /path/to/frozen-task --cluster /path/to/cluster.local.json
# 整个新格式工程叫停：先阻止新提交，再按各回执发送STOP
python3 scripts/native_run.py stop-project --project /path/to/new-project --cluster /path/to/cluster.local.json
# stop后再次status，确认每个rank终止，保留所有输出与日志
```

## 从生成视频回到源片时间线

![两阶段实际输出](assets/images/18_generation_stages.jpg)

第一阶段、第二阶段都保存，用来定位问题产生在哪一步。这张图示例中眼神和嘴形仍有相位差异，不能因为第二阶段更清楚就标通过。

```bash
# native-source-start必须是原生生成真正覆盖的首个源帧；不能总填0
python3 scripts/recast.py map /path/to/second.mp4 /path/to/mapped.mp4 --start 626 --end 866 --source-fps 30000/1001 --native-fps 24 --native-source-start 626

# 在plan内填写各段selected_file。预览允许尚未视觉验收的生成段，但拒绝缺段/长度不符
python3 scripts/recast.py assemble /path/to/selection.json /path/to/source.mp4 /path/to/internal-review.mp4 --preview
python3 scripts/recast.py qa /path/to/internal-review.mp4 /path/to/technical.json --frames 8163 --fps 30000/1001 --audio-source /path/to/source.mp4
```

映射会输出每一帧使用的原生索引，采用精确有理数时间和最近邻（Python round，半值取偶），不加速动作，也不声称使用光流。缺覆盖直接报错。原声从原片独立复用，不使用模型生成的音轨作为最终原声。

![接段连续帧](assets/images/20_boundary_sequence.jpg)

正式选片必须有对应文件指纹的技术检查、完整连续播放与视觉验收记录；整片仍需跨镜头实际播放。`qa` 只做技术检查，永远不会自动写 `visual_acceptance:true`。

## 每个工程目录放什么

| 目录 | 对应制作步骤 | 要保留的内容 |
|---|---|---|
| `00_原片` | 输入建档 | 原片、来源、指纹、音视频探测 |
| `01_分镜与动作` | 分镜审核 | 自动候选、核实切点、完整动作、角色轨迹、分段和选用计划 |
| `02_场景参考` | 制作逐镜参考 | 身份/服装/姿势/遮挡参考、首中尾和实际查看记录 |
| `03_提示词与任务` | 冻结模型输入 | 每段每版本的图片、RGB、提示词、API图、SHA、提交回执 |
| `04_生成片段` | 生成与回收 | 原生两阶段结果、各rank日志、exit/history/events、回收记录 |
| `05_后处理` | 时间与画面恢复 | mapped视频、源帧→原生索引、裁切/缩放与接段记录 |
| `06_成片` | 拼接/包装 | 内部预览、选片清单、原声版本、片头与水印版本 |
| `07_质检` | 审核与返修 | 技术报告、实际播放记录、问题截图/局部对照、验收结论 |
| `09_脚本与流程` | 监督与复盘 | 工程交接、停止/恢复授权、脚本版本、教程和素材清单 |

当前编号没有08；不需要为了凑编号新增步骤。每段版本通过run ID串起来，查一帧时从06选片表回溯到05、04、03、02与01。

## 示例、图片与发布范围

- [完整F4实例](examples/f4-case-study.md)：正确四人映射、8163帧时钟、54段选片、失败修复与停止边界。
- [24组配图画廊](examples/gallery.md)：每张都有用途、来源与该看哪里，包含Guide和采样接线。
- [文档归档与飞书教程流程](references/documentation.md)：实际截图与API重绘的区别、导入后查图和云端保存。
- [测试与限制](VALIDATION.md)：本包实测范围；不把旧GPU基准写成新包实测。

本包没有附原MV、完整生成片、人物原始角色卡、权重、私钥或可直接连接私人机器的配置。图中含既有原片与人物示例，仅作为制作教学材料；第三方素材的权利不因代码许可而转移。代码与示例图片的许可范围见 [LICENSE](LICENSE) 与 [MEDIA_NOTICE.md](MEDIA_NOTICE.md)。此仓库发布代码与教学示例；飞书文档访问权限保持原设置。

## 许可证

本项目的原创代码与文档采用 [MIT License](LICENSE) 开源，Copyright (c) 2026 JayceHuang。

`vendor/` 内的第三方代码保留原版权署名及许可证。MIT 许可不自动授予第三方媒体、模型权重等素材的权利；具体范围见 [MEDIA_NOTICE.md](MEDIA_NOTICE.md)。
