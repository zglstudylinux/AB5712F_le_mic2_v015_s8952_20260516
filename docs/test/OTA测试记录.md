# OTA（USB HID 升级）测试记录

> 工程：`app/projects/microphone`（CONFIG_CLIP_LE_MIC，AB5712F）
> 原理与代码说明：`docs/peripheral/OTA/OTA原理与代码说明.md`
> 测试日期：2026-09-22。结论：**adapter 端 USB HID OTA 升级全链路验证通过 ✅**（T1 通道打通 / T2 复位加载 / T3 自编译固件交叉升级闭环）

## 1. 测试环境

| 项 | 内容 |
|---|---|
| 固件 | `ADAPTER_USB_UPDATE_EN=1`，reporter_id=0x03，4 参 `usb_device_hid_send_user(..., FOTA_DATA)` |
| 上位机 | `docs/peripheral/OTA/usb_tool1.1_20241205/usb_tool.exe` |
| 升级文件 | `Output/bin/rx_ota.fot`（0x51400 字节，原厂工具生成，MD5 前 4 字节 A3C247C0） |
| 连接 | USB 线（设备枚举为 Composite Device，VID 0x0C21/PID 0x0124），usb_tool 选 **Usage Page 0xff90（Reporter ID 0x03）** 的 "USB Mirophone" 条目 |
| 串口 | PB3，`PRINTF_PB3`（config_clip_mic.h），观察升级打印 |
| 注意 | Reporter ID **0x41**（0xff82 页）连接无效——发 0x91 后设备虽解析但不回 0x92，进度 0%。**必须用 0x03** |

## 2. 测试原理

设备固件内置 HID 升级服务器（`usb_device_update.c`）：上位机发 0x91（带文件大小+MD5 hash+升级对象）启动升级，之后设备按 512B/包用 0x92 拉数据（0x93 回），全部写入 flash（FOTA 双备份，库内 `fot_write` 校验）后回 0x94 成功状态；上位机发 0x95 触发设备 1.5 秒后软复位加载新固件。升级全程不需要按键/跳线，插 USB 即可。

## 3. 测试步骤与结果

### T1 升级主流程 ✅

1. 烧录含升级功能的固件，USB 连 PC，usb_tool 连 0xff90 条目。
2. 加载 rx_ota.fot，点【升级adapter端】。
3. usb_tool 显示进度条走完，日志输出 **"Adapter 升级成功！"**（收到 0x94 sta=0）。

串口实际打印（完整关键字序列）：

```
usb_dev_update_init adapter
file_size: 51400
file_hash: a3c247c0
adapter ota            ← 每收到一包0x93打一次（共约百余行，略）
-->addr: 0x51000, len: 0x200
adapter ota ...（最后一包）
-->addr: 0x51200, len: 0x200
-->fot update ok       ← 写入+校验通过，回0x94
```

`-->addr` 从 0 递增到 0x51200 共 0x51400 字节 = 文件大小，无缺口；`fot_get_err()==0`。

### T2 复位加载 ✅

usb_tool 点【重启设备】（发 0x95）：

```
usb_dev_update_reset
ADAPTER RESET          ← 1.5s 后软复位
Hello Platform / startup
SDK : v0150, LIBS: v0150
WDT reset
func_adapter ... 正常进入适配器模式
EVT_PC_INSERT          ← USB 重新枚举成功
```

设备复位后重新枚举、正常启动运行。

### T3 版本交叉升级闭环 ✅（2026-09-22）

目的：验证"PC 收到我们自编译固件转的 FOT → 升级后新固件真正生效"，即完整闭环。

1. [main.c:24](../../app/projects/microphone/main.c) 增加版本标识打印 `APP_VER: OTA-TEST-V2 (__DATE__ __TIME__)`，重新构建得 `app_ota_test_v2.dcf`（CODE SIZE 328KB）。
2. 原厂 FOT 打包工具将 dcf 转为 `OTA-TEST-V2.fot`（0x51400 字节，Hash B6A17E7C）。
3. usb_tool 连 0xff90（Report ID 0x03）升级 → 工具显示"Adapter 升级成功！"→ 点【重启设备】。

串口打印（复位后新固件生效）：

```
-->addr: 0x51200, len: 0x200
-->fot update ok
usb_dev_update_reset
ADAPTER RESET
Hello Platform / startup
Hello BT893X: 88010039
SDK : v0150, LIBS: v0150
APP_VER: OTA-TEST-V2 (Sep 22 2026 15:04:40)   ← 新固件标识出现，交叉升级闭环成功
WDT reset
func_adapter ... 正常运行
EVT_PC_INSERT
```

**结论：自编译固件 → FOT → USB HID OTA → 设备运行新固件，全链路验证通过。**

## 4. 判定标准（通过/失败）

| 检查点 | 通过标准 | 本次结果 |
|---|---|---|
| 0x91 应答 | 串口打印 `usb_dev_update_init adapter` + size/hash 与工具显示一致 | ✅ |
| 传输完整 | `-->addr` 最终值 + 0x200 == file_size，中途无 `USB_DEV_ERR`/`fot_get_err:N` | ✅ |
| 写入校验 | `-->fot update ok`，工具收到升级成功 | ✅ |
| 复位 | `ADAPTER RESET` 后正常启动、USB 重新枚举 | ✅ |

## 5. 遗留测试项（待做）

| # | 用例 | 状态 |
|---|---|---|
| L1 | ~~版本交叉验证~~ | ✅ 已完成，见 T3 |
| L2 | 篡改 FOT 文件后升级：确认失败行为（固件侧未见 hash 校验分支，预期靠库 CRC） | 未做 |
| L3 | 升级中途拔 USB 线：设备不死机、可重新升级（无断点续传，预期整包重来） | 未做 |
| L4 | 升级 device 端（UPDATE_OBJ=1）：需 `ADAPTER_AB_FOT_DEVICE_SUPPORT` 及成对环境 | 未做 |
| L5 | 升级后全功能回归（无线收发/蓝牙/按键/LED） | 未做 |
| L6 | ~~确认升级固件生效~~ | ✅ 已完成，见 T3（APP_VER 打印验证） |

## 6. 历史问题记录（供排查参考）

| 现象 | 根因 | 解决 |
|---|---|---|
| 进度条 0%，串口只见 `usb_dev_update_init` 无后续 | 固件用 Report ID 0x55 发 0x92，描述符中无此 ID，Windows 拒发 | reporter_id 改 0x03（usb_device_update.c） |
| `usb_device_hid_send_user` 3/4 参不匹配，第 4 参为栈垃圾 | 头文件声明少一个 `int type` 参数 | usb_device_update.h:19 补全，调用传 `FOTA_DATA` |
| usb_tool 设备列表多个条目选不对 | 复合设备 4 个 HID 接口（0x06/0x07/0x03/0x41） | 选 VID 0x0c21 且 Usage Page 0xff90 的条目 |
| 原厂首次给的 libplatform.a 无 HID 符号 | 发错库（HID 在 libdrivers.a）；后发的 libdrivers.a 与存量库逐字节相同（仅打包时间戳不同） | 无需替换，当前库已支持 |
