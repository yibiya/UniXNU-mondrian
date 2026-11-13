# K60（mondrian）Goodix 触摸移植方案

> 依据：`docs/devices/marble-milacy.md`、`docs/devices/fuxi-oddlylabs.md`、`docs/devices/j716f.md`、`docs/devices/evb10.md`、`docs/plans/K60-TOUCH-PARAMS.md`，以及 `patches/xnu/zzzz…-pong-touch-console.patch`。
> 结论先行：硬件上最接近 K60 的官方页是 **marble-milacy 的 Goodix GT9916S**（SPI `0x990000`、CS0、15 MHz、IRQ GPIO21、RESET GPIO20、GCC `0x100000`、GPI `0x900000`）；但 marble 页与 fuxi 页都没有可用的 Goodix 驱动源码。可执行路线只能是：**marble 的板级事实 + pong 的 Berlin-D 协议核心 + fuxi 的分阶段 GENI/HID 工程骨架 + K60 自己的无 RSC 上电策略**。

## 1. 证据索引：官方页里到底有什么、能直接拿什么

### 1.1 总表

| 页面 | 具体仓库与 commit | 触摸相关文件/脚本 | 有没有 Goodix 驱动源码 | 能否直接拿 |
|---|---|---|---|---|
| `marble-milacy.md` | `https://github.com/MiLacy/UniXNU-marble`，`78f4e7bf00e88129f8b6ef5680d81770d386832d`（`porting.toml`） | `board.toml`、`configs/marble_defconfig`、`docs/HARDWARE.md`、`docs/MIGRATION.md`；无 `drivers/*touch*` 目录 | **没有**。`Input` 区明确写：`QcomGeniSpi` 只作为控制器组件选中，**没有 Goodix protocol 或 IOHIDDevice publisher**；`docs/HARDWARE.md` 只给 GT9916S 的 SPI/GPI/时钟/GPIO 数值 | 只能拿**板级参数**，不能拿驱动代码。`QcomGeniSpi` 本身是独立官方组件（`UniXNU/QcomGeniSpi@eac47baefe6a5d2368a70351e6ad1b6361bedbe2`），也不是 Goodix 协议 |
| `marble.md`（legacy，补充 GPI 细节） | `https://github.com/sing0626/UniXNU-marble`，`a63260314d130ebf02a987f0785f5ae3258c219c` | `touch/MARBLE-GPI-DMA-PATH.md`；笔记提到 `marble_smmu_stream.{c,h}`、`marble_gpi_spi_tre.{c,h}`，但它们是 host 侧解码/构包工具，**没有集成进 XNU** | **没有**。原文第一句：`No GPI command or touch-controller transaction has been issued by the XNU port` | 只能拿**GPI/DMA 路径事实**：GPI `0x900000/0x60000`、TX0/RX1、SEID4、protocol1、64 TREs；以及 `GENI_IF_DISABLE=1` 时 Linux 选 GSI DMA 的结论。不能拿完整 Goodix 驱动 |
| `fuxi-oddlylabs.md` | `https://github.com/OddlyLabs/unixnu-fuxi`，`78f30b651c0ca6ea325682832bb25767523212f5`；触摸子模块 `https://github.com/OddlyLabs/FuxiFtsTouch`，`a923703792a94196cefaf49293c162a4919e202a` | `drivers/FuxiFtsTouch/FtsProtocol.c`、`FtsTouchCore.c`；`package/FuxiTouchProbe.cpp`、`package/FuxiTouchBoard.h`、`package/NativeHidInput.cpp`、`package/FuxiHidProbe.cpp`；`scripts/build-fuxi-touch.py`、`scripts/build-fuxi-hid-input.py`、`patches/xnu/fuxi-touch-probe.patch` | **不是 Goodix**：控制器是 ST FingerTip **FTM5**（chip id `36 40`，固件 `53.42` 在芯片自己的 flash 中） | **协议层不能拿**；但它是唯一完整跑通的 **GENI SPI + RPMh + GPIO + FIFO + HID + 分阶段日志** 工程模板。K60 应复用其分层、staging、HID probe 结构，不复用 FTS 协议 |
| `j716f.md` | `https://github.com/Denis0x00/UniXNU-j716f`，`b0b1409c2752d0791802f441bdc87ab4a9e050ce` | 同 commit 树：`package/TouchBoard.h`、`package/TouchProbe.cpp`、`package/TouchState.cpp`；页面 Evidence 只直接指 `research/native-fast-vm-done-20261029.json` | **有，但为 I2C**：Goodix **GT7386**（`goodix,gt738x`），addr `0x5D`、10 点；`package/TouchProbe.cpp` 是完整 GENI I2C + RPMh + GPIO + FIFO/SE-DMA 驱动。**页面 `:856` 写 `SE4 0x00990000`，但同 commit 的 `package/TouchBoard.h` 写的是 QUP1 SE5 `0xa94000`（wrapper `0xac0000`）；两者冲突，以源码为准** | **不能直接当 SPI 总线驱动**；可作为 GT7386/GTX8 的识别、校验、触点解析、`native_touch_*` 快照发布参考。页面正文明确“touch coordinates and HID reports are not yet adapted” |
| `evb10.md` | `https://github.com/Ken-u/UniXNU-evb10`，`37befa9162d6b4b080cb63a95cccff0ebb4d34f5` | `package/Evb10Touch/Evb10Touch.cpp`；`evidence/touch-400khz-tk4-20261030.txt`、`evidence/boot-v1-20261030.txt` | **有，但为 I2C**：Goodix **GT1158**（GT9xx/GT1x 寄存器族），I2C5 `0xfead0000`，addr `0x14`（也试 `0x5d`、`0x5a`），10 点，轮询 10 ms | **不能直接当 SPI 总线驱动**；`0x8140/0x8048/0x814e/0x814f` 的 GT9xx 读法、产品 ID 解码、`native_touch_*` 发布可直接作为 fallback 协议参考 |
| `pong-touch-console.patch` | `E:/hackios/UniXNU-mondrian/patches/xnu/zzzz…-pong-touch-console.patch`（本地仓库；`bsd/dev/pong_goodix.c` 899 行） | `bsd/dev/pong_goodix.c`、`bsd/dev/pong_input.c`、`bsd/sys/pong_input.h`、`bsd/kern/bsd_init.c` 调用、`userland/touch.c` | **有，且是唯一完整 Berlin-D over GENI SPI**：`0xF1` 读、`0xF0` 写、`0x10014` 固件版本、`0x10070` IC info、26 字节 report 头 + 8 字节触点、`/dev/pong-touch` ring | **协议核心可以直接拿**；但它的 RSC/ldoc3/ldoc12 电源、`PONG` 编译域、char-device sink 和 ADT 断言都是 Pong 的，不能原样作为 K60 整机驱动 |

### 1.2 逐页要点

**marble-milacy（最接近 K60 的 Goodix 页）**

- `Input` 硬件（页面 `:303-305`）：Goodix GT9916S on SPI `0x990000`，SPI 605，IRQ GPIO21，reset GPIO20，GPI `0x900000`，clocks `94/134/135`。
- `docs/HARDWARE.md`（页面 `:334` 的 Evidence）补全为：
  - `compatible = "goodix,gt9916S"`；SPI `0x990000`，size `0x4000`，SPI 605，CS0，15 MHz max。
  - IRQ TLMM GPIO21，flags `0x08`（`goodix,irq-flags = 2`，下降沿）；reset TLMM GPIO20，active low。
  - touch panel range `0x438 x 0x960` = **1080 x 2400**（注意是 1× 面板像素，不是 K60 的 10× raw）。
  - GCC IDs `94 = se-clk`、`134 = m-ahb`、`135 = s-ahb`；GPI `0x900000/0x60000`，TX0/RX1，SEID4，protocol1，64 TREs。
- `Input` 状态是 `planned`；`Development findings` 说：`The vendor DT gives a complete wiring description, but the GPI/SMMU mapping and the Goodix firmware/config images are not implemented. The legacy port reached register reads and a clock snapshot but no physical touch.`（页面 `:317-318`）
- **结论：这是 K60 硬件参数的黄金来源，但不是驱动来源。** 复制其数值前仍要按 K60 live FDT 复核（K60 的 `panel-max` 是 14400×32000，不是 1080×2400）。

**fuxi-oddlylabs（最好的工程模板，但不是 Goodix）**

- 触摸控制器是 ST FTM5，不是 Goodix；`package/FuxiTouchProbe.cpp` 的头部说明它按 `fuxi-touch=<n>` 分 4 级：1 只读快照，2 RPMh + QUP clocks + pins + GENI identity，3 GENI SPI + reset + chip id，4 start sensing + interrupt + FIFO + publish。
- Stage 2 的 RPMh 细节：apps RSC DRV2、`tcs-offset 0xd00`、active TCS 2；先 vote vdd 再 vote avdd，等 3 ms；再 vote QUP clocks、配置 pins（页面 `:360-363`）。
- Stage 3：GENI SPI 12 MHz；reset low 10 ms；wait event `03`（2 ms polls，100 ms，三次，然后 ICR 4-wire fallback 和 chip id）；读 chip id（页面 `:362`）。
- Stage 4：`A0 00 00`、50 ms、`A0 00 01`；下降沿中断；drain FIFO；按 `ShengHidInput` ABI 发布（页面 `:363`）。
- HID 文件：`package/NativeHidInput.cpp` 是 `ShengHidInput 71a40e7` 的未改副本；`package/FuxiHidProbe.cpp` 加载 `IOHIDFamily`、`IOHIDEventDriver`，等 `IOHIDResource`，注册 metaclass，起 touch-only device；`scripts/build-fuxi-hid-input.py` 构建；`ci/check-hid-bindings.py` 校验 ABI（页面 `:353-356, 379`）。
- **结论：K60 的 stage 编号、日志行、HID probe 的生命周期可以照这个骨架；协议层必须换成 Goodix。**

**j716f（有 Goodix，但是 I2C）**

- 页面 `:856-857`：2026-11-04 验证 Goodix **GT7386** I2C 触摸（页面写 `SE4 0x00990000`，addr `0x5D`，10-point multi-touch）；但“touch coordinates and HID reports are not yet adapted”。**注意页面与源码不一致**：同 commit `package/TouchBoard.h` 定义 `TB_QUP_WRAPPER_BASE 0x00ac0000`、`TB_SE_BASE 0x00a94000`（QUPv3 Wrapper 1，SE5，i2c@a94000），不是 `0x990000`；采用时以源码为准，并在引用页面数值时标注这个冲突。
- 同 commit 的 `package/TouchBoard.h` 给出：`GTX8_ADDR_PRIMARY 0x5D`、`GTX8_REG_VERSION 0x8240`、`GTX8_REG_TOUCH_DATA 0x824D`、`GTX8_REG_CMD 0x8040`；`package/TouchProbe.cpp` 是完整 GENI I2C + RPMh + GPIO + FIFO/SE-DMA 驱动，并使用 `native_touch_*` 快照。
- **结论：不能把 `0x8240/0x824d` 当作 Berlin 的 `0x10014/0x10070`；GT7386 是另一套 GTX8 寄存器族。只借它的 Goodix 校验、触点状态机和 `native_touch_*` 发布方式。**

**evb10（最完整的 Goodix GT9xx 轮询参考）**

- 页面 `:319-359`：Goodix GT1158 在 I2C5，轮询；坐标网格 720×1280 映射到 1080×1920；400 kHz 扫描见 `0x14/0x5d`，product id 1158，10 点，352 个 HID report。
- `package/Evb10Touch/Evb10Touch.cpp` 的寄存器：`0x8140` product id、`0x8048` 配置（x/y max、触点数）、`0x814e` 状态（bit7 有新报告，低 4 位点数）、`0x814f` 触点；每触点 8 字节；`publish()` 写 `native_touch_seq/count/xs/ys/ids`。
- **结论：K60 若 Berlin 不应答，第二 ABI 应直接按这套 GT9xx 读法实现；不要自己发明。**

**pong（唯一完整 Berlin-D SPI 实现）**

- `pong_goodix.c` 的 `configuration()`（补丁 `:57-65`）要求 `/arm-io/pong-touch` 的 9 个属性：`spi-base/tlmm-base/gcc-base/rsc-base/cmd-db-base/gpio-offset/pins[6]/max-x/max-y`。本地文件把断言改成了 K60 值：`se_phys==0x990000`、`tlmm_phys==0xf000000`、`gcc_phys==0x100000`、`rsc_phys==0x17a20000`、`cmd_db_phys==0x80860000`、`gpio_offset==0x100000`、`pins=={16,17,18,19,21,20}`、`max_x==14400`、`max_y==32000`。
- `pong_goodix.c` 已实现完整的 `goodix_read`/`goodix_write`/`goodix_probe`/`goodix_poll`/`goodix_report`，且 `pong_input.c` 提供 `/dev/pong-touch` 的 512 条 ring；`userland/touch.c` 是 20F66/20G81 世代的 virtual HID bridge，`userland/build.py:374` 已把它编进 `libpong-ui.dylib`。
- `pong_goodix.c` 头部明确：`No firmware upload`；`bring_up()` 先试 GENI FIFO，如果 `GENI_IF_DISABLE` 为 1 则回退到 GPIO bitbang；`io_supply_on()` 是 Pong 的 `ldoc3 2700 mV` + `ldoc12 1800 mV`。
- **结论：这是 K60 的协议代码基底，但不是 K60 的板级/电源基底。**

### 1.3 一句话回答“marble/fuxi 的 Goodix 驱动源码在哪”

- **marble**：不存在。`MiLacy/UniXNU-marble` 的 `Input` 只有 `QcomGeniSpi` 控制器选择；仓库树没有触摸驱动目录。legacy `sing0626/UniXNU-marble` 只有 `touch/MARBLE-GPI-DMA-PATH.md` 的 GPI 路径笔记和 host 侧构包工具，明确没有发出过 GPI 命令或触摸事务。
- **fuxi**：不存在。`OddlyLabs/FuxiFtsTouch` 是 ST FTM5 协议，不是 Goodix；但它是 K60 最值得照抄的 **GENI SPI + RPMh + GPIO + FIFO + HID** 工程模板。
- **真正有 Goodix 代码的是**：`pong_goodix.c`（Berlin-D，SPI，唯一完整）、`j716f/package/TouchProbe.cpp`（GT7386，I2C）、`evb10/package/Evb10Touch/Evb10Touch.cpp`（GT1158，I2C）。

## 2. K60 与 marble 的硬件差异

### 2.1 对照表

| 项 | K60（live FDT / K60-TOUCH-PARAMS） | marble-milacy（GT9916S） | fuxi（FTM5，工程参考） | pong（Berlin-D，代码参考） |
|---|---|---|---|---|
| SoC | SM8475 | SM7475-AB（Cape） | SM8550 | SM8475 |
| 触摸型号 | `xiaomi,m11a-spi`；固件名 `goodix_firmware_m11a`；家族按 Berlin/GT9916 处理，**未硬确认** | Goodix GT9916S，`goodix,gt9916S` | ST FTM5，chip id `36 40` | Goodix Berlin-D |
| 总线 | GENI SPI `spi@990000`（SE4），`0x4000`，CS0；控制器上限 50 MHz，设备 15 MHz | SPI `0x990000`，CS0，15 MHz max，SPI 605 | QUP1 SE4 `0xa90000`，12 MHz | GENI SPI `0x990000`，4 MHz bring-up |
| IRQ / RESET | GPIO21 / GPIO20 | GPIO21 / GPIO20 | GPIO25 / GPIO24 | GPIO21 / GPIO20 |
| AVDD / IOVDD | AVDD GPIO71（active high）；IOVDD `pm8350c_l2`（ldoc2，1.8 V） | 页面未列触摸电源；只有 SPI/GPI/clock 清单 | RPMh `ldob12` 1.8 V + `ldom6` 3.2 V，TCS2 `0xd00` | `ldoc12` 1.8 V + `ldoc3` 2.7 V；RSC 0x17a20000 |
| GPI | live FDT 的 9 个触摸属性没有 GPI；需按 `0x900000/0x60000` 另行确认 | GPI `0x900000/0x60000`，TX0/RX1，SEID4，protocol1，64 TREs | 无 GPI 说明；QUP1 SE4 + SMMU stream `0xa3` | 无 GPI；FIFO 或 bitbang |
| GCC | `clock-controller@100000`；direct vote `0x62008` bits 6/7/8/9/14 | GCC IDs `94/134/135`（se/m-ahb/s-ahb） | QUP1 branch votes（同 Sheng） | `0x62008` bits 6/7/8/9/14 |
| 面板 | 1440×3200；Goodix raw `14400×32000`（10×） | 1080×2400；touch range `0x438×0x960` = 1080×2400（1×） | 1080×2400；FTS raw 10800×24000（10×） | 1080×2412（pong 原值）；本地补丁断言 14400×32000 |
| RSC | `rsc@17a00000`，drv-2 `0x17a20000`；**首读 TCS 整机死**（0246/0248/LOCKSCREEN） | **触摸页不使用 RPMh**；没有 rail vote | 使用 apps RSC DRV2 TCS2，但 fuxi 上可读 | 使用 `rsc-base=0x17a20000`，TCS `0xd00` |
| 触摸状态 | `planned` | `planned` | `implemented`（FTM5） | source 完整但未在 K60 probe |

### 2.2 GPI 与 clock ID：哪些能照抄，哪些不能

1. **marble 的 GPI 是独立 DMA 引擎**：`0x900000/0x60000`，GPI stream SID `0x5b6`（pool `0x100000/0x100000`），QUP wrapper `0x9c0000` SID `0x5a3`（pool `0x40000000/0x10000000`），TX0/RX1、SEID4、protocol1、64 TREs。`MARBLE-GPI-DMA-PATH.md` 还记录：在 marble 上实测 `GENI_IF_DISABLE = 1`，所以 Linux 的 `select_xfer_mode()` 选 GSI DMA；`QcomGeniSpi` 的 SE-DMA 在该设备状态下不是有效 fallback。
2. **K60 不能默认自己也有 GPI**：K60 live FDT 的触摸 9 属性没有 GPI 节点，`spi@990000` 也没有 `iommus`（gamma2 记录）；因此 first bring-up 不能建 GPI ring，也不能假设 SID。必须在 stage 1 读 `GENI_IF_DISABLE_RO`（SE + `0x64`）：
   - `bit0 == 0`：FIFO 可用，走 pong 的 `geni_transfer` FIFO 路径；这是最短路径。
   - `bit0 == 1`：FIFO 不可用。pong 会退到 GPIO bitbang（只能做低速 ID/诊断）；生产路径要按 fuxi 的 `GeniSpi` SE-DMA，或按 marble 的 GPI 路径实现 TX/RX/event rings。**在 36 字节版本事务成功前，不宣称 GPI 可用。**
3. **clock ID 不是关键路径**：marble 页的 `94/134/135` 是 clock framework 的 ID，K60 live FDT 只给 `qcom,cape-gcc@100000`；pong/sweet 走的是 `GCC +0x62008` 的五个 vote 位（`0x43c0`）和 `0x27004/0x27008/0x274dc` 的 halt bit31。只要 vote 回读包含 `0x43c0` 且三个 CBCR 的 bit31 全为 0，就说明 QUP0 SE4 时钟已经开；`94/134/135` 只能作为同 GCC 家族的假设，不能作为上机判据。
4. **引脚角色**：K60 pinctrl 只写 `gpio16/17/18` 属于 `qup4`，不逐线命名；`K60-TOUCH-PARAMS` 指出 pong 把 `pins[0..2]` 解释为 MISO/MOSI/CLK，而 sweet 0240 解释为 CLK/MOSI/MISO。QUP 模式下不影响；只有 bitbang fallback 才需要按实测/原理图确认 16/17 谁是 MISO、谁是 MOSI。

### 2.3 面板坐标：14400×32000 与 1440×3200 不是同一层

- K60 live FDT：`goodix,panel-max-x = 0x3840 = 14400`，`goodix,panel-max-y = 0x7d00 = 32000`；面板是 **1440×3200**。所以 14400/32000 是 Goodix 的 raw（10×面板像素），不是面板尺寸。
- marble 的 GT9916S：touch range `0x438×0x960` = `1080×2400`，与它 1080×2400 面板一致，是 1×。**不要把 marble 的 1080/2400 当作 K60 的 max。**
- pong 的 `goodix_report()` 做的是 `x * 65535 / max_x`，所以 K60 必须传 `max_x=14400, max_y=32000`，让 raw 先映射到 0..65535；如果填 `max_x=1440`，任何 raw x>1440 都会饱和到 65535。
- `userland/touch.c` 再把 0..65535 除以 65535 变成归一化坐标（`:234-235`），所以 **内核这一层不要先除 10**；`touch_flip=1` 左右镜像、`touch_flip=2` 上下镜像；方向最后按实机四角测试确定。

### 2.4 RSC：marble 没有用 RPMh，K60 必须绕过

- **marble**：`Input` 页只列 SPI/GPI/IRQ/reset/GCC，没有 `vdd-supply`、没有 regulator、没有 RPMh/TCS 操作；`Development findings` 说的是 GPI/SMMU 和 firmware 未实现。它不是“巧妙绕过 RPMh”，而是**第一 console 根本没有实现触摸电源**；触摸在 marble 上也是 `planned`，不能把“没有 rail vote”当作已验证方案。
- **fuxi**：使用 apps RSC DRV2 的 active TCS2（`tcs-offset 0xd00`），先 vdd 后 avdd；并且用 `fuxi-touch-power=0` 证明，跳过 rail vote 时控制器三次 reset 都不应答、chip id `00 00`，恢复 vote 后才读到 `36 40`（页面 `:378`）。这个“必须投票”的结论对 K60 只适用于有 RSC 的板；K60 的 RSC 不可读。
- **K60 的硬事实**：`0x17a00000` 首读整机死；`0x17a20000` 的 TCS 首次读也可能整机死（`0246` 注释原文：“这台机器上 RSC 一个寄存器都读不得：首次读 TCS 就整机死”；`0248` 的只读探针也以首读死告终）。这不是“暂时没读到”，而是**不能在 XNU 里通过 RSC 上电**。
- **K60 首通策略**：只做 **warm boot**——从 Android 已经打开 `pm8350c_l2` 的状态进 fastboot，再 `fastboot boot`；驱动 `k60rpmhoff=1`，**不 map RSC、不读 cmd-db、不写 TCS**。AVDD 走 GPIO71（这是安全的 TLMM 输出），IOVDD 假定为 Android 留下的 1.8 V。若冷启动不回答，先回 Android 热重启再 boot，不要读 `0x17a00000` 排查。
- **pong 的 `io_supply_on()` 不能直接用于 K60**：它按 Pong 的 `ldoc3 2700 mV` + `ldoc12 1800 mV`，而 K60 是 `pm8350c_l2`（ldoc2，1.8 V）+ GPIO71。即使 RSC 可读，它也会投错轨。K60 版本必须把这个函数改成：`AVDD GPIO71 high`；`IOVDD` 仅在显式打开 `k60touch_rsc=1` 时才走 `ldoc2`，默认整个函数跳过。

## 3. 目标实现：把 pong 的协议核装进 K60 的板级层

### 3.1 现状与缺口

当前 `UniXNU-mondrian` 已经把三个 pong 触摸补丁放进 `patches/xnu/series:194-196`：

- `zzzz…-pong-touch-console.patch`
- `zzzz…-pong-goodix-resume.patch`
- `zzzz…-pong-physical-keys.patch`

但触摸仍然是 `planned`，原因是三个具体缺口：

1. `scripts/make-pong-adt.py:207-217` 仍发射 Pong 的 `max-x=1080 / max-y=2412`；本地 `pong-touch-console.patch:65` 断言的是 `max_x==14400 && max_y==32000`。两者不一致，`configuration()` 会返回 false，日志是 `Pong touch: missing/incompatible Pong DT; SKIP`。**这是第一个必须修的 bug。**
2. `board.toml:125` 的 `kernel_args` 没有 `pong_touch=1`（也没有 `touch_flip`）；驱动虽然编译进内核，但 `pong_goodix_init()` 一开始就 `PE_parse_boot_argn("pong_touch", ...)`，没有参数直接返回。
3. `pong_goodix.c` 的 board/power 层是 Pong 的：`io_supply_on()` 用 `ldoc3/ldoc12`；`touch_thread` 无条件 `ml_io_map(RSC_PHYS, RSC_SIZE)`；`goodix` 只走 FIFO，没有 firmware blob。K60 必须替换这些，而不是启用原样。

### 3.2 推荐文件布局

在 `UniXNU-mondrian` 内新增一个 K60 专用的 in-tree patch（不要与 pong 补丁同时启用两个 `pong_goodix_init`）：

```text
patches/xnu/zzzz…-k60-touch-console.patch
  bsd/dev/k60_goodix.c        # 协议核：goodix_read/write/probe/poll/report
  bsd/dev/k60_input.c         # /dev/pong-touch 兼容 ring（可直接复制 pong_input.c）
  bsd/dev/k60_touch.c         # 线程、boot args、stage、日志
  bsd/sys/k60_touch.h         # K60_TOUCH_* 常量 + struct k60_touch
  bsd/conf/files              # 登记上述文件
  bsd/kern/bsd_init.c         # devfs 后调用 k60_touch_init()
```

如果不想新增 patch，也可以把 `pong_goodix.c` 原地改名为 `k60_goodix.c` 并保留 `pong_input.c`；但**不要保留两个初始化入口**。推荐新建 `k60-touch-console.patch`，把 pong patch 留在历史/参考，K60 series 用新 patch。

外部构建脚本（可选，模仿 fuxi 的 archive 方式）：

```text
scripts/build-k60-touch.py   # 编译 k60_goodix.c + k60_touch.c，生成 libK60Touch.a
configs/mondrian_defconfig   # 若走 archive，加 CONFIG_XNU_EXTRA_LD_ARCHIVES 的门
```

当前 `build-xnu-4k-baseline.sh` 以 `DEVELOPMENT_ARM64_PONG` 为 machine config，in-tree `bsd/conf/files` 方式最省改动；先按 in-tree patch 做，archive 只作为后续清理。

### 3.3 ADT 节点（`scripts/make-pong-adt.py` 必须改）

把 `:207-217` 的 `touch` 节点改成：

```text
/arm-io/pong-touch {
    compatible = "pong,goodix-berlin-d";
    spi-base = <0x00990000>;
    tlmm-base = <0x0f000000>;
    gcc-base = <0x00100000>;
    rsc-base = <0x17a20000>;        /* 仅为 schema 兼容；k60rpmhoff=1 时不读 */
    cmd-db-base = <0x80860000>;     /* 同上，不读 */
    gpio-offset = <0x00100000>;
    pins = <16 17 18 19 21 20>;
    max-x = <14400>;
    max-y = <32000>;
    avdd-gpio = <71>;               /* K60 新增，pong 原 schema 没有 */
};
```

- `compatible` 可以保留 `pong,goodix-berlin-d`，因为驱动按 `/arm-io/pong-touch` 查找；改名只会增加不必要的 ABI 变更。
- `max-x/max-y` 必须用 **raw** 14400/32000，不是面板 1440/3200。
- `pins[0..3]` 的 16/17/18/19 是 K60 pinctrl 的 SPI 四线；`pins[4]=21` INT、`pins[5]=20` RESET；`avdd-gpio=71` 是新增项，驱动要读取并 `gpio_output(71,true,...)`。
- `rsc-base`/`cmd-db-base` 保留只是为了让 `configuration()` 的 9 属性 schema 通过；K60 版本在 `k60rpmhoff=1` 下不能 `ml_io_map` 它们，也不能读 TCS。

### 3.4 驱动板级常量（`bsd/sys/k60_touch.h`）

```c
#define K60_TOUCH_SPI_BASE        0x00990000ULL
#define K60_TOUCH_SPI_SIZE        0x4000
#define K60_TOUCH_GIC_SPI         605
#define K60_TOUCH_CS              0
#define K60_TOUCH_SPI_MAX_HZ      15000000
#define K60_TOUCH_SPI_BRINGUP_HZ  4000000

#define K60_TOUCH_TLMM_BASE       0x0f000000ULL
#define K60_TOUCH_GPIO_OFFSET     0x00100000
#define K60_TOUCH_GPIO_MISO       16
#define K60_TOUCH_GPIO_MOSI       17
#define K60_TOUCH_GPIO_CLK        18
#define K60_TOUCH_GPIO_CS         19
#define K60_TOUCH_GPIO_INT        21
#define K60_TOUCH_GPIO_RESET      20
#define K60_TOUCH_GPIO_AVDD       71

#define K60_TOUCH_GCC_BASE        0x00100000ULL
#define K60_TOUCH_GCC_SIZE        0x70000
#define K60_TOUCH_GCC_VOTE        0x62008
#define K60_TOUCH_GCC_VOTE_BITS   0x43c0
#define K60_TOUCH_GCC_M_AHB_CBCR  0x27004
#define K60_TOUCH_GCC_S_AHB_CBCR  0x27008
#define K60_TOUCH_GCC_SE4_CBCR    0x274dc
#define K60_TOUCH_GCC_SE4_CFG     0x274e8
#define K60_TOUCH_GCC_SE4_CMD     0x274e4

#define K60_TOUCH_PANEL_X         1440
#define K60_TOUCH_PANEL_Y         3200
#define K60_TOUCH_RAW_X           14400
#define K60_TOUCH_RAW_Y           32000
```

这些值逐项来自 `K60-TOUCH-PARAMS` 的 live FDT 表（`:31-60`、`:84-88`、`:106-125`、`:157-166`）和 pong 驱动（`pong_goodix.c:283-298`）；`GCC_VOTE_BITS=0x43c0` 来自 `0241`/pong 的位 6/7/8/9/14 组合。`K60_TOUCH_GIC_SPI 605` 只是 SE 中断，不用于 touch IRQ（touch IRQ 是 TLMM GPIO21）。

### 3.5 启动参数与 stage

沿用 `K60-TOUCH-PARAMS` 的 `k60touch*` 命名，避免再造一套：

| 参数 | 默认 | 作用 |
|---|---|---|
| `k60touch` | `1` | 总开关；0 = 完全不跑 |
| `k60touchstage` | `2` | 1 pins；2 +IOVDD；3 +GCC；4 +SPI/ID；5 +poll/HID |
| `k60rpmhoff` | **`1`** | 跳过 RSC/IOVDD vote；K60 首通必须为 1 |
| `k60touchdelay` | `8` | bsd_init 后延迟秒数，等 devfs/日志 |
| `k60touchhz` | `4000000` | 首次 SPI 频率；ID 通过后再试 15 MHz |
| `k60touchid` | `1` | stage 4 只打印 ID 摘要后退出，不轮询 |
| `k60touchflip` | `0` | 1 左右镜像，2 上下镜像 |
| `k60touhid` | `0` | stage 5 启动 `/dev/pong-touch` + userland HID bridge |
| `k60touchdump` | `0` | 打印 IC info 原始长度/misc 地址，用于 report 地址确认 |

`k60touchstage=4 k60rpmhoff=1 k60touchid=1 k60touchdelay=8 k60touchhz=4000000` 是第一条上机命令；它对应 `K60-TOUCH-PARAMS` 4.1 的同一组参数，不需要 `k60touchstage=5`。

## 4. Goodix 协议层要做什么

### 4.1 传输原语（Berlin-D）

```c
/* 读：0xF1 + 4 字节大端地址 + 3 字节 0xFF，然后 8 字节 dummy + data */
static bool k60_gd_read(uint32_t addr, uint8_t *buf, uint32_t len);
/* 写：0xF0 + 4 字节大端地址 + data */
static bool k60_gd_write(uint32_t addr, const uint8_t *data, uint32_t len);
```

- 读帧的 tx buffer 至少 `8 + 1024`：`tx[0]=0xF1`，`tx[1..4] = addr >> 24/16/8/0`，`tx[5..7]=0xFF`，`tx[8..]=0`；`geni_transfer(tx, rx, 8+len)` 后 `memcpy(buf, rx+8, len)`。
- 写帧 `tx[0]=0xF0`，`tx[1..4]=addr`，`tx[5..]=data`；长度 `5+len`，不需要 rx。
- `k60_gd_read` 必须限制 `len <= 1024`；返回 false 时不要重试超过 3 次，防止 SE 总线挂死。

### 4.2 芯片识别

读取 `0x10014` 的 28 字节 `struct goodix_berlin_fw_version`（Linux `goodix_berlin_core.c` 同布局）：

```c
struct k60_gd_fw_version {
    uint8_t rom_pid[6];        /* 0x00..0x05，通常 "911" + NUL */
    uint8_t rom_vid[3];        /* 0x06..0x08 */
    uint8_t rom_vid_reserved;  /* 0x09 */
    uint8_t patch_pid[8];      /* 0x0A..0x11，固件字符串 */
    uint8_t patch_vid[4];      /* 0x12..0x15 */
    uint8_t patch_vid_reserved;/* 0x16 */
    uint8_t sensor_id;         /* 0x17 */
    uint8_t reserved[2];       /* 0x18..0x19 */
    uint16_t checksum;         /* 0x1A..0x1B，LE */
};
```

判据：

- `goodix_checksum_ok(version, 28)`：前 26 字节和 == 末尾 LE16；全 0/全 0xFF 视为无应答。
- 成功日志：`K60 TOUCH: id: berlin fw chip "911" firmware "..." 1.00.00.00 sensor N`。
- 失败时再试 GT9xx：`0x8140` 读 4 字节 product id（常见 `"9111"`），`0x814E` 读状态，`0x8150` 读触点；这套 fallback 只在 Berlin 无应答时启用。
- 若两套都无应答：`K60 TOUCH: id: abi none`，先查 IOVDD/SPI/引脚，**不要**读 RSC。

### 4.3 IC info 与 report 地址

读取 `0x10070`：先读 2 字节长度（LE16），再读完整 block（长度 < 1024），校验和通过后解析：

```c
unsigned off = 2 + 16 + 10 + 4;  /* length + version(16) + feature(10) + 4 counts */
for (int i = 0; i < 5; i++)
    off += 1 + 2 * info[off];    /* 5 个变长 u16 列表 */
uint8_t *misc = info + off;
uint32_t touch_addr   = le32(misc + 44);
uint32_t command_addr = le32(misc + 0);
uint32_t fw_state     = le32(misc + 12);
uint32_t fw_buffer    = le32(misc + 18);
uint16_t fw_buf_max   = le16(misc + 22);
```

- 必须校验 `touch_addr != 0`；`misc + 48 == 8`（touch_data_head_len）、`misc + 50 == 8`（point_struct_len）；`misc + 52/54` 是 IC 自报的坐标上限（若非 0，优先于 DT 的 14400/32000）。
- 成功日志：`K60 TOUCH: id: berlin touch report at 0x..., command at 0x..., axes 14400 x 32000`。
- `k60touchdump=1` 时打印 `ic info len=%u misc_off=%u touch=%#x cmd=%#x fw_state=%#x fw_buf=%#x/%u`，用于人工确认 report 地址不是 0 或 0xFFFFFFFF。

### 4.4 report 解析

与 Linux `goodix_berlin_core.c` 的 26 字节头一致：

```c
/* 第一次读：8 字节 header + 2 * 8 字节 touch + 2 字节 checksum = 26 */
k60_gd_read(touch_addr, report, 26);
if (report[0] == 0) return;                 /* no event */
count = report[2] & 0x0f;                  /* request_type low nibble */
if ((report[0] & 0x80) && count > 2 && count <= 10)
    k60_gd_read(touch_addr + 8, report + 8, 8 * count + 2);
k60_gd_write(touch_addr, &zero, 1);        /* ack: clear status */
if (!checksum_ok(report, 8)) bad_header++;
if (report[0] & 0x40) {                    /* request event */
    if ((report[2] & 0x0f) == 3) touch_reset();
    return;
}
if (!(report[0] & 0x80)) return;
if (count > 10 || (count && !checksum_ok(report + 8, count * 8 + 2))) bad_report++;
```

每个触点 8 字节：

```c
uint8_t id   = t[0] >> 4;       /* 0..9 */
uint8_t type = t[0] & 0x0f;     /* 1/3 = stylus，跳过 */
uint16_t x   = le16(t + 2);
uint16_t y   = le16(t + 4);
uint16_t w   = le16(t + 6);     /* 压力/宽度，可只记录 */
```

- 上报前把 raw 转 0..65535：`x = x >= 14400 ? 65535 : x * 65535 / 14400`；y 同理用 32000。不要在这里除 10。
- 维护 `down/seen/x/y` 的 10 个 slot；上一帧 down 但本帧不在 = UP；新出现 = DOWN；坐标变化 = MOVE。Goodix 固件已做跟踪，不需要矩阵算法。
- 对 `report[0] & 0x40` 的 request code 3 调 `touch_reset()`（reset low 2 ms，high 100 ms），并重新 probe；这是 `pong-goodix-resume.patch` 的逻辑。

### 4.5 固件加载：`goodix_firmware_m11a`

K60 DT 只给文件名，不给文件本体；当前仓库也没有 `proprietary-files.txt` 条目。实现分两级：

**P1（首通必须）——运行固件识别，不刷写：**

- 读取 `0x10014` 的 28 字节并校验；如果通过，说明芯片已有运行固件（与 fuxi 的 FTS 一样，Goodix 通常把固件放在芯片自己的 flash）。日志 `K60 TOUCH: fw running "..."`。
- 不调用任何写 flash 的命令；`pong_goodix.c` 原本就是 `No firmware upload`，首通沿用这个保守策略。

**P2（只有 P1 失败才做）——加载 `goodix_firmware_m11a`：**

- Android 侧文件位置通常为 `/vendor/firmware/goodix_firmware_m11a`（或 `/odm/firmware/`）；用 root 只读导出：

```sh
adb shell su -c 'ls -l /vendor/firmware/goodix* /odm/firmware/goodix* 2>/dev/null'
adb shell su -c 'dd if=/vendor/firmware/goodix_firmware_m11a' > goodix_firmware_m11a.bin
adb shell su -c 'dd if=/vendor/firmware/goodix_cfg_group_m11a' > goodix_cfg_group_m11a.bin
sha256sum goodix_firmware_m11a.bin goodix_cfg_group_m11a.bin
```

- 放入设备仓的 `firmware/` 目录，并登记到 `proprietary-files.txt`；用 `scripts/build-k60-touch.py` 生成 `K60GoodixFirmware.c`（`xxd -i` 或等价）把 blob 编进内核。**不要把几百 KB 的固件塞进 `board.toml` 或 ADT。**
- 运行时先做 Goodix boot confirm：在 `0x10000` 写 8 个 `0xAA`，读回 8 字节必须相同（Linux `goodix_berlin_dev_confirm()`）；然后按 IC info 的 `fw_state_addr` / `fw_buffer_addr` / `fw_buffer_max_len` 分包写。固件文件头的 magic/长度/校验和必须先用 `xxd -l 256`、`binwalk` 和厂商 `goodix_ts` 源码确认；**具体 firmware update command id 不从 pong/Linux mainline 猜，标为 `[GOODIX_FW_CMD_UPDATE]`，由 Android 厂商驱动核对后再填**。
- 若文件缺失或格式未知：日志 `K60 TOUCH: fw blob absent; running-firmware mode`，继续只读 ID/报告，不阻塞 HID 调试。

### 4.6 HID 发布

K60 的 `userland/build.py:374` 已把 `userland/touch.c` 编进 `libpong-ui.dylib`（target `arm64-apple-ios16.6.1`，`20G81`），这是当前仓库最贴近 K60 OS 的 HID 路径：

- 内核 `k60_input.c` 发布 `/dev/pong-touch`（`struct pong_touch { uint64_t time; uint32_t type, finger, x, y; }`；`0444`，root:wheel；512 条 ring）。
- `userland/touch.c` 打开 `/dev/pong-touch`，创建 `IOHIDVirtualServiceClient`：
  - digitizer page `0x0d` usage `0x04`；
  - consumer page `0x0c` usage `0x01`（tap-to-wake / power / volume）。
- 成功日志：`Pong touch HID: display-integrated digitizer opened`、`Pong touch HID: touch detection mode 0x1`、`Pong touch HID: type=1 finger=0 xy=...`。
- 若以后要与 fuxi 对齐改成 kernel `NativeHidInput`：必须先用 K60 20G81 的 kernelcache 重对 `IOHIDFamily` ABI（`ci/check-hid-bindings.py` 的 K60 版），不能直接把 fuxi 17.7.11 的 `NativeHidInput.cpp` 编译进来。**首通不采用 kernel HID，采用 userland bridge。**

## 5. 分阶段上机计划（只读 ID → report 地址 → HID）

### 5.1 构建前置

```sh
cd /e/hackios/UniXNU-mondrian
# 修改 make-pong-adt.py 的 touch 节点（见 3.3），确认 ADT：
python3 scripts/make-pong-adt.py --help   # 按当前脚本实际参数调用
# 确认 12 个值：
grep -n "max-x\|max-y\|spi-base\|pins" scripts/make-pong-adt.py
# 构建内核（K60 的现有流程）：
./unixnu build kernel -D ../UniXNU-mondrian --ipsw <K60 的 iPadOS 16.6.1/20G81 IPSW>
```

判据日志：ADT dump 中 `touch` 节点必须出现 `max-x = 14400`、`max-y = 32000`、`pins = 16 17 18 19 21 20`、`avdd-gpio = 71`。若仍是 1080/2412，直接停止构建，先修 `make-pong-adt.py`。

### 5.2 P1：只读 ID（stage 4，不轮询）

**启动参数：**

```text
k60touchstage=4 k60rpmhoff=1 k60touchid=1 k60touchdelay=8 k60touchhz=4000000
```

**只做什么：**

1. 映射 TLMM `0x0f000000 + 0x100000`，配置 AVDD GPIO71、RESET GPIO20、IRQ GPIO21。
2. **不 map RSC，不读 cmd-db，不写 TCS**；日志固定打印 `iovdd vote skipped (k60rpmhoff=1)`。
3. GCC vote `0x62008` 五位 + 读三个 CBCR 的 bit31；SE4 `0x990000` 配 SPI（4 MHz，mode 0，8-bit，CS0）。
4. 读 `0x10014` 28 字节、`0x10070` IC info，打印 chip/firmware/sensor、touch report 地址和 axes；然后 `k60_touch_done(4)` 退出，**不轮询、不写 report、不启 HID**。

**必须看到（按顺序）：**

```text
K60 TOUCH: init: stage 4 (1 pins, 2 +iovdd, 3 +clocks, 4 +spi/id, 5 +poll), delay 8 s, spi 4000000 Hz, flush 1
K60 TOUCH: stage 1: avdd gpio71, reset gpio20, irq gpio21
K60 TOUCH: powered: avdd=1 reset=1 irq level 0
K60 TOUCH: iovdd vote skipped (k60rpmhoff=1)
K60 TOUCH: stage 3: gcc 0x100000 window 0x70000, no SE access yet
K60 TOUCH: gcc vote 0x0 -> 0x43c0, bits 0x43c0
K60 TOUCH: gcc halt m/s/se 0x.../0x.../0x...        # 三个 bit31 全为 0
K60 TOUCH: clocks ready: ...
K60 TOUCH: se4 fw 0x102 version 2 protocol 1 (SPI=1)
K60 TOUCH: se4 spi cfg word 0x4 mclk 0x51 (source 19200000 Hz / 5)
K60 TOUCH: se4 pins gpio16-19 -> qup
K60 TOUCH: se4 status 0x0 fifo 64 words
K60 TOUCH: id: berlin fw chip "911" firmware "..." 1.00.00.00 sensor N
K60 TOUCH: id: berlin touch report at 0x..., command at 0x..., axes 14400 x 32000
K60 TOUCH: abi berlin selected, axes 14400 x 32000, panel 1440 x 3200
K60 TOUCH: stage 4: goodix id ok, abi 1
K60 TOUCH: stage 4 done (...), thread exits
```

**判读：**

- `iovdd vote skipped` 是预期，不是失败；如果芯片不答，先确认是从 Android 热重启进 fastboot，而不是冷启动。
- `gcc halt` 任一 bit31=1：GCC vote/RCG 未成功，停在 SE 之前；不要继续读 SE。
- `se4 fw 0x102` 不对：SE4 不是 SPI 固件；检查 `spi@990000` 是 SE4 与 QUP0。
- `id: berlin fw chip "911"` 是 Berlin ABI；若出现 `id: gt9xx`，说明是 GT9xx 族，按 evb10 的寄存器协议走 fallback，并更新本计划。
- `id: abi none`：先怀疑 IOVDD/SPI/GPIO，不要读 RSC。

### 5.3 P2：report 地址确认（stage 4 + dump）

**启动参数：**

```text
k60touchstage=4 k60rpmhoff=1 k60touchid=1 k60touchdump=1 k60touchdelay=8 k60touchhz=4000000
```

**目的：** 不靠“驱动解析成功”这一句，直接看 IC info 的原始 offset 和 `touch_addr/command_addr`；判据：

```text
K60 TOUCH: ic info len=..., misc_off=..., touch=0x..., cmd=0x..., fw_state=0x..., fw_buf=0x.../...
K60 TOUCH: id: berlin touch report at 0x..., command at 0x..., axes 14400 x 32000
```

- `touch != 0`、不等于 `0xFFFFFFFF`，且 `misc+48 == 8`、`misc+50 == 8`。
- 如果 `touch` 是 0：说明 IC info 解析偏移错；对照 Linux `goodix_berlin_core.c` 的 `goodix_berlin_parse_ic_info()`，不要猜。
- 如果 `touch` 合理但后续 report 读全 0：检查 SPI mode/频率/CS 和 `report[0]` 是否被 ack 清零。

### 5.4 P3：首触点（stage 5，轮询 + HID）

**启动参数：**

```text
k60touchstage=5 k60rpmhoff=1 k60touchdelay=8 k60touchhz=4000000 k60touhid=1
```

**必须看到：**

```text
K60 TOUCH: stage 4: goodix id ok, abi 1
K60 TOUCH: polling, irq 1, abi 1, every 8 ms
K60 TOUCH: report status 0x80 count 1
K60 TOUCH: touch finger 0 at 7200, 16000 -> 720, 1600 px
K60 TOUCH: touch down 0 (...)
K60 TOUCH: touch move 0 (...)
K60 TOUCH: touch up 0 (...)
Pong touch HID: display-integrated digitizer opened
Pong touch HID: touch detection mode 0x1
Pong touch HID: type=1 finger=0 xy=...
```

**判读：**

- `report status 0x80 count 0` 连续出现：芯片在报空帧，检查 reset/AVDD/固件状态；不要把它当触摸。
- `bad report header` / `bad report (N touches)`：SPI 时序、CS 保持、校验和或 `touch_addr` 错误；先降 `k60touchhz` 到 1 MHz 排除信号完整性。
- 坐标差 10 倍：说明 `/10` 提前做了；本方案要求内核输出 raw→65535，`userland/touch.c` 负责归一化。
- HID 没有 `display-integrated digitizer opened`：检查 `libpong-ui.dylib` 是否在 20G81 root 中、`pong_touch_start()` 是否被注入、`/dev/pong-touch` 是否 0444。

### 5.5 P4：稳定性/多指

在 P3 相同镜像上测试：

- 单指四角 + 中心，记录 raw 与 px，确认无镜像/轴向错误；需要时用 `k60touchflip`。
- 两指交叉滑动，确认触点 ID 稳定、抬一指后另一指不跳变。
- 边缘滑动（返回手势）确认 `userland/touch.c` 的 edge 标记生效。
- 连续运行 10 分钟，`bad report` 计数不增长、SE 无 transfer failed、无 panic。
- 若某次运行出现 SE 访问挂起：记录最后一行（`gcc halt` / `se4 fw` / `se4 spi cfg` / `se4 status`），不要在同一轮继续读 RSC。

## 6. 与 pong 的取舍：谁更适合作为 K60 基底

| 维度 | `pong-touch-console.patch` | K60 应采用 |
|---|---|---|
| Goodix 协议 | Berlin-D 完整：`0xF1/0xF0`、`0x10014`、`0x10070`、26 字节 report、8 字节触点、checksum、reset request | **保留**；这是唯一完整的 SPI Berlin-D 实现，直接从 `pong_goodix.c` fork |
| 总线 | GENI SPI FIFO 4 MHz；`GENI_IF_DISABLE=1` 时只退到 GPIO bitbang | K60 先读 `GENI_IF_DISABLE`；FIFO 可用就保留；不可用则 fuxi `GeniSpi` SE-DMA 或 marble GPI，不把 bitbang 当生产路径 |
| 板级 | `configuration()` 硬断言 9 属性；`make-pong-adt.py` 仍是 1080/2412，和本地补丁的 14400/32000 冲突 | 改 ADT；新增 `avdd-gpio=71`；保留 `/arm-io/pong-touch` 路径以减少 ABI 变更 |
| 电源 | `io_supply_on()` 用 `ldoc3 2700` + `ldoc12 1800`；`touch_thread` 无条件 map RSC | **整段替换**：K60 AVDD GPIO71；IOVDD 默认跳过，只允许 warm boot；`k60rpmhoff=1` 时 RSC 零 map |
| HID | `/dev/pong-touch` ring + `userland/touch.c`（20F66/20G81） | **保留 userland bridge**；`userland/build.py:374` 已编译；kernel `NativeHidInput` 是 P2，不是首通 |
| 固件 | 无上传；只读版本/IC info | 首通同样只读；`goodix_firmware_m11a` 作为 P2 可选 blob，不阻塞 |
| 编译域 | `#if defined(PONG)`，当前 K60 用 `DEVELOPMENT_ARM64_PONG` | 改名 `K60` 或 `K60_TOUCH`，避免 PONG 语义；若保留 `PONG` 需确认 `board_config.h` |
| 与 series 的关系 | 当前 `series:194-196` 已包含 pong 三补丁 | 新增 `k60-touch-console.patch` 替换 pong 补丁；不要同时留两个 init |

**结论：**

- **如果问题只是“谁有完整可运行的 Goodix 代码”**：`pong-touch-console.patch` 是唯一答案，marble/fuxi 都没有 Goodix 驱动；j716f/evb10 有 Goodix 但都是 I2C。
- **如果问题是“谁最适合当 K60 整机基底”**：不是 pong 整机驱动，而是 **pong 的协议核 + K60 的板级层**；pong 的 `ldoc3/ldoc12`、RSC 和无条件 map 必须去掉。
- **如果问题是“哪个官方页最接近 K60”**：`marble-milacy` 的 GT9916S 页，SPI/CS/15 MHz/IRQ21/reset20/GCC `0x100000` 全部同值；但它只配当“硬件规格页”，不能当“驱动源码页”。

## 7. 风险与未决项

| # | 风险/未决 | 现状 | 处置 |
|---:|---|---|---|
| 1 | Goodix 具体型号/ABI | DT 只有 `xiaomi,m11a-spi`、`goodix_firmware_m11a`；K60-TOUCH-PARAMS 说未硬确认；外部 ZCODE 报告称 compatible `goodix,gt9916`，但 live FDT 为 `xiaomi,m11a-spi` | P1 读 `0x10014`；`"911"` = Berlin，`0x8140` `"9111"` = GT9xx；两套都实现，以实际应答为准 |
| 2 | RSC 不可读 | `0x17a00000` 首读整机死；`0x17a20000` TCS 首读也可能死 | 首通 `k60rpmhoff=1`，RSC 零 map；冷启动另行解决 UEFI/SPMI 供电，不在本计划 |
| 3 | IOVDD 冷启动 | 无 UEFI PMIC/regulator 驱动；只能假定 ABL/Android 留电 | 从 Android 热重启进 fastboot；冷启动不回答时回 Android 再 boot，不读 RSC |
| 4 | `GENI_IF_DISABLE` | K60 未测；marble 实测为 1 | P1 读 SE+0x64；0=FIFO，1=SE-DMA/GPI；bitbang 只做 ID 诊断 |
| 5 | 引脚 16/17/18 角色 | pinctrl 只标 `qup4`，不逐线命名；pong/sweet 解释相反 | QUP 模式不影响；bitbang 时按实测/原理图确认 MISO/MOSI |
| 6 | 15 MHz 信号完整性 | DT 上限 15 MHz，首验 4 MHz | ID 通过后逐级 4→8→15 MHz，记录 `transfer failed`/bad checksum |
| 7 | 固件 blob 格式 | `goodix_firmware_m11a` 本体不在仓库；无 `proprietary-files.txt` 条目 | P2 从 Android `/vendor/firmware` 只读导出；先 `xxd`/`binwalk`/厂商 `goodix_ts` 核对，不猜 command id |
| 8 | HID ABI | `userland/touch.c` 按 20F66 原型，K60 OS 20G81；kernel `NativeHidInput` 是 17.7.11 | 首通用 userland bridge；kernel HID 只在 K60 20G81 ABI 核对后做 |
| 9 | ADT 与驱动断言漂移 | `make-pong-adt.py` 1080/2412 vs 补丁 14400/32000 | 构建前 `check-wiring.py`/dump 校验 12 值；不一致不 boot |
| 10 | pong 补丁与 K60 补丁重复 | 当前 series 已有 pong 三补丁 | 新 K60 patch 替换 `pong-touch-console` + `pong-goodix-resume`；保留 `pong-physical-keys` 独立 |

## 8. 验收清单

- [ ] `scripts/make-pong-adt.py` 的 touch 节点：`spi-base 0x990000`、`tlmm-base 0x0f000000`、`gcc-base 0x00100000`、`rsc-base 0x17a20000`、`cmd-db-base 0x80860000`、`gpio-offset 0x00100000`、`pins 16 17 18 19 21 20`、`max-x 14400`、`max-y 32000`、`avdd-gpio 71` 全部出现。
- [ ] `board.toml` 的 probe 镜像包含 `pong_touch=1`（或 `k60touch=1`）与 `k60touchstage=4 k60rpmhoff=1`，默认生产镜像暂不打开。
- [ ] P1 日志：`gcc vote ... bits 0x43c0`、`gcc halt ...` 三个 bit31=0、`se4 fw 0x102 protocol 1 (SPI=1)`、`id: berlin fw chip "911"`、`id: berlin touch report at 0x...`、`stage 4 done`。
- [ ] P2 日志：`ic info len=... misc_off=... touch=0x... cmd=0x...`，`touch != 0` 且 `misc+48/50 == 8`。
- [ ] P3 日志：`report status 0x80 count 1`、`touch finger 0 at ... -> ... px`、`touch down/move/up`。
- [ ] HID：`Pong touch HID: display-integrated digitizer opened`、`touch detection mode 0x1`、`type=1 finger=0`；SpringBoard 手指/多指/抬起正常。
- [ ] 全程 `0x17a00000`/`0x17a20000` 零读；没有 `iovdd vote` 行（或明确 `skipped`）。
- [ ] 失败可回滚：仅 `fastboot boot`，不写分区；保留上一版 boot.img。

## 9. 本轮改动边界

本文件只定义方案，不包含代码提交。实际实现时建议按以下顺序落 patch：

1. `docs(touch): plan the Goodix port from the marble/fuxi references`（本文件）。
2. `fix(touch): align the pong-touch ADT with K60 raw axes`——只改 `scripts/make-pong-adt.py:207-217`，把 `max-x/max-y` 改成 14400/32000，加 `avdd-gpio=71`。
3. `feat(touch): fork the Berlin-D protocol into a K60 board layer`——新增 `bsd/dev/k60_goodix.c`、`k60_input.c`、`k60_touch.c`、`bsd/sys/k60_touch.h`，替换 pong 三补丁中的触摸两件；保留 `pong-physical-keys`。
4. `feat(touch): add the optional goodix_firmware_m11a blob`——只在 P1 不能读到有效版本时启用。
5. `feat(touch): wire the 20G81 userland HID bridge`——如果 `libpong-ui.dylib` 还没有在 K60 root 中自动启动 `pong_touch_start()`，补 `userland/touch.c` 的启动接线。

前两步做完就可以上 P1；不要等固件加载和 kernel HID 都完成才做第一次只读 ID。

## 10. 2026-11-13：初次握手失败与源码差异

以当前 `gpuJ` 日志为准：`SE4 fw=0x102 protocol1 interfaces1`，FIFO不可用后
实际GPIO SPI握手读回00三次；不再是cand15的metadata缺失。Android实读
BERLIN/9916R、05800263；按名称找到pm8350c_l2已启用1800000uV（本轮regulator.27）。
debugfs确认RESET20输出高、IRQ21输入高、AVDD71输出高。`/dev/mem`不存在；
调试GPIO文本不暴露eGPIO ownership，不能由它声称XNU已控制pad。

[Cape头文件](https://github.com/NothingOSS/android_kernel_msm-5.10_nothing_sm8475/blob/sm8475/t/drivers/pinctrl/qcom/pinctrl-cape.h#L13-L55)
确认1MiB GPIO偏移正确；[msm_pinmux_set_mux](https://github.com/NothingOSS/android_kernel_msm-5.10_nothing_sm8475/blob/sm8475/t/drivers/pinctrl/qcom/pinctrl-msm.c#L241-L248)
在bit11 present时置bit12启用APPS。当前驱动补此条件操作并打印cfg前/后与io，
保留其他位；不清bit12、不再扫描未验证的引脚排列。

Linux v6.12 `goodix_berlin_core.c:247-279` 初次上电：IOVDD→3ms→AVDD→15ms→
释放RESET→4ms→八字节AA握手→100ms，再读固件。当前驱动此前先release+100ms，
随后才配置SPI；改为RESET保持低直到现有transport准备完成，再按原协议握手。
恢复reset仍保留low2/high100ms。该顺序来源证明时序差异，不证明9916R有硬4ms截止。
GPIO角色17/18/16/19仍未由成功握手证明，若仍失败必须看真实pad/供电读数。


Device-confirm仅在初次power-on执行一次；恢复reset之后直接读checksum有效的固件
与IC-info，不再在100ms之后重写ROM boot-option。GPU `gprobe`暂停（置0），
正常GPU渲染、拷贝与CPU回退不变。Host实际GPIO配置函数的四种present/enable
组合验证通过：present时置APPS位，保留未关联控制位。


Host检查：实际GPIO配置函数present/enable四组合与实际`bring_up`控制流夹具通过，
总线初始化时RESET仍低、4ms后握手、成功100ms后读ID，失败保留RESET低。
kernel/payload构建通过，`boot-touch-reset-20261113.img` sha256
`6883848d04357e21d8916555502d78c5505511f580fd5c115cd51a8c9fe0c765`，
源码`e6a63f50446c0616455fdbf32ba5dc827ec8cf63`，编译kernel`6d72dc8b`，
payload kernel`d48f594b`。从确认LDO2启用的Android热重启，只用fastboot boot。


真机`console-20261113-194925.log`（198068B）：初始化时序修正已运行，但八字节AA
仍读回00三次，用户确认触摸无效。GPIO20/21/71及16–19的原始cfg均从0x1开始，
bit11未置位，因此此板上eGPIO条件启用未改变ownership，不能当作此次故障根因。
AVDD71 IO=3仅证明GPIO电平，不证明IOVDD1.8V。下一步需要XNU中实际LDO2状态或
独立的SPI角色证据；不重复相同镜像、不把GPIO电平当供电测量。


## 11. 2026-11-13：SM8475引脚表纠正旧排列推断

[SM8475/SM8475P Data Sheet80-27620-1 Rev.AJ](https://www.scribd.com/document/995754119/80-27620-1-AJ-SM8475-SM8475P-Data-Sheet)
Table2-4 printed p44直接列出QUP[4]：GPIO16=L0/MISO，17=L1/MOSI，18=L2/SCLK，
19=L3/CS0；pin定义C30/E29/C31/D31分别对应这四个lane。
这是Qualcomm文档的第三方镜像，不是已认证官方下载；不依赖通用QRB5165文档
互相矛盾的MISO/MOSI表。[Xiaomi mondrian-s-oss pinctrl-cape.h](https://github.com/MiCode/Xiaomi_Kernel_OpenSource/blob/mondrian-s-oss/drivers/pinctrl/qcom/pinctrl-cape.h#L1641-L1648)
独立确认16–19的qup4 function slot1与正确寄存器布局。

旧元组17/18/16/19错误；00或ff只表示某次采样电平，不证明MISO或成功事务。
修改ADT元组为16/17/18/19，只重建payload并沿用`boot-touch-reset`的同一内核。
不改电源/频率/时序，不扫描排列；AA读回和checksum有效ID是下一轮判据。


Pin-trial image `boot-touch-pins-20261113.img` sha256
`ddfc759b3fd3d92fdeb7b544b82929f39239440730ab04ae702bfd7f05e9ff91`；
payload kernel仍`d48f594b`、编译kernel仍`6d72dc8b`，没有重编内核或改boot参数。
源码`1010e8b3939a14201e4bc01424861b0cee6e4245`，仅ADT触摸角色值改为16/17/18/19。


`console-20261113-200959.log`（152483B）确认16/17/18/19已实际运行，仍AA读回00
三次，未到chip ID。与前轮同一内核，说明单纯元组修正未闭合故障。
不能继续把GPIO电平推断为rail或总线应答。转向实际Android `goodix_core.ko`
`brl_power_on/brl_select_spi_mode`与固件PMIC接口，逐项核对9916R实际启动协议。


## 12. 2026-11-13：以实际Android模块纠正Berlin族启动协议

`goodix_core.ko`实际ELF反汇编：`goodix_get_ic_type`0x18fa4读compatible，
0x190b0/0x190c4分别检测`9916`/`spi`并到0x19284返回6。字符串重定位
`.rodata+0x66bb=9916`、`+0xc49d=spi`；真实`xiaomi,m11a-spi`因此走6/BerlinD。
`goodix_spi_probe`0x174c只为numeric4选择9-byte读；6走8-byte普通读，当前帧头正确。
Goodix公开枚举A=4/B=5/D=6：[pinned header](https://github.com/goodix/goodix_ts_berlin/blob/58cc7874cb5e13490a6c8720b308fa56847a1f51/goodix_ts_core.h#L65-L73)。

实际`brl_power_on`0x111d4非4直接跳到0x11660等待100ms，初次上电没有8×AA。
0xc900与0xd807/0xd806/0xd006/0xd808是BerlinA-only，不能用于9916R。
删除未匹配本机模块的AA强制门禁，释放RESET后100ms读28字节firmware和有界
IC-info；全0/ff、checksum错误和描述ABI错误仍拒绝。恢复reset仍2ms/100ms。
这是协议匹配，不是把dummy数据当成功，也不增加fallback。实际应答仍待上机。


Host实际`bring_up`控制流检查通过：transport初始化时RESET保持低，release后
100ms读真实identity；identity失败传播，无合成成功。kernel/payload构建成功；
`boot-touch-berlind-20261113.img` sha256
`f5aa2f8ff0fd59c387f4f2c2aa9b4b5c0f88d7e8c47586d418c696af151f8835`，
源码`bae9df2ed044ad85db71b19d802bd67c2057464f`，编译kernel`3d47ba01`，
payload kernel`d80ab402`，UEFI/DTB保持原样。实际应答仍以上机日志为准。


实际`console-20261113-203646.log`：GPIO16/17/18/19配置执行，RESET/AVDD路径
执行，普通8-byte读在100ms后到firmware0x10014；仍只有总线默认字节，
checksum-valid身份未出现，没有/dev/pong-touch。AA门禁已移除但不是全部根因。
UI继续启动且实际GPU scanout完成sequence1/2/3，不能据此声称触摸成功或帧率合格。
下一步只取实际GPIO/SPI/电源状态证据，不套用BerlinA或猜PMIC寄存器。


## 13. GPIO-only阶段7：已知输出pad状态取证

实际模块`spi_setup`前mode字段写0（0x1710），bits_per_word=8（0x1714）；
普通read在0x1da8–0x1dd8发F1/地址4/FF3。当前frame/mode无已证明差异。
stage7在RESET低时仅对已知CS19做101、SCLK18/MOSI17做010，恢复idle，
记录CTL及IO的out-latch(bit1)/sampled-pad(bit0)。不驱动MISO16/IRQ21。
原100ms启动等待不变，记录RESETrelease即时/100ms后状态和实际firmware字节。
不是芯片ACK、供电或continuity判据；该阶段不访问GCC/SE/RSC/PMIC，不发布input。

Android当前debugfs在`/dev/un_qslpi/traku_tu`，不是`/sys/kernel/debug`；
按name确认当前regulator.28=pm8350c_l2（旧27现在是l1），enabled/1800000uV，
regulator_summary consumer为spi0.0-vdd。cached cmd-db ldoc2=0x42c00 aux[01 2a 00 00]
只说明VRM资源42，不是SPMI peripheral base。固件/XNU状态仍未实测。


Host bounded-pad控制流检查通过：只写19/18/17，RESET保持低，CS在CLK/MOSI
检查时保持高，最后恢复CS高/CLK低/MOSI低。内核构建成功`f8683700`，
payload kernel`8a9c9f88`；实际FAT参数只有`k60_touch_se=7`，无残留6。
`boot-touch-padprobe-20261114.img` sha256
`5622eb59cc1eebfa82f570c89372648b8e262c06d7511250d2ddc235c2d754fc`，
源码`3cf75ef30dba317baa637291ed692b742a258603`，UEFI/DTB未改。

TOML evidence中历史JSON冒号已修为inline-TOML等号，文件解析通过。
`scripts/check-porting-paths.py --quiet`仍报5012未认领路径（包括XNU源码/headers），
不是触摸协议结果；没有为掩盖检查而增添宽泛路径认领。


实际`console-20261114-021326.log`205330B（源码3cf75ef，image5622eb59）：
CS19 IO=3/0/3，SCLK18/MOSI17 IO=0/3/0，证明这些pad的sampled输入跟随
已知输出latch；RESET20 release与100ms后IO=3。IRQ1/MISO0。
28-byte firmware实际全00；没有chip ID、IC-info、devfs或HID。用户确认仍无触摸。
这排除了本轮GPIOpad不翻转/RESET未释放；不证明板级continuity或IOVDD保留。
capture在310.85s读错误后回fastboot，后续ADB已恢复Android。


## 14. 只读SID2:C200 observer：测XNU中的真实数字状态

Android当前单外设读：C200header00030101048700、statusC208=87，C240/41和
C242/43均0807（1800mV），mode07、enable80。这是实际LDO，不是空总线；
raw APPS RW地址映射返回EPERM，不说明RPMh投票权限或具体owner。
QTI源码`qti-regmap-debugfs.c`507–511确认address/count只改debugmetadata，
只有写data才调用regmap_write；本轮没有data写且恢复metadata。

actualAOP sha`b3c2fc3e8255a9ceb673c8af0ff96b45dcf93e24d4b9906e8546c4b26aecf69f`、
config sha`0a3fbde21eb78060fdb2caf462ee88e4b63aed67d6ec67fde20cc5ee64c0f950`。
config CRC目录LDO_NUM(e7b5f30e)的PMICslot2 count13；LDO_RSRC(0990fd58)
slot2包含NULL,ldoc1,ldoc2…ldoc13。实际AOP80822010按firstbase/stride100索引；
资源consumer8081ccb2–ccb6确实用index+1取name。C200对应ldoc2仍标为推断，
不把Android读header等同于AOP启动时检测接受，也不推断UEFI/XNU电源保留。

复用physical-keys patch的observer，lazy初始化独立于按键线程，mutex串行化。
新增内部pong_spmi_find/read，按键迁移同backend；success外不输出data，保留
raw arbiter status和ENXIO/EINVAL/ETIMEDOUT/EACCES/EIO。observer只写读命令，
不写任何PMIC配置寄存器。stage7读取PPID2c2的header并核对type4/sub87后，
才读ready8/control40八字节；日志区分arbiterstatus和PMICstatus、数字电压和物理电压。


observer镜像已构建：编译kernel`e0efc97b`，payload kernel`df0c6d07`；
`boot-touch-supplyread-20261114.img` sha256
`da864ab948d6a2897b8bcc11d24cb6f29af3f6e0f0719c87271c2335d18a9f88`，
源码`ec82bc9f8ca5a5c1b24848c6d640bb2eeaef18dc`。UEFI/DTB保持原样，参数stage7。
身份/ready/control三次observer读以真实成功status为门，失败不输出合成供电值。


实际`console-20261114-032407.log`190597B：observer身份/ready/control均
err0 arb1，APID181。XNU中C200ready=0、enable=0，programmed1800mV但valid0、
mode7、status3；Android同外设ready1/enable1/valid1800。这是数字外设供电状态
未保留的实际证据，不再仅依据SPI00推测；电压ADC/板级continuity仍未测。
SPI仍28字节全00，用户确认无触摸。实体power键有真实down/up，up/down键
peripheral资格通过；本轮不能把driver45fps称为可见显示合格。


用户要求后续boot必须自动复位、明早检查成果：board默认k60_autoreset=300，
使用现有0334b独立内核线程和fastboot restart priming；不新增watchdog，
仍只fastboot boot。正常输入配置仍stage6，stage7是明确不发布input的取证路径。


## 15. 修复实测丢失的IOVDD：共享RPMh owner

当前编译provider确为device-ui/drivers/PongAdreno/PongAdrenoPlatform.cpp；
其run412–427实际direct RSC17a20000，GPU22阶段已实测成功；top securehelper
是死代码残留。触摸不得自行map旧17a00000或用被拒绝的SPMI RW。
使用同一TCS3 owner串行化GPU/触摸，保留现有idle/conflict/freshcompletion判据。
ldoc2 cmdDB VRM地址42c00、aux012a0000；两条owner请求：voltage1800mV、
enable1@42c04，不是PMIC EN_CTL80。mode/headroom/其它rail不改，保留已存在
1800..1980更高vote，失败不重用未确认inflight事务。ACK后observer实测
type4/sub87、ready/enable/valid1800..1980再释放RESET，最后验证chip/IC-info和HID。

actualAOPconfigCRC LDO_RAIL=2fbb5c4d→PMICslot2ptr8088031c（file3d0），
消费者8081d56c/570按36bytes/rail取Accessbit0；ldoc2零索引1=file3f4
recordword0=6101（Access1），min/max1800/1980，+1c owner mask4。
消费者8081d6a0/6a4将mask交0b00351c逐bit映射，不以SPMI写owner推断RPMh权限。
这是从本机AOP/config解出的目标值，不套旧PMICbase或他机mask。


已实现独立于GPU启动的pong_touch_iovdd_enable：shared mutex保护RSC/DB映射和
整个TCS3事务。匹配VRM0100/42c00/aux012a0000才发两条带响应的1800mV/enable1；
mode不变，未知retained值/occupied/conflict拒绝，未确认事务保持quarantine。
Goodix在RESET低时请求供电；ACK后核对observer ready/enable/valid再释放RESET。
不是保留电源假设，也不绕过chip checksum/config/HID资格门。
target mask翻译表0b0e0ed0实际为0,1,2…，owner mask4因此允许DRV2。

自动复位试验console-20261114-034402.log147952B：absolute300/idle0、fastboot
primed和armed均出现，317.65s USB断开，capture末态fastboot。expiry行未抓到，
因此定时原因由配置/时间及末态支持，不把USB断开单独当复位证明。

