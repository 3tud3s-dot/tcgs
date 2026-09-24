# Mac 轻量工作区迁移记录

日期：2026-09-23（Asia/Shanghai）。本轮范围仅为 workspace migration、sync policy 与中文 AGENTS.md。

## 工作区

- Mac：`/Users/qdtrr/projects/tcgs`，文件系统规范路径为 `/Users/qdtrr/Projects/tcgs`。已核验两者 inode 相同，未创建重复目录。
- Server：`/data1/userdata/tcweng/projects/tcgs`，通过已有 SSH alias `amax` 访问。
- 根目录原为空目录，未初始化 Git；三个子仓库独立管理。

## 迁移前服务器审计

实际执行了 `du -sh *`、`find . -maxdepth 2 -type d | sort`，并单独检查隐藏备份。大小为 du 舍入后的磁盘占用。

| 目录 | 服务器大小 | Git 状态 | 处理 |
|---|---:|---|---|
| PhysTwin | 32G | main / clean；ahead origin/main 2 | 同步 Git 源码，内部数据与实验产物不复制 |
| SoMA | 343M | main / clean | 同步 Git 源码，构建与运行产物不复制 |
| deform360 | 11M | main / clean | 同步 Git 源码 |
| environment | 1.1G | 非 Git | 81 个已列明文本文件/脚本/小日志 |
| datasets | 515M | 非 Git | server-only，包含 soma_sample |
| outputs | 87M | 非 Git | server-only，包含 soma_sample |
| .PhysTwin.rsync-backup-20260921 | 111M | main / dirty，780 条状态记录 | server-only；未修改、未覆盖 |

隐藏备份 origin 为 `https://github.com/Jianghanxiao/PhysTwin.git`，HEAD 为 `81c718790a37e5e0102eb77af2c6edd34a9db25f`。它不是本次迁移来源。

浅层目录中，PhysTwin 另有 data、downloads、experiments、experiments_optimization、experiments_refit、gaussian_output；SoMA 另有 soma.egg-info；environment/soma-20260923 中有 src、wheels 与构建产物。这些未通过目录镜像传输。

未发现原有 workspace-level AGENTS.md 或顶层 docs。源码仓库中已有 docs 随 Git 恢复；本目录为本次新建的迁移记录。

## 计划与执行

执行前预算约 160–200 MiB：三个仓库的已跟踪文件分别约 54.09、19.81、5.73 MiB，加 Git 对象、约 1.31 MiB 轻量记录和少量文档。最大的已跟踪文件是 9,975,332 字节的官方展示 GIF；没有已跟踪的 PLY、HDF5、checkpoint、wheel 或编译后的 CUDA 二进制。

源码仓库均以官方 remote clone 重建。直接 GitHub 包下载缓慢，因此 SoMA 用 Git 经 SSH 从服务器取得临时 bare 对象缓存，再通过官方 remote 的 `clone --reference ... --dissociate` 完成。PhysTwin 的服务器历史本身为 shallow，不能直接作为 reference；因此先从官方 remote 做无 checkout、blob:none 的浅 clone，再从服务器 Git 缓存完整 fetch 所需对象，checkout 服务器同一 HEAD。验证所有可达对象无缺失后取消了临时 partial-clone 配置。两份临时 bare 缓存已删除，正式仓库没有 alternates 依赖。

PhysTwin 服务器比官方 origin/main 多两个提交，已通过 Git 保留。未新建开发分支、未 commit、未 push，未 rsync `.git/`。服务器已有的 `codex-sync/*` 辅助分支不属于本次要求的活动 main 同步范围，未在 Mac 额外创建。

非 Git 文件使用逐文件清单 `environment-allowlist.txt`。筛选排除了 src、wheels、build、缓存、二进制和符号链接；只允许已检查为 UTF-8 的文本/脚本，每个不超过 1 MiB。实际最大文件为 598,421 字节的构建日志。

先执行并检查：

```bash
rsync -rtiv --dry-run \
  --files-from=docs/migration-20260923/environment-allowlist.txt \
  --max-size=1048576 -e 'ssh -o BatchMode=yes' \
  amax:/data1/userdata/tcweng/projects/tcgs/ /Users/qdtrr/Projects/tcgs/
```

确认只有清单内 81 个文件（另有 8 个目录项）后，以相同参数去掉 `--dry-run` 执行复制。没有使用 `--delete`。共 1,368,488 字节，全部 SHA256 匹配。AGENTS.md 也单独先 dry-run 再复制到服务器原本不存在的目标文件。

## 最终 Git 核验

| 仓库 | 两端 origin（fetch/push） | 分支 | HEAD | 状态 | 已核验 tracked 文件 |
|---|---|---|---|---|---:|
| PhysTwin | https://github.com/Jianghanxiao/PhysTwin.git | main | `84d5adb5646097a7836faa3aeed01b3c33318d21` | clean / ahead 2 | 819 |
| SoMA | https://github.com/Wrioste/SoMA.git | main | `8e8772a98f5eeb332745d9b6dd008e6927f050af` | clean | 231 |
| deform360 | https://github.com/lhy0807/deform360.git | main | `d8522a4403b766aeb387510c04e89032a56fdf35` | clean | 84 |

SoMA 与 deform360 与用户历史 HEAD 一致；PhysTwin 原交接未给 HEAD，本次以服务器读数为准。两端 Git tree 一致；Mac 1,134 个 tracked 文件逐个计算 Git blob hash，与服务器 HEAD tree 匹配。`git fsck --full --no-dangling` 通过，没有缺失的可达对象。服务器最终重新审计的仓库和轻量文件快照与初始审计完全一致。

详细机器可读证据：`server-audit.json`、`verification.json`。JSON 中空 status 表示 clean。迁移报告、审计、核验和 environment allowlist 这四个文件也通过明确清单同步至服务器 `docs/migration-20260923/`，先 dry-run，再复制并校验。

## AGENTS.md

- Mac：`/Users/qdtrr/projects/tcgs/AGENTS.md`
- Server：`/data1/userdata/tcweng/projects/tcgs/AGENTS.md`
- 两端 SHA256：`a63de9cf24e14ee9ee936a261ce76361a2af16c96fd2c022f6278166b3f69304`

包含项目目标、研究阶段、双端分工、目录规范、Git 与同步规则、禁同步项、SoMA/Deform360 状态、v0 候选映射、tactile 后续、实验安全与 Codex 工作规则。研究交接中尚未现场验证的数据候选明确标为待核验。

## 占用、限制与风险

Mac 工作区约 159 MiB（含 Git 对象）：PhysTwin 约 107 MiB、SoMA 39 MiB、deform360 11 MiB、environment 1.5 MiB，剩余为文档及文件系统开销。大型临时 Git 缓存已清理。

未同步：datasets、outputs、checkpoint、raw/processed data、SoMA sample、PLY/HDF5、Linux 环境、构建缓存和 wheels，以及 dirty 历史备份。environment 内的 Linux/CUDA 激活和实验脚本仅作记录，不应直接在 Mac 执行。

PhysTwin 为浅历史仓库，且有两个未推到官方上游的本地提交；以后仅 clone 官方 remote 无法保证得到当前研究状态。所有 dirty working tree 均不得因同步而覆盖。本次没有建立自动双向同步，今后仍需先检查两端状态。

完整 outputs 日志未同步；rollout=9 allocator 重试的状态通过服务器 `outputs/soma_sample/logs/trial-a-25485.metrics.json` 只读核验，并写入 AGENTS.md 历史摘要。没有将其解释为新的运行授权。

下一步仅建议：在新的开发任务中先审阅 SoMA 数据接口与已保存记录，再创建 `deform360-adaptation` 分支并设计无 tactile 的最小 adapter。本次未下载 Deform360、未运行训练或测试 CUDA、未写 adapter、未修改模型与环境。
