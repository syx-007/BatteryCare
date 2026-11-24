[README.md](https://github.com/user-attachments/files/35321508/README.md)
# 电池保护（BatteryCare）

设置充电上限。无需后台服务。

![界面](v1.1.1.png)

一个 14 KB 的单文件 PowerShell 工具：通过华硕驱动公开的 ACPI 接口，把充电上限写进
主板 EC。
开发初衷：现有方案大多依赖臃肿官方套件，本工具追求轻量化，支持多场景一键切换阈值。

## 用法

| 操作 | 命令 |
|---|---|
| 打开界面 | 双击 `BatteryCare.cmd` |
| **一键切换 80% ↔ 100%** | `BatteryCare.cmd -Toggle` |
| 设为 80% | `BatteryCare.cmd -Set 80` |
| 设为 60% | `BatteryCare.cmd -Set 60` |
| 查看当前上限和电量 | `BatteryCare.cmd -Get` |
| 关闭保护（允许充满） | `BatteryCare.cmd -Set 100` |
| 看 EC 原始返回（排查用） | `BatteryCare.cmd -Diag` |

界面为三个按钮，单击立即生效，没有确认框，当前生效的那项前面有 `●`：

- `60%（严格保护）` —— 最保守，长期插电办公用
- `80%（保护）` —— 推荐
- `100%（充满）` —— 关闭保护，恢复原样

所有操作会自动弹 UAC 提权。

## 原理

ASUS 在 `ASUS System Control Interface` 驱动里提供了 `\\.\ATKACPI` 设备，通过
`DeviceIoControl`（控制码 `0x0022240C`）调用 EC 的 `DEVS`/`DSTS` 方法，设备号
`0x00120057` 便是充电上限。本项目只实现了这一个设备号。

写入的是主板 EC 的寄存器，所以重启、关机、拔电源后依然有效；只有重刷 BIOS 或
EC 复位才会被还原。

## 本机实测情况（FX608JMR，天选6Pro）

| 项目 | 结果 |
|---|---|
| 接口存在 | 是，`AsusOptimization` 服务运行中 |
| 写入 | EC 返回 `1`（成功）；非管理员也能写，但工具默认仍提权 |
| 回读 | **不支持**：`DSTS(0x00120057)` 永远返回 `00 00 89 00`，与设定值无关 |
| MyASUS 同步 | 写入 `ChargingRate` 注册表（管理员时生效） |

因为回读不可靠，工具会把最近一次设定值记在
`%LOCALAPPDATA%\BatteryCare\state.json`，界面显示时标注"，本地记录"。


### 首次使用请做一次 30 秒验证

回读不可靠时，只能靠物理现象确认写入真的生效：

1. 插上电源，确认电量在 80% 以上（本例可设 60%）。
2. 运行 `BatteryCare.cmd -Set 60`。
3. 观察 Windows 电池图标：几十秒内应从"正在充电"变成"已接通电源，未在充电"或"全智能充电"。

- 变成"未在充电"→ 写入确实生效，工具可用。
- 仍在充电并冲到 100% → 该固件不接受这条写入（`-Set` 会显示成功但实际无效）。

## 安全性

该脚本和在 MyASUS 里使用"电池健康充电"功能一致，只是
界面不同，且优点显著：

- **绝对轻量级**：总文件大小14KB，界面清爽专注；
- **无需重复设置**：不用每次重启或插电都重设，上限写在主板 EC 里，重启、关机、睡眠、拔插电源都不会丢；
- **不刷 BIOS、不改固件**：只调用华硕驱动公开的 ACPI 接口；
- **不常驻后台**：没有服务、没有托盘、没有开机自启，跑一次即退出；
- **不联网**：全文没有任何下载或上传代码（可搜 `Invoke-WebRequest`、`http` 验证）；
- **可随时还原**：切回 100% 就是原始状态，文件夹里就是所有文件，可随时删掉文件夹，不留任何痕迹；
- **代码可审计**：单文件 13 KB，C# 部分 90 行，只碰 `0x00120057` 一个设备号；
- **和Armoury不冲突**：不存在其他软件运行后和奥创中心权限竞争产生的卸载后遗留问题。


需要留意的两点：

1. 它需要**管理员权限**（`\\.\ATKACPI` 的写权限），弹 UAC 是正常的；
2. 万一固件不认这条写命令，**不会报错**，所以要按上面做一次物理验证。这是唯一
   的真实风险，而且后果只是"以为设了其实没设"，不会损坏电池或主板。

## 兼容性

- 华硕 ROG / TUF / Vivobook 大多使用同一个设备号；非华硕机器请勿运行，
  `\\.\ATKACPI` 不存在时会直接报错退出，不会改任何东西。

## 注意

- 建议管理员运行（脚本会自动请求提权），非管理员时注册表同步会静默失败。
- 一键切换的方向来自本工具记录的当前值；如果你中途用其他工具修改过上限，
  先 `-Set 80` 或 `-Set 100` 明确设一次，让记录对齐。
- 与其他工具的充电设置会互相覆盖，建议只保留一个来源。
- 彻底移除：删掉这个文件夹即可，再顺手删掉 `%LOCALAPPDATA%\BatteryCare` 里的记录文件。

## 免责声明

本项目与 ASUSTeK 无关，使用的是华硕驱动公开的 ACPI 接口。仅在一台 ASUS TUF Gaming
FX608JMR 上实测有效，其它机型可能不支持（表现为 `-Set` 返回成功但电量继续充到 100%）。
使用前请先做上面那 30 秒验证；因使用本工具产生的任何后果由使用者自行承担。

## License

[MIT](LICENSE)

---

## English

A minimal single-file PowerShell tool that sets the battery charge limit on ASUS
laptops by calling the vendor's `\\.\ATKACPI` interface (device `0x00120057`).

```
BatteryCare.cmd            # GUI: 60% / 80% / 100%
BatteryCare.cmd -Set 80    # set limit
BatteryCare.cmd -Get       # show current limit
BatteryCare.cmd -Toggle    # switch 80% <-> 100%
```

Requires administrator rights. Tested on ASUS TUF Gaming FX608JMR (2025); the EC on
that model does not report the limit back, so the tool keeps the last value in
`%LOCALAPPDATA%\BatteryCare\state.json` for display only.
