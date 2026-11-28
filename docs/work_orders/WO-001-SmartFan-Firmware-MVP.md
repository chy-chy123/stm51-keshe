# 司令部技术规格工单：WO-001

- **工单编号**：`WO-001-SmartFan-Firmware-MVP`
- **项目阶段**：**未发布原型开发阶段**（禁止形式主义空壳防御代码、禁止私改接口契约、禁止编写历史兼容层）
- **流水线定位**：
  - 本工单交付 **AWS Quick（全程智能档）** 搭建 Keil C51 工程与全套模块化初版固件，**或**直接交由 **Codex (GPT)** 编写/复核整改；
  - 完成后统一交回 **朱鸢兄（Antigravity）** 执行 Keil uVision5 命令行真编译与逐行 Code Review 验收。
- **依据规范**：严格遵循 [AGENTS.md](../../AGENTS.md) 与已冻结的 [HARDWARE_AND_INTERFACE_SPEC.md](../HARDWARE_AND_INTERFACE_SPEC.md)。

---

## 一、 目标与目录领地边界

### 1. 本工单要交付什么
在 `firmware/` 目录下构建完整的 Keil uVision5 C51 工程 `SmartFan.uvproj` 及其全部模块化源代码，实现《基于单片机的智能温控调速风扇与环境监测系统设计》全功能 MVP，并成功编译输出 `firmware/Objects/SmartFan.hex`。

### 2. 文件读写权限边界
- **仅允许写入以下路径**：
  - `firmware/SmartFan.uvproj`
  - `firmware/SmartFan.uvopt`（可选，若用标准模板生成或 Keil 编译自动生成）
  - `firmware/src/main.c`
  - `firmware/src/config/pin_config.h`
  - `firmware/src/drivers/` 下全部 `.c/.h`
  - `firmware/src/app/` 下全部 `.c/.h`
- **严禁触碰**：
  - `/程序源码/`（仅可只读参考其中 `11.0592MHz` 相关延时与协议实现）
  - 根目录 `DS1302.c/h`、`LCD1602.c/h`、`171ccd6...jpg`、`fc67236...jpg`、`A2开发板原理图.pdf`
  - `.gitignore`、`AGENTS.md`、`WORKSPACE_MAP.md`、`README.md`、`docs/HARDWARE_AND_INTERFACE_SPEC.md`、`paper/`

---

## 二、 Keil 工程构建规格（`firmware/SmartFan.uvproj`）

可参考 `/程序源码/10-1 DS1302时钟/Project.uvproj` 的标准 MCS-51 XML 格式编写 `firmware/SmartFan.uvproj`，核心字段必须准确无误：

1. **Target / Device**：
   - `TargetName`：`SmartFan`
   - `Device`：`AT89C52`（完全兼容 STC89C52RC，`RegisterFile` 为 `REGX52.H`）
   - `Cpu`：`IRAM(0-0xFF) IROM(0-0x1FFF) CLOCK(11059200)`
   - `OutputDirectory`：`.\Objects\`
   - `OutputName`：`SmartFan`
   - `CreateHexFile`：`1`
   - `ListingPath`：`.\Listings\`
2. **C51 编译选项（`<C51>`）**：
   - `IncludePath`：`.\src\config;.\src\drivers;.\src\app`（让各模块可直接 `#include "pin_config.h"`、`#include "LCD1602.h"` 等）
   - 优化等级与内存模式：默认 `Small` 模式（严格控制 `DATA` 变量体积，长字符串常量尽量使用 `code` 修饰符存入 Flash ROM，避免占用宝贵 DATA RAM！）
3. **文件分组（`<Groups>`）**：
   - `Config`：`./src/config/pin_config.h`
   - `Drivers`：`Delay.c`, `Timer2.c`, `LCD1602.c`, `DS18B20.c`, `DS1302.c`, `AT24C02.c`, `IR.c`, `Motor.c`, `Buzzer.c`, `Key.c`, `UART.c`
   - `App`：`App_Config.c`, `App_Control.c`, `App_Display.c`
   - `Core`：`./src/main.c`

---

## 三、 逐模块极细粒度开发清单与防坑规格

### 1. `firmware/src/config/pin_config.h`
- `#include <REGX52.H>`
- 定义晶振宏与系统参数：
  - `#define SYS_FOSC 11059200UL`
  - `#define TEMP_LOW_DEFAULT   250`（25.0C）
  - `#define TEMP_HIGH_DEFAULT  300`（30.0C）
  - `#define TEMP_ALARM_DEFAULT 350`（35.0C）
  - `#define TEMP_MIN_LIMIT     150`
  - `#define TEMP_MAX_LIMIT     600`
  - `#define TEMP_HYSTERESIS    10`（1.0C 报警回差）
- 按 `HARDWARE_AND_INTERFACE_SPEC.md` 第一节一字不差定义全部 `sbit` 与 `LCD_DATA` (`P0`)。

### 2. 驱动层 `firmware/src/drivers/`（11 个独立模块）

1. **`Delay.c / Delay.h`**：
   - `void Delay_ms(unsigned int ms);` 按 **11.0592MHz** 精确校准 1ms（机器周期 $1.085\,\mu\text{s}$，内部循环次数按 STC-ISP `@11.0592MHz` 校准：`i=2; j=199;` 约 1ms）。
   - 不使用任何浮点运算。
2. **`Timer2.c / Timer2.h`**：
   - 配置 Timer2 为 16 位自动重装模式：`T2CON = 0x00; RCAP2H = 0xFF; RCAP2L = 0xA4; TH2 = 0xFF; TL2 = 0xA4; ET2 = 1; PT2 = 0; TR2 = 1; EA = 1;`。
   - 导出并清零软定时标志：`volatile bit g_Flag20ms;`, `volatile bit g_Flag500ms;`, `volatile bit g_Flag1s;`。
   - `void Timer2_Routine(void) interrupt 5` 入口**第一行必须写 `TF2 = 0;`**！
   - 中断内按计数分频：每次调用 `Motor_PwmTickISR()` 与 `Buzzer_TickISR()`；每 200 次（20ms）置 `g_Flag20ms = 1`；每 5000 次（500ms）置 `g_Flag500ms = 1`；每 10000 次（1s）置 `g_Flag1s = 1`。
3. **`LCD1602.c / LCD1602.h`**：
   - 定义并导出互斥标志：`volatile bit g_LcdBusy = 0;`。
   - 写命令与写数据内部顺序：
     ```c
     g_LcdBusy = 1;
     LCD_RS = 0; /* 或 1 */
     LCD_RW = 0;
     LCD_DATA = val;
     LCD_EN = 1;
     LCD_Delay1ms();
     LCD_EN = 0;
     LCD_Delay1ms();
     g_LcdBusy = 0;
     ```
   - 提供 `LCD_Init`、`LCD_ShowChar`、`LCD_ShowString`、`LCD_ShowNum`（移除不用的 `LCD_ShowSignedNum`/`LCD_ShowHexNum`/`LCD_ShowBinNum` 或保持精简以节省 8KB ROM）。
4. **`DS18B20.c / DS18B20.h`**（可直接合并单总线逻辑或附带内部 static 辅助函数，对外只暴露 `DS18B20.h`）：
   - **严禁使用 `float`**！
   - 提供：`void DS18B20_ConvertT(void);`、`bit DS18B20_ReadTempX10(int *pTempX10);`
   - 转换算法（定点整数，0.1C 精度）：
     - `raw = (int)((TMSB << 8) | TLSB);`
     - 若存在复位无应答（`Ack != 0`），返回 `0` 且不改 `*pTempX10`。
     - 成功时：`*pTempX10 = (raw * 10) / 16;`（若 `raw < 0` 也需正确取整或保持符号；对正温 `(raw * 10L + 8) >> 4` 或 `(int)(((long)raw * 10) / 16)`，注意 16 位 `raw * 10` 在室温下最大 `1600 * 10 = 16000 < 32767`，当 `T > 204.7C` 才会超 `int`，为稳妥可用 `long` 或限幅，避免溢出）。
   - 单总线时序（`OneWire_Init`, `SendBit`, `ReceiveBit`）按 **11.0592MHz** 精确校准微秒循环，并在关键窗口执行 `EA = 0; ... EA = 1;`。
5. **`DS1302.c / DS1302.h`**：
   - 复用根目录验证过的引脚定义（`P3^6/P3^4/P3^5`）与 BCD 转换逻辑。
   - 默认初始时间设为 `26年11月28日 20:00:00 星期6`（或合理默认值 `{26, 11, 28, 20, 0, 0, 6}`）。
   - 注意在 `DS1302_Init()` 或上电检测：若读出秒寄存器最高位 `CH=1`（时钟停止）或非法值，再写入默认时间，避免每次复位覆盖时间（若无备用电池则可上电检查 CH 位决定是否初始化）。
6. **`AT24C02.c / AT24C02.h`**（内含或调用静态 I2C 函数）：
   - 引脚：`I2C_SCL = P2^1`, `I2C_SDA = P2^0`（禁止动其他 P2 引脚）。
   - 写字节后自带 `Delay_ms(5)` 等待 EEPROM 内部写周期完成，防止连续写丢失数据！
7. **`IR.c / IR.h`**：
   - 直接整合 `Timer0`（16 位模式，不开中断）和 `INT0`（`IT0=1, EX0=1, PX0=1, EA=1`）。
   - 必须使用 **11.0592MHz** 判定阈值：
     - Start：`12442 ± 500`
     - Repeat：`10368 ± 500`
     - Bit 0：`1032 ± 500`
     - Bit 1：`2074 ± 500`
   - 当 `IR_State == 1` 或 `IR_State == 2` 遇到错误脉冲超时，必须重置 `Timer0_Run(0); IR_State = 0; IR_pData = 0;`（修复江协示例在出错时直接 `IR_State=1` 导致后续状态机卡死的小瑕疵！）。
   - `IR.h` 完整保留遥控器键码宏：`IR_POWER (0x45)`, `IR_MODE (0x46)`, `IR_MUTE (0x47)`, `IR_EQ (0x07)`, `IR_VOL_MINUS (0x15)`, `IR_VOL_ADD (0x09)`, `IR_0 (0x16)`, `IR_1 (0x0C)`, `IR_2 (0x18)`, `IR_3 (0x5E)` 等。
8. **`Motor.c / Motor.h`**：
   - 上电默认停转（`MOTOR_PWM = 0`, 档位 0）。
   - 4 档映射常量表（`code unsigned char GEAR_DUTY[4] = {0, 45, 75, 100};`）。
   - `Motor_PwmTickISR()` 由 Timer2 每 100us 调用：`Counter = (Counter + 1) % 100; MOTOR_PWM = (Counter < Compare) ? 1 : 0;`。
9. **`Buzzer.c / Buzzer.h`**：
   - 与 `LCD1602` 通过 `extern volatile bit g_LcdBusy;` 共享 `P2^5` 互斥保护。
   - **非阻塞发声**：
     - `Buzzer_BeepMs(unsigned int ms)` 设置剩余半周期或毫秒计数（提示音如按键反馈 30ms），绝不在主循环里死等阻塞！
     - `Buzzer_SetAlarmState(bit Enable)` 控制超温报警间歇鸣响（如每 1s 内前 300ms 鸣响、后 700ms 静音）。
     - `Buzzer_TickISR()` 在 Timer2 中断中每 500us（每 5 次 100us 中断）若允许发声且 `!g_LcdBusy`，则翻转 `BEEP_PIN = !BEEP_PIN`；不发声时确保 `BEEP_PIN = 0`（注意：不要在 `!g_LcdBusy` 为假即 LCD 正在写时强制写 `BEEP_PIN = 0`，避免干扰 LCD_RW 极短写窗口）。
10. **`Key.c / Key.h`**：
    - 在 20ms 节拍 `Key_Tick20ms()` 中扫描 `KEY_MODE (P3^1)`、`KEY_UP_GEAR (P3^0)`、`KEY_SET_MUTE (P3^3)`；**严禁扫描 `P3^2`**！
    - 当 `g_UartTxBusy == 1` 时跳过 `K1/K2` 采样。
    - 具备消抖与 `K4` 长按（>= 50 次 20ms = 1000ms）检测，区分 `KEY_4_SHORT` 与 `KEY_4_LONG`（`K4` 短按仅在松手且未触发长按时触发）。
11. **`UART.c / UART.h`**：
    - 波特率 **9600 bps @ 11.0592MHz**：`SCON = 0x50; PCON &= 0x7F; TMOD = (TMOD & 0x0F) | 0x20; TH1 = 0xFD; TL1 = 0xFD; ET1 = 0; TR1 = 1; ES = 1; EA = 1;`。
    - 导出 `volatile bit g_UartTxBusy = 0;`。
    - 发送：`UART_SendByte()` 置 `g_UartTxBusy = 1`，等待 `TI` 置位后清 `TI = 0; g_UartTxBusy = 0;`。
    - 接收中断 `interrupt 4`：仅接收 ASCII 指令行（缓冲 12 字节，`Buf[12]`），遇到 `\r` 或 `\n` 置 `RxReady = 1`；主循环调用 `UART_FetchCommand()` 取出处理。

### 3. 应用层 `firmware/src/app/`（3 个独立模块）

1. **`App_Config.c / App_Config.h`**：
   - 全局配置结构 `AppConfig`：`TempLowX10`、`TempHighX10`、`TempAlarmX10`。
   - AT24C02 存储映射（Magic `0xA5`、3 个阈值的高低字节或整度数、XOR 校验和）。
   - `AppConfig_Load()`：校验 Magic 与校验和，失败则恢复默认值 `250 / 300 / 350` 并写回 EEPROM。
   - `AppConfig_Save()`：写入并计算校验和；提供 `AppConfig_AdjustThreshold(Mode, Delta)` 确保始终满足 `150 <= TempLow < TempHigh < TempAlarm <= 600`。
2. **`App_Control.c / App_Control.h`**：
   - 状态枚举：`SYS_MODE_AUTO`, `SYS_MODE_MANUAL`, `SYS_SET_TL`, `SYS_SET_TH`, `SYS_SET_TA`。
   - 上电初始化序列：`LCD_Init` -> `DS1302_Init` -> `AT24C02`/`AppConfig_Load` -> `DS18B20_ConvertT`（先启动一次转换）-> `Delay_ms(750)` -> 首读温度 -> `Motor_Init` -> `Buzzer_Init` -> `IR_Init` -> `UART_Init` -> `Timer2_Init` -> `AppDisplay_Init`。
   - 主循环周期调度（无长阻塞）：
     - 每 20ms：`Key_Tick20ms()` 采样 + 消费按键/红外/串口指令。
     - 每 1s（结合 500ms/1s 标志）：读取一次 `DS1302_ReadTime()`；奇数秒发 `DS18B20_ConvertT()`、偶数秒（间隔>=1s）调 `DS18B20_ReadTempX10()`（或者第 1s 启动转换，下个 1s 读取后立刻启动下次转换，确保转换时间 >= 1000ms > 750ms！）；每 2s 经 UART 上报一帧状态；刷新报警与自动档位。
   - 报警逻辑：若温度有效且 `TempX10 >= TempAlarmX10`，进入报警态；仅在本次报警期间未按 Mute 才触发 `Buzzer_SetAlarmState(1)`；按下 `IR_MUTE` 或 `KEY_4_SHORT` 则置 `Muted = 1` 并关闭蜂鸣器；当 `TempX10 < TempAlarmX10 - 10` 后清除报警和 `Muted` 标志。
3. **`App_Display.c / App_Display.h`**：
   - 严格遵循 `HARDWARE_AND_INTERFACE_SPEC.md` 第五节第 2 点布局。
   - 全程采用定点数整除/取余拆字符（禁用 `sprintf` 浮点，尽量不用或谨慎用 `stdio.h` 以省 ROM，手写极简字符填充函数直接写入 16 字节行缓存并与上次缓存比较或按段刷新，彻底消除 LCD 闪烁）。

### 4. `firmware/src/main.c`
- 保持极简干净：仅调用 `AppControl_Init();` 并在 `while (1)` 中调用 `AppControl_Loop();`。

---

## 四、 验收标准与提交要求

1. **编译验收命令**（在工作区根目录执行）：
   ```powershell
   D:\Keil_v5\UV4\UV4.exe -b firmware\SmartFan.uvproj -o build.log
   ```
2. **硬性通过条件**：
   - `build.log` 末尾显示 `0 Error(s)`，无隐式类型截断/未定义函数等高危 Warning。
   - 成功生成 `firmware\Objects\SmartFan.hex`。
   - `Program Size` 中 `code <= 8192`（STC89C52RC 8KB Flash 上限）且 `data <= 128`（内部直接 RAM 不溢出）。
   - 全程无 `float` / `double`、无 `P2 = ...` 整口操作、无 `K3` 扫描、无未清 `TF2 = 0` 的 Timer2 中断。
3. **交付物**：
   - 执行 Agent 仅汇报真实修改的文件清单、关键实现要点与编译输出日志摘要（禁止修改 `docs/` 及根目录无关文件），随后交由 **朱鸢兄** 执行真编译和逐行 Code Review 验收。
