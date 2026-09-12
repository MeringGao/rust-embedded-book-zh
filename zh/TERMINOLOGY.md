# 翻译术语表

> 维护说明: 第一次出现某术语时, 中文 + 英文括号注释, 例如 "外设 (Peripheral)". 后续只保留中文.

## 类别 A: 保留英文 (技术名词/品牌/工具)

| 英文 | 说明 |
|------|------|
| `no_std` | Rust 内置属性, 不翻译 |
| `bare metal` | 裸机, 在术语中保留英文 |
| `Cortex-M` | ARM 处理器架构, 品牌 |
| `QEMU` | 模拟器 |
| `OpenOCD` | 调试工具 |
| `GDB` | 调试器 |
| `cargo` | Rust 包管理器 |
| `rustc` | Rust 编译器 |
| `linker` | 链接器, 保留英文 |
| `linker script` | 链接脚本 |
| `RTOS` | 实时操作系统 |
| `HAL` | 硬件抽象层 |
| `PAC` | 外设访问 crate |
| `svd2rust` | 工具名 |
| `embedded-hal` | crate 名 |
| `Embassy` | 异步嵌入式框架, 品牌 |
| `STM32` / `nRF52` / `MSP430` / `AVR` | 芯片型号, 保留 |
| `GPIO` | 通用输入输出, 通用缩写, 保留 |
| `UART` / `SPI` / `I2C` / `USART` | 总线协议, 保留 |
| `USB` / `CAN` / `Ethernet` | 协议, 保留 |
| `PWM` / `ADC` / `DAC` | 硬件模块, 保留 |
| `DMA` | 直接内存访问, 保留缩写 |
| `FPU` | 浮点单元, 保留 |
| `LLVM` / `ARM` / `Thumb` | 工具链 / 架构 |
| `Rust 2018 edition` | Rust 版本, 保留 |
| `#[...]` / `fn` / `let` / `mut` 等 | 代码语法, 不翻译 |
| `trait` / `struct` / `enum` / `impl` | Rust 关键字, 保留 |

## 类别 B: 翻译并首次加注英文

| 中文 | 英文 | 备注 |
|------|------|------|
| 外设 | Peripheral | 硬件设备 |
| 寄存器 | Register | 内存映射 I/O 中的寄存器 |
| 微控制器 | Microcontroller (MCU) | 简称 MCU |
| 中断 | Interrupt | |
| 异常 | Exception | 注意 Cortex-M 中 Exception 是包含 Interrupt 的上位概念 |
| 半主机 | Semihosting | 调试时主机与目标通信机制 |
| 工具链 | Toolchain | |
| 交叉编译 | Cross Compilation | |
| 内存映射 | Memory Mapped | |
| 单例 | Singleton | 编程模式 |
| 类型状态 | Typestate | 编程模式 |
| 借用检查器 | Borrow Checker | |
| 静态保证 | Static Guarantee | |
| 零成本抽象 | Zero Cost Abstraction | |
| 设计契约 | Design Contract | |
| 状态机 | State Machine | |
| 互操作性 | Interoperability | |
| 可预测性 | Predictability | |
| 类型状态编程 | Typestate Programming | |
| 命名 | Naming | |
| 清单 | Checklist | |
| 优化 | Optimization | |
| 速度与大小 | Speed vs Size | |
| 异常处理 | Panicking | rust 中的 panic 机制 |
| 看门狗 | Watchdog | |
| 时钟 | Clock | |
| 复位 | Reset | |
| 引导程序 | Bootloader | |
| 链接脚本 | Linker Script | |
| 段 | Section | 内存段, 如 .text / .data |
| 原子操作 | Atomic Operation | |
| 信号量 | Semaphore | |
| 互斥锁 | Mutex | |
| 临界区 | Critical Section | |
| 优先级反转 | Priority Inversion | |
| 死锁 | Deadlock | |
| 实时 | Real-Time | |
| 并发 | Concurrency | |
| 并行 | Parallelism | |
| 多线程 | Multithreading | 嵌入式场景中较少使用, 强调 cooperative 时也翻译 |
| 调度器 | Scheduler | |
| 任务 | Task | RTOS 概念 |
| 协程 | Coroutine | |
| 异步 | Async | |
| 阻塞 | Blocking | |
| 非阻塞 | Non-blocking | |
| 无锁 | Lock-free | |
| 字节序 | Endianness | |
| 大端 | Big-endian | |
| 小端 | Little-endian | |
| 二进制文件 | Binary | |
| 十六进制 | Hexadecimal | |
| 字节序 | Byte Order | |
| 触发 | Trigger | |
| 边沿触发 | Edge Triggered | |
| 电平触发 | Level Triggered | |
| 优先级 | Priority | |
| 抢占 | Preemption | |
| 协作式 | Cooperative | |
| 抢占式 | Preemptive | |
| 看门狗定时器 | Watchdog Timer | |
| 串行 | Serial | |
| 并行 | Parallel | |
| 总线 | Bus | |
| 主机 | Host | 与 Target 相对 |
| 目标 | Target | 嵌入式设备 |
| 探针 | Probe | 调试硬件 |
| 烧录 | Flash | 烧录固件 |
| 固件 | Firmware | |
| 调试 | Debug | |
| 单步 | Single Step | |
| 断点 | Breakpoint | |
| 观察点 | Watchpoint | |
| 栈 | Stack | 调用栈 |
| 堆 | Heap | 动态内存 |
| 内存布局 | Memory Layout | |
| 向量表 | Vector Table | |
| 异常向量 | Exception Vector | |
| 重定位 | Relocation | |
| 位置无关代码 | Position-Independent Code (PIC) | |
| 加载地址 | Load Address | |
| 入口点 | Entry Point | |
| 退出点 | Exit Point | |
| 启动 | Startup | |
| 终止 | Termination | |
| 清理 | Cleanup | |
| 钩子 | Hook | |

## 类别 C: 代码块翻译规则

- Rust 代码: **保留**所有英文, 注释翻译为中文 + 英文双语
  ```rust
  // 开启 LED
  // Turn on the LED
  led.set_high();
  ```
- 配置 / TOML 代码块: 保留原值, 注释翻译
- 终端输出 (`console` 块): 保留原样 (这是实际命令输出)
- 文件路径: 保留英文
- URL: 保留英文

## 类别 D: 一般中文写作约定

- 中英文之间加空格
- 中文标点规则: 顿号 (、) 保留, 其余标点一律使用英文 + 空格
- 章节标题使用 # / ## / ###
- 引号使用英文双引号
- 强调使用 `*文本*` (org 加粗) 或 rustdoc 风格 `**文本**`