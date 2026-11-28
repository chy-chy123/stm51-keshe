# 多 Agent 协同开发军规（AGENTS.md）

本项目严格执行 **“司令部工单验收制”**。任何参与本仓库研发的 Agent 在读写文件或提交产出前，必须无条件遵守以下全部规范。

---

## 一、 当前项目阶段声明

- **当前阶段**：**未发布原型开发阶段**
- **阶段约束**：
  1. 严格按照冻结后的 `docs/HARDWARE_AND_INTERFACE_SPEC.md` 编码实现，严禁私自改动引脚定义、定时器分配、头文件函数签名、LCD 显示布局或串口通信协议。
  2. 严禁编写多余的“历史兼容层”来掩盖接口改错或数据结构破坏；严禁堆砌形式主义的空壳防御代码和假测试脚本。
  3. 仅在各自工单指定的目标目录与文件范围内操作，禁止越界修改。

---

## 二、 角色分工与文件领地隔离

| 角色 | 职责定位 | 允许写入路径 | 禁止触碰路径 |
| :--- | :--- | :--- | :--- |
| **朱鸢兄（Antigravity）** | 技术总监 + 架构冻结 + 逐行 Code Review + Keil 真编译验收 + LaTeX 论文主笔 | `docs/`、`AGENTS.md`、`WORKSPACE_MAP.md`、`README.md`、`.gitignore`、`paper/`、验收修复时的 `firmware/` | 不得破坏 `程序源码/` 与原始参考资料 |
| **AWS Quick（智能档）** | 免费期初版体力主力（Keil 工程搭建、模块骨架与初版实现） | 仅限工单指定的 `firmware/` 下新建文件 | 严禁修改 `/程序源码/`、根目录参考示例 `DS1302.*`/`LCD1602.*`、`.gitignore`、`docs/HARDWARE_AND_INTERFACE_SPEC.md` |
| **Codex (GPT)** | 嵌入式底层复核与整改主力（时序精度、状态机、中断互斥、ROM/RAM 优化） | 仅限工单指定的 `firmware/` 下文件 | 同上，严禁越界触碰参考目录与冻结规范文档 |
| **DSH / WorkBuddy** | 单点轻量脚本、环境整理与辅助资料 | 仅限各自工单明确授权路径 | 未授权路径一律只读 |

---

## 三、 C51（STC89C52RC @ 11.0592MHz）编码硬规

1. **严格模块化**：
   - 每个硬件驱动（`Delay`、`Timer2`、`LCD1602`、`DS18B20`、`DS1302`、`AT24C02`、`IR`、`Motor`、`Buzzer`、`Key`、`UART`）与每个业务模块（`App_Config`、`App_Control`、`App_Display`）必须拆分为独立的 `.c` 和 `.h` 文件。
   - 所有 `.h` 文件必须使用 `#ifndef __XXX_H__ ... #endif` 防重复包含宏，引脚集中定义在 `firmware/src/config/pin_config.h`。
2. **文件编码与 Keil C51 语法安全**：
   - 所有 `.c` / `.h` 源文件必须保存为 **UTF-8（无 BOM）**；禁止在代码行尾直接写带特殊符号的中文注释（防止 Keil C51 在 GBK 环境下因多字节末尾 `0x5C` 触发续行吞下一行代码）。
   - LCD1602 显示与 UART 串口输出一律使用 **纯 ASCII 字符**（如温度单位用 `C`，不用 `℃`）。
3. **禁用浮点 `float` 与严控资源开销**：
   - STC89C52RC 仅 **8KB Flash ROM** 与 **512B RAM**（其中内部直接寻址 DATA 仅 128B）。
   - 严禁在任何驱动或业务代码中使用 `float` / `double` 或引入 `printf` 浮点格式化，避免链接庞大浮点库导致 ROM 溢出！温度统一采用 **16 位有符号定点数（放大 10 倍）**，例如 `265` 表示 `26.5C`。
4. **硬件复用与中断铁律（红线，违者验收直接打回）**：
   - **禁止对 `P2` 口整口赋值**（如 `P2 = 0xFE;`）：`P2` 同时挂接了 LCD 控制线、I2C EEPROM 与蜂鸣器，全工程仅允许对 `P2` 进行 `sbit` 逐位操作。
   - **`P2^5`（`LCD_RW` 与 `BEEP` 复用）互斥保护**：LCD 写入前必须置位 `g_LcdBusy = 1` 且显式 `LCD_RW = 0`，写完后拉低 `LCD_EN = 0` 并释放 `g_LcdBusy = 0`；Timer2 中断中的蜂鸣器翻转仅在 `g_BuzzerTicks > 0 && !g_LcdBusy` 时执行。
   - **`P3^2`（红外 `INT0` 与独立按键 `K3` 复用）专口专用**：`P3^2` 固定分配给 VS1838B 红外外部中断 `INT0`，按键驱动严禁扫描 `K3`。
   - **`P3^0/P3^1`（UART 与 `K1/K2` 复用）串口保护**：UART 正在发送数据时置 `g_UartTxBusy = 1`，`Key_Scan()` 在 `g_UartTxBusy` 为真时跳过 `K1/K2` 采样。
   - **Timer2 中断标志清零**：8052 的 Timer2 硬件**不会**自动清零 `TF2`，`Timer2_Routine() interrupt 5` 入口处必须手动执行 `TF2 = 0;`。
   - **单总线时序临界区**：DS18B20 复位与字节收发必须在 11.0592MHz 下校准延时，位操作临界区短暂关闭总中断 `EA = 0;` 后立即恢复 `EA = 1;`。

---

## 四、 实证验收标准

下属 Agent 交付后，必须通过以下实证检查方可视为合格：
1. **Keil 命令行真编译通过**（0 Error）：
   ```powershell
   D:\Keil_v5\UV4\UV4.exe -b firmware\SmartFan.uvproj -o build.log
   ```
2. 生成有效固件 `firmware\Objects\SmartFan.hex`（或工单指定输出路径），且 `build.log` 中 `Program Size` 显示 `code <= 8192`、`data <= 128`（不溢出报错）。
3. 朱鸢兄逐行 Code Review 核查 `git diff`，确认无硬件冲突、无浮点库引入、无越界改文件、逻辑与 `HARDWARE_AND_INTERFACE_SPEC.md` 100% 一致。
