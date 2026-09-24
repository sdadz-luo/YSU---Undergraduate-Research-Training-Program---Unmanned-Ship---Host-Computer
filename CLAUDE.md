# CLAUDE.md — Host Computer（无人船遥控上位机）

供 Claude Code 使用的项目说明。面向人的介绍见 `README.md`。

## 项目身份

本仓库是**三合一**目录，三部分互不相关，改动前先确认目标：

| 子目录 | 内容 | 状态 |
| --- | --- | --- |
| `RA/` | RA6M5 上位机主控工程 | 主力开发中 |
| `STM32/` | STM32H750 串口屏显示固件 | 可用 |
| `上位机/` | 淘晶驰串口屏素材（HMI 工程、图片、字库） | 设计资源，非代码 |

RA 工程参数：

- **MCU**：Renesas RA6M5（R7FA6M5BF），Cortex-M33
- **FSP**：v6.4.0，经 RASC 生成
- **工具链**：Keil MDK-ARM V5.43 + ARMClang V6.24，C11
- **架构**：裸机超级循环，无 RTOS
- **TrustZone**：已启用安全分区

## 构建

| 目标 | 工程文件 | 说明 |
| --- | --- | --- |
| RA6M5 上位机 | `RA/FSP_Project.uvprojx` | Keil 打开后 Build |
| STM32H750 屏显 | `STM32/MDK-ARM/uart_test.uvprojx` | 同上 |

无 Makefile / CMake / CI。改外设须走 RASC：编辑 `RA/configuration.xml`
→ 生成 → FSP 重写 `RA/ra_gen/` 与 `RA/ra_cfg/`。

## RA 工程结构

```text
RA/src/
├── hal_entry.c          主循环：屏帧解析 + ADC 扫描 + LoRa 分派
├── hal_warmstart.c
├── debug_uart/          UART3 接收（DMAC + 环形缓冲）+ UART2 LoRa 发送
├── debug_gpt/           GPT 定时器，2 ms 周期驱动 ADC 扫描
├── debug_adc/           4 通道 ADC + 死区判断 + 摇杆方向合成
└── debug_lora/          LoRa 协议封装（CRC8 + 打包）
```

`ra_gen/`、`ra_cfg/`、`ra/` 由 RASC / FSP 生成，**不要手改**。

## 通信协议

### 屏包（串口屏 → 上位机 → LoRa）

```text
EE <船号> <组件> <数据> CRC8 FF        共 6 字节，CRC 覆盖 EE~数据 4 字节
```

船号：`0x01` = 小白（船 1），`0x02` = 小黑（船 2）。

| 组件 | 值 | 数据含义 |
| --- | --- | --- |
| `COMP_NONE` | 0x00 | 无 |
| `COMP_LIGHT` | 0x01 | 灯光模式 |
| `COMP_PUMP` | 0x02 | 水泵开关 |
| `COMP_PTZ_UP_DOWN` | 0x03 | 云台俯仰 |
| `COMP_PTZ_LEFT_RIGHT` | 0x04 | 云台转向 |
| `COMP_ARM_SERVO` | 0x05 | 机械臂舵机 |
| `COMP_ARM_DUTY` | 0x06 | 机械臂占空比 |
| `COMP_SPEED` | 0x07 | 速度 0–10 |
| `COMP_NEW_CMD` | 0x08 | 控制锁 |

### 摇杆包（上位机 → LoRa）

```text
CC 01 <方向1> 02 <方向2> CRC8        共 6 字节，CRC 覆盖 CC~方向2 5 字节
```

方向值：0 停止 / 1 前进 / 2 后退 / 3 左转 / 4 右转。
CRC8 多项式 `0x31`，初值 `0x00`。常量定义在 `RA/src/debug_lora/lora_proto.h`。

## ADC 与摇杆

- 4 通道：AN04(P004)、AN05(P005)、AN10(P010)、AN12(P014)，12 位
- 摇杆 1 = X(AN04)/Y(AN05)，摇杆 2 = X(AN10)/Y(AN12)
- 死区为半量程的 30%（`ADC_DEAD_ZONE_HALF = 614`，对称分布）
- 上电后 `Adc_CalibrateCenter(8)` 采样 8 次取平均作为中点阈值
- 双轴同时越界时，比较偏离量决定优先方向
- ADC 扫描周期 2 ms（由 GPT0 驱动）

## 引脚分配

| 引脚 | 功能 | 说明 |
| --- | --- | --- |
| P004 / P005 | AN04 / AN05 | 摇杆 1 X / Y |
| P010 / P014 | AN10 / AN12 | 摇杆 2 X / Y |
| P706 / P707 | SCI3 RXD / TXD | UART3，接串口屏（DMAC 接收） |
| P301 / P302 | SCI2 RXD / TXD | UART2，接 LoRa 模块 |

## 已知陷阱

| 现象 | 原因 | 解法 |
| --- | --- | --- |
| 定时器周期误解 | `gpt.c` 与 `hal_entry.c` 注释写「10 ms」，但 FSP 实际配置是 **2 ms** | 以 `ra_gen/hal_data.c` 的 `period_counts` 为准 |
| UART2 发送冲突 | LoRa 发送是阻塞等待，重入会丢包 | 用 `Uart2_IsIdle()` 判断，避免在回调里发送 |
| IntelliSense 报错 | ARMClang 专有关键字无法识别 | 在 `c_cpp_properties.json` 的 `defines` 中手动补充 |
| .gitignore 不生效 | `ra_gen/`、`ra_cfg/`、`RTE/` 等规则只对**未跟踪**文件有效，这些目录早已被跟踪 | 需要真正排除时先 `git rm -r --cached` |

## 源码编码

RA 与 STM32 的源文件均为 **GBK**。编辑时必须用能保持原编码的工具——
按 UTF-8 读写会把中文注释变成替换字符。

## 约定

- 注释与交流用中文，标识符用英文
- 模块目录用 `debug_` 前缀；函数名 `PascalCase_WithUnderscore`（如
  `Uart_Init()`），回调函数用 `snake_case`（如 `adc_callback()`）
- 宏用 `UPPER_CASE` + 模块前缀（如 `UART3_RX_BUF_SIZE`）
- 头文件保护宏为 `_MODULE_NAME_H_`

## 测试

无自动化测试。验证方式：Keil 编译 + 烧录实测。
