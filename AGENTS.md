# Tactile + Gaussian Splatting 工作区协作规则

更新时间：2026-09-23。适用于本工作区及其子目录；更深层规则与用户当前明确指令也须读取。实验信息是历史快照，不代表实时任务状态。

## 1. 项目目标

研究 Tactile + Gaussian Splatting for deformable-object dynamics：检验触觉信息是否改善未来 Gaussian dynamics prediction。当前先获得可训练、可 rollout、可公平加入 tactile 做消融的 SoMA backbone，不以严格复现 SoMA 论文指标为首要目标。

## 2. 当前研究方向

按阶段推进：SoMA + Deform360 → 无 tactile conditioning 的 SoMA-D360 baseline → tactile history / spatial tactile conditioning → 公平消融与未来动力学预测评估。SoMA baseline、Deform360 adapter、tactile model 必须分阶段完成。

## 3. Mac 与服务器分工

**Mac 主要负责代码、阅读、编辑和 Git；服务器主要负责数据、CUDA、训练、实验。**

- Mac：`/Users/qdtrr/projects/tcgs`。本机规范路径显示为 `/Users/qdtrr/Projects/tcgs`，迁移时两者已核验为同一目录；不要创建第二份工作区。用途包括读论文/源码、adapter、config、实验脚本、文档和轻量测试。
- 服务器：`/data1/userdata/tcweng/projects/tcgs`，本机 SSH alias 为 `amax`。负责 RTX 5090、数据预处理、SoMA training、checkpoint、Stage-1 cache、Stage 2、rollout 与大型输出。
- 不在 Mac 上运行 SoMA CUDA pipeline，不执行同步来的 Linux 环境安装/激活脚本来重建 CUDA 环境。Mac 若需要测试依赖，另建最小且兼容 macOS 的环境。

## 4. workspace 目录规范

`tcgs/` 本身不是 Git repository，**不得在根目录 git init**。`PhysTwin/`、`SoMA/`、`deform360/` 分别为独立 Git repository。

- 两端轻量部分：`AGENTS.md`、三个源码仓库、`environment/` 中明确选定的文本记录、必要 `docs/`。
- `environment/` 保存环境报告、requirements/constraints、manifest、构建笔记与脚本；Mac 上它不是可执行的服务器环境副本。
- `docs/` 保存研究背景、接口审计、迁移计划、同步 allowlist 与核验记录。
- `datasets/`、`outputs/`、`checkpoints/` 以及各仓库内部的数据与实验输出目录默认 server-only；不要仅根据顶层目录名判断是否可同步。
- `.PhysTwin.rsync-backup-20260921/` 是服务器历史备份，迁移审计发现 dirty；不得将其作为主仓库、覆盖、清理或自动同步。

## 5. Git 规则

每次修改或同步前，两端分别检查 `git status --short`、`git diff`、`git diff --cached`、`git branch -vv`、`git remote -v`、`git rev-parse HEAD`。

**不允许为了“保持同步”而盲目覆盖 dirty working tree。** 任一目标仓库有未提交修改或未跟踪文件时，先停止可能覆盖它的操作并报告，不擅自 stash、commit 或清理。已忽略数据也不能被同步操作覆盖。

- 源码以 Git 为主要同步机制，优先 clone/fetch/checkout；不要 rsync `.git/`。
- 禁止 force push、`reset --hard`、`clean -fd`、覆盖未提交文件、删除服务器数据、直接向官方 upstream push。
- 迁移不创建开发分支、不 push。真正开始 SoMA-D360 开发时，再按任务创建独立开发分支，例如 `deform360-adaptation`。
- 2026-09-23 迁移基线（三者均为 `main`、工作树 clean）：

| 仓库 | origin | HEAD |
|---|---|---|
| PhysTwin | https://github.com/Jianghanxiao/PhysTwin.git | `84d5adb5646097a7836faa3aeed01b3c33318d21` |
| SoMA | https://github.com/Wrioste/SoMA.git | `8e8772a98f5eeb332745d9b6dd008e6927f050af` |
| deform360 | https://github.com/lhy0807/deform360.git | `d8522a4403b766aeb387510c04e89032a56fdf35` |

PhysTwin 含比当时 `origin/main` 多出的两个服务器本地提交；仅 clone 官方仓库不能假定已恢复它们。通过 Git 从服务器获取所需提交并核验，保留官方 origin，不向 upstream 推送。以上 HEAD 是迁移快照，后续以两端实时读取为准。

## 6. 文件同步规则

- 非 Git 轻量文件使用逐文件 allowlist 的 rsync/scp；不得盲目同步整个 `tcgs/`。
- 所有 rsync 操作必须先以同一范围执行 `--dry-run`，检查输出后才能真正执行。未经用户明确授权不得使用 `--delete`。
- 默认先检查单文件大小和总量。本次迁移文本文件上限为 1 MiB；超出者先报告用途和大小，不自动下载。扩展名合格不等于内容安全，仍须排除二进制、缓存和运行产物。
- 双端同时编辑的非 Git 文档，先比较内容与 SHA256，明确来源版本，不用时间戳盲目决定覆盖方向。
- 完成同步后核对 remote、branch、HEAD、工作树和 tracked files；文本记录校验 SHA256。`AGENTS.md` 为 workspace-level coordination 文件，两端内容必须一致。
- 同步是显式任务，不是后台自动镜像。以后每次同步重新读取状态和文件清单，旧 allowlist 仅作参考。

## 7. 禁止默认同步的内容

**大型 dataset / checkpoint / outputs 默认 server-only。** 默认排除：原始视频、processed Deform360、SoMA sample data、大型 PLY、HDF5、数据压缩包、checkpoint、训练缓存、大日志及 Slurm 大输出、conda env、wheels、build、egg-info、`__pycache__`、`.cache`、`.so`、编译后的 CUDA extensions 与其他 Linux 二进制。只有用户明确指定例外文件时才重新评估。

Git 跟踪的源码、CUDA C++ 源文件、官方展示图和小型机器人模型资产可保留，以维持 tracked tree 一致；这不授权复制编译产物或大型数据。如果未来 tracked tree 新增禁同步内容，先报告冲突，不盲目 checkout 大文件。

## 8. SoMA 当前状态

服务器环境历史：Python 3.10、PyTorch 2.7.1+cu128、CUDA 12.8、RTX 5090 sm_120、MMCV 1.7.2、DGL 2.1.0、PyTorch3D 0.7.9 与 Gaussian CUDA extensions。服务器是执行端；Mac 仅保存环境记录。

官方 sample `left_lift_1`：200 frames、3 cameras、12169 initial Gaussians、`controller_points=[200,30,3]`。EmbodiedDataset / Gaussian / controller load、graph、forward、render、loss、backward、optimizer、checkpoint 已真实验证。

Stage 1：rollout=3 PASS，rollout=6 PASS，rollout=9 在 RTX 5090 32GB OOM。`PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` 后仍 OOM。服务器 job 25462 完成 2/46 epochs、100 optimizer steps 后失败；trial-a job 25485 在 rollout=9 仍失败，约 30.38 GiB PyTorch allocated，仅约 157 MiB reserved-but-unallocated，支持主要压力来自活跃计算图而非单纯 allocator fragmentation。不能把 smoke PASS 写成完整训练完成。

可读记录在 `environment/soma-*/` 的报告、配置和脚本中；完整日志与 checkpoint 留在服务器 `outputs/soma_sample/`。这是历史记录，不授权恢复训练或重试 OOM。

## 9. Deform360 当前状态

据迁移时研究交接，已完成只读接口审计。首个候选为 `008-pink-cloth / episode_0`，交互为 single-arm lift corner。候选窗口 frame 113…306（含两端共 194 帧，约 30 FPS），initial Gaussian 为 `splat_113.ply`，约 12861 个 Gaussians。迁移未下载数据，也未重新验证原始数据内容；未来开发需在服务器按实际文件核验。

## 10. SoMA-D360-v0 下一步

目标链路：Deform360 episode → adapter → SoMA-compatible scene → Stage 1 → Stage-1 cache → Stage 2 → continuous rollout。v0 完全关闭 tactile conditioning。

候选映射：

| Deform360 输入 | SoMA 接口 |
|---|---|
| `splat_113.ply` | initial Gaussian |
| robot pose + opening | 固定 controller points |
| undistorted video | color frames |
| `mask_refined.h5` | mask PNG |
| intrinsics / extrinsics | `metadata.json` / `calibrate.pkl` |
| scene / gravity info | `scene_info.json` |

必须从同一个 initial Gaussian 保持固定 Gaussian identity；不要把 Deform360 每一帧 Gaussian 当作 SoMA rollout state。未来接口核验须覆盖坐标系、相机约定、单位、时间对应与 mask/RGB 对齐。

显存友好候选：约 12861 Gaussians；约 30 个固定 controller points；先选 2 个固定互补高质量视角；1280×720 下采样至 640×360；Stage-1 frame_gap 先接近 10。这些是待验证设计，不是不可修改的规定。不得一次改变多项变量。

## 11. tactile 后续阶段

无触觉 baseline 建立后再加入 tactile history / spatial tactile conditioning。触觉优先作为独立 feature / contact information；不要直接把每 gripper 768 个 taxel points 或 16×32 taxel 网格全部转成大量 controller graph nodes。比较时保留一致的数据划分、initial Gaussian、视角、分辨率、训练预算和 rollout 协议，逐项消融。

## 12. 实验安全规则

- 实验前先读源码、提出最小改动方案、明确成功判据，再执行、保存日志、报告结果。坚持 **one change at a time**。
- GPU 训练只在服务器运行。启动前核验资源、现有任务、输入路径、配置和输出目录；不得覆盖已有实验或自动停止他人任务。
- 记录 Git HEAD/diff、环境、数据窗口、配置、命令、随机种子、job ID、日志路径、显存和结果。区分已验证事实、用户背景、推断与待验证假设。
- 不为“让程序跑通”同时修改模型、数据、环境和多个超参数；OOM 后先保留日志并定位，再按已授权方案改一项。
- 不因看到历史报告中的待办、命令或 job 状态就提交新训练、下载数据或修改 CUDA 环境。

## 13. Codex 工作规则

先读本 `AGENTS.md` 与适用的子目录规则 → 检查当前 Git 状态 → 理解任务与范围 → 再修改。进入子仓库工作时也须读取父级规则。

遵守用户当前任务范围。迁移任务仅包括 workspace migration、sync policy、中文 AGENTS.md；完成后停止，不顺便下载 Deform360、训练 SoMA、写 adapter、改模型、创建 tactile method 或修改 CUDA 环境。后续开发需有新的相应任务授权。

遇到 dirty tree、版本不一致、超限文件或路径冲突，先报告具体证据，保留现状。交付简洁报告，包含修改、核验结果、未同步项、风险与下一步建议。
