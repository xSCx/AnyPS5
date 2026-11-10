# AnyPS5 五个关键问题的代码级可行性分析

基线：本地仓库，main @ 70aab1bf93（2026-11-10），行号以该提交为准。只读分析，没有构建也没有运行；性能数字全部来自 issue 里别人的实测，我没有复现。

标注约定：**【代码】** 是我在这个提交里读到的；**【issue】** 是 issue 作者报告的测量；**【推断】** 是我根据前两者做的判断或估算。人周数字都属于【推断】，按"已经熟悉这套代码的人"估，外部贡献者要再加 2 到 4 周上手时间。

## 汇总表

| 问题 | 能否解决 | 难度 | 粗略工作量 | 预期收益 | 第一个可独立合并的 PR |
|---|---|---|---|---|---|
| 1. 重度 3D 的 CPU 瓶颈（单线程 committer） | **部分能**：并行准备可行；30/60 fps 短期不能 | 并行准备：高；换掉逐 draw 模型：很高 | 并行准备 8–12 人周；换模型以人月计（6 个月以上） | CPU 受限场景约 1.3–2 倍。Astro Bot Crash Site 推算 10 → 约 16 fps，之后被 GPU 每帧 62 ms 限制 | draw 的资源构建改成"先无锁段、后有锁段"两段，仍在同一线程跑，行为不变 |
| 2. 着色器精确模拟 / N 卡 f64 / 巨型管线 | **部分能**：f64 插值在 main 上已基本解决；体积和编译时间能逐项降，不能根除 | 中到高 | 单项 2–4 人周；整个系列 10–16 人周 | f16 特殊值规则加守卫，上限是 Palworld 场景 +22% 帧率；巨型管线首次编译能降多少没有依据 | compute 管线"先关优化快速编译、后台再编优化版"的试验，附三家驱动的编译时间 |
| 3. 显存 / 内存压力 | **能**，逐项修 | 中 | 每项 1–3 人周，合计 6–10 人周 | 16 GB 机器和 8 GB 显卡从换页、崩溃变成能跑。Elden Ring 最多省 2.5 GiB，Wolverine 存储图像常驻约 5.9 → 1.3 GiB（均为 issue 作者估算） | 只读堆镜像不再保留 16 MiB 主机副本 |
| 4. macOS | **部分能**：轻量游戏已可玩；重度 3D 短期不能 | 高 | main 上直接可构建并进 CI：4–6 人周；mesh 回退：4–6 人周 | 2D 和轻 3D 30–50 fps（已有实测）；重度 3D 不会好于 Windows 同档机器的个位数帧率 | 合完剩下的 macOS 构建 PR，把 macOS 全量构建加进 CI |
| 5. 治理（约 2000 个 open PR） | 不是技术问题 | — | — | 决定上面每一项的落地速度 | 动手前先在对应 issue 认领，拆成一天内能评审完的小 PR |

## 总的判断

10 月 19 日有人在 RTX 5070 Ti + 7800X3D 上跑 Astro Bot 的 Crash Site：每帧约 3000 个 draw，约 10 fps。GPU 每帧只忙 62 ms，而负责把 draw 录进 Vulkan 命令缓冲的那个线程（代码里叫 draw committer）每帧 102 ms 里忙了 88 ms，其中真正录命令的只占四分之一。

**这个项目慢的主要原因是：每个 draw 都要在 CPU 上、在一把全局锁里，证明它读的 guest 数据没有变。这部分工作可以分到多个线程上，能换来一到两倍，换不来 30/60 fps。** 其余几项里，内存压力能逐项修掉，着色器体积能逐项降，macOS 能做到轻量游戏可玩；重度 3D 在哪个平台上都还差一个数量级，要等逐 draw 的模型本身被换掉。

---

## 问题 1：单线程 committer 能不能并行化？

**能并行的是"准备"，不是"录制"。准备并行化可行，主线已经做了一半的准备；按 render pass 用 secondary command buffer 这条路现在不值得做。**

### 根因在代码里的位置

一个 draw 从 PM4 包到 Vulkan 命令最多经过三个线程，最后一段只有一个线程，并且全程持有全局 GPU 锁。【代码】

| 环节 | 位置 | 说明 |
|---|---|---|
| queue 0 的包处理线程 | `core/libs/prx/libSceAgcDriver/Execution/src/Driver/Packets/PacketExecution.cpp:183-185` | 只有 queue 0 启用 draw pipeline；解码、抓取描述符、查 draw cache、物化着色器阶段都在这个线程 |
| 可选的 finisher 线程 | `core/libs/prx/libSceAgcDriver/Execution/src/Driver/Draw/DrawPipeline.cpp:181-233` | 一个线程，深度 16，只有设了 `APS5_DRAW_PIPELINE` 才启用；间接 draw 不走它（`.../Execution/src/Driver/Draw/Draw.cpp:739` 的 `handOff` 条件含 `!drawParameters.indirect`） |
| committer 线程 | 同文件 `DrawPipeline::run`，432-485；默认深度 8（262-271） | `DrawPipeline::Queue0()` 是单例，只有一个 `std::thread`。队列满时包处理线程在 `enqueue` 里等（354-364） |
| committer 对每个 draw 做的事 | `core/libs/prx/libSceAgcDriver/Execution/src/Driver/Draw/Draw.cpp:1381-1402` `Driver::commitDraw` | 1384 行拿 `GuestMemory::GpuMutex()`，整个 `Graphics::Draw` 都在锁内 |
| `Graphics::Draw` | `core/libs/prx/libSceAgcDriver/Graphics/src/Draw.cpp:4154` | 已经拆成 `prepareDraw`（3262）和 `recordPreparedDraw`（3701），但两段都在 committer 上、锁内串行跑 |
| 耗时最大的三块 | 同文件 `resolveDrawResources`（2114，其中 2166 行调 `ShaderResources::Revalidate`）、`prepareDrawInputs`（1723，顶点和索引拷贝）、`refreshResidentTarget`（2019，渲染目标刷新） | 对应 #7895 的 lookup 2.0 s、vertex 0.82 s、readTarget 0.66 s（每 10 s），record 只有 1.47 s【issue】 |

并行化要面对的共享状态和锁：【代码】

| 共享状态 | 位置 | 现在靠什么保护 |
|---|---|---|
| 全局 GPU 锁 | `core/libs/prx/libSceAgcDriver/Execution/include/GuestMemory.hpp:187` `GpuMutexType`，递归锁 | draw、dispatch、present、写回全部串行。"已证明未变"的记录默认不跨 draw 复用，只有设了 `APS5_DRAW_PIPELINE` 才在一个 epoch 内复用；即使开了，任何非 committer 线程每次最外层加锁都会把它们全部作废（`.../Execution/src/GuestMemory.cpp:3010-3015` 调 `BreakProofs()`） |
| Recorder | `core/libs/prx/libSceAgcDriver/Graphics/include/Recorder.hpp:39`（注释：每次调用都在 GPU 锁内）、927（唯一的 open batch）、938 和 959（draw 输入快照） | 只靠 GPU 锁。全进程一个 active recorder，一个 `VkCommandPool`（`.../Graphics/include/Context.hpp:157`） |
| 资源缓存 `ResourceCache` | `core/libs/prx/libSceAgcDriver/Graphics/include/ShaderResources.hpp:819`，自带 mutex（879） | 查找可以无锁；`Insert` 和 `Revalidate` 要求在 GPU 锁内（804-812 的注释） |
| 存储图像缓存、采样纹理缓存、待写回登记表 | `core/libs/prx/libSceAgcDriver/Graphics/src/ShaderResources.cpp:701`（预算）、1275（驱逐）；`.../Graphics/src/Texture.cpp:5664` `PendingSerial` | 各自有 mutex，但刷新、上传、flush 都经过 recorder，所以实际要 GPU 锁 |
| 写监视与证明序号 | `core/libs/prx/libSceAgcDriver/Execution/src/GuestMemory.cpp:2177-2187`：`ProofSerial` 是全局原子，`CollectEpoch` 是线程局部 | 序号是全局的，任何一次写回或 CPU 写都让所有证明失效 |
| draw cache | `core/libs/prx/libSceAgcDriver/Execution/src/Driver/Draw/Draw.cpp:409` 等处的 `drawCacheMutex` | 独立 mutex，已能在锁外用 |

### 主线已有的进展

- **prepare / record 已经拆开。** #7895 提议的第一步（把 `Graphics::Draw` 拆成准备和录制、不改行为）在 main 上已经存在。【代码】
- **dispatch 已经有"两段式构建"，draw 没用上。** `ShaderResources` 的构建分 `buildPrepare`（不需要 GPU 锁：绑定计划、buffer、描述符集）和 `buildComplete`（锁内：图像查找、上传）。compute 走延迟构造，中间还有一次无锁的预同步（`ShaderResources.cpp:1839-1876`，头文件 505-513 和 408-415 的注释）。draw 的构造函数（1819-1826）直接调 `build()`，两段都在锁内跑。【代码】
- **顺序和依赖信息已经在队列里。** pipeline 的每个条目带写范围和读范围（`core/libs/prx/libSceAgcDriver/Execution/include/Driver/Draw/DrawPipeline.hpp` 的 `Item`；范围由 `.../Execution/src/Driver/Memory/WrittenBuffers.cpp:157-198` 算出），label、CP 存储、DMA、end-of-pipe、提交结束都已经作为有序条目排队，并提供 `OverlapsAny`、`PendingLabel`、`StoredBytes` 这些查询。并行准备要用的"谁依赖谁"已经可查。【代码】
- **环境开关**：`APS5_PIPELINED_DRAWS`（深度，默认 8）、`APS5_DRAW_PIPELINE`（finisher 加上"一个 collect epoch 内复用证明"，默认关）、`APS5_PIPELINE_DISPATCH`（默认关）、`APS5_NO_PIPELINED_INDIRECT`、`APS5_HANDOFF_SPIN_US`、`APS5_VERIFY_DRAW_PIPELINE`、`APS5_SAMPLE_COMMITTER`（Windows 上采样 committer 的栈）。
- **技术债文档**：`docs/dev/debt/libSceAgcDriver.md:439` 只把 finisher 记为 opt-in，没有并行准备的条目。
- **没进 main 的**：#6754 POC 的"3 线程准备池 + ticket 有序提交"（main 里搜不到）；#9508 的 GPU 描述符堆（`APS5_GPU_HEAP`，174 条评论，仍 open，main 里搜不到）；#17533 作者说的"把加锁即作废证明收窄"的本地改动。

### 两条路的判断

**方案 A（推荐）：并行准备第 N+1 到 N+k 个 draw，单线程按序录制，录制前校验。** 依据有三条。第一，可挪走的工作占 committer 一半左右：Astro Bot 是 lookup + vertex + readTarget 合计 3.48 s 对 record 1.47 s；#8068 在 Yakuza 0 上采样 committer，`resolveDrawResources` 36.1%、`prepareDrawInputs` 17.3%、`recordDraw` 19.7%【issue】。第二，87.5% 的 draw 不读前 16 个 draw 写的任何范围【issue】，而这个判断用的读写范围主线已经在算。第三，dispatch 的两段式构建是现成的同类先例。

**方案 B（现在不做）：按 render pass 用 secondary command buffer 并行录制。** 录制只占 committer 的 20% 到 25%，就算录制完全免费也只省这么多。录制里除了 `vkCmd*` 调用，还有 pass 冲突检查、对象保活、写范围登记，这些必须按顺序更新 recorder（`core/libs/prx/libSceAgcDriver/Graphics/src/Recorder.cpp:1843` `StartDrawPass`）。pass 又很短：AoM 约 830 个 draw 分在约 300 个 pass 里【issue】。主线没有任何 secondary buffer 的基础设施（全库搜不到 `VK_COMMAND_BUFFER_LEVEL_SECONDARY` 和 `vkCmdExecuteCommands`）【代码】。等方案 A 做完、录制成为新瓶颈时再评估。

### 方案 A 的步骤

| 步骤 | 内容 | 要动的模块 |
|---|---|---|
| PR 1 | draw 的 `ShaderResources` 改成延迟构造：committer 上先 `buildPrepare` 再 `buildComplete`，同线程、行为不变；加一个对比校验开关 | `ShaderResources.cpp/.hpp`、图形层 `Draw.cpp` 的 `resolveDrawResources` |
| PR 2 | 缓存未命中的 draw，把无锁段挪到入队之前（包处理线程或 finisher）。用到 BDA 地址表的着色器整段留在锁内，和 dispatch 的 `lockedBuild` 一致 | 驱动层 `Draw.cpp` 的 `finishDraw`、`DrawPipeline` |
| PR 3 | 顶点和索引拷贝挪出锁：把 recorder 里的 draw 输入快照表换成线程安全的分段环形缓冲，提交时按写监视代数校验 | `Recorder`、`prepareDrawInputs`、`CopyDrawInput` |
| PR 4 | 缓存命中的 draw 预先做 `Revalidate`：准备线程记下当时的 epoch、证明序号、待写回序号，committer 录制前比较，变了走原路径。前置条件是把"任何线程加 GPU 锁就作废全部证明"收窄成按范围作废 | `GuestMemory.cpp` 的 `GpuMutexType::acquired`、`ShaderResources::Revalidate`、`Texture.cpp` |
| PR 5 | 2 到 3 个线程的准备池加 ticket 有序提交，把包处理线程上的解码、抓取、物化也并行化。每个 draw 要带一份寄存器状态快照 | 驱动层 `Draw.cpp`、`DrawCapture.cpp`、`DrawPipeline`、`PacketExecution.cpp` |

### 正确性风险

| 风险 | 具体情况 | 应对 |
|---|---|---|
| guest 内存写观测 | 准备提前读 guest 内存，CPU 可能在准备和录制之间写。现在入队前有一次 `capturedMemoryChanged` 复查（驱动层 `Draw.cpp:1333-1336`），变了就重抓 | 复查挪到 committer 录制前，或依赖 PR 4 的序号比较 |
| label 与 WAIT_REG_MEM 顺序 | POC 遇到过：label 任务越过了还在准备中的 draw（#6754 第 5 条）。主线的 WAIT 逻辑读排队中的 label 值（`PacketExecution.cpp:315-338`） | 所有非 draw 条目都排在准备 ticket 之后；读者等待写它范围的排队条目 |
| 读到前面 draw 写的东西 | 12.5% 的 draw 读前 16 个 draw 写的范围，其中 10.5% 依赖紧挨着的前一个【issue】 | 这些 draw 的准备等前一个录完，或直接回到串行路径 |
| DCC key | 目标刷新每次要扫 DCC key，可能要等已录制的 key 写入（`core/libs/prx/libSceAgcDriver/Graphics/src/DccMetadata.cpp:909` `WaitForKeyWriters`） | 留在有锁段 |
| 堆镜像和 BDA 地址表 | 地址表构建要 pin、镜像已注册内存，还会把 buffer 退给 recorder | 整段留在锁内 |
| 缓存驱逐 | 准备阶段拿到的图像在录制前可能被驱逐 | `shared_ptr` 持有，录制前复查是否仍在缓存；draw recipe 路径已有"目标没了、模板没了就回退"的处理可以沿用 |
| 测试覆盖 | CI 只有 lavapipe；这类改动要在同一台机器、同一块 GPU 上对比 main 和改动的验证层日志 | 每步都留开关和对比校验模式，像 `APS5_VERIFY_DRAW_PIPELINE` 那样 |

### 结论

- **能否解决**：部分能。并行准备能做；30/60 fps 做不到。
- **难度**：高。改的是全库最热的四个文件（图形层 `Draw.cpp` 4453 行、`ShaderResources.cpp` 7070 行、`Recorder.cpp` 4952 行、`Texture.cpp` 7484 行），每一步都要在真实游戏上验证画面。
- **工作量**：PR 1 到 PR 3 约 4–6 人周，做到 PR 5 约 8–12 人周。
- **收益**【推断】：把 lookup、vertex、readTarget 挪走并留一部分校验成本，Astro Bot 的 committer 从每帧 88 ms 降到约 55–60 ms，CPU 上限约 17 fps，随后被 GPU 的 62 ms 限制在约 16 fps。Yakuza 0（每帧 8700 个 draw、3.3 fps、GPU 73–90 ms）只做 committer 这一侧大约到 4.7 fps，因为包处理线程自己每帧要忙约 210 ms，会成为新的瓶颈；做完 PR 5 才到 6–7 fps。GTA POC 里准备池那一步是 12.0 → 15.3 fps【issue】。合起来是 1.3 到 2 倍。
- **到不了 30/60 fps 的原因**：#17533 在 AoM 上量到每个 draw 约 390 µs，60 fps 的预算是 8–20 µs；作者估算即使完美分到三个线程也只有约 15 fps，GPU 时间线还把它限制在 4 fps 附近【issue】。GTA V 在 main 上是 2–3 fps，其中一个光追 compute 每帧占 100 ms，关掉光追也只有约 4 fps【issue】。这些要靠"CPU 不再逐 draw 碰 guest 数据"的新模型（GPU 侧读描述符，#9508 和 #17533 的方向），属于以人月计的重写。
- **并行准备解决不了的一类**：CPU 准备 dispatch 时要读 GPU 刚写出的数据，只能等 GPU（GTA V 里 queue 0 三分之一的时间花在这上面，#5694【issue】）。

### 外部贡献者切入点

从 PR 1 下手，并且先在 #7895 认领。它不改行为，有 dispatch 的现成写法可以照着做，评审成本低。测量不需要改代码：`APS5_PROFILE_DRAW=1` 看 `[draws]` 各阶段，`APS5_SAMPLE_COMMITTER` 采 committer 的栈，`tools/trace_digest.py` 出对比。没有游戏数据的话，`tests/execution/` 下 `DrawPipeline*`、`Pipelined*`、`FinisherLabelOrder` 这组测试是顺序语义的现成样例。

---

## 问题 2：精确模拟的代价、N 卡 f64、巨型管线，能不能按设备降级或拆开编译？

**"按设备能力选一条结果逐位相同的路径"可行，也是这个项目默认唯一接受的形态；"降精度"只能做成 opt-in。N 卡 f64 插值的问题 main 上已经基本解决。巨型管线没有单一办法能解决，能做的是减体积和两档编译。**

### 根因在代码里的位置

| 现象 | 位置 | 现状【代码】 |
|---|---|---|
| f64 精确插值 | `core/shader/recompiler/SpirvBackend/src/SpirvAlu/SpirvAluEmitterFloat64.cpp:305` `Interpolate` | 用 f64 乘加再舍入一次。但调用入口 `CallInterpolate`（1390）先走 `FastInterpolateFunction`（1377）：`InterpolateFast`（1271）用 f32 的无误差变换算结果，指数范围守卫（1292-1295）成立就直接用，不成立才调不内联的精确函数。只有定向舍入的 f16 结果直接走精确函数 |
| v_fma_mixlo / mixhi | `core/shader/recompiler/Translation/src/FloatInstructions.cpp:283-285` | 发成 `FPInterpolateF16`，和上面是同一条路 |
| 普通 f32 FMA | `core/shader/recompiler/SpirvBackend/src/SpirvAlu/SpirvAluEmitterMath.cpp:2122` | 直接用主机的 `Fma`，不经过 f64 |
| packed f16 特殊值规则 | `FloatInstructions.cpp:13` `packedFloat16` | NaN 选择、默认 NaN、FP16_OVFL 规则对每条指令无条件展开，没有守卫 |
| f16 FMA | `FloatInstructions.cpp:261-281`；`core/shader/recompiler/Recompiler.cpp:278` | 设备有 `OpFmaKHR` 时走原生 packed f16 FMA（`nativeFloat16Fma`），否则在 f32 里做 round-to-odd |
| 设备能力探测 | `core/libs/prx/libSceAgcDriver/Execution/src/VulkanDevice.cpp:1616-1630`；`core/shader/recompiler/Recompiler.hpp:339-384` `SpirvTarget` | 启动时跑探针（f32 flush、f32 denorm preserve、f16 转换的 NaN），结果是 `float32FlushToZero`、`float32DenormPreserve`、`float16WidenNans` 等标志，全部进缓存键（`.../recompiler/CacheKey.hpp:381-387`） |
| 表图像访问体积 | `core/shader/recompiler/SpirvBackend/src/SpirvImageEmitter.cpp:2397-2470` `EmitImage` | 每次访问对每个运行时模式展开一个 `OpSwitch` 分支，分支体内联。#12089 量到 28 个 `ImageWrite` 发了 136 万 word【issue】 |
| compute 管线创建 | `VulkanDevice.cpp:6075-6138` `createComputePipeline`；标志在 2801-2803 | 同步创建，调用方等到完成；标志只有统计和 dispatch base，没用"关优化"标志 |
| 预编译 | `VulkanDevice.cpp:2877` `PrebuildComputePipeline`；`.../Execution/src/Driver/Dispatch/DispatchAhead.cpp` | 往前扫 dispatch，在线程池里抢先编译；管线 journal 在下次启动时重放。首次遇到的巨型管线仍然要等 |
| 图形管线两档 | `core/libs/prx/libSceAgcDriver/Graphics/src/PipelineLibrary.cpp:338` `OptimizedPipeline::Schedule` | 图形管线先快速链接、后台再做 LTO 重链（N 卡上不做 LTO）。compute 没有对应机制 |

### 主线已有的进展

- **#6754 的第 1 条发现在 main 上已经不是原样。** 当时 N 卡上 `v_fma_mix` 和 f16 插值整条走 f64；现在 f64 只在守卫不成立的 lane 上执行，结果仍然精确【代码】。#6754 的评论里有人在两块 N 卡上验证过一条"带守卫的 f32 路径"与 f64 路径 0 差异，并说它会单独进上游；main 上这条路径和它形态一致，是不是同一份代码我没有核对。
- **原生 packed f16 FMA**（#11183 第 2 步）已在 main。
- **已有一个 opt-in 的非精确模式**：`APS5_FAST_F32`（`core/shader/recompiler/Translation/src/DispatchInstructions.cpp:110-116`）关掉 f32 的软件 denormal flush 和 RDNA NaN 载荷，磁盘缓存分开存（`.../recompiler/ShaderDiskCache.cpp:1920`），有单独注册的测试。f16 没有对应开关。
- **还没做的**：f16 特殊值规则的守卫（#4042，前一次尝试 #5057 因为模块反而大了 17% 被关）；f32 NaN 和 denormal 规则的值域分析（#11183 第 5 步，占 GTA 语料 16.9% 的 word，全库搜不到相关代码）；wave64 每次调用两个 lane 的布局占 34%，更省的布局对很多程序被拒绝【issue】。
- **项目原则**写在 `docs/dev/TechnicalDebt.md`："每个函数要么完全正确，要么抛异常"。#11183 给这个系列定的规则是：能力、规范保证或探针只能在两条结果相同的路径之间做选择，不能放宽语义。

### 可行方案

**精度这一侧，按收益排序：**

1. f16 特殊值规则加守卫，在 SPIR-V 后端发射：快路径加一次"输入或结果是否 NaN、是否溢出"的判断，成立才进原来的规则代码。结果逐位相同。硬约束是模块体积不能变大。改 `FloatInstructions.cpp` 的 `packedFloat16` 和后端发射器。2–3 人周。
2. f16 的 opt-in 快速模式，照 `APS5_FAST_F32` 的做法：原生 f16x2 运算、主机 NaN 载荷。POC 在一个 GTA compute 上量到 104 万 → 32.4 万 word、8.25 → 1.2 ms【issue】。不精确，只能默认关。1–2 人周。
3. f32 的值域分析：证明某个结果不可能是 denormal 或 NaN 时省掉修复代码。占比最大，工作量也最大。4–6 人周。

**巨型管线这一侧：**

1. 减体积：表图像访问改成"每个模式一个 SPIR-V 函数，各访问点调用"。对 RADV 无效，Mesa 会把函数全部内联，#13538 的 250 万 word 模块就是内联后把编译器内存耗尽的【issue】；对 NVIDIA 和 AMD 闭源驱动的效果没有数据。2–3 人周。
2. compute 两档编译：先带 `VK_PIPELINE_CREATE_DISABLE_OPTIMIZATION_BIT` 建一份马上用，后台建优化版再替换。同一份 SPIR-V，不改语义，和图形管线的"快速链接 + 后台重链"同构。各驱动能省多少没有数据【推断】，所以第一个 PR 应该是带测量的试验。2–3 人周。
3. 减少同一程序的第二份特化（#14160：11 个程序因为描述符字段不同各编两次，冷启动多花约 300 s【issue】）。
4. "把巨型管线拆成几个小管线"不可行：一个 compute 程序对应一次 dispatch，状态是共享的。"异步编译、编完之前先跳过"违背精确原则，也不可行。能做的只有把首次编译挪到加载阶段，journal 重放已经让第二次运行受益。

### 对正确性测试的影响

精确路径的逐位测试不用动。守卫方案的慢路径就是原代码，需要补一个"强制所有 lane 走慢路径"的测试和一个统计守卫命中次数的计数。opt-in 模式要单独注册测试，先例是 `agc_driver_f32_arithmetic_nan_fast`。任何新探针的结果都要进缓存键和序列化请求。CI 只有 lavapipe，N 卡和 A 卡的结果要贡献者自己跑并写进 PR。

### 结论

- **能否解决**：部分能。N 卡 f64 已基本解决；f16 的体积和 GPU 时间能降；巨型管线首次编译的几十秒没有根治办法。
- **难度**：中（守卫、opt-in、两档编译）到高（值域分析）。
- **工作量**：单项 2–4 人周，整个系列 10–16 人周。
- **收益**：f16 守卫的上限是 Palworld 标题画面 +22% 帧率，那是 GPU 受限场景下"完全去掉规则"的测量【issue】，实际会低一些。原生 f16 FMA 那一步在 GTA 语料上减了 7.7% 的 word，一个程序的 N 卡管线创建从 5445 ms 降到 3743 ms【issue】。Mafia 的 37 s、Ratchet 在 RADV 上编不出来，能改善多少没有依据。
- **风险**：守卫让模块变大（#5057 的教训）；不同驱动对同一段 SPIR-V 的行为不同，`docs/dev/debt/shader.md:239` 记着两种写法各让一组 N 卡驱动挂死。

### 外部贡献者切入点

`agc_shader_replay` 和 `agc_compute_pipeline_replay` 可以离线重放保存下来的着色器请求和管线，不需要跑游戏。手上有 N 卡或 A 卡的人，最划算的是做 compute 两档编译的测量：改动集中在 `createComputePipeline` 一处，产出是一张"三家驱动、关优化前后的编译时间"表，不管结论正反都有价值。

---

## 问题 3：显存和内存压力能不能缓解？

**能，而且每一项都能单独修。压力来自几套各管各的预算，再加上同一份 guest 数据在主机和设备上被保存多份。**

### 根因在代码里的位置

| 机制 | 位置 | 现状【代码】 |
|---|---|---|
| 存储图像缓存 | `core/libs/prx/libSceAgcDriver/Graphics/src/ShaderResources.cpp`：`SurfaceKey` 1290、`storageBudget` 701、LRU 驱逐 1427-1432、`evictStorage` 1275 | 缓存键含宽和高，动态分辨率每一档是一张独立图像。预算每秒读一次 `VK_EXT_memory_budget`，下一次插入时才裁剪 |
| 堆镜像的主机副本 | `core/libs/prx/libSceAgcDriver/Graphics/src/ResidentMirror.cpp:126` | 每个 16 MiB 的设备内存 slab 配一个同样大的主机数组，只读镜像也配 |
| 堆镜像预算 | `core/libs/prx/libSceAgcDriver/Graphics/src/GuestBufferMemory.cpp:2773-2780` `heapMirrorBudget` | 固定 24576 MiB，不随物理内存和显存变 |
| host import 预算 | 同文件 `HostImportBudget` 4427-4435 | 取"6 GiB 和物理内存一半的较大者"，再受主机堆大小限制；64 MiB 窗口，LRU 退役 |
| 设备本地 staging 副本 | 同文件 `deviceLocalHasRoom` 5812-5830、`stagingBuffer` 5832，三个调用点 6150、6584、6659 | 只在新分配时检查"离预算是否还剩八分之一"。5807-5809 的注释写明每个副本是独立的 `VkDeviceMemory`，由缓存的构建持有 |
| 资源缓存 | `core/libs/prx/libSceAgcDriver/Graphics/include/ShaderResources.hpp:819-896` | 只按条目数（1024 起，最多 16384）和表元素数限制，没有字节预算 |
| 分配失败后的恢复 | `docs/dev/debt/libSceAgcDriver.md:155、208` | 先回收空闲 buffer，再放缓存；仍失败就退到主机内存类型，而且不会再搬回设备内存 |

各 issue 和代码的对应：

- **#15086（Elden Ring，16 GiB 机器提交 22 GiB）**：161 个 slab 乘 16 MiB 等于 2.5 GiB 的主机副本，就是 `ResidentMirror.cpp:126` 那一行。【代码】【issue】
- **#6099、#5961（Wolverine，8 GB 卡）**：约 260 个基地址上有约 1460 个存储图像，同一地址最多 40 种分辨率；主机侧图像合计 5.9 GiB，对应的 guest 内存只有 1.2 GiB【issue】。原因是缓存键带宽高【代码】。
- **#18020（Astro Bot 从 3 fps 掉到 0.7 fps）**：`vkAllocateMemory buffer` 这一类从 100 MiB 涨到 4068 MiB【issue】。这个类别名就是 `Buffer` 构造函数分配时用的标签（`core/libs/prx/libSceAgcDriver/Graphics/src/Resources.cpp:68`），staging 副本也走它（58）。我的判断是：涨的是被资源缓存条目持有的 staging 副本，而资源缓存没有字节预算【推断，issue 作者也没确认】。关掉四个 staging 开关后不再上涨，与这个判断一致。
- **#7047（780M 核显）**：主机堆只有 3 GiB，堆镜像和 import 都从这里分配，设备本地堆反而空着一半【issue】。

### 主线已有的进展

预算已经在读 `VK_EXT_memory_budget`，分配失败有恢复路径，`APS5_TRACE_MEMORY_BUDGET` 能按类别打印占用。可调开关：`APS5_TEXTURE_CACHE_MIB`、`APS5_STORAGE_CACHE_ENTRIES`、`APS5_HEAP_MIRROR_MIB`、`APS5_HOST_IMPORT_MIB`、`APS5_IMAGE_POOL_MIB`、`APS5_STAGING_POOL_MIB`、`APS5_UNIT_SHADOW_MIB`，以及四个关闭 staging 的开关（`APS5_NO_WRITTEN_SHADOW`、`APS5_NO_ATOMIC_STAGING`、`APS5_NO_DRAW_STAGING`、`APS5_NO_BDA_STAGING`）。技术债文档如实写着这些阈值（八分之一、一半、2 GiB 下限）各自只在一两个游戏、一两块显卡上验证过，还写着"连续多个 GPU 使用者共用一份设备本地副本"没有实现（debt 文档 374 行）。

### 可改进点

| 改进 | 做法 | 工作量 | 风险 |
|---|---|---|---|
| 只读堆镜像不留主机副本 | 刷新时把 guest 字节直接拷进 staging，只有可写镜像保留主机数组 | 1–2 人周 | 低。要先统计可写镜像占多少，才知道实际能省多少 |
| 堆镜像预算随机器缩放 | 默认值从固定 24 GiB 改成按物理内存和设备堆预算算 | 1 人周以内 | 低，但默认值改动要有游戏数据证明不回退 |
| staging 副本不再被缓存条目无上限持有 | 资源缓存加字节预算，或者副本改从共享池取、跨连续使用者保留 | 2–3 人周 | 中。动资源缓存会影响 draw 命中率 |
| 同一地址多种分辨率共用一块设备内存 | 每个基地址一块 `VkDeviceMemory`，各分辨率的图像绑定到它；按块记账 | 3–4 人周 | 高。切换别名前要把上一个的待写回结果落到 guest 内存，同一批次里两个别名之间要加 barrier。更便宜的替代是按大小复用旧分配，省分配不省上传 |
| 被驱逐的图像尽快释放 | 查清驱逐后仍持有它的 1 到 3 个所有者（候选是资源缓存条目和图像表），驱逐时一并移除 | 1–2 人周 | 中 |
| 核显把堆镜像放设备本地 | 主机堆小的设备上，镜像放设备本地，刷新走 staging 拷贝 | 2–3 人周 | 中。issue 作者本地做过，去掉了分配失败，但那台机器随后撞上了物理内存上限 |

### 结论

- **能否解决**：能，逐项修。
- **难度**：中。
- **工作量**：合计 6–10 人周，每项可以独立合并。
- **收益**：Elden Ring 最多省 2.5 GiB，但作者自己指出省完仍要约 20 GiB，16 GiB 机器还得继续削另外两块【issue】。Wolverine 的存储图像常驻从约 5.9 GiB 降到约 1.3 GiB 是作者对"共享内存"方案的估算【issue】。Astro Bot 那次，关掉 staging 后帧率稳在 3.3 fps，说明修好之后是"不再掉到 0.7"，不是变快。
- **风险**：这些阈值目前靠少数游戏的数据定，改默认值很容易在别的游戏上回退；项目要求性能类 PR 附同机同场景的对比。

### 外部贡献者切入点

只读堆镜像去掉主机副本最合适：改动集中在 `ResidentMirror.cpp` 和 `GuestBufferMemory.cpp` 里使用主机块的几处，有现成测试（`agc_driver_resident_heap_mirror`），收益在任何用到堆镜像的游戏上都能用 `APS5_TRACE_MEMORY_BUDGET` 直接读出来，不依赖高端显卡。

---

## 问题 4：macOS 能做到什么程度？

**轻量游戏在 Apple Silicon 上已经能玩；重度 3D 短期做不到，而且 macOS 上的上限会一直低于同档 Windows 机器，因为整个运行时连同游戏代码都在 Rosetta 2 下翻译执行。**

### 现状

| 事实 | 位置【代码】 |
|---|---|
| 全量构建在 macOS 上强制 x86-64。系统库、AGC 驱动、着色器重编译器和游戏代码在同一个进程里，全部经 Rosetta 2 执行 | `CMakeLists.txt:2-4、734-735` |
| relinker 能输出 Mach-O；`--to-rosetta` 把 Rosetta 不支持的指令改写掉 | `core/relinker/elfpatcher/src/macos/MacOsImage.cpp`；`core/relinker/codegen/src/x86/` 下各个 `*Lowering.cpp` |
| relinker 的代码生成只有 x86 指令改写，没有 ARM64 后端 | `core/relinker/codegen/src/` |
| libkernel 有成套的 macOS 实现：guest 跑在第二线程、主线程留给 AppKit；主栈搬出 guest 地址空间；guest TLS；写监视和 label 读陷阱 | `docs/dev/debt/libkernel.md:174、259、260`；`docs/dev/debt/libSceAgcDriver.md:218` |
| 驱动里已有一批 MoltenVK 和 KosmicKrisp 的特判 | `VulkanDevice.cpp:1616-1626`；debt 文档 9、92、145、200、201、214、349 行 |
| mesh 路径硬依赖 `VK_EXT_mesh_shader`；#6528 的 compute 回退不在 main | `core/libs/prx/libSceAgcDriver/Graphics/src/Draw.cpp:1659`；`.../Graphics/src/ShaderValidation.cpp:680` |
| CI 在 macOS 上只跑 relinker 的构建和测试，全量构建不在 CI 里 | `CONTRIBUTING.md:35`；`.github/workflows/build.yml` |

实测【issue，#1 的评论】：

| 游戏 | 机器 | 结果 |
|---|---|---|
| Planet of Lana | M4 Pro | 进游戏，28.9 fps |
| God of War Sons of Sparta | M4 Pro | 到 Cyclops 战，40.6 fps（冷缓存） |
| Balatro | M2 Air | 菜单约 47 fps，GPU 每帧约 21 ms |
| SuperTuxKart | M4 | 约 6 fps，每帧约 106 次小提交 |
| 全量 ctest | M4 | 2818 个里 2773 个通过 |

### 剩余阻碍

1. **构建还没完全进 main。** 10 月 27 日的清点是 main 还要叠 8 个 open PR 才能构建和启动【issue】。其中一部分现在已在 main（`GuestMemory.cpp:58-60` 对 `link.h` 的保护、`VulkanDevice.cpp:1426` 加载 `libvulkan.1.dylib`）【代码】。今天的 main 能否直接构建，我没有验证。
2. **macOS 特有的死锁。** #15742：import 路径在持有写监视锁和 import 锁时去解读陷阱，两把锁都不可重入。修法要改大部分 import 入口（先找陷阱、放锁、解陷阱、重试）。Stray 约十次出现一次【issue】。
3. **MoltenVK 缺的特性。** mesh shader 缺失影响最大：PS5 的 NGG 几何路径在主机上靠 mesh shader 跑，多数 3D 游戏会走到。其余还有 sparse 绑定（堆镜像退化成整份拷贝）、`nullDescriptor`（用占位图，写入会被保留）、`depthBounds`（部分情况用深度测试模拟）、设备地址上的原子操作（并发更新会丢）、PS5 的固定地址 0xfe0000000 被 macOS 占用（一部分着色器因此无法链接）。这些都在技术债文档里逐条记着。
4. **只能靠 Rosetta。** 没有 x86 到 ARM 的翻译后端，新增一个相当于另起一个项目。Rosetta 自身的差异已经遇到几处：带掩码的存储在非首元素上出错会直接结束进程（`docs/dev/debt/relinker.md:51`），支持哪些指令随 macOS 版本变（同文件 59-60）。另外按我的背景知识，Apple 已宣布 Rosetta 2 作为通用工具只提供到 macOS 27，之后只保留面向老游戏的子集。这不是仓库里的内容，是否覆盖这种用法**待确认**。
5. **性能。** 问题 1 的 CPU 瓶颈在 macOS 上更重，因为 committer 和重编译器本身也是翻译执行的；MoltenVK 上每次提交的开销也更高，SuperTuxKart 那组数据说明了这点。

### 可行方案

- **第一步**：把还开着的 macOS 构建 PR 合完，再把 macOS 全量构建和一组不依赖显示设备的测试加进 CI。没有 CI，Linux 和 Windows 侧每天几十个合入会不断把 macOS 弄坏。4–6 人周，多数是协调和 rebase。
- **第二步**：实现 #6528 的 mesh 回退（compute 把顶点和图元写进 buffer，再做一次带索引的 draw）。它同时让没有 mesh shader 的 Windows 显卡受益（RDNA1 等），不是 macOS 专用。4–6 人周。
- **第三步**：按 #15742 的思路重排 import 入口的加锁顺序。2–3 人周，风险在于会碰到大部分 import 代码。

### 结论

- **能否解决**：部分能。"2D 和轻 3D 可玩"已经达到，做完第一步和第二步后能覆盖更多中等负载的 3D 游戏。
- **难度**：高，主要高在平台差异多、没有 CI。
- **现实上限**【推断】：Apple Silicon 上 2D 和轻量 3D 在 30–60 fps；重度 3D 在 RTX 5070 Ti 级别的 Windows 机器上是 3–10 fps，macOS 上不会比这更好。原生 ARM64 不在可预见范围内。
- **风险**：Rosetta 的长期可用性；MoltenVK 行为随版本变化（技术债文档里的若干条目绑定在 1.4.2 这个版本上）。

### 外部贡献者切入点

有 Apple Silicon 机器的人：在 #1 里认领一个失败的测试并修掉，那边已经形成"先在评论里宣布、再开 PR"的做法；或者在 `--to-rosetta` 里补一条还没改写的指令，只需要 relinker-only 构建，这部分 macOS 有 CI。没有 Mac 的人做 mesh 回退，在一块没有 mesh shader 的 Windows 显卡上就能开发和验证。

---

## 问题 5：治理状况对落地有什么影响？

**对这几项工作影响最大的不是评审慢，而是"大改动必须拆成很多个有先后依赖的小 PR，每个都要在一两天内合进去，否则就冲突"。**

事实：

- 每位作者的 open PR 数量有上限：没有已合并 PR 时 1 个，有 1 个后 20 个，20 个后不限。PR 的 Linux 和 Windows 全量构建不会自动跑，要有写权限的人手动触发，一次最多合并 40 个 PR 一起测。评审队列每小时取最老的 30 个。（`CONTRIBUTING.md:37、60、65`）【代码】
- 性能类 PR 必须附 main 和改动在同一台机器、同一场景的摘要行和 trace；AGC、重编译器、VideoOut 的改动必须附同一块 GPU 上的验证层对比；CI 只有 lavapipe。（同文件 73、74 行）【代码】
- #12521 的统计：1362 个 open PR 里 274 个冲突；PR 第一天冲突概率 3%，两天 26%，十天以上 49%；合并耗时中位数 24 小时【issue】。它提议的"每个 AGC 测试一个 CMake 片段"已经合入，`tests/cmake/` 下现在有 339 个片段【代码】。技术债文档仍然是冲突最多的地方：`libSceAgcDriver.md` 有 322 KB，每个条目是一个长行。

对各问题的影响：

| 问题 | 影响 |
|---|---|
| 1. 并行准备 | 影响最大。要在最热的四个文件里连续合入 5 步以上、彼此有依赖的改动，每一步都会和几十个 open PR 重叠。#7895 本身就是在点名问负责这块的两个人"要不要、以什么形式接收"。#9508 开了三周、174 条评论还没合，说明架构级的大 PR 很难整体进去，只能拆 |
| 2. 着色器 | 中等。重编译器的改动相对局部，但每个 PR 都要 N 卡和 A 卡的硬件数据，没硬件的人只能做一半 |
| 3. 内存 | 较小。每项都能独立合并，适合外部贡献者，但默认阈值的改动需要多个游戏的数据 |
| 4. macOS | 中等。没有全量 CI，macOS 的修复要靠贡献者在评论里互相协调合并顺序 |

对个人贡献者的含义：第一个 PR 要小，合了才有 20 个名额；动手前用 `tools/pr_overlap.py` 查重叠并在 issue 里认领；性能改动先准备好能复现的场景和 trace，否则会被打上 `perf numbers missing` 标签，进不了评审队列。

---

## 方法和局限

- 读过的资料：`docs/dev/ARCHITECTURE.md`、`TechnicalDebt.md`、`TRACING.md`、`ENVIRONMENT.md`、`CONTRIBUTING.md`、`AGENTS.md`，`docs/dev/debt/` 下 AGC 驱动、着色器、relinker、libkernel 四个文件的相关条目，离线整理的 issue 正文和评论，以及上面列出的源码位置。
- 没有做的：没有构建和运行，行为判断来自读代码和代码里的说明。`gh` 没有登录，open PR 的情况只来自离线数据（确认了 #9508 仍 open），可能漏掉最近几天新开的相关 PR。
- 性能预期都是按 issue 里的阶段占比推算的，没有实测支撑；并行准备引入的校验成本我按"被挪走工作量的 15% 到 20%"估，这个比例没有依据，实际收益可能更低。
