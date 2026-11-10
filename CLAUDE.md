# CLAUDE.md（个人 fork：xSCx/AnyPS5）

回复一律用中文。

这个文件只存在于个人 fork，用来在不同电脑之间延续上下文，不属于上游项目。上游写给 agent 的规则如下，必须遵守：

@AGENTS.md

## 这个 fork 在做什么

- 上游 boykopovar/AnyPS5 是重链接工具，不是模拟器：
  - 把 PS5 可执行文件重链接成 Linux 或 Windows 原生程序。
  - 自己实现 PS5 系统库（`core/libs/prx`）。
  - GPU 层（AGC，`libSceAgc` / `libSceAgcDriver`）把 GCN/RDNA 命令流和着色器转成 Vulkan / SPIR-V。
- 2026-11-10 在 Mac（M3）上完成了 fork 和一次只读分析；2026-11-17 同步上游主线至 `3dceb9c2dd`（+8,018 commits），**还没改过任何项目代码**。
- 接下来在 Windows 机器上继续，显卡 RTX 5070 Ti，和 #7895 的测试机同一款。在那里测基线，并尝试修复。

## Git 工作流（必须遵守）

- **remote**
  - `origin` = `git@github.com:xSCx/AnyPS5.git`，个人 fork，推送走这里。
  - `upstream` = `git@github.com:boykopovar/AnyPS5.git`，只拉不推。
  - 防误推：`git remote set-url --push upstream DISABLED_no_push_to_upstream`。
- **提交身份**
  - 仓库级 `user.name` / `user.email` 用个人身份，和本文件所在提交的作者一致（`git log -1 --format='%an <%ae>' -- CLAUDE.md`）。
  - 不要用全局 git 身份提交，也不要改全局配置。
- **fork 的 main**
  - 内容 = 上游 main，加上 CLAUDE.md 和 `notes/` 这一个笔记提交。
  - 同步：`git fetch upstream && git rebase upstream/main`，再 `git push --force-with-lease origin main`。
- **给上游提 PR 的分支一律从 `upstream/main` 拉出**，不要从 fork 的 main 拉。否则 CLAUDE.md 和 `notes/` 会混进 PR。上游 CONTRIBUTING 规定：调查笔记、报告、日志写在 PR 里，不进仓库。
- **这类分支上没有 CLAUDE.md，靠 CLAUDE.local.md 补上**
  - 首次设置：`git show origin/main:CLAUDE.md > CLAUDE.local.md`，并把 `CLAUDE.local.md` 写进 `.git/info/exclude`。这样切到任何分支，Claude Code 都能读到这些上下文。
  - CLAUDE.md 更新后重新复制一次。
  - 在 main 上两份会同时加载，内容重复，不影响。
- **AGENTS.md 里的 `--base origin/main` 在本 fork 要换成 `--base upstream/main`**：`python3 tools/check_conventions.py --base upstream/main`。开工前查重叠：`python3 tools/pr_overlap.py --names <function>`（注意在 `blob:none` 部分克隆上遍历全量 open PR 较慢，可先用 `gh search prs` 筛查）；开 PR 前跑 `python3 tools/pr_overlap.py --branch <branch>`。
- **技术债修改必须走碎片文件**：不再直接改 `docs/dev/debt/<library>.md`，必须在 `docs/dev/debt/changes/` 下新建碎片文件（格式见 `docs/dev/TechnicalDebt.md#fragments`）。
- **推送和开 PR 前先问用户**
- **上游的 PR 门槛**（CONTRIBUTING.md / AGENTS.md）
  - 开工前先在对应 issue 认领，并确认没有 open PR 在做同样的事。
  - 每位作者的 open PR 数量按已合并数量分档：0 个已合并时只能开 1 个，至少 1 个时 20 个，满 20 个后不限。超限的 PR 会被自动关闭。
  - PR 的全量 Linux / Windows 构建，要有写权限的协作者手动触发。
  - 声称提速的 PR，必须附同一台机器、同一个场景下 main 和改动各一份 `tools/trace_digest.py` 摘要，以及 trace 文件。
  - AGC、着色器重编译器、VideoOut 的改动，必须在同一块 GPU 上对比 main 和改动的验证层日志（命令在 AGENTS.md）：必须带 `VK_LOADER_DEBUG=layer` 并确认日志含 `Insert instance layer "VK_LAYER_KHRONOS_validation"`，且验证层版本 `>= 1.4.335`（对应 `3rdparty/Vulkan-Headers` `VK_HEADER_VERSION` 335）。

## Windows 首次设置

1. 克隆：`git clone git@github.com:xSCx/AnyPS5.git`，需要这台机器的 SSH key 已加到 xSCx 账号；或者 `gh auth login` 之后 `gh repo clone xSCx/AnyPS5`。然后在仓库根目录执行 `git submodule update --init --recursive`（现共 38 个子模块，11月中旬新增了 `libwpe`、`icu`、`pixman`、`cairo`、`libpng`），全量构建需要它。
2. 按上面的 Git 工作流配置 upstream、禁推、仓库级身份和 CLAUDE.local.md。
3. 构建要点，完整说明在 `docs/dev/BUILD.md`：
   - **编译器：** 要 x86-64 MinGW-w64 GCC（POSIX 线程）。可用 WinLibs GCC 15.2.0（`x86_64-ucrt-posix-seh`，release `15.2.0posix-14.0.0-ucrt-r7`），或 MSYS2 UCRT64 GCC。Scoop 的独立 GCC 15.2.0 构建不了运行时。
   - **杀软：** 会隔离 relinker 测试生成的 exe，要把构建目录下的 `core/relinker/test-temp` 加进排除。#16960 提到 `relinker.exe` 本身也会被 Defender 隔离。
4. 还需要：
   - Vulkan SDK（验证层须 `>= 1.4.335`），跑验证层要用。
   - 自己 dump 的 PS5 游戏，项目不提供。手上有哪些游戏，待向用户确认。

## 项目现状快照（2026-11-17，`upstream/main` @ `3dceb9c2dd`）

- **规模：** 仓库 2026-08-03 创建，最新 release **v0.1.5（11-11）**。5.46 万 star（54,626），已合并 PR **13,799**（较 11-10 +3,285），open PR **1,644**（较 11-10 净减约 341）；issue 共 1,601，open 291。
- **系统库完成度 41.73%**（16,476 / 39,484，较 11-10 +5.82%）：libc 99.5%，libkernel 94.8%，libSceAgc 98.8%，libSceAgcDriver 58.1%，libSceLibcInternal 48.0%，libSceAudioOut 35.6%，libSceVideoOut 26.2%。着色器指令形式 1460/1460 全部解码与翻译，未支持 throw 从 148 降至 **124**。
- **2D 和轻量独立游戏能流畅玩（30–60 FPS）**：Dreaming Sarah 60 FPS（#361）；Hades II 主城 30 FPS，加本地多线程补丁约 48 FPS（#5485）。
- **中等负载 3D / CRPG 约 20–35 FPS**：Divinity: Original Sin 2 ~23 FPS、FF Resonance 20–27 FPS、LEGO Builder's Journey ~20 FPS（#17533）。
- **重度 3A 仍为个位数帧率（1–10 FPS），但稳定性持续推进**

| 游戏 | 现状（截至 2026-11-17） | 来源 |
|---|---|---|
| GTA V | 主线 Fidelity 约 1 FPS，切 **Performance 模式开放世界 6.1–7.5 FPS**（RTX 4070 Ti）；旧 POC 约 21 FPS 未合并 | #6754 / #17533 |
| ASTRO BOT 第一关 | **#18020 已关闭（11-14）**：`#18356` + `#18388` 限制了 staging shadow 显存，不再掉到 0.7 FPS；RTX 5070 Ti 稳在 3.4–3.5 FPS（叠 open `#19100` 可跑满 600 s） | #18020 |
| ASTRO BOT Crash Site | 约 10 FPS（RTX 5070 Ti + 7800X3D，和本机同卡） | #7895 |
| Lies of P | 调低 `settleFloorBytes` 临时绕过存储图像报错后可无崩溃打过首个 Boss 进主城 | #22009 |
| GT7 | 买二手车后崩溃（仍 open） | #16650 |
| 对马岛 | 黑屏 / 0.2 FPS（仍 open） | #5166 |

## 分析结论（Fable 5.2，基于 70aab1bf93；附 11-17 进展）

**总判断：** 3A 慢，主要因为每个 draw 都要在 CPU 上、在一把全局锁里证明它读的游戏内存没变，且频繁打断 render pass 往返拷贝。把准备分给多个线程最多换来 1.3–2 倍，到不了 30/60 FPS；真正跑满 30/60 FPS 要换用“GPU 直读描述符 + 按脏页同步镜像”模型（`#17533` / `rewrite/direct` 原型方向）。显存和内存类问题可以逐项修好，修好后是不再崩溃、不再换页，不是变快。

- 11-10 完整报告在 `notes/fable-analysis-2026-11-10.md`（行号以 `70aab1bf93` 为准）。
- 人周和收益是推断。性能数字是 issue 作者的实测，没有复现过。

| 问题 | 能否解决 | 难度 | 工作量 | 预期收益 | 11-17 上游状态与切入点 |
|---|---|---|---|---|---|
| 1. 单线程 draw committer | 部分能 | 高 | 并行准备 8–12 人周 | 1.3–2 倍；Crash Site 约 10 → 16 FPS，之后被 GPU 每帧 62 ms 卡住 | 默认流水线深度已升至 32，已合入 `#10678`（提前建 key）、`#17738`（复用附件）、`#9875`、`#17786`；`#18173`（异步编码）与 `#9508`（GPU 堆）仍 open。draw `ShaderResources` 两段式构建仍未做 |
| 2. 着色器精确模拟 / N 卡 f64 / 巨型管线 | 部分能；f64 插值主线已基本解决 | 中到高 | 10–16 人周 | f16 规则加守卫上限约 +22%（Palworld）；巨型管线首编降幅无数据 | 新合入 `#15719`（启动 f16 NaN 探针）和 `#17733`（f32 NaN 修复加守卫）。`#4042`（packed f16 守卫）与 compute 两档编译仍未有人做 |
| 3. 显存 / 内存压力 | 能，逐项修 | 中 | 6–10 人周 | 8 GB 显卡、16 GB 内存机器从崩溃换页变成能跑 | `#18020` 已由 `#18356`/`#18388` 修复关闭；**堆镜像去主机副本已有 open PR `#15296`/`#15295`/`#15297` 占位** |
| 4. macOS | 部分能 | 高 | 进 CI 4–6 人周 | 2D 和轻量 3D 30–50 FPS；重度 3D 不会好于 Windows | M4 叠剩余 2 个 open PR（`#17425`、`#11049`）已跑通全量构建；`#6528` 有 open PR `#9356`，`#15742` 有 open PR `#20580`；尚未进全量 CI |
| 5. open PR 积压 | 不是技术问题 | — | — | 决定上面每项落地速度 | open PR 从 ~1,985 降至 1,644；拆成一天内能评审完的小 PR |

- 问题 1 不考虑 secondary command buffer：录制本身只占 committer 时间的 20–25%，项目里也完全没有相关代码。
- 问题 2 的项目原则：只接受和硬件结果逐位一致的优化，降精度只能做成 opt-in 开关（先例 `APS5_FAST_F32`）。

## 关键代码位置（`3dceb9c2dd`，2026-11-17）

`Execution/`、`Graphics/` 都在 `core/libs/prx/libSceAgcDriver/` 下。

- **draw 管线线程模型**（queue 0 包处理 `Execution/src/Driver/Packets/PacketExecution.cpp:187` → 可选 finisher → committer，默认深度已升至 **32**）：`Execution/src/Driver/Draw/DrawPipeline.cpp:208`（`DrawFinisher::run`）、`267-276`（`Depth`）、`461`（`DrawPipeline::run`）。
- **全局 GPU 锁：**
  - committer 在 `Execution/src/Driver/Draw/Draw.cpp:1519-1522`（`Driver::commitDraw`）拿 `GuestMemory::GpuMutex()`，整个 `Graphics::Draw` 都在锁内。
  - 锁的定义在 `Execution/include/GuestMemory.hpp:213、239`。
  - 非 proof 线程每次最外层加锁都会调 `BreakProofs()` 作废证明：`Execution/src/GuestMemory.cpp:3265-3270`（定义在 `2366`）。
- **`Graphics::Draw`**（`Graphics/src/Draw.cpp:4813`）：
  - 已拆成 `prepareDraw`（`3809`）和 `recordPreparedDraw`（`4341`），在 `4859` 行仍串行执行。
  - 热点：`prepareDrawInputs`（`1794`）、`refreshResidentTarget`（`2109`）、`resolveDrawResources`（`2205`）。
- **Recorder 的共享状态**（唯一的 `open` batch、上传缓冲、快照表）：`Graphics/include/Recorder.hpp:1058、1070、1079-1090`。
- **ResourceCache 约束**：`Insert` 和 `Revalidate` 必须在锁内，见 `Graphics/include/ShaderResources.hpp:845-875`（`Revalidate` 声明在 `374`）。
- **dispatch 两段式先例**（`buildPrepare` 无锁 / `buildComplete` 有锁）：`Graphics/src/ShaderResources.cpp:2032-2069`（`buildPrepare` 在 `2425`、`buildComplete` 在 `2857`）。draw 构造函数在 `2012-2018` 行，仍直接调 `build()`（`2079`）。
- **f32 快速插值路径：** `core/shader/recompiler/SpirvBackend/src/SpirvAlu/SpirvAluEmitterFloat64.cpp:1400`（`FastInterpolateFunction`）和 `1413`（`CallInterpolate`）。
- **`APS5_FAST_F32`：** `core/shader/recompiler/Translation/src/DispatchInstructions.cpp:107`。
- **管线创建：**
  - compute 管线同步创建：`Execution/src/VulkanDevice.cpp:6180-6243`（预编译在 `2950`）。
  - 图形管线已有快速链接加后台重链：`Graphics/src/PipelineLibrary.cpp:339`。
- **内存：**
  - 堆镜像预算固定 24576 MiB，可用 `APS5_HEAP_MIRROR_MIB` 覆盖：`Graphics/src/GuestBufferMemory.cpp:3129-3136`。
  - staging shadow 活跃上限（`#18356`，默认堆预算 1/16，下限 256 MiB，`APS5_STAGING_SHADOW_MAX_MIB`）：同文件 `6269-6295`。
  - 每个 slab 配一份主机副本：`Graphics/src/ResidentMirror.cpp:127`。
  - 存储图像缓存键含宽高：`Graphics/src/ShaderResources.cpp:1453`（`SurfaceKey`），预算在 `745`，LRU 驱逐在 `1438`、`1595-1602`。
- **mesh 路径硬依赖 `VK_EXT_mesh_shader`：** `Graphics/src/Draw.cpp:1723`（open PR `#9356` 在做 compute 回退）。

## 测量工具

- 环境变量：`APS5_PROFILE_DRAW`（现默认带出 `[lock]` 各线程等待与持锁位点归因，`APS5_NO_LOCK_PROFILE` / `APS5_NO_HOLD_PROFILE` 可关）、`APS5_SAMPLE_COMMITTER`、`APS5_PROFILE_GPU`、`APS5_TRACE_SECONDS`、`APS5_TRACE_SUMMARY_EVERY=10`、`APS5_TRACE_MEMORY_BUDGET`、`APS5_AB=<seconds>:<RuntimeSwitch>`。
- 汇总与分析：`tools/trace_digest.py`、`tools/ab_split.py`、`tools/memory_trace_address.py`。
- 说明文档：`docs/dev/TRACING.md`、`docs/dev/ENVIRONMENT.md`。
- 离线重放着色器与 compute 管线（不用跑游戏）：`agc_shader_replay`、`agc_compute_pipeline_replay`。

## 下一步候选（2026-11-17 更新撞车状态）

- ~~**原 A（已撞车，跳过）：问题 3 堆镜像去主机副本 / 显存约束。**~~ 上游已有 `@Prophet069` 的 open PR `#15296`、`#15295`、`#15297` 覆盖，不要重复开 PR。
- **C（推荐优先）：在 Windows + RTX 5070 Ti 先测基线。** 用最新 main（已含 `#18356` 修复和 `[lock]` 持锁分析）跑手头游戏（如 ASTRO BOT）拿到基准 trace；若长时运行遇到主机堆分配失败可对照 open PR `#19100`。
- **D（新推荐小切口）：问题 2 的 compute 两档编译测量。** 在 `VulkanDevice.cpp:6180` `createComputePipeline` 试验 `VK_PIPELINE_CREATE_DISABLE_OPTIMIZATION_BIT` 先快速建管线、后台编优化版替换，可用 `agc_compute_pipeline_replay` 在 RTX 5070 Ti 上离线测编译耗时，目前无 open PR 撞车、改动集中在一处。
- **B：问题 1 的第一步。** `ShaderResources.cpp:2012` draw 资源构建拆成 `buildPrepare` / `buildComplete` 两段（仍在同线程跑，行为不变），目前无同名 open PR，但 `Draw.cpp` / `ShaderResources.cpp` 每日合入密集，动手前先在 `#7895` 认领并紧跟 rebase。
