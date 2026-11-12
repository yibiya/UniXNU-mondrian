# UniXNU-mondrian

Redmi K60（代号 **mondrian**，高通 **SM8475**）的 UniXNU 设备支持仓库。

**当前目标：iOS 16.6.1（20G81）/ darwin22 上跑通完整硬件。**

> 本 README 最后更新：2026-11-09。历史路线（darwin24 / iPadOS 18.7.10）已归档到 `archive/darwin24-main-20261103` 分支，不再维护。

## 分支

| 分支 | 用途 |
|---|---|
| **`main`** | 主线（默认分支），所有可工作的阶段性成果都合到这里 |
| `archive/first-frame-20261107` | 历史里程碑：iOS 16.6.1 用户态首帧 |
| `archive/darwin24-main-20261103` | 已废弃的 darwin24（iPadOS 18）路线 |

提交规范见 `AGENTS.md` 铁律 11：**改动后及时 commit，阶段性成果及时 push**（push 用 Windows 侧 git）。

## 当前状态（2026-11-12）

| 项 | 状态 | 证据 |
|---|---|---|
| Boot chain（UEFI → UniXNU loader → XNU → launchd） | ✅ | 启动日志 |
| UFS / APFS 三窗（OS / SystemOS / Data） | ✅ | `PONG APFS: OS root binding PASS`、`K60DATA mounted ro=0 fstype=apfs` |
| 显示首帧（1440×3200） | ✅ | `K60 DISPLAY: first frame presented (swap 1)` |
| 背光（DCS 0x11/0x29/0x51） | ✅ | `K60 PANEL: bring-up complete` |
| 桌面 / SpringBoard | 设备通过 | cand7：用户看到状态栏、App Store、Settings、Dock；主循环 `turns=3889`、持续 BGRA swaps |
| launchd 稳定 | ✅ | 禁用 8 个 App-cryptex job 后不再 `initproc exited` |
| **实体键（电源/音量）** | 🔄 | 驱动已编译/链接/调用（`bsd_init.o` 有 `U _pong_buttons_init`），`configuration()` 结果不可观测 |
| **触摸** | 待设备验证 | 已补实际 DT 节点、GPIO71、raw14400×32000、受检 Berlin 报告；删除错误 Pong 供电/RSC 路径。FIFO 禁用需要真实 GPI 或已证实的 GPIO 线序 |
| GPU（Adreno730v3） | 待设备验证 | 启用实际归档；使用本机 GMU/SQE/ZAP 固件，APSS RSC 仅通过文档定义的 Qualcomm monitor I/O，不直接读挂机寄存器；须验证真实 CP/fence/Metal |
| WiFi / 蓝牙 / 音频 / 相机 / 传感器 / 电源 | 📋 | 八步调研已完成，实现进行中 |

详细状态与缺口矩阵见 [`E:\k60-work\docs\PORTING-PIPELINE.md`](file:///E:/k60-work/docs/PORTING-PIPELINE.md)。

## 结构

| 路径 | 内容 |
|---|---|
| `board.toml` | `schema_version = 2` 设备配置与组件选择 |
| `porting.toml` | 14 个子系统的输入、发现、证据与完成判据 |
| `feeds.toml` | 固定的 SoC 源 |
| `configs/mondrian_defconfig` | 设备内核设置 |
| `dts/mondrian.dts` | 内存、CPU、UFS 硬件；`board.toml [adt.arm-io]` 提供板级触摸元数据 |
| `patches/xnu/` | 设备补丁（16.6.1 系列，见 `patches/xnu/series`） |
| `scripts/` | 设备构建与检查工具 |
| `drivers/` | PongAdreno / NativeUfs / NativeRamBlock / ArmCpu / ArmDTPlatform / PongMetal |

## 四层仓库

| 层 | 仓库 | 内容 |
|---|---|---|
| 框架 | [UniXNU/UniXNU](https://github.com/UniXNU/UniXNU)（`main`） | `unixnu` 命令、共享脚本、profiles、组件目录 |
| SoC | UniXNU-sm8475-common | `soc.toml`、`configs/sm8475.config`、`dts/sm8475.dtsi` |
| 设备 | 本仓库 | `board.toml`、`dts/mondrian.dts`、`patches/xnu/` |
| 固件 | uefi-mondrian | EDK2 / Mu-Silicium 构建树 |

## 构建

```bash
# 内核（Linux/WSL）
tr -d '\r' < E:/k60-work/scratch/buildv15.sh > /tmp/b.sh && bash /tmp/b.sh
# 产物
#   /opt/k60/trees/k60build/out-baseline/mondrian/ipados-16.6.1-20G81/mondrian/kernel/kernel.macho
# 打包 boot 镜像
tr -d '\r' < E:/k60-work/scratch/packv15.sh > /tmp/p.sh && bash /tmp/p.sh
#   → E:/k60-work/usb-a-stage/out/boot-v15-20261108.img
```

UI 兼容库部署前必须同步源码和 `0340` 内嵌数组，不能只修改 `ui_compat.c`：

```bash
# Linux/WSL：使用真实 iOS SDK；不使用声明 shim 或 fake codesign
python3 userland/build.py ui --sdk /path/to/iPhoneOS.sdk --out ../work/ui
# macOS：使用 Xcode 的 iPhoneOS SDK
python3 userland/build.py ui --developer /Applications/Xcode.app/Contents/Developer --out ../work/ui
```

此命令编译 `ui_compat.c`、`icons_cpu.c`、`touch.c`，校验 4 KiB 签名页，更新
`patches/xnu/0340-k60-uilib-deploy.patch` 的数组与 hunk 长度，并保留部署调用点。
提交补丁后重建内核。最终 FAT 中的内核必须包含这份库，并经过
`scripts/patch-armcpu-validate.py --apply`；不能用未处理的 prelink 内核替换 payload。

参考 [Pong iOS16 的真实用户态构建](https://github.com/Chengdong-Yu/UniXNU-pong/blob/7798f1e5f261e7121d01d7e4807ed73512737ded/userland/build.py)
保持 20G81 ABI；[HakiOS 的 main-queue 初始化经验](https://github.com/oosexgang/hakios-xnu-kernel/blob/main/userland/src/inject_sb.c#L2249-L2262)
说明依赖 UIApplication/RunLoop 的调用应等主循环可运行后执行，不照搬其 20F66 私有地址。

选中 job 的缓存也必须保留 `PublishesEvents`。20G81 notifyd 只有收到 initial
publisher barrier 后才连接 Mach listeners；缺少字段会触发 `Event publisher error`。
恢复实际 prepared descriptors 的字段（保留其他缓存策略与全部选中 jobs）：

```bash
python3 userland/build.py launchd_cache --jobs /path/to/pong/LaunchDaemons --out ../work/cache
```

触摸首轮沿用桌面 cand7 的固件、内存与 root/Data，仅增加 canonical
`/arm-io/pong-touch`。`k60_touch_se=5 k60_touch_rcg=1` 只读实际 ID；默认
stage6 才发布 `/dev/pong-touch` 并把校验正确的触点交给 backboardd HID。
`bitbang-pins` 必须来自独立确认的 MISO/MOSI/CLK/CS 线序，不能扫描排列猜。
ARM64 的启动参数容量是608字节，新增阶段参数前仅移除诊断用
`applog=1 applog_name=SpringBoard`；不改 Data UUID、内嵌 PID1 或桌面兼容策略。

GPU 归档从实际 kernel tree 构建，固件目录默认 `.cache/firmware/k60-gpu`，
可用 `PONG_ADRENO_FIRMWARE_DIR` 指定本机提取目录。`pong_adreno_start=1`
是独立硬件试验，`pong_touch=0` 隔离触摸；只有真实 GMU/HFI、PAS13/ZAP、
CP timestamp fence 全部通过后才能打开 Metal/scanout。monitor 拒绝访问是
真实失败，不能当作“已上电”继续，也不代表 GPU 加速通过。

**路径铁则（重要）**：构建读取的是 `/tmp/usb-cdc-mainline/device`（= `/opt/k60/trees/usb-cdc-mainline/device`，同一 inode）。
`/mnt/e/hackios/UniXNU-mondrian` 是**另一份 git 副本**。改补丁前必须 `ls -li` 对比 inode、`sha256sum` 比对两侧一致后才可构建。
详见 `AGENTS.md` 铁律 12。

## 作业纪律

1. **八步流水线**（铁则 10）：同型研究 → 官方文档 → 反编译+真机数据 → 广域验证 → 方案 → 深挖 → 核验 → 真机。**缺一步不得上机**。
2. **调研驱动**（铁则 5）：任何上机/改代码前必须有已落盘的调研文档。
3. **坑必记录**（铁则 1）：`E:\k60-work\docs\PITFALLS.md`（P-NN）+ `PITFALLS-FROM-HISTORY.md`。
4. **先查 Pong 仓库**（铁则 8）：`Chengdong-Yu/UniXNU-pong` 的 `main`/`ios16` 分支与 `drivers/*` 子模块。
5. **路径核对**（铁则 12）：见上。

## 相关文档

- 项目总闸：`D:\Android\AGENTS.md`（铁律 1–12）
- 文档入口：`E:\k60-work\docs\INDEX.md`
- 现状页：`E:\k60-work\docs\HANDOFF.md`
- 八步矩阵：`E:\k60-work\docs\PORTING-PIPELINE.md`
- 坑台账：`E:\k60-work\docs\PITFALLS.md`

## 数据安全

- bring-up 全程只用 `fastboot boot`（RAM 临时引导），**不 flash 任何分区**。
- XNU 的 root/Data 使用独立 `unixnu` 分区：SHNGROOT 里的 OS/SystemOS 窗口 + 尾部 Data 窗口，不写安卓 `userdata`。
- **绝对禁止** `fastboot flash persist` / `flash all`（TEE 密钥永久损坏）。

## 许可

设备级代码采用 BSD-2-Clause-Patent。导入组件保留各自 SPDX 标记与原始许可证。
