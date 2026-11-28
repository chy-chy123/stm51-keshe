# 硬件引脚与模块接口冻结规格（HARDWARE_AND_INTERFACE_SPEC）

> **状态**：已冻结（Frozen） | **适用目标**：STC89C52RC @ 11.0592MHz（鹿小班-开天 / 普中 A2 开发板）  
> **课题名称**：基于单片机的智能温控调速风扇与环境监测系统设计

---

## 一、 硬件引脚冻结表

经实物丝印（`171ccd6fd3ba6a845ea2005e221b8f4d.jpg`）与 `A2开发板原理图.pdf` 双重核对，全系统外设引脚冻结如下（集中定义于 `firmware/src/config/pin_config.h`）：

| 外设模块 | 引脚信号 | 单片机 IO | 备注与硬件连接说明 |
| :--- | :--- | :--- | :--- |
| **LCD1602** | `LCD_RS` | `P2^6` | 寄存器选择 |
| **LCD1602** | `LCD_RW` | `P2^5` | 读写控制（**与蜂鸣器复用，写前强置 0**） |
| **LCD1602** | `LCD_EN` | `P2^7` | 使能脉冲（空闲保持 0） |
| **LCD1602** | `LCD_DATA` | `P0` | 8 位并行数据口（P0.0~P0.7） |
| **DS18B20** | `ONEWIRE_DQ` | `P3^7` | 单总线温度传感器（板载丝印 `DHT11/DS18B20 P37`） |
| **DS1302** | `DS1302_SCLK` | `P3^6` | 时钟同步脉冲 |
| **DS1302** | `DS1302_IO` | `P3^4` | 双向数据线 |
| **DS1302** | `DS1302_CE` | `P3^5` | 片选使能（复位线） |
| **130 直流电机** | `MOTOR_PWM` | `P1^0` | 接 ULN2003D 步进/直流驱动板载排针（`P10`） |
| **AT24C02 (EEPROM)** | `I2C_SCL` | `P2^1` | I2C 时钟线（板载丝印 `EEPROM SCL:P21`） |
| **AT24C02 (EEPROM)** | `I2C_SDA` | `P2^0` | I2C 数据线（板载丝印 `EEPROM SDA:P20`） |
| **无源蜂鸣器** | `BEEP_PIN` | `P2^5` | 经 ULN2003D 驱动（**与 LCD_RW 复用**） |
| **VS1838B 红外接收** | `IR_IN` | `P3^2` | 外部中断 0（`INT0` 下降沿触发） |
| **独立按键 K1** | `KEY_MODE` | `P3^1` | 模式切换（AUTO / MANUAL）— **与 UART TXD 复用** |
| **独立按键 K2** | `KEY_UP_GEAR` | `P3^0` | 手动加档 / 设置项数值加 — **与 UART RXD 复用** |
| **独立按键 K3** | **禁用** | `P3^2` | **禁止作为独立按键扫描，专供红外 INT0** |
| **独立按键 K4** | `KEY_SET_MUTE` | `P3^3` | 短按静音消警 / 长按或特定状态切入设置模式 |
| **UART 串口** | `RXD / TXD` | `P3^0 / P3^1` | 接 CH340G USB 转 TTL 下载/调试口 |

---

## 二、 三定时器黄金分配方案（11.0592 MHz）

STC89C52RC 拥有 Timer0、Timer1、Timer2 共 3 个定时器，按功能正交分配，互不抢占：

| 定时器 | 工作模式 | 核心用途 | 11.0592MHz 精确配置参数 | 中断向量 |
| :--- | :--- | :--- | :--- | :--- |
| **Timer0** | 16 位定时器（`TMOD` 低 4 位 `0x01`） | 红外 NEC 脉宽测量（供 `INT0` 读取计数） | 机器周期 $\frac{12}{11.0592\text{MHz}} \approx 1.085069\,\mu\text{s}$；不开中断 `ET0=0`，由 `INT0` 启停清零；判定阈值：Start `12442±500`，Repeat `10368±500`，Bit0 `1032±500`，Bit1 `2074±500` | 不开中断 |
| **Timer1** | 8 位自动重装（`TMOD` 高 4 位 `0x20`） | UART 串口波特率发生器 | **9600 bps**：`SCON=0x50`，`PCON&=0x7F`（`SMOD=0`），`TH1=0xFD`，`TL1=0xFD`，`ET1=0`，`TR1=1`，`ES=1`（误差 0.00%） | UART `interrupt 4` |
| **Timer2** | 16 位自动重装（`T2CON=0x00`） | **100us 系统基础节拍**：100 级电机 PWM（~100Hz）+ 非阻塞蜂鸣器 + 系统软定时器 | 重装值 `65536 - 92 = 65444 = 0xFFA4`（实测周期 $92 \times 1.085069\mu\text{s} \approx 99.83\,\mu\text{s}$）；`RCAP2H=0xFF`，`RCAP2L=0xA4`，`TH2=0xFF`，`TL2=0xA4`，`ET2=1`，`TR2=1` | `interrupt 5`（**必须手动清 `TF2=0`**） |

### 中断优先级配置
- **高优先级**：`PX0 = 1`（`INT0` 红外脉冲捕获不能被 100us 定时器延迟）
- **普通优先级**：`PT2 = 0`（Timer2）、`PS = 0`（UART）

---

## 三、 硬件冲突规避铁律

1. **严禁对 `P2` 进行整口赋值**：全工程禁止出现 `P2 = ...;`。所有操作必须通过 `sbit` 逐位完成，避免误改 `I2C_SCL`/`I2C_SDA`/`LCD_RS`/`LCD_EN`。
2. **`P2^5`（`LCD_RW` 与 `BEEP` 复用）互斥锁**：
   - 定义全局标志 `volatile bit g_LcdBusy`（在 `Timer2` 与 `LCD1602` 间共享）。
   - `LCD_WriteCommand()` 与 `LCD_WriteData()` 内部在拉高使能前必须：`g_LcdBusy = 1; LCD_RW = 0; ... LCD_EN = 0; g_LcdBusy = 0;`。
   - `Timer2` 中断中的蜂鸣器驱动仅在 `g_BuzzerActive && !g_LcdBusy` 时翻转 `BEEP_PIN`；不发声时保持 `BEEP_PIN = 0`（关断 ULN2003 驱动）。
3. **`P3^0/P3^1`（UART 与 `K1/K2` 复用）互斥**：
   - 定义全局标志 `volatile bit g_UartTxBusy`。
   - `UART_SendByte()` 发送前设 `g_UartTxBusy = 1`，发送完成 `TI=0` 后清 `0`。
   - 按键扫描函数检测到 `g_UartTxBusy == 1` 时跳过 `K1`/`K2` 读取（或仅读取 `K4`）。
4. **`P0` 总线独占**：
   - 本工程使用 LCD1602 作为主显示，**严禁启用动态数码管或 8×8 点阵**（不驱动 `74HC138`/`74HC595`），确保 `P0` 口纯粹归 LCD1602 使用。
5. **DS18B20 单总线时序保护**：
   - 11.0592MHz 机器周期与 12MHz 略有差异（快约 8%），`OneWire` 初始化（480~500us 低电平、60~70us 等待应答）和位读写（10us/50us）须按 11.0592MHz 重新校准空循环次数，并在关键微秒窗口内 `EA=0` 关中断、窗口结束立刻 `EA=1`。

---

## 四、 模块划分与 `.h` 接口契约

所有模块均置于 `firmware/src/` 下：

### 1. 配置层 `firmware/src/config/pin_config.h`
- 统一声明所有 `sbit` 引脚与系统常数（晶振 `11059200UL`、默认阈值 `DEFAULT_TEMP_LOW = 250`、`DEFAULT_TEMP_HIGH = 300`、`DEFAULT_TEMP_ALARM = 350`、PWM 档位占空比表 `0, 45, 75, 100`）。

### 2. 驱动层 `firmware/src/drivers/`
| 模块 | 核心导出接口 |
| :--- | :--- |
| `Delay.h` | `void Delay_ms(unsigned int ms);`（按 `@11.0592MHz` 校准） |
| `Timer2.h` | `void Timer2_Init(void);`、导出 `volatile bit g_Flag20ms;`、`volatile bit g_Flag500ms;`、`volatile bit g_Flag1s;`（由主循环消费并清零） |
| `LCD1602.h` | `void LCD_Init(void);`、`void LCD_ShowChar(u8 Line, u8 Col, char Char);`、`void LCD_ShowString(u8 Line, u8 Col, char *Str);`、`void LCD_ShowNum(u8 Line, u8 Col, u16 Num, u8 Len);`（含 `g_LcdBusy` 互斥） |
| `DS18B20.h` | `void DS18B20_ConvertT(void);`、`bit DS18B20_ReadTempX10(int *pTempX10);`（返回 `1` 成功、`0` 传感器未响应，绝不使用 `float`） |
| `DS1302.h` | `void DS1302_Init(void);`、`void DS1302_SetTime(void);`、`void DS1302_ReadTime(void);`、`extern unsigned char DS1302_Time[7];` |
| `AT24C02.h` | `void AT24C02_WriteByte(unsigned char WordAddr, unsigned char Data);`、`unsigned char AT24C02_ReadByte(unsigned char WordAddr);`（I2C 逻辑可独立 `I2C.c/h` 或内聚于驱动） |
| `IR.h` | `void IR_Init(void);`、`unsigned char IR_GetDataFlag(void);`、`unsigned char IR_GetRepeatFlag(void);`、`unsigned char IR_GetCommand(void);`（含遥控器键码宏定义） |
| `Motor.h` | `void Motor_Init(void);`、`void Motor_SetGear(unsigned char Gear);`、`unsigned char Motor_GetGear(void);`、`void Motor_PwmTickISR(void);`（供 Timer2 调用） |
| `Buzzer.h` | `void Buzzer_Init(void);`、`void Buzzer_BeepMs(unsigned int ms);`、`void Buzzer_SetAlarmState(bit Enable);`、`void Buzzer_TickISR(void);`（供 Timer2 非阻塞调用） |
| `Key.h` | `void Key_Tick20ms(void);`、`unsigned char Key_GetEvent(void);`（返回无按键 `0`、短按 `KEY_1_SHORT`/`KEY_2_SHORT`/`KEY_4_SHORT`、长按 `KEY_4_LONG`） |
| `UART.h` | `void UART_Init(void);`、`void UART_SendByte(unsigned char Byte);`、`void UART_SendString(const char *Str);`、`bit UART_FetchCommand(char *Buf, unsigned char MaxLen);` |

### 3. 应用层 `firmware/src/app/`
| 模块 | 职责与接口 |
| :--- | :--- |
| `App_Config.h` | EEPROM 结构体持久化（地址 `0x00~0x07`）：Magic(`0xA5`)、`TempLowX10`、`TempHighX10`、`TempAlarmX10`、校验和；`void AppConfig_Load(void);`、`void AppConfig_Save(void);`、阈值合法性约束：`150 <= TempLow < TempHigh < TempAlarm <= 600`（步进 `10`，即 1℃） |
| `App_Control.h` | 系统核心状态机：`SYS_MODE_AUTO` / `SYS_MODE_MANUAL` / `SYS_SET_TL` / `SYS_SET_TH` / `SYS_SET_TA`；统一处理按键、红外、串口三路控制事件；自动温控分档规则；温度转换防阻塞调度（先发 `ConvertT`，至少间隔 800ms 后读出温度）；超温报警与静音消警 |
| `App_Display.h` | `void AppDisplay_Init(void);`、`void AppDisplay_Refresh(void);`（局部字符刷新，禁止主循环频繁整屏清屏导致闪烁） |

---

## 五、 业务规则与三端交互契约冻结

### 1. 自动温控分档逻辑（`SYS_MODE_AUTO`）
设当前温度为 $T$（单位 `0.1C`），阈值为 $T_L$（默认 `250`=25.0C）、$T_H$（默认 `300`=30.0C）、$T_A$（默认 `350`=35.0C）：
- $T < T_L$：**0 档**（停转，PWM = 0%）
- $T_L \le T < \frac{T_L + T_H}{2}$：**1 档**（低速，PWM = 45%）
- $\frac{T_L + T_H}{2} \le T < T_H$：**2 档**（中速，PWM = 75%）
- $T \ge T_H$：**3 档**（高速，PWM = 100%）
- $T \ge T_A$：触发 **超温报警**（`AL:HI`，蜂鸣器 1Hz 间歇鸣响；若用户主动静音则显示 `AL:MU`；当温度回落至 $T < T_A - 10$ 即回差 1.0C 后自动解除报警）

### 2. LCD1602 显示布局（16 列 × 2 行，纯 ASCII）
- **运行模式（AUTO / MANUAL）**：
  - 第 1 行：`T:26.5C 12:30:45`（若 DS18B20 异常显示 `T:--.-C `）
  - 第 2 行：`M:AUT G:2 AL:OK `（模式：`M:AUT` / `M:MAN`；档位：`G:0`~`G:3`；报警：`AL:OK` / `AL:HI` / `AL:MU`）
- **阈值设置模式（SET）**：
  - 第 1 行：`SET L:25* H:30  `（当前光标选中项后面加 `*`，如 `L:25*`、`H:30*`）
  - 第 2 行：`ALARM:35*C [SAV]`（选中报警阈值时 `*` 跟在 `35` 后）

### 3. 红外遥控器按键映射（`IR.h` 键码）
| 红外按键 | 键码 | 功能行为 |
| :--- | :--- | :--- |
| `IR_POWER` | `0x45` | 风扇急停（切到 MANUAL 0 档） |
| `IR_MODE` | `0x46` | 切换 `AUTO` / `MANUAL` 模式 |
| `IR_MUTE` | `0x47` | 静音/解除当前超温蜂鸣器报警 |
| `IR_EQ` | `0x07` | 进入/轮换设置模式：`SET_TL -> SET_TH -> SET_TA -> 保存至 AT24C02 并返回` |
| `IR_VOL_ADD` | `0x09` | `MANUAL` 模式下档位 +1；`SET` 模式下当前阈值 +1℃ |
| `IR_VOL_MINUS` | `0x15` | `MANUAL` 模式下档位 -1；`SET` 模式下当前阈值 -1℃ |
| `IR_0`~`IR_3` | `0x16, 0x0C, 0x18, 0x5E` | 切入 `MANUAL` 模式并直设 0/1/2/3 档 |

### 4. 独立按键映射（`K3` 禁用）
| 按键 | 引脚 | 短按功能 | 长按功能（>= 1s） |
| :--- | :--- | :--- | :--- |
| `K1` | `P3^1` | 切换 `AUTO` / `MANUAL` 模式 | — |
| `K2` | `P3^0` | `MANUAL` 模式循环切档（0->1->2->3->0）；`SET` 模式当前阈值 +1℃ | — |
| `K4` | `P3^3` | 报警态消警静音；`SET` 模式切换下一阈值项（与 `IR_EQ` 等效） | 进入/保存退出阈值设置模式 `SET` |

### 5. UART 串口通信协议（9600 bps, 8N1, ASCII）
- **主动上报（每 2 秒 1 帧）**：
  `T=26.5,M=A,G=2,L=25,H=30,A=35,ALM=0\r\n`
  - `M=A`（AUTO）或 `M=M`（MANUAL）；`ALM=0`（正常）、`1`（超温报警）、`2`（已静音）
- **下行指令（以 `\r` 或 `\n` 结尾，最长 12 字节缓冲）**：
  - `M0`：切换自动模式（回复 `OK\r\n`）
  - `M1`：切换手动模式（回复 `OK\r\n`）
  - `G0` / `G1` / `G2` / `G3`：切手动模式并设置对应档位（回复 `OK\r\n`）
  - `Lxx` / `Hxx` / `Axx`（如 `L24`、`H31`、`A36`）：设置对应阈值并保存至 AT24C02，校验通过回复 `OK\r\n`，越界或倒挂回复 `ERR\r\n`
  - `Q`：立即上报一帧当前状态
  - 其他未知指令：回复 `ERR\r\n`
