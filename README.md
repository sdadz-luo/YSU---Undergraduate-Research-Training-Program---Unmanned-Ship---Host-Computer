# 无人船遥控上位机（Host Computer）

基于淘晶驰串口屏与 Renesas RA6M5 的无人船遥控上位机：采集串口屏触摸指令
与双摇杆姿态，经 LoRa 转发给船体。仓库内另含一版 STM32H750 串口屏显示
固件，以及屏端设计素材。

## 硬件平台

| 组件 | 型号 | 说明 |
| --- | --- | --- |
| 主控 MCU | Renesas RA6M5（R7FA6M5BF） | Cortex-M33，上位机主控 |
| 备用平台 | STM32H750VBT6 | 串口屏显示固件所用 MCU |
| 串口屏 | 淘晶驰 USART HMI | 触摸交互，发送组件指令 |
| LoRa 模块 | UART 接口 | 无线数传，连接船体 |
| 摇杆 ×2 | 模拟量摇杆 | 四通道 ADC 采集 |

## 项目结构

```text
Host computer/
├── RA/                       RA6M5 上位机工程（主力开发）
│   ├── src/
│   │   ├── hal_entry.c       主循环：帧解析 + ADC 扫描 + LoRa 分派
│   │   ├── debug_uart/       UART3 接收（DMAC）+ UART2 LoRa 发送
│   │   ├── debug_gpt/        GPT 定时器（2 ms，驱动 ADC）
│   │   ├── debug_adc/        4 通道 ADC + 死区判断 + 摇杆合成
│   │   └── debug_lora/       LoRa 协议封装（CRC8 + 打包）
│   ├── ra_gen/               FSP 生成代码（勿手改）
│   └── FSP_Project.uvprojx   Keil 工程
├── STM32/                    STM32H750 串口屏显示固件
│   └── MDK-ARM/uart_test.uvprojx
└── 上位机/usart_lcd/         屏端素材：HMI 工程、图片、字库
```

## 数据流

```text
串口屏触摸指令
    │ UART3 (DMAC)
    ▼
┌───────────────┐    帧检测      ┌──────────────────┐
│ UART3 环形缓冲 │ ──EE…FF──→   │ 解析船号/组件/数据 │
└───────────────┘                └────────┬─────────┘
                                          │
摇杆1 (X=AN04, Y=AN05)                    │
摇杆2 (X=AN10, Y=AN12)                    │
    │ ADC (2 ms)                          │
    ▼                                     ▼
┌───────────────┐   三态(0/1/2)    ┌──────────────┐
│   死区判断     │ ─────────────→  │  LoRa 打包    │
│ （半量程 30%） │                  │  （CRC8）     │
└───────────────┘                  └──────┬───────┘
                                          │ UART2
                                          ▼
                                   LoRa 模块 ──→ 船体
```

## 通信协议

### 屏包（串口屏 → 上位机 → LoRa）

```text
EE <船号> <组件> <数据> CRC8 FF     共 6 字节
```

船号：`0x01` 小白（船 1）、`0x02` 小黑（船 2）。

| 组件 | 值 | 数据 |
| --- | --- | --- |
| 灯 | 0x01 | 0 关 / 1 红 / 2 白 / 3 绿 / 4 紫 / 5 青 / 6 彩 |
| 水泵 | 0x02 | 0 关 / 1 开 |
| 云台上下 / 左右 | 0x03 / 0x04 | 占空比 |
| 机械臂舵机 / 占空比 | 0x05 / 0x06 | 舵机号 / 占空比 |
| 速度 | 0x07 | 0–10 |
| 控制锁 | 0x08 | — |

### 摇杆包（上位机 → LoRa）

```text
CC 01 <方向1> 02 <方向2> CRC8      共 6 字节
```

| 方向值 | 含义 |
| --- | --- |
| 0x00 | 停止 |
| 0x01 | 前进 |
| 0x02 | 后退 |
| 0x03 | 左转 |
| 0x04 | 右转 |

### CRC8

- 多项式 `0x31`（x⁸ + x⁵ + x⁴ + 1），初值 `0x00`
- 屏包校验范围 `EE 船号 组件 数据`（4 字节）
- 摇杆包校验范围 `CC 01 方向1 02 方向2`（5 字节）

## 软件模块

| 模块 | 主要接口 | 说明 |
| --- | --- | --- |
| `debug_uart` | `Uart_Init()` / `Uart3_GetData()` / `Uart2_SendData()` | UART3 用 DMAC 收进 256 字节环形缓冲；UART2 发送靠中断回调管理忙状态 |
| `debug_gpt` | `Gpt_Init()` / `Gpt_TimeoutCheck()` | 2 ms 周期中断，置位超时标志供主循环轮询 |
| `debug_adc` | `Adc_CalibrateCenter()` / `Adc_ProcessAll()` / `Adc_JoystickDir()` | 原始值 → 三态方向 → 摇杆方向（0–4）；死区为半量程 30% |
| `debug_lora` | `Lora_BuildScrPacket()` / `Lora_BuildJoyPacket()` | CRC8 计算与协议打包 |

主循环两件事：屏包到达即转发；摇杆方向变化即发摇杆包。

## 构建

1. 用 Keil MDK-ARM 打开 `RA/FSP_Project.uvprojx`
2. Build 并通过 J-Link 烧录

修改引脚 / 时钟 / 中断优先级 / DMA 通道需用 **RASC** 改
`RA/configuration.xml` 后重新生成代码。

## 引脚分配

| 引脚 | 功能 | 说明 |
| --- | --- | --- |
| P004 / P005 | AN04 / AN05 | 摇杆 1 X / Y |
| P010 / P014 | AN10 / AN12 | 摇杆 2 X / Y |
| P706 / P707 | SCI3 RXD / TXD | UART3，接串口屏 |
| P301 / P302 | SCI2 RXD / TXD | UART2，接 LoRa |

## 开发环境

- Keil MDK-ARM V5.43 + ARMClang V6.24
- Renesas FSP v6.4.0
- 屏端 UI 用**淘晶驰编辑器**修改 `上位机/usart_lcd/*.HMI`

## 相关仓库

同一无人船项目的其他工程：

- [Black_ship][black-ship] — 黑船固件（RA6M5）
- [White_ship][white-ship] — 白船工程

[black-ship]: https://github.com/sdadz-luo/YSU---Undergraduate-Research-Training-Program---Black_ship
[white-ship]: https://github.com/sdadz-luo/YSU---Undergraduate-Research-Training-Program---White_ship
