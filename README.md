# 基于单片机的智能温控调速风扇与环境监测系统设计

本项目为单片机课程设计工程仓库，基于 **鹿小班-开天 51 开发板（兼容普中 A2 硬件布局）** 与 **STC89C52RC** 单片机（`@ 11.0592 MHz` 晶振），采用模块化 C51 架构实现智能温控调速风扇与环境监测系统。

## 硬件平台与核心参数

- **主控芯片**：STC89C52RC（兼容标准 8051 / `REGX52.H`，8KB Flash ROM / 512B RAM）
- **系统晶振**：`11.0592 MHz`
- **串口通信**：CH340G USB 转 TTL，`9600 bps`（8N1，Timer1 Mode 2，0% 误差）
- **核心外设**：
  - **DS18B20**：高精度单总线温度采集（0.1℃ 定点数处理）
  - **DS1302**：实时时钟显示（时:分:秒）
  - **LCD1602**：16×2 字符液晶实时显示温度、时间、工作模式、档位与报警状态
  - **130 直流电机 + 扇叶（ULN2003D 驱动）**：Timer2 100Hz 100 级软件 PWM 调速（0/1/2/3 档）
  - **VS1838B + 红外遥控器**：NEC 协议红外遥控（急停、模式切换、静音消警、阈值设置、直选档位）
  - **独立按键**：辅助本地模式切换、档位/阈值调节与设置确认
  - **AT24C02 (I2C EEPROM)**：掉电保存温度阈值（低温启动、高温全速、超温报警）
  - **蜂鸣器**：超温非阻塞间歇声光/蜂鸣报警，支持手动静音消警

## 仓库目录结构

```text
单片机课设/
├── 171ccd6fd3ba6a845ea2005e221b8f4d.jpg   # 开发板实物主板图
├── fc67236887bb00b45c6634b850ee92a8.jpg   # 套件外设实物图（LCD1602/马达/遥控器等）
├── A2开发板原理图.pdf                      # 开发板完整硬件原理图
├── README.md                              # 项目总览与编译说明
├── AGENTS.md                              # 多 Agent 协同开发军规与编码红线
├── WORKSPACE_MAP.md                       # 工作区目录地图与文件归属
├── docs/
│   ├── HARDWARE_AND_INTERFACE_SPEC.md     # 引脚、定时器、硬件冲突规避与接口冻结规格
│   └── work_orders/                       # 司令部技术规格工单
└── firmware/                              # 新建模块化 Keil C51 工程与源码（SmartFan.uvproj）
```

> 注：本地参考例程（`程序源码/`）、根目录旧示例（`DS1302.*`、`LCD1602.*`）、结业论文模板与 `paper/`（LaTeX 结业论文）、`backups/`、`archive/` 以及 Keil 编译中间产物均已写入 `.gitignore`，仅在本地保留。

## Keil 编译与烧录说明

1. 使用 **Keil uVision5 (C51)** 打开 `firmware/SmartFan.uvproj`（或命令行构建）：
   ```powershell
   D:\Keil_v5\UV4\UV4.exe -b firmware\SmartFan.uvproj -o build.log
   ```
2. 编译输出固件为 `firmware/Objects/SmartFan.hex`（`SmartFan.hex` 纳入版本控制，编译中间文件忽略）。
3. 使用 **STC-ISP** 烧录时：
   - 芯片型号选择 `STC89C52RC / LE52RC`
   - 晶振频率按硬件选择 `11.0592 MHz`
   - 串口调试助手波特率设为 `9600 bps`（8N1）
