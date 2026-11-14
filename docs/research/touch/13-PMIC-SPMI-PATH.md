# 触摸供电：未验证的SPMI候选路径

> 历史方案，不是可直接执行的寄存器配方。实际PM8350C SID2已由Android设备树确认；LDO2 raw base、APID和XNU可访问性未确认。当前固件未派发PMIC协议producer。

## 1. 问题

FDT 明确：触摸 VDD = `pm8350c_l2`（PM8350C 的 LDO2，1.8 V），
由 `rpmh-regulator-ldoc2`（`qcom,rpmh-vrm-regulator`）管理。

**RPMh 稳压器只能通过 RPMh（RSC）写入**，而 `0x17a00000/0x17a20000`
首读整机死（PITFALLS P-49 / HARD-RULES HR-01）。

**未确认项**：XNU未测量LDO2保留状态；SPI全00/ff不能证明供电关闭。Android侧enabled=1、1.8V不能证明bootloader/UEFI/XNU中仍保持。

## 2. 合法替代：PMIC 仲裁器（SPMI）

FDT 提供独立的 SPMI 控制器：

```
qcom,spmi@c42d000 {
    compatible = "qcom,spmi-pmic-arb";
    reg = <0xc42d000 0x4000      /* cnfg   */
           0xc400000 0x3000      /* core   */
           0xc500000 0x400000    /* chnls  */
           0xc440000 0x80000     /* obsrvr */
           0xc4c0000 0x10000>;   /* intr   */
    reg-names = "cnfg\0core\0chnls\0obsrvr\0intr";
    qcom,ee = <0x00>;
    qcom,channel = <0x00>;
    cell-index = <0x00>;
};
```

它与RSC窗口地址独立，但地址独立不证明访问安全或通道归属；没有目标映射和状态证据前不执行SPMI事务。

## 3. 实现步骤

### 3.1 映射窗口（4 个，建议各 4 KiB）
```c
#define SPMI_CNFG_PHYS   0x0c42d000UL   /* 0x4000 */
#define SPMI_CHNLS_PHYS  0x0c500000UL   /* 0x400000，只需前 64 KiB */
#define SPMI_OBSRVR_PHYS 0x0c440000UL   /* 0x80000，只需前 4 KiB */
#define SPMI_CORE_PHYS   0x0c400000UL   /* 0x3000 */
```
注意 io_submap 预算（P-63）：每个窗口只映射实际用到的 4 KiB。

### 3.2 执行前缺失的目标证据

- Android实际`/sys/bus/spmi/devices/0-02/uevent`：PM8350C位于`qcom,pm8350c@2`，SID2；SID不等于UEFI logical PMIC index。
- 尚未得到本机PM8350C LDO2 peripheral base/APID；不能从其它PMIC的L2=0x4100推导PPID0x241。
- RPMh regulator的`ldoc2`命名和cached voltage不是SPMI物理地址。
- observer读同样需要发命令及验证channel status；不能把任意observer地址视为无副作用内存读取。
- 必须先得到目标来源、只读测量路径和错误语义，再考虑电源控制；不执行旧文中通用写命令配方。

**来源边界**：MiCode mondrian-s-oss目前可取得的FDT/cmd-db只确认RPMh资源；
尚无本机PM8350C LDO2 SPMI peripheral表。Android debugfs实际挂载点为
`/dev/un_qslpi/traku_tu`；regulator_summary/sysfs不能补出raw base。

## 4. 替代方案（更简单，但可能不可行）

Observer路径不是被动监听：它发起SPMI读命令，并需要目标PPID/APID、
权限和status/error证据。已工作的UEFI keypad observer只证明PON/GPIO路径，
不能据此读取尚未识别的LDO2。当前不执行任何猜测的PMIC事务。

## 5. 判据

| 步骤 | 成功标志 |
|---|---|
| 状态测量 | 目标channel、DONE/error语义和真实enable/voltage回读均有证据 |
| 电源判断 | GPIO/IRQ电平或cached vote不足以证明1.8V物理供电 |
| 触摸响应 | checksum-valid firmware和IC-info，随后实际down/move/up/HID |

## 6. 风险

- **写错 SPMI 通道/地址可能影响其它 PMIC 供电域**（如 CPU/内存/UFS），有整机死风险
- 建议**先只读**（obsrvr / 回读），确认安全后再写
- 严格遵守 HR-03：一次只加一个未知 MMIO 访问
