# K60 Adreno 730 GPU 修复与验收计划

> 当前计划：2026-11-12，mondrian / SM8475 / Adreno730v3 / iOS 16.6.1 (20G81)。
> 本轮仅调研和制定计划，不改实现、不构建、不上机、不写设备分区。
> 本节为当前执行入口；后面的旧 scanout 调研仅作历史参考，不再代表现状。

## 当前修复计划（2026-11-12）

### 结论与范围

先打通 **电源/时钟 → SMMU/GMU/HFI → SQE/ZAP/CP → 原生用户客户端 → Metal 渲染 → GPU 上屏 → 系统壁纸与稳定性**。
不是重写整套 GPU 驱动，也不是同时打开所有启动参数。已构建的 Turnip/Metal 桥可以复用，但构建成功不等于 GPU 执行成功。

壁纸不必然以 GPU 为唯一前提；当前日志只能证明没有观测到壁纸可用输出。`PBUIWallpaperImage` 存在、原生 setter completion 返回、壁纸层 `surface=0`，都不能排除 GPU、软件 renderer、资源解码或 scene 路径的问题。先前“问题肯定不在 GPU/合成器”的判断撤回。

### 已知事实与缺口

| 项 | 证据 | 当前判定 |
|---|---|---|
| 可恢复基线 | `E:/k60-ios/boot-cand24-20261112.img`；SHA256 `3ecba34b4f41496ba7e1bafc01f0a44d0bdd7eccdd4483cdc5bf5d69e8e9fba8`；`console-20261112-182641.json` 的 fastboot rc=0、COM8、84552 B | 软件显示及三键已有板上证据，不作为 GPU 证据 |
| 本机硬件和固件 | `work/device-evidence/be74ba07/bringup-research.json:24-29`；`firmware-manifest.json:8-36` | A730v3 `0x07030002`，本 K60 的 GMU/SQE/ZAP 文件已读取并记录来源 |
| 当前第一实测失败 | `console-20261112-013119.log:491-494`：secure RSC read `0x17a2000c` 返回 `0xfffffffffffffffe`，随后 `FAIL parent votes` | 还没有 GMU/HFI、ZAP、CP 或 Metal 执行证据；不重复运行已知失败路径 |
| 当前源码 | `PongAdrenoPlatform.cpp:412-427` 已换成 direct RSC mapping；同文件 `:2-5` 却仍说禁止 EL1 访问 | 访问安全证据冲突必须先解决；不能因 secure-IO 被拒绝就默认直读安全 |
| 内核实现 | `configs/mondrian_defconfig:52` 为 `y`；`patches/xnu/series:195-197` 已列 scanout/GPU-only/pitch | 旧文档“没链接/缺三个补丁”已过时；仍须核对实际候选的编译配置和链接结果 |
| 用户态构建 | `work/gpu-userspace/build-report.json:235,244,414`；`work/touch-gpu/gpu-userspace-integration.json:34-54` | Turnip、桥、bundle 已构建；publication=false、hardware=false，不重装已成功的工具链 |
| 同 SoC 参考 | 本轮成功 fetch `Chengdong-Yu/UniXNU-pong` 的 `ios16`，得到 `7798f1e5f261e7121d01d7e4807ed73512737ded` | 其 `drivers/PongAdreno/README.md:3-8,229-237` 同样未确认 GPU 渲染；可复用实现，不可借用成功结论 |

证据路径：日志在 `E:/k60-ios/work/cap/`；表中的 `work/...` 相对 `/root/k60`；驱动和配置路径相对此设备仓库。

### 阶段 0：固定基线、消除证据冲突

1. 保留 cand24 不覆盖；记录每个候选的 device/core revision、kernel/payload/boot SHA、实际 FAT 中 `unixnu.toml`、固件和用户态版本。
2. 对照现有 K60 `live-fdt.dtb`、command-db、固件清单和 Nothing SM8475 源码，核对 APSS DRV2/TCS3、CX/MX 资源、GPUCC、GMU、GPU SMMU、IRQ 和保留区。DT 地址声明不等于 XNU 访问授权；SID 数量不等于 bank 所有权。
3. “22-step 全 result=0”已追到 `work/up-device/porting.toml:147-167` 转述 `usblog-20261110-234429`，不是 Pong 原始流；该原始 capture/sidecar 未找到，不能绑定本机 be74ba07。历史挂死报告来自 [ios-sweet-k60 d11a3c0e](https://github.com/XTOM0706/ios-sweet-k60/commit/d11a3c0e515a440bc7ef5dcb4d7c40552e1658ed)，报告写首读 `0x17a00000`，0248 探针却首读 DRV2 `0x17a20000`，没有隔离的 map/read 原证。因此两者都不能证明当前直读安全；后续补齐设备/镜像/loader/冷热状态与访问地址来源，不复试挂死路径来补。
4. 核对 framebuffer、ZAP、ring、页表和 BO 池的物理范围及保留状态。framebuffer 位于 managed RAM 时必须证明已从 VM 分配器排除，不能只凭一条范围日志继续。
5. 核对最终启动参数长度：当前 ARM64 boot-args 上限 608 B（`board.toml:185-188`）；保留工作链所需参数，不截断，也不把所有 GPU 参数一次塞进去。

**出口：** 唯一基线和可追溯资源表齐全；RSC 访问选择有本板证据，危险访问不得用“再试一次”来确认。仓库 `docs/ops/HARD-RULES.md:24-46` 现行规则禁止 direct RSC，本计划不授权违反；先实现有契约的替代路径，或以新证据按项目流程正式解决该限制，之后才可选 MMIO。证据缺口先走现有可达的固件/源码/只读记录调查，不上机猜地址。

### 阶段 1：修电源投票与时钟

主要文件：`PongAdrenoPlatform.cpp`、`PongAdrenoPower.c`、`PongAdrenoClocks.c`。

- 在阶段 0 的结果上选择唯一实际后端：已授权的 MMIO，或有真实接口契约的固件代办。不能虚构可用 SCM 接口；若需要 UEFI handoff，必须实现真实投票及可验证的交付契约。
- 使用真实 command-db/AUX 及保留票值；不得降低其他所有者的请求，不超过固件许可等级，不借“最高 corner”掩盖资源匹配错误。
- 只租用确认空闲的 active TCS；保留 sleep/wake 与其他 driver 请求。完成判据必须来自本次事务的 ISSUED/COMPL、idle/进展，不接受旧状态或伪 ACK。
- 修正后候选只请求 GPU startup，Metal/scanout/GPU-only 保持关闭；遇到首个失败阶段就停止后续资格认定。

**出口：** `parent ARC votes acknowledged result=0`、`CX clocks result=0`，GPUCC/PLL 状态符合本板，USB/系统存活且无新异常。MMIO 总线访问本身可能硬挂，软件 polling timeout 不能防住，必须在访问前解决授权风险。

### 阶段 2：GMU、地址空间和 HFI

主要文件：`PongAdrenoPlatform.cpp`、`PongAdrenoSmmu.c`、`ArmSmmu4K/`、`PongAdrenoGmu.c`、`PongAdrenoHfi.c`。

- 保持真实 cold-master 检查：GX off、CM3 reset。若现场不满足，研究实际 handoff，不删除检查强行接管。
- 4 KiB CPU/DMA 页、真实物理页表、cache 属性与 clean/invalidate 一致；GMU 与 GPU 使用各自确认可租用的 context/ASID。
- 区分 GPU SMMU `0x03da0000` 与 UFS/USB 的 SMMU；USB 日志不能作为 GPU SMMU 健康证据。保留 secure SID 和其他活动 stream。
- 上传本板 GMU 固件，采用 SM8475 的队列布局及 PERF/BW/CORE_FW_START 顺序；要求新鲜、类型/序号匹配的真实 ACK。

**出口：** `GMU native4K context result=0`、`GMU firmware/HFI result=0`、`GMU power handshake result=0`，无 GPU SMMU fault。仅 GMU 成功仍不发布 GPU-ready。

### 阶段 3：SQE、ZAP、CP 与原生 BO 客户端

主要文件：`PongAdrenoCp.cpp`、`PongAdrenoZap.c`、`submit/`、`PongAdrenoNativeGlue.cpp`。

- 复用 gen7-4-0/2 MiB GMEM 配置；不用 A740 初始化表或 FD730 假 ID。SQE payload 和整页 backing 分开处理，固件来自本 K60。
- 保持当前板级启动顺序：CP_ME_INIT → RPTR drain/RBBM idle → 真正 PAS13 ZAP 认证 → nonsecure switch → CP 写内存与 timestamp fence（`PongAdrenoCp.cpp:352-403`）。bootstrap drain 不等于渲染 fence。
- 在合法原生 ABI 上单独验证 query、BO allocate/import/map、submit、wait、unmap/free；已完成的写入值和边界必须正确，释放后再次分配/映射不能读到旧页。
- 故障或 timeout 后隔离队列，DMA 未证实停止就保留 BO/页表，不在同一次启动里假装可恢复。

**出口：** `CP ready; authenticated nonsecure fence result=0` 和 `CP/user-client startup result=0`；原生用户态提交产生真实新 fence 与正确结果，正常 teardown 无残留活动映射。CP 工作不等于 shader 工作。

### 阶段 4：20G81 Metal / Turnip 真实渲染

主要文件：`drivers/PongMetal/`、`scripts/build-pong-userspace-linux.py`、`scripts/describe-pong-metal-abi.py`、`userland/init.c`、`userland/ui_compat.c`。

- 复用已构建的 Mesa `85fda3b3` 与 metal2vulkan `720ef1ce` 配套来源/补丁；不替换成最新 A740/iOS27 栈，不重复成功的依赖安装。
- 完成当前 guest 20G81 ABI 核对：真实 dyld cache、`_MTLDevice` extent/selector、IOKit/Metal 导入、IOAcceleratorES prelink 绑定，以及 `MTLPixelFormatGetInfo` 七 word 返回的调用约定和 QuartzCore policy。复用 `work/touch-gpu/wallpaper-native20G81.json:442` 已成功的 sealed fext/cache 读取方法，不重新假定整个 cache 不可读；SDK 可链接不能代替 guest ABI 资格。
- 保持 publication 默认 0；用阶段专用、受限诊断客户端验证真实 Vulkan/Metal device、pipeline、shader draw 和完成后的像素。采用简单纹理/三角形图案，并读回已知像素及未触碰区域，不能只做 clear/blit 或“device 非 nil”。
- 在 shader/IOSurface import 验证后再启用目标系统客户端。保留 CPReady/MetalEnabled/runtime ABI guard，不能手写 readiness 属性绕过。
- 完成安装/发现闭环：`init.c:461-480` 的现有 views 没有 Metal Extensions mount，`:516,543-547` 只向 backboardd 注入入口；必须让实际系统客户端可见完整 `PongAdrenoMetal.bundle`、sibling dylib、root-owned clients/publish。核对其他内核 staging 后补真实只读 Extensions 发现路径，保留 stock lower；不用不同名称的 NativeAdrenoMetal/offline broker 替代。`PongMetalPlugin.mm:174-181` 优先读取 Extensions，合法 marker=0 会直接拒绝，因此禁止留下遮蔽 Data=1 的旧 marker。
- 核对加载与初始化时序：`ui_compat.c:843-850` 会缓存 renderer=null；`init.c` 也有 Metal 入口注入。必要的改动是 readiness-aware 切换及正确进程启动时机，避免过早缓存永久软件模式；不先删掉所有软件兼容修复。

**出口：** 指定 20G81 客户端完成真实 shader pipeline、GPU command buffer、正确像素、IOSurface import；冷/热 pipeline 分开记录。bundle 编译成功、device 创建成功和 CP 内存写入均不单独算本阶段完成。

### 阶段 5：GPU scanout 上屏

主要文件：`PongAdrenoScanout.cpp`、`PongMetalScanout.h`、既有 framebuffer 三个 GPU 补丁。

- 阶段 4 通过后打开 scanout，先保留 CPU 的安全回退。复核实际物理 framebuffer、1440×3200、pitch、BGRA、旋转和寿命。
- 区分 offer、deliver、GPU submit、fence、complete 和真正屏幕变化；顺序必须一致。用上一阶段的已知输出作单层全屏资格画面。
- 当前 scanout 只支持单层全屏无缩放 BGRA（`PongAdreno/README.md:68-73`）；不把多层 CPU 回退当 GPU 全合成。
- 再用一次受控 `pong_gpu_only=1` 排除 CPU 绘制：画面持续真实更新且完成序号递增。试验后恢复普通安全策略；已交付但 DMA 未退休的帧必须隔离保留，不能 CPU 同时覆盖。

**出口：** `PONG SCANOUT: GPU frame completed ... status=0x0`、匹配的实际屏摄，以及 GPU-only 场景持续更新，无错误退休或条带。仅 `first frame presented` 和 FPS 不算 GPU 证据。

### 阶段 6：系统壁纸、合成和稳定性

- GPU/Metal 资格通过后验证 backboardd、SpringBoard 及实际壁纸渲染进程；按真实 scene 链确定客户端，不凭进程名字猜。
- 撤除或按 readiness 限定阻断加速的 renderer/poster 策略；保持与 GPU 无关的设备兼容修复。核对 HEIC 解码、纹理上传、AIR/pipeline、绘制、输出 surface 与合成提交。
- 若 GPU 已工作而壁纸仍黑，沿以上链定位第一失败点；若 CA 已把多层场景合成为单 surface，现有 scanout 可用；否则真实多层 GPU 合成是本计划的实现项，不把单层拷贝当最终交付。
- 验证锁屏和桌面壁纸实际像素、亮灭屏、切场景、客户端退出/重启、映射回收和冷/热启动。先固定 60 Hz；120 Hz、DVFS、preemption 和性能优化不混入首次修复。

**最终出口：** 连续 30 分钟包含上述操作，壁纸正确且有 shader/fence/上屏证据，三键、UFS/USB 无回归，无 GPU/SMMU fault、挂起或持续增长的未退休 BO。需要的 warm/reset/resume 路径以真实实现闭环，不能靠跳过功能通过。

### 每轮执行与停止规则

- 一个候选只改变一个失败层；源码与调研先提交/push，构建后核对实际镜像内容，再 RAM-only `fastboot boot` 并抓完整日志/屏摄。每轮更新 `porting.toml` 的证据与 next_action，不提前标记 hardware-qualified。
- 未提交 GPU 阶段可继续软件路径；已提交且 DMA 状态不确定必须隔离并重启，不能强行回退写同一 framebuffer。
- 禁止 flash 安卓分区、userdata、persist；不改变现有 Data UUID/根卷布局，不为 GPU 混入触摸、电池、120 Hz 或其他未请求改动。
- 对已知的 secure-IO `-2` 和 RSC 硬挂不复试以“确认”；缺失证据和拟实施动作分开记录。

### 参考与适用限制

- [Pong ios16 GPU](https://github.com/Chengdong-Yu/UniXNU-pong/blob/7798f1e5f261e7121d01d7e4807ed73512737ded/drivers/PongAdreno/README.md)：同 SoC 实现与尚未确认的阶段，不是本板渲染成功证据。
- [Nothing SM8475 gen7 GMU](https://github.com/NothingOSS/android_kernel_msm-5.10_nothing_sm8475/blob/sm8475/t/drivers/gpu/msm/adreno_gen7_gmu.c)：板族硬件顺序来源；APSS RSC 与 GPU RSCC 是不同窗口。
- [UniXNU GPU adaptation](https://unixnu.com/porting/gpu)：分层硬件验收方法可复用；页面中的 A740/iOS17/27 固件、初始化表、ABI、默认性能策略不能直接套到 A730/20G81。

## 历史 scanout 调研（已过时，不作为执行清单）

以下保留旧补丁分析。它把缺少补丁、未传参数等旧状态当现状；当前以本页前面的修复计划、实际镜像和实机证据为准。


## 阅读约定

| 简称 | 完整路径 |
|---|---|
| `SERIES` | `patches/xnu/series` |
| `SCANOUT` | `patches/xnu/zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz-pong-gpu-scanout.patch` |
| `GPUONLY` | `patches/xnu/zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz-pong-gpu-only-display.patch` |
| `PITCH` | `patches/xnu/zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz-pong-source-row-pitch.patch` |
| `UNION` | `patches/xnu/zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz-pong-union-lower-directory.patch` |
| `ADDON` | `patches/xnu/zzy-native-gpu-addon-mount.patch` |
| `BROKER` | `patches/xnu/offline-native-gpu-broker.patch` |
| `BUILD` | `scripts/build-xnu-4k-baseline.sh` |
| `SCANOUTCPP` | `drivers/PongAdreno/PongAdrenoScanout.cpp` |
| `PLATFORMCPP` | `drivers/PongAdreno/PongAdrenoPlatform.cpp` |
| `CPCPP` | `drivers/PongAdreno/PongAdrenoCp.cpp` |
| `GMUH` | `drivers/PongAdreno/PongAdrenoGmu.h` |
| `PLAN120` | `docs/120HZ-PORT-PLAN.md` |

---

## ① 结论摘要

### 1.1 能不能一次开

**构建可以一次做完，运行不要一次开。**

- 构建侧：一次镜像可以同时包含 Adreno 平台归档、三个 scanout 补丁和 `0232`；这些不冲突。
- 但三个 scanout 补丁**不能原样 apply**：`SCANOUT` 的第二个 hunk 与 `0210` 改的是同一段 `start()` 上下文；`GPUONLY` 的第一个 hunk 与 `0211` 改的是同一个 `pong_software_ui` 启动块。按 `SERIES` 现有顺序直接追加，`git apply --check` 会失败，`BUILD` 会退回 `patch --fuzz=3`（`BUILD:152-157`），fuzz 可能把 hunk 贴到错误位置。**必须 rebase，不能靠 fuzz。**
- 运行侧：先只开 Adreno 启动（`pong_adreno_start=1`，`pong_gpu_scanout=0`），拿到 `PONG ADRENO: ...` 的 CP 成功日志；再开 `pong_gpu_scanout=1`，拿到 `PONG SCANOUT: GPU frame completed`；最后才用 `pong_gpu_only=1` 做“是否真的 GPU 上屏”的决定性实验。一次全开如果黑屏，日志会同时缺少三层证据，无法定位。

### 1.2 最小改动集（先给结论）

1. `configs/mondrian_defconfig:52` 保持 `CONFIG_XNU_PONG_ADRENO_PLATFORM=y`。当前已提交值就是 `y`；它经 `scripts/unixnu/kernel.py:17-26` 变成 `XNU_PONG_ADRENO_PLATFORM=1`，`BUILD:957-983` 因此编译并 `-force_load` `libPongAdrenoPlatform.a`。
2. 在 `SERIES` 末尾追加（顺序不能反）：
   - `SCANOUT`
   - `GPUONLY`
   - `PITCH`
   - `UNION`（只有在使用 `ADDON` 或 SystemOS union 时才需要；它本身 apply 干净，见 ②）
3. 对 `SCANOUT` 的 hunk 2 和 `GPUONLY` 的 hunk 1 做两处 rebase（④ 给出精确位置）。
4. 保留 `0232`。它是 `pong_ramoops_diagnostic_write` 的唯一实现；`SCANOUTCPP:12,26` 和 `PLATFORMCPP:23,46` 都引用它，`PongAdreno` 被 `-force_load` 时没有它就会链接失败。`SERIES:126` 已经在列。
5. **不要**把 `0231` 的 `#if` 删掉；**不要**为了“打开 GPU”去改 `0231`。原因见 1.3。
6. 第一轮**不要**启用 `ADDON` 和 `BROKER`。`ADDON` 的构建门是 `XNU_NATIVE_OFFLINE_GPU=1`（`BUILD:114-117`），而它又强制 `NATIVE_VIRTIO_CONSOLE=1`（`BUILD:468-475`）；这会把离线 GPU broker 一起拖进来，和 PongAdreno scanout 不是同一条路。`BROKER` 在 `SERIES:52` 已应用，但 `XNU_NATIVE_OFFLINE_GPU=0`（`configs/mondrian_defconfig:21`）把它编译成空调用（`BROKER:7-10`）。
7. 启动参数（按阶段加，不要一次全加）：
   - 阶段 A：`pong_adreno_start=1`
   - 阶段 B：`pong_adreno_start=1 pong_gpu_scanout=1`
   - 阶段 B+Metal：再加 `pong_metal=1`
   - 决定性实验：再加 `pong_gpu_only=1`（只用于一轮，确认后去掉）
   - **不要**在这个阶段加 `fb_hz=120`；120 Hz 是另一条 DSI 面板路线，见 ⑥。

### 1.3 先纠正一个前提：`0231` 不是 GPU 的总开关

`0231` 的内容是：

```c
MiniIOMobileFramebufferStart(provider);
#if XNU_K60_ACCEL_STUB
	MiniAcceleratorPublishStub(provider);
#endif
```

证据：`patches/xnu/0231-k60-accel-stub-adreno-gate.patch:7-11`。

`XNU_K60_ACCEL_STUB` 不是独立配置，而是由 PongAdreno 是否链接推导出来的：

```sh
EXTRA_CFLAGS="$EXTRA_CFLAGS -DXNU_K60_ACCEL_STUB=$((1 - PONG_ADRENO_PLATFORM))"
```

证据：`BUILD:1066`。而 `PONG_ADRENO_PLATFORM` 来自 `configs/mondrian_defconfig:52` 的 `CONFIG_XNU_PONG_ADRENO_PLATFORM=y`（`BUILD:957`，映射见 `scripts/unixnu/kernel.py:17-26`）。所以当前提交的配置是：

- `PONG_ADRENO_PLATFORM=1` → `XNU_K60_ACCEL_STUB=0`
- `0231` 的 `#if` 为假，**空 stub 的调用点被删掉**
- `0212` 的整个 `MiniAcceleratorPublishStub` 定义也在 `#if ... XNU_K60_ACCEL_STUB` 内（`0212:25-29`，闭合在 `:132`），同样被编译掉
- 真正发布加速器的是 PongAdreno 的晚初始化路径（`PLATFORMCPP:389-397`，`PongAdrenoNativeGlue.cpp:35-51`）

因此，**如果现场看到的是空 stub 在应答，先查这次构建是不是把 `XNU_PONG_ADRENO_PLATFORM` 覆盖成了 0**，而不是去改 `0231`。把 `0231` 的 `#if` 删掉在 `stub=0` 时会直接产生 `MiniAcceleratorPublishStub` 未定义符号；在 `stub=1` 时则仍然发布空 stub。两种都不是“打开 GPU”。

当前 K60 仍走软件 framebuffer 的直接原因不是 `0231`，而是：

- `SERIES` 没有三个 scanout 补丁（`SERIES:1-144`，GPU 相关只有 `:87` 的 UI foundation 和 `:96` 的 panel-selection），所以 `MiniIOMobileFramebuffer::show()` 里没有 `pong_scanout_frame` 调用点；
- `pong_adreno_start=1` 没有传，`pong_adreno_start_optional()` 直接 return（`PLATFORMCPP:389-397`）；
- `pong_gpu_scanout=1` 没有传，`pong_scanout_initialize()` 直接 return（`SCANOUTCPP:30-33`）。

`0232` 不是“可选诊断”：它是链接依赖。`0231` 不是“要移除的门”：它是 stub 与真驱动的互斥保护。把这两个判断反过来，会得到错误的改动集。**如果一定要用“绕过 0231”来描述，正确形式是让 `XNU_K60_ACCEL_STUB=0`（即 `CONFIG_XNU_PONG_ADRENO_PLATFORM=y`），而不是编辑 `0231` 的 `#if` 行。**

---

## ② 六个补丁逐条说明

### 2.0 hunk 总表

| 补丁 | 文件 | hunk 起始（patch 文件行） | 目标函数/位置 |
|---|---|---|---|
| `SCANOUT` | `iokit/Kernel/MiniIOMobileFramebuffer.cpp` | `:3` `@@ -41,6 +41,10 @@`；`:14` `@@ -386,6 +390,8 @@`；`:23` `@@ -801,9 +807,11 @@`；`:35` `@@ -859,6 +867,30 @@`；`:66` `@@ -905,7 +937,7 @@` | 文件头 extern；`start()`；`show()` 的 swap 等待；`show()` 的 GPU 交接块；`show()` 的 `drawLayer` 调用 |
| `GPUONLY` | `iokit/Kernel/MiniIOSurface.cpp`；`iokit/Kernel/MiniIOMobileFramebuffer.cpp` | `:3` `@@ -1917,12 +1917,12 @@`；`:20` `@@ -242,6 +242,7 @@`；`:28` `@@ -351,6 +352,9 @@`；`:38` `@@ -868,7 +872,7 @@`；`:47` `@@ -889,8 +893,17 @@`；`:67` `@@ -906,9 +919,9 @@`；`:79` `@@ -937,7 +950,7 @@` | `native_pong_ui_services_start()`；类字段；`start()`；`show()` 的注释/帧判定/清屏/绘制 |
| `PITCH` | `iokit/Kernel/MiniIOMobileFramebuffer.cpp` | `:3` `@@ -882,7 +882,7 @@`；`:12` `@@ -893,7 +893,7 @@` | `show()` 的 stride 判定与 `pong_scanout_frame` 参数 |
| `UNION` | `bsd/vfs/vfs_lookup.c` | `:3` `@@ -1539,7 +1539,16 @@` | `lookup_traverse_union()` 的 `namei()` 结果（XNU 8796 `bsd/vfs/vfs_lookup.c:1511-1517`） |
| `ADDON` | `bsd/kern/kern_exec.c`；`bsd/miscfs/bindfs/bind_vfsops.c`；`bsd/miscfs/bindfs/bind_vnops.c`；`bsd/miscfs/bindfs/bindfs.h`；`osfmk/kern/startup.c` | `:5` `@@ -7193,6 +7193,82 @@`；`:88` `@@ -7250,6 +7326,7 @@`；`:100` `@@ -78,6 +78,8 @@`；`:109` `@@ -139,8 +141,9 @@`；`:121` `@@ -230,6 +233,9 @@`；`:135` `@@ -105,6 +105,14 @@`；`:154` `@@ -81,6 +81,9 @@`；`:168` `@@ -470,10 +470,12 @@` | `native_gpu_addon_mount()`/`native_gpu_addon_prepare()`；`native_owner_data_prepare()`；`bindfs_mount()`；`bindfs_getattr()`；`BINDM_FORCE_ROOT`；`kernel_bootstrap()` |
| `BROKER` | `bsd/kern/bsd_init.c` | `:3` `@@ -912,6 +912,10 @@` | `bsd_init()` 的 DTrace 之后 |

### 2.1 `SCANOUT`：把 framebuffer 的物理页交给 GPU

`SCANOUT` 在 `MiniIOMobileFramebuffer.cpp` 做五件事：

1. 声明三个 C 符号：`pong_scanout_initialize`、`pong_scanout_frame`、`pong_scanout_quarantined`（`SCANOUT:7-10`）。这三个符号来自 `drivers/PongAdreno/PongAdrenoScanout.cpp`，随 `libPongAdrenoPlatform.a` 链接（`BUILD:979-983`，源文件清单 `scripts/build-pong-adreno-platform.py:83`）。
2. 在 `IOMobileFramebuffer::start()` 里，拿到 `fBase/fRowBytes/fPanelWidth/fPanelHeight` 之后调用 `pong_scanout_initialize`，把 boot framebuffer 的物理范围、stride、面板尺寸登记给 scanout 模块（`SCANOUT:18-19`）。
3. 在 `show()` 的 swap 循环里加 `surfacesReady`：只要有一个 surface 还被 GPU 画着（`MiniIOSurfaceDrawn` 超时），就**不**把这个 swap 当成可交接帧（`SCANOUT:27-33`）。
4. 在 `show()` 的 `SWAP_LAYERS_CLEARED` 日志之后、layer 循环之前，插入 GPU 交接块（`SCANOUT:39-62`）：
   - 先查 `pong_scanout_quarantined()`；一旦隔离，直接解锁返回，不写 CPU（`SCANOUT:43`）。
   - 只接受“单层、全屏、无缩放、BGRA、4 字节/像素、planeCount=0、source/destination 都是 0,0,panelW,panelH”的 swap（`SCANOUT:48-53`）。
   - 取 `base->client ? base->client : base->memory` 作为 `IOMemoryDescriptor`，算 `base->pixels()` 相对 `kpixels/kbase` 的偏移（`SCANOUT:54-55`）。
   - 第一次交接时把 console 切到 graphics mode 并 release screen（`SCANOUT:56-60`）。
   - 调 `pong_scanout_frame(memory, offset, fRowBytes, fPanelWidth, fPanelHeight)`；返回值非 0 就置 `gpuFrame`（`SCANOUT:61`）。
5. 在 layer 循环里，`gpuFrame` 为真时跳过 `drawLayer()`（`SCANOUT:70-71`），避免 CPU 与 GPU 同时写同一块 framebuffer。

**它不做什么**：不编程 DSI/VIG，不设置 panel，不改 refresh rate，不分配 BO。它只把 bootloader 已经点亮的 framebuffer 物理页交给 pong 的 scanout 协议。`SCANOUTCPP:30-58` 的初始化也只做 `PE_current_console()` 校验和 `IOMemoryDescriptor::withPhysicalAddress()`；没有 DSI 寄存器。

**依赖**：
- `SCANOUTCPP` 必须被编译进 `libPongAdrenoPlatform.a`（`scripts/build-pong-adreno-platform.py:83` 的 `names` 包含 `PongAdrenoScanout`）。
- `pong_ramoops_diagnostic_write` 必须存在（`SCANOUTCPP:12`），即 `0232`。
- `MiniIOSurface` 的 `info.bytesPerRow`、`pixelFormat`、`pixels()` 字段必须和 pong 的 `MiniIOSurface.h` 一致；K60 用的是同一份 `zzzz...pong-hackios-ui-foundation` 展开的 MiniIOSurface，字段一致。

### 2.2 `GPUONLY`：GPU 帧的语义与“只走 GPU”策略

`GPUONLY` 有 7 个 hunk，分两类。

**第一类：启动门（`MiniIOSurface.cpp`）**

- `:3-17` 把 `native_pong_ui_services_start()` 的启动参数从 `pong_software_ui` 改成 `pong_display`，并把日志从 “software display startup skipped” 改成 “native display startup skipped”。
- 这与 K60 的 `0211` 直接冲突：`0211:5-19` 把同一个块改成 `uint32_t enable = 1;`，默认开启，只有显式 `pong_software_ui=0` 才关。`GPUONLY` 期望的是 pong 原版 `uint32_t enable = 0; if (!parse(...)) return;`，所以这个 hunk 在 K60 上**不能原样应用**。④ 给出两种处理。

**第二类：帧语义（`MiniIOMobileFramebuffer.cpp`）**

- `:20-26` 加 `bool fGpuOnly;` 字段，注释明确 “explicit bring-up policy, never the default”。
- `:28-34` 在 `start()` 里解析 `pong_gpu_only`，默认 0。
- `:38-45` 只改注释，把 “other layouts keep the existing CPU composition path” 改成 “unless explicitly disabled”。
- `:47-63` 是核心：
  - `gpuFrame = pong_scanout_frame(...) == 1;`（只有 1 算完成；2 是 DMA 不确定，0 是 CPU 可回退）。
  - 调用后**再查一次** `pong_scanout_quarantined()`；隔离就解锁返回，保留上一帧。
  - 如果 `fGpuOnly && !gpuFrame`，打印 `PONG GPU ONLY: no completed GPU frame; CPU composition disabled`（最多 4 次），解锁返回，不再 CPU 合成（`GPUONLY:58-62`）。
- `:67-77` 在 `sid == 0` 的“底层被拿走”分支，只有 `!gpuFrame` 时才 `bzero` 清屏；GPU 帧时不清，避免 CPU 覆盖 GPU 刚写的内容。
- `:79-84` 只有 `!gpuFrame` 才 `drawLayer`。

**依赖**：必须在 `SCANOUT` 之后。它引用 `gpuFrame`、`pong_scanout_quarantined`、`pong_scanout_frame`，这些都是 `SCANOUT` 引入的。`PITCH` 又必须在它之后，因为 `PITCH` 改的是 `GPUONLY` 写下的 `==1` 行和 `bytesPerRow` 行。

### 2.3 `PITCH`：接受 IOSurface 自己的 stride

`PITCH` 只有两个 hunk，都是 `show()`：

- `:3-11` 把交接条件从 `base->info.bytesPerRow == fRowBytes` 改成 `base->info.bytesPerRow >= uint64_t(fPanelWidth) * 4`。
- `:12-18` 把 `pong_scanout_frame(..., fRowBytes, ...)` 改成 `pong_scanout_frame(..., base->info.bytesPerRow, ...)`。

**为什么需要**：`SCANOUT` 原来要求 IOSurface 的 stride 与 framebuffer 的 `fRowBytes` 完全相等；只要 CA 给 surface 加了 padding，GPU 交接就会被拒绝，静默回 CPU。`PITCH` 放宽为“至少 width*4”，并把真实 stride 传给 scanout。`SCANOUTCPP:137` 也会检查 `stride < width*4`，所以下限仍然有保护。

**依赖**：必须在 `GPUONLY` 之后。它修改的正是 `GPUONLY` 的 `gpuFrame = ... == 1;` 行；在 `SCANOUT` 原版 `!=0` 上应用会失败。

### 2.4 `UNION`：union lower 是符号链接时返回 ENOENT 而不是 ENOTDIR

`UNION` 只改 `bsd/vfs/vfs_lookup.c` 的 `lookup_traverse_union()`：

- `:3-18` 在 `namei()` 成功后检查 `nd.ni_vp->v_type != VDIR`；不是目录就 `vnode_put` 并返回 `ENOENT`，而不是把非目录 vnode 赋给 `*new_dvp`。

**作用**：base system 在 `/System/Library/Extensions` 之类的位置可能放的是符号链接，而 mounted system 放的是目录；union 查找如果拿到符号链接 vnode，会把后续 lookup 变成 ENOTDIR。`UNION` 让这种情况干净地落到 ENOENT，让 union 继续走 lower/upper 的其他分支。

**依赖**：源码级独立，不依赖 `SCANOUT/GPUONLY/PITCH`；`git apply --check` 在当前树上直接通过（本次实测 exit 0）。但如果要使用 `ADDON` 的 `/System/Library/Extensions` union，它应当在 `ADDON` 之前或同一批启用；否则 union 的 lower 符号链接问题会让 `ADDON` 的可见性检查失败。它不是 GPU scanout 的必需项；**只在用 `ADDON` 或 SystemOS union 时加**。

### 2.5 `ADDON`：把 owner Data 里的 Metal bundle/launchd job 以只读 union 盖到系统目录

`ADDON` 是六个补丁里最大的一个，184 行，5 个文件：

- `kern_exec.c:5-83` 新增 `native_gpu_addon_mount()` 和 `native_gpu_addon_prepare()`：
  - 只有 `sheng-gpu-addon=1` 才继续（`:71`）。
  - 源目录固定为 `/private/var/ngpu/addons` 和 `/private/var/ngpu/daemons`；挂载点固定为 `/System/Library/Extensions` 和 `/System/Library/LaunchDaemons`（`:74-75`）。
  - 用 `mount("bindfs", ..., MNT_RDONLY | MNT_UNION)` 做只读 union（`:35-44`）。
  - 挂载后检查 sealed lower 仍可见（`/System/Library/Extensions/AGXMetalA10.bundle`、`/System/Library/LaunchDaemons/com.apple.backboardd.plist`），再检查 owner 的条目可见（`NativeAdrenoMetal.bundle/Info.plist`、`org.blu-sharky.adrenoprobed.plist`）（`:51-64`、`:76-82`）。
  - 失败只记录 `stage/status`，不 panic（`:27-31`、`:46-49`、`:65`）。
- `kern_exec.c:88-92` 在 `native_owner_data_prepare()` 的 Data 挂载成功后调用 `native_gpu_addon_prepare()`。**它必须在 owner Data 挂载之后**；`BUILD:40-41` 明确说这个补丁改的是 “owner Data mount function inside the systemos patch”，所以它要排在 `offline-owner-systemos-mount.patch`（`SERIES:84`）之后。
- `bind_vfsops.c:100-130`：
  - 新增全局 `native_bindfs_boot_mount`（`:104`）。
  - `bindfs_mount()` 的 entitlement 检查改为：只有 boot mount 标志为假时才要求 `com.apple.private.bindfs-allow`（`:113-119`）。这把“启动期内核自己挂”与“用户态进程挂”区分开。
  - boot mount 时加 `BINDM_FORCE_ROOT`（`:125-127`）。
- `bind_vnops.c:135-149`：`bindfs_getattr()` 在 `BINDM_FORCE_ROOT` 时把 uid/gid 报成 0/0，让 launchd 和 loader 接受 owner Data 里的文件。
- `bindfs.h:154-161`：定义 `BINDM_FORCE_ROOT 0x80000000ULL`。
- `startup.c:168-182`：把 `-no_slto_panic` 的处理从 `if (debugger_enabled)` 块里移出来。这是与 GPU 无关的附带改动，但因为它在这个补丁里，启用 `ADDON` 会一起带上。

**它需要什么**：
- `/private/var/ngpu/addons` 和 `/private/var/ngpu/daemons` 必须**预先存在**。补丁只做 `vnode_lookup(source)`；源目录不存在就 `status = error` 返回（`ADDON:27-31`），不会创建目录。仓库里没有任何脚本创建这两个目录（`grep -rn ngpu` 只命中补丁本身和 cryptex 相关补丁），所以必须由 Data seed/安装步骤预置。
- `sheng-gpu-addon=1` 启动参数。
- 构建门：`BUILD:114-117` 在 `XNU_NATIVE_OFFLINE_GPU != 1` 时直接跳过 `ADDON`。也就是说，当前构建脚本下，**只把 `ADDON` 加进 `SERIES` 不会生效**；要让它生效，必须同时开 `XNU_NATIVE_OFFLINE_GPU=1`，而后者又要求 `NATIVE_VIRTIO_CONSOLE=1`（`BUILD:468-475`）。这是一个耦合，不是 `ADDON` 本身的需求。

**结论**：第一轮 scanout 不需要 `ADDON`。如果 Metal 插件必须从 owner Data 注入，再单独处理这个耦合；不要在“打开 GPU”这一轮同时开。

### 2.6 `BROKER`：离线 GPU broker 的启动调用

`BROKER` 只有 4 行有效改动：

- `:3-10` 在 `bsd_init()` 的 `dtrace_postinit()` 之后，如果 `(DEVELOPMENT || DEBUG) && defined(XNU_NATIVE_OFFLINE_GPU) && XNU_NATIVE_OFFLINE_GPU`，调用 `native_offline_gpu_publish()`。

**它是什么**：`native_offline_gpu_publish()` 属于 `E:/hackios/UniXNU/drivers/AdrenoMetal/NativeOfflineGPU.cpp` 的离线 broker 路径：它通过 VirtIO console 把 Metal 请求转发到主机侧渲染器，不是 PongAdreno 的 CP/scanout。它的 `start()` 要求 `provider->open()` 成功（`NativeOfflineGPU.cpp:129-134`），传输层是 `NativeVirtioConsoleTransport`（`:8`）。

**依赖**：
- `XNU_NATIVE_OFFLINE_GPU=1` 才会链接 `libNativeOfflineGPU.late.a`（`BUILD:1042`）。
- 同时要求 `NATIVE_VIRTIO_CONSOLE=1`（`BUILD:473-475`）。
- 当前 `configs/mondrian_defconfig:21` 是 `n`，`:30` 也是 `n`，所以 `BROKER` 虽然已在 `SERIES:52`，但编译为空。

**结论**：不要为了 scanout 启用它。它和 PongAdreno 是两条独立的 GPU 路线；同时开会让“GPU 是否接管”这个问题变成两个变量。

---

## ③ Pong 专属 vs K60 需改

### 3.1 面板选择

| 项 | Pong | K60 | 结论 |
|---|---|---|---|
| 面板尺寸 | `pong-touch-console.patch:61-65` 要求 `max_x==1080 && max_y==2412`（触摸坐标上限，不是 GPU 补丁） | `0210:22-24` 定义 `K60_PANEL_WIDTH 1440`、`K60_PANEL_HEIGHT 3200`，注释写明 DSC video mode、60 Hz | **不改 GPU 补丁**。`SCANOUT` 用的是 `fPanelWidth/fPanelHeight`（来自 `vinfo`），不是硬编码；`0210` 负责把 canvas 固定为 1440×3200（`0210:37-60`）。不要把 pong 的 1080×2412 抄过来。 |
| 面板选择 selector | `pong-panel-selection.patch:30-35` 只接受 `SetDisplayDevice(0)`，其他返回 unsupported | `0213:34-40` 的 `GetDisplayDevice` 返回 `kFBDisplayID=1` | 二者不冲突：一个管 set，一个管 get；K60 没有第二块面板，`SetDisplayDevice(0)` 语义正确。无需改。 |
| 画布/缩放 | 通用 `MiniIOMobileFramebuffer` 有 `fbwidth/fbheight/fbscale` 分支（`0210:17` 的注释：K60 上这些值被拒绝，`k60_panel=0` 才回到 generic） | `0210:44-56` 在 `k60PanelNative()` 时拒绝 `fbwidth/fbheight/fbscale`，canvas 就是面板 | **K60 专属，保持**。GPU scanout 只接受 `source/destination == panel` 的 swap（`SCANOUT:48-53`）；如果允许缩放，GPU 路径会直接拒绝并回 CPU。 |

### 3.2 DSI / VIG

- 六个补丁里**没有**任何 DSI/VIG 寄存器写。`SCANOUTCPP:30-58` 只校验 `PE_current_console()` 并把物理地址包成 `IOMemoryDescriptor`；`SCANOUTCPP:135-163` 只做 offer/map/complete。`PLATFORMCPP` 的窗口列表（`PongAdrenoGmu.c:10-13`）是 GPU/GMU/RSCC/PDC/GPUCC/SMMU/AO，不含 DSI。
- K60 的 panel 是 ABL 点亮的；`PLAN120:44-46` 记录“整棵 K60 内核没有一行 DSI/MDSS/PLL 代码”，面板当前是 ABL 点亮、纯软件 memcpy 到 boot framebuffer。
- 120 Hz 的 DSI 切档在 `E:/hackios/UniXNU-zizhan` 的 `PongAdrenoScanout.cpp` 里，不在 pong16 的六个补丁里（`PLAN120:203-211` 引用的是 zizhan 路径）。

**结论**：GPU scanout 不需要移植 DSI/VIG；它只要求 boot framebuffer 仍然有效。120 Hz 是后续独立工作，见 ⑥。

### 3.3 GMEM / BO 池

| 项 | 值 | 证据 | K60 是否改 |
|---|---|---|---|
| GPU chip ID | `0x07030002`（Adreno730v3 / FD725） | `GMUH:19`；`PLATFORMCPP:365` | 不改。K60 同 SM8475/A730；如果实际 ID 不同，`PongAdrenoGmu.c:70` 会拒绝。 |
| GMEM | `0x200000` = 2 MiB | `GMUH:19` | 不改。`CPCPP:234-235` 明确 gen7_4_0 的 `uche_gmem_alignment=0`，不要学 Sheng/A740 去编程 16 MiB UCHE 窗口。 |
| BO 池 | 默认 256 MiB，`pong_gpu_pool_mib` 可调，失败按 16 MiB 步长减半 | `CPCPP:310-315` | 先不改。K60 的 4 GiB 窗口（`porting.toml:61`）如果分配失败，日志 `PONG ADRENO: native BO pool ...` 会给出实际大小；必要时用 boot arg 降到 128/64。 |
| GPU 地址窗口 | 1 GiB（`UINT64_C(0x100000000)`） | `CPCPP:323`；`README.md:41` | 不改。这是 GPU VA，不是物理保留。 |

### 3.4 SID / SMMU

| 项 | Pong 值 | 证据 | K60 是否改 |
|---|---|---|---|
| `gfx3d_user` | SID 0 / mask 0x400 | `CPCPP:28-34`；`GMUH:19-21` | 同 SoC，先不改；**未核实 K60 实际 SMR/S2CR**。 |
| `gfx3d_lpac` | SID 1 / mask 0x400 | 同上 | 同上。 |
| `gfx3d_secure` | SID 2 / 0x402，不租 | `CPCPP:31`；`GMUH:20` | 不改；secure 路由不碰。 |
| GMU | SID 5 / 0x405 | `PLATFORMCPP:207-239`；`GMUH:21` | 同上。 |
| SMMU 物理地址 | `0x03da0000` / `0x40000` | `PongAdrenoSmmu.c:7`；`PongAdrenoGmu.c:13` | 同 SoC，先不改；如果 K60 的 GPU SMMU 基址不同，`PongAdrenoSmmuStart` 会拒绝。 |
| ACTLR | `0x32b`（qcom,qsmmu-v500） | `CPCPP:302-305` | 不改；这是 Pong DT 的值，K60 同 SoC 但 **未核实 K60 DT 的 actlr**。 |

`PLATFORMCPP:198-205` 会打印 `GPU SMMU actual IDs`，并把 groups/banks 限制到 24/21；`PLATFORMCPP:210-223` 如果发现 foreign stream 占用 ASID 1/2 会打印 `SKIP ASID lease conflicts with foreign stream` 并返回 false。**这是安全失败，不会抢正在用的 SMMU**。如果 K60 的 bootloader 已经给 GPU 建了 domain，第一轮就会停在这里。

### 3.5 固件路径

- 固件不是运行时路径，而是**构建时嵌入**：`scripts/build-pong-adreno-platform-linux.py:83-86` 把 `gmu_gen70000.bin`、`a730_sqe.fw`、`a730_zap.mdt`、`a730_zap.b01`、`a730_zap.b02` 生成 C 数组；默认目录是 `/mnt/d/Android/k60-ios/firmware/k60-gpu`（`:38`）。
- 本机 `D:\Android\k60-ios\firmware\k60-gpu` 实际存在这五个文件（`a730_sqe.fw` 74676 B、`gmu_gen70000.bin` 57272 B、`a730_zap.b01/b02/mdt` 2236/3416/3564 B）。K60 目录名已经是 `k60-gpu`，说明构建侧已经按 K60 取固件。
- **未核实**：这五个文件的 SHA-256 是否与 Pong 参考树完全一致。没有做哈希比对，不能声称“同一份固件”。如果 K60 的 ZAP 签名/fuse 不同，`PongAdrenoZap` 的 PAS13 会在 `CP/user-client startup` 之前失败。

### 3.6 `max_x` / `max_y`

- 只出现在 `pong-touch-console.patch:43,61-65,745-746,780-786`，是 **Goodix 触摸**的坐标上限，不是 GPU/scanout 的一部分。
- K60 的触摸面板是 1440×3200（`0210:22-24`）；如果把这个补丁原样带进 K60，`:61-65` 的 `max_x==1080 && max_y==2412` 会直接拒绝面板，触摸不工作。
- **结论**：GPU 启用不碰 `max_x/max_y`；如果以后移植 pong 的 touch，必须把这两个值改成 K60 的实测值。它不是“Pong 专属要改的 GPU 参数”，而是“不要误带的触摸参数”。

### 3.7 `configs/pong-iogpu-20G81.json`

这个文件里**没有** panel、DSI、GMEM、SID、BO 池之类的“设备能力项”。它只有 20G81 的 IOGPU/IOAcceleratorES ABI 布局：

- `guest`、`instance_bytes=136`、`virtual_entries=168`（`:2-4`）
- `metaclass`、`vptr`、`get_meta_class`、`constructor`、`metaclass_constructor`、`destructor`（`:5-10`）
- `operational_overrides=[]`（`:11`）
- `constructor_aliases`（`:12-15`）
- `original_prelink_base`（`:16`）

消费点是 `scripts/build-pong-adreno-platform.py:94-103`：它把这些地址加上 `mac-build/native-ufs/bindings.json` 的 `fixed_prelink_base` 与 `original_prelink_base` 的差值，生成 `PongIOAcceleratorBindings.S`，用于绑定真正的 `IOAcceleratorES` 构造/析构/元类。也就是说，它是**内核缓存 ABI 绑定**，不是硬件能力表。

**K60 是否改**：K60 与 Pong 用同一个 20G81 guest；如果 K60 的 prelink base 不同，`slide` 会自动处理（`:95-96`）。**不要**在这里找面板/GMEM/SID 的“能力项”，也不要把 Pong 的地址当成 K60 的硬件值。

### 3.8 启动参数与路径（Pong 惯例 vs K60）

| 参数/路径 | Pong 用法 | K60 现状 | 建议 |
|---|---|---|---|
| `pong_adreno_start=1` | 触发 Adreno 冷启动（`PLATFORMCPP:389-397`） | 未传 | 阶段 A 加。 |
| `pong_gpu_scanout=1` | 启用 framebuffer 交接（`SCANOUTCPP:33`） | 未传 | 阶段 B 加。 |
| `pong_metal=1` | 发布 Metal 加速器（`PongAdrenoNativeGlue.cpp:44-49`） | 未传 | 阶段 B+ 加；只影响 Metal 发布，不影响 scanout 本身。 |
| `pong_gpu_only=1` | 禁止 CPU 合成（`GPUONLY:28-34,58-62`） | 未传，默认 0 | 只用于决定性实验；确认后去掉。 |
| `pong_display=1` | pong 的 native display 启动门（`GPUONLY:3-17`） | K60 用 `pong_software_ui` 默认开启（`0211:9-18`） | 第一轮保留 `0211`，不要改成 `pong_display`；否则每次启动都要记得加参数。 |
| `pong_software_ui=0` | 关显示服务 | K60 默认 1 | 调试时可用来隔离显示服务。 |
| `sheng-gpu-addon=1` | 挂 owner Data 的 bundle/daemon（`ADDON:71`） | 未传 | 第一轮不用。 |
| `/private/var/ngpu/addons`、`/private/var/ngpu/daemons` | owner Data 源目录（`ADDON:74-75`） | 仓库无创建者 | 只有用 `ADDON` 时才预建并填充。 |
| `pong_gpu_pool_mib` | BO 池大小（`CPCPP:312`） | 未传，默认 256 | 分配失败时再调。 |
| `fb_hz` | 发布给 CA 的刷新率（`PLAN120:186-191`） | 默认 60 | 不要和 GPU 启用一起改；120 Hz 需 DSI 切档。 |

---

## ④ 启用顺序与最小 diff

### 4.1 应用顺序

**源码依赖（硬约束）**：

1. `SCANOUT`
2. `GPUONLY`（依赖 `SCANOUT` 的 `gpuFrame`/scanout 符号）
3. `PITCH`（依赖 `GPUONLY` 的 `==1` 行和 `bytesPerRow` 行）
4. `UNION`（源码独立；只在用 `ADDON`/SystemOS union 时加）
5. `ADDON`（如果启用；必须在 owner Data mount 之后，`BUILD:40-41`）
6. `BROKER`（独立；但当前编译为空，不要为 scanout 打开）

pong 的 format-1 glob 顺序也是这个方向：`zz*pong*` 先反序撤销（`BUILD:61-72`），再按 glob 的字典序应用（`BUILD:79-157`）；`zzy...` 的 `ADDON` 在 `zzzz...SCANOUT` 之前；`SCANOUT`（48 个 z）→ `GPUONLY`（52 个 z）→ `PITCH`（54 个 z）→ `UNION`（54 个 z，字典序在 `PITCH` 后）。但**不要**依赖 z 的个数来排序；以 4.1 的源码依赖为准。

**建议在 `SERIES` 的位置**：追加在末尾（`SERIES:144` 之后），即所有 K60 显示/内存补丁和 `0232` 都已应用之后。这样：

- `0232` 已在 `:126`，`pong_ramoops_diagnostic_write` 先存在；
- `0210/0211/0213` 已在 `:101/:102/:104`，rebase 的目标上下文已经稳定；
- `0231` 已在 `:125`，`XNU_K60_ACCEL_STUB` 的配对关系不被打破；
- `0236` 已在 `:130`，framebuffer 诊断先跑。

### 4.2 两处必须 rebase 的 hunk

**A. `SCANOUT` hunk 2（`SCANOUT:14-21`）**

pong 原上下文：

```c
fPanelWidth = vinfo.v_width;
fPanelHeight = vinfo.v_height;
fRowBytes = vinfo.v_rowbytes;
pong_scanout_initialize((uint64_t)fBase, (size_t)fRowBytes*fPanelHeight,
    fRowBytes, fPanelWidth, fPanelHeight);
fWidth = gFBWidth ? gFBWidth : fPanelWidth;
```

K60 在 `0210:37-60` 把 `fWidth/fHeight` 改成了 `if (k60PanelNative()) { ... } else { ... }`，所以最后一行不再紧跟 `fRowBytes`。rebase 只需要把 `pong_scanout_initialize` 插到 `fRowBytes = vinfo.v_rowbytes;` 之后、`if (k60PanelNative())` 之前：

```c
fPanelWidth = vinfo.v_width;
fPanelHeight = vinfo.v_height;
fRowBytes = vinfo.v_rowbytes;
pong_scanout_initialize((uint64_t)fBase, (size_t)fRowBytes*fPanelHeight,
    fRowBytes, fPanelWidth, fPanelHeight);
if (k60PanelNative()) {
    ...
```

这是**同一句 pong 代码**，只是换了插入点；不新增逻辑。注意 `fBase` 在 `start()` 更早的 `:385` 已赋值（XNU 树行号），所以此时可用。

**B. `GPUONLY` hunk 1（`GPUONLY:3-17`）**

pong 原逻辑是：

```c
uint32_t enable = 0;
if (!PE_parse_boot_argn("pong_display", &enable, sizeof(enable)) || !enable) {
    return;
}
```

K60 `0211:9-18` 已经改成：

```c
uint32_t enable = 1;
if (PE_parse_boot_argn("pong_software_ui", &enable, sizeof(enable)) && !enable) {
    IOLog("K60 DISPLAY: display services disabled by pong_software_ui=0\n");
    return;
}
```

两种处理，**推荐 A**：

- **A（最小回归，推荐）**：丢掉 `GPUONLY` 的第一个 hunk，保留 `0211` 的“默认开启、`pong_software_ui=0` 才关”。`GPUONLY` 的其余 6 个 hunk 全部保留。这样显示启动语义不变，GPU-only 策略仍然可用。这个改动是“去掉一个与 K60 既有补丁冲突的 hunk”，不是新逻辑。
- **B（严格 pong 参数）**：把 `0211` 的 `pong_software_ui` 改成 `pong_display`，并把默认值改回 0，然后每次启动都加 `pong_display=1`。这更接近 pong 的 boot args，但会让“忘记加参数 = 没有显示服务”，对 K60 是回归。只有在你要逐字复刻 pong 的启动脚本时才选 B。

### 4.3 最小 diff 清单

```text
configs/mondrian_defconfig
  52  CONFIG_XNU_PONG_ADRENO_PLATFORM=y          # 已是 y；确认没有本地覆盖

patches/xnu/series
  末尾追加（顺序固定）：
    ...-pong-gpu-scanout.patch
    ...-pong-gpu-only-display.patch
    ...-pong-source-row-pitch.patch
    ...-pong-union-lower-directory.patch         # 可选，用 ADDON/union 时加

patches/xnu/...-pong-gpu-scanout.patch
  hunk 2：把 pong_scanout_initialize 插到 K60 的 fRowBytes 之后、if (k60PanelNative()) 之前

patches/xnu/...-pong-gpu-only-display.patch
  hunk 1：删除（推荐，保留 0211）；或改成 pong_display 并加 boot arg（严格复刻）

patches/xnu/0231-k60-accel-stub-adreno-gate.patch
  不改。保留 #if，让 stub 与 PongAdreno 互斥。

patches/xnu/0232-k60-pong-adreno-ramoops-bridge.patch
  不改，保留。它是 pong_ramoops_diagnostic_write 的唯一实现。

patches/xnu/0212-k60-empty-accelerator.patch
  不改。PONG_ADRENO_PLATFORM=1 时它整体编译为空。
```

### 4.4 运行阶段

**阶段 A：只验证 Adreno 启动，不交接**

```text
pong_adreno_start=1
# pong_gpu_scanout 不传（默认 0）
```

期望：`PONG ADRENO: startup requested; no accelerator published`（`PLATFORMCPP:334`）→ `GPU SMMU actual IDs`（`:199`）→ `GMU native4K context`（`:293`）→ `GMU firmware/HFI`（`:371`）→ `GMU power handshake`（`:374`）→ `native BO pool`（`CPCPP:318`）→ `GPU native4K/SQE prepared`（`:347`）→ `Pong CP_ME_INIT bootstrap submitted`（`:369`）→ `CP fence`（`:127`）→ `CP/user-client startup`（`PLATFORMCPP:377`）→ `CP ready; authenticated nonsecure fence`（`CPCPP:402`）→ `PONG METAL: ... status=0x0`（如果加 `pong_metal=1`，`PongAdrenoNativeGlue.cpp:46-48`）。

如果这一步失败，不要开阶段 B；先按 ⑤ 的回退判据定位。

**阶段 B：打开交接，但保留 CPU 回退**

```text
pong_adreno_start=1
pong_gpu_scanout=1
pong_metal=1          # 需要 Metal 时
# pong_gpu_only 不传（默认 0）
```

期望：`PONG SCANOUT: actual framebuffer handoff available sequence=0 status=0x0`（`SCANOUTCPP:57`）→ `PONG SCANOUT: GPU frame completed sequence=N status=0x0`（`:159`，前 3 帧）→ `K60 DISPLAY: first frame presented ...`（`0213:27-30`）。如果只看到 `first frame presented` 而没有 `GPU frame completed`，说明还在 CPU 路径。

**阶段 C：决定性实验（只做一轮）**

```text
pong_adreno_start=1
pong_gpu_scanout=1
pong_metal=1
pong_gpu_only=1
```

判断：屏幕照常刷新，且没有 `PONG GPU ONLY: no completed GPU frame; CPU composition disabled`（`GPUONLY:60`）→ GPU scanout 真的在接管。屏幕冻结且该行出现 → 仍在 CPU 路径或 GPU 没交付。做完这轮把 `pong_gpu_only` 去掉，恢复 CPU 回退。

---

## ⑤ 判据日志与回退条件

### 5.1 正向判据（必须逐条出现）

| # | 日志/行为 | 证据 | 说明 |
|---|---|---|---|
| P1 | `PONG ADRENO: startup requested; no accelerator published` | `PLATFORMCPP:334` | 只证明启动线程进来了，不证明成功。 |
| P2 | `PONG ADRENO: GPU SMMU actual IDs result=0 a=0x... b=0x...` | `PLATFORMCPP:198-199` | 证明 SMMU 可读；后面还要看 lease。 |
| P3 | `PONG ADRENO: GMU native4K context result=0 ...` | `PLATFORMCPP:291-293` | GMU 的 native4K context 建立成功。 |
| P4 | `PONG ADRENO: GMU firmware/HFI result=0 ...` | `PLATFORMCPP:370-371` | GMU 固件上传和 HFI 成功。 |
| P5 | `PONG ADRENO: GMU power handshake result=0 ...` | `PLATFORMCPP:373-374` | PERF/BW/CORE_FW_START 的 ACK 成功。 |
| P6 | `PONG ADRENO: CP/user-client startup result=0 ...` | `PLATFORMCPP:376-377` | CP 启动成功；这是“GPU 起来了”的关键。 |
| P6a | `PONG ADRENO: native BO pool result=0 ...` | `CPCPP:318` | BO 池分配成功；失败会继续用 scattered BO，不是致命。 |
| P6b | `PONG ADRENO: GPU native4K/SQE prepared result=0 ...` | `CPCPP:347` | SQE/固件 BO 准备好；`LPAC native4K shared context` 紧随其后（`:348`）。 |
| P6c | `PONG ADRENO: Pong CP_ME_INIT bootstrap submitted result=0 ...` | `CPCPP:369` | 第一次真正的 CP **submit**；没有这行就没有提交链。 |
| P6d | `PONG ADRENO: CP fence result=0 ...` | `CPCPP:127` | 真实 timestamp fence 完成；这是 submit 真正被 GPU 执行的证据。 |
| P7 | `PONG ADRENO: CP ready; authenticated nonsecure fence` | `CPCPP:402` | 真正的 nonsecure fence 和 client 发布。 |
| P8 | `PONG METAL: NativeAdreno complete runtime publication status=0x0` | `PongAdrenoNativeGlue.cpp:46-48` | 只有 `pong_metal=1` 才有；0 才成功。 |
| P9 | `PONG SCANOUT: actual framebuffer handoff available sequence=0 status=0x0` | `SCANOUTCPP:57` | **交接初始化成功的唯一证据**。没有这行就是初始化被静默拒绝。 |
| P10 | `PONG SCANOUT: GPU frame completed sequence=N status=0x0` | `SCANOUTCPP:158-159` | 前 3 帧；证明真正的 GPU copy 完成。 |
| P11 | `K60 DISPLAY: first frame presented (swap ..., 1440x3200 canvas on 1440x3200 panel)` | `0213:27-30` | 只证明 `show()` 出了第一帧，不证明 GPU。要和 P10 一起看。 |
| P12 | `MiniIOMobileFramebuffer: ... frames in ... ms (... a second), ... us each in the driver ...` 的 `us each in the driver` 明显下降 | `pong-hackios-ui-foundation` 的统计行；`PLAN120:77` 引用 | 间接证据：CPU 合成被跳过。 |

### 5.2 回退判据（出现就退回软件路径）

| # | 日志/行为 | 证据 | 动作 |
|---|---|---|---|
| F1 | `PONG SCANOUT: GPU copy not retired; retain pages and previous frame` | `SCANOUTCPP:154-157` | DMA 不确定，`poisoned=true`；此后所有帧返回 2，**只能重启**。去掉 `pong_gpu_scanout=1` 重启，回到 CPU。 |
| F2 | `PONG GPU ONLY: no completed GPU frame; CPU composition disabled` | `GPUONLY:58-62` | 只在 `pong_gpu_only=1` 时出现；说明 GPU 没交付。去掉 `pong_gpu_only`，恢复 CPU 回退。 |
| F3 | `PONG SCANOUT: client closed with outstanding GPU copy; retain pages` | `SCANOUTCPP:72-75` | presenter 退出时还有未完成 copy；进入隔离。重启。 |
| F4 | `PONG SCANOUT: actual framebuffer handoff available` **没有出现** | `SCANOUTCPP:30-58` 的静默 return | 初始化被拒绝（地址不对齐、`PE_current_console` 宽高/stride/depth 不符、`v_rotate` 非 0、bytes 不匹配）。不要继续开 `pong_gpu_only`；先查 `0210` 的 `fRowBytes` 和 `0236` 的 `K60 FB:` 行。 |
| F5 | `PONG ADRENO: SKIP ...` / `FAIL ...` / `SMMU FAULT` | `PLATFORMCPP:308` 及各处 `record()` | GPU 没起来；`pong_gpu_scanout=1` 不会有效果。回到软件路径，按具体阶段修。 |
| F6 | `PONG METAL: ... status=0x...` 非 0 | `PongAdrenoNativeGlue.cpp:46-49` | Metal 发布失败；scanout 仍可能工作，但 CA 不会用 Metal。先修 Metal 或退回软件 UI。 |
| F7 | `K60 FB: ... -> INSIDE managed range` | `0236:37-48` | **硬停止**。framebuffer 在 XNU 管理的物理范围内，VM 可能把页面分给用户态/页表；GPU 继续扫描会造成条带、PTE/heap 损坏。先解决保留问题，再开 GPU。 |
| F8 | `PONG GPU ONLY` 没有出现但屏幕黑/冻结 | 行为判据 | 可能是 GPU 帧被提交但没显示，或 presenter 卡住；查 `GPU frame completed` 和 `PongMetalScanout` 的 `actual GPU scanout copy completed`（`PongMetalScanout.h:44`）。 |

### 5.3 两层路径的独立判据

- 内核 scanout 路径：P9 → P10。`SCANOUTCPP` 自己打 `PONG SCANOUT:`。
- Metal presenter 路径：`PongMetalScanout.h:44` 的 `actual GPU scanout copy completed`（前 3 次），失败分支 `:59/69/75/79/85/91` 分别是 boot policy 禁用、连接不可用、mode 拒绝、mapping 拒绝、import 不可用、worker 创建失败。
- 两条路任一生效都说明“GPU 接管了显示”，但**必须至少有一条日志**；只有 `K60 DISPLAY: first frame presented` 不能证明 GPU。

---

## ⑥ 风险与未核实项

### 6.1 风险（按严重度）

**R1（最大风险）：framebuffer 落在 XNU managed range 内，GPU DMA 与 VM 竞争同一批物理页。**

`0236:10-19` 的注释记录了 sibling K60 port 的 0170 发现：ABL 的显示控制器持续扫描 panel buffer；这些页一旦进入用户映射或页表，就会被控制台文本覆盖，表现为条带面板和 PTE/pthread/heap 损坏。`0236:37-48` 只打印 `K60 FB: ... -> INSIDE/outside managed range`，`k60_capmanaged` 只是可选地缩小 `mem_size`（`:50-56`），**没有把 framebuffer 从 VM 里保留出来**。如果日志是 `INSIDE`，开 GPU scanout 会让“ABL 扫描 + GPU DMA + CPU 控制台”三方写同一块内存，后果比不开 GPU 更严重。

**处置**：把 `K60 FB: ... -> INSIDE` 作为硬停止条件。先按 `0236` 的注释思路把 framebuffer 页从 free list/allocator 中保留（这是另一个补丁的工作，不在本次范围），确认 `outside` 后再开 `pong_gpu_scanout=1`。

**R2：补丁 fuzz 误贴。**

`SCANOUT`/`GPUONLY` 与 `0210/0211` 的冲突是结构性冲突；`BUILD:152-157` 在 `git apply --check` 失败后会 `patch --fuzz=3`。fuzz 可能把“插入 `pong_scanout_initialize`”贴到错误的 `fRowBytes` 附近，或把 `pong_display` 贴到别的 `pong_software_ui` 块。必须按 4.2 手工 rebase，并在构建日志里确认没有 `Hunk #... succeeded ... with fuzz`。

**R3：GPU 不是 cold 状态，启动直接 skip。**

`PLATFORMCPP:346-347`：

```c
uint32_t gx=rd(windows[PongGpuClock],0x905c);
if(gx&0x80000000){record("SKIP GX already powered; require cold boot",1,gx);return;}
```

K60 的 ABL 可能为了 boot splash 已经给 GPU 上电；如果这样，`pong_adreno_start=1` 会打印 `SKIP GX already powered` 并退出。此时不是补丁问题，是启动时序问题；需要让 ABL 不在 XNU 启动前给 GPU 上电，或接受 warm GPU 的另一条启动路径（当前代码没有）。

**R4：SMMU/SID 与 K60 实际路由不符。**

`PLATFORMCPP:207-239` 只接受 exact 5/0x405 的 GMU 路由和 0/0x400/1/0x401 的 GPU 路由；发现 foreign stream 占用 ASID 1/2 就 `SKIP ASID lease conflicts` 并返回 false。K60 的 GPU SID 路由**未核实**；如果不同，驱动会安全拒绝，但 GPU 不会起来。需要第一轮日志 `GPU SMMU actual IDs` + `SKIP ASID lease conflicts` 来定标。

**R5：固件/ZAP 认证失败。**

`build-pong-adreno-platform-linux.py:83-86` 嵌入的是 `D:/Android/k60-ios/firmware/k60-gpu` 下的文件；本机文件存在，但**没有和 Pong 参考树做哈希比对**。如果 K60 的 ZAP 签名/fuse 与 Pong 不同，`PongAdrenoZap` 的 PAS13 会失败，`CP/user-client startup` 非 0。此时不要反复重试；保留日志，先比对固件哈希和 ZAP 头。

**R6：电源/GMU 的 command-db / TCS3 假设不成立。**

`PLATFORMCPP:107-170` 假设 APSS DRV2 的 TCS3 布局、`count>=8`、`ncpt>=2`、active TCS3 空闲；`PongAdrenoReadPowerInputs` 从 `0x80860000` 读 command-db（`:335-339`）。K60 的 bootloader 可能给出不同的 command-db 或 TCS 状态，日志会是 `SKIP command-db`、`SKIP APSS DRV2 capacity`、`SKIP active TCS3 busy`、`FAIL parent votes`。这些是安全失败，不会写坏硬件，但 GPU 不会启动。

**R7：`pong_gpu_only=1` 会让一次 presenter 卡顿变成永久黑屏。**

`SCANOUTCPP:135-163` 在持有 `fLock` 的情况下等 presenter 最多 2000 ms（`:151`）；`GPUONLY:58-62` 在 `fGpuOnly` 且没有完成帧时直接返回，不画 CPU。如果 userspace presenter 卡住/退出，屏幕会停住；`poisoned` 后只能重启（`:139`、`:154-157`）。所以 `pong_gpu_only` 只用于一轮决定性实验，不要留在日常启动参数里。

**R8：120 Hz 与 GPU 启用混在一起会得到错误结论。**

`0210:66-67` 的日志用 `refreshHz()`，默认 60；`PLAN120:186-191` 说明 `fb_hz` 只改 CA 节拍，不改面板扫描率；`PLAN120:16-18` 明确“只改 `fb_hz=120` 会让 CA 按 8.33 ms 画、面板仍 60 Hz 扫，CPU 拷贝翻倍、撕裂概率升高”。GPU scanout 本身不依赖 refresh（`SCANOUTCPP:135-163` 只有 2 s 完成超时；`PongMetalScanout.h:19` 是 500 ms wait），所以**先 60 Hz 打通 GPU，再按 `PLAN120` 做 DSI 切档**。不要在 GPU 启用这一轮加 `fb_hz=120`。

**R9：`ADDON` 的构建耦合会把离线 broker 拖进来。**

`BUILD:114-117` 用 `XNU_NATIVE_OFFLINE_GPU` 作为 `ADDON` 的开关；`BUILD:468-475` 又要求 `NATIVE_VIRTIO_CONSOLE=1`；`BUILD:1042` 会链接 `libNativeOfflineGPU.late.a`。如果只是为了把 `NativeAdrenoMetal.bundle` 放进 `/System/Library/Extensions`，这条路会同时启用一个完全不同的 GPU broker。第一轮不要用；如果必须用，先决定是改构建门（把 `ADDON` 从 `XNU_NATIVE_OFFLINE_GPU` 解耦）还是走 sealed SystemOS/PID1 helper 的注入路径。

**R10：`ADDON` 没有日志。**

`ADDON` 只写 `native_gpu_addon_stage/status` 全局变量（`:16-19`），没有 `IOLog`。如果 mount 失败，串口/ramoops 里看不到原因，只能靠调试器读变量或看后续 Metal 发现失败。这也是第一轮不建议启用它的原因之一。

### 6.2 未核实项（不要当成结论）

1. **K60 的实际 GPU SID / SMMU 路由**：代码里只有 Pong 的 0/0x400、1/0x401、5/0x405；K60 的 SMR/S2CR 没有现场读数。第一轮 `GPU SMMU actual IDs` 和 `SKIP ASID lease conflicts` 才能定标。
2. **K60 的 GPU 固件哈希**：`D:/Android/k60-ios/firmware/k60-gpu` 的五个文件存在，但没有与 Pong 参考树比对 SHA-256；不能声称同一份固件。
3. **K60 的 boot framebuffer 几何**：`0210` 假设 1440×3200；`SCANOUTCPP:44-48` 会校验 `PE_current_console` 的 width/height/stride/depth/rotate。实际 stride 是否 5760、`v_baseAddr` 是否页对齐，未核实；如果 `PONG SCANOUT: actual framebuffer handoff available` 不出现，先查这里。
4. **K60 的 GPU 是否 cold**：ABL 是否在 XNU 前给 GPU 上电，未核实；`SKIP GX already powered` 会告诉你。
5. **K60 的 panel 实际扫描率**：`0210` 注释写 60 Hz，`refreshHz()` 默认 60；K60 面板有 60/90/120 三档（`PLAN120:7-11`），当前内核没有 DSI 切档代码（`PLAN120:44-46`）。不要从 `K60 DISPLAY: ... %u Hz` 反推面板真的在 120 Hz 扫。
6. **K60 的 Data seed 是否有 `/private/var/ngpu`**：仓库没有创建者；`ADDON:27-31` 在源目录不存在时只记录 status。未核实。
7. **`XNU_PONG_ADRENO_PLATFORM` 是否被本地构建覆盖为 0**：当前 `configs/mondrian_defconfig:52` 是 `y`，但如果现场看到空 stub，先查环境变量/本地配置，不要改 `0231`。
8. **`pong_gpu_pool_mib=256` 在 K60 的 4 GiB 窗口里是否一次分配成功**：`CPCPP:310-315` 会减半，日志 `PONG ADRENO: native BO pool ...` 给出实际大小；未实测。
9. **`PongAdrenoWindows` 的硬编码地址是否与 K60 的 DT 完全一致**：`PongAdrenoGmu.c:10-13` 是 Pong live.dts 的值；K60 同 SoC，但未逐项比对 K60 的 ADT/DT。
10. **`configs/pong-iogpu-20G81.json` 的 ABI 地址是否匹配 K60 当前 prelink**：脚本用 `mac-build/native-ufs/bindings.json` 的 `fixed_prelink_base` 算 slide（`build-pong-adreno-platform.py:95-96`），但未对 K60 镜像做符号级验证。

---

## 附：一句话回答

**能开，但不是“改一行 0231”**：当前 defconfig 已经把 PongAdreno 链进来并让 `XNU_K60_ACCEL_STUB=0`；真正缺的是把三个 scanout 补丁 rebase 进 `SERIES`、传 `pong_adreno_start=1`/`pong_gpu_scanout=1`，以及先确认 `K60 FB` 不在 managed range 内。最小集是“`SERIES` 加三个补丁 + 两处 rebase + 两个 boot arg”，`0232` 保留，`ADDON`/`BROKER` 第一轮不开。最大风险是 **R1：framebuffer 在 XNU managed range 内时，GPU DMA 与 VM 抢同一批页**，会从“GPU 没起来”升级成内存/页表损坏；`K60 FB: ... -> INSIDE` 必须当硬停止条件。
