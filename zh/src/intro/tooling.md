# 工具链

处理微控制器会涉及使用多种不同的工具, 因为我们将面对的架构不同于你的笔记本电脑,
并且需要在 *远程* 设备上运行和调试程序.

我们将使用下面列出的所有工具. 当没有指定最低版本时, 任何较新版本都可以工作, 但我们列出了经过测试的版本.

- Rust 1.31、1.31-beta 或更新版本的工具链, 加上 ARM Cortex-M 编译支持.
- [`cargo-binutils`](https://github.com/rust-embedded/cargo-binutils) ~0.1.4
- [`qemu-system-arm`](https://www.qemu.org/). 测试版本: 3.0.0
- OpenOCD >= 0.8. 测试版本: v0.9.0 和 v0.10.0
- 带 ARM 支持的 GDB. 强烈建议使用 7.12 或更新版本. 测试
  版本: 7.10、7.11、7.12 和 8.1
- [`cargo-generate`](https://github.com/ashleygwilliams/cargo-generate) 或 `git`.
  这些工具是可选的, 但会让跟随本书更加容易.

下面的文字解释了我们为什么使用这些工具. 安装说明见下一页.

## `cargo-generate` 或 `git`

裸机程序是非标准 (`no_std`) 的 Rust 程序, 需要对链接过程做一些调整, 才能让程序的内存布局正确.
这需要一些额外的文件 (如链接脚本 (Linker Script)) 和设置 (如链接器 (Linker) 标志).
我们已经为你把它们打包在了一个模板 (Template) 中, 这样你只需要填写缺失的信息 (例如项目名称和目标硬件的特性).

我们的模板与 `cargo-generate` 兼容: `cargo-generate` 是一个用于从模板创建新 Cargo 项目的 Cargo 子命令.
你也可以使用 `git`、`curl`、`wget` 或你的网页浏览器来下载模板.

## `cargo-binutils`

`cargo-binutils` 是一组 Cargo 子命令的集合, 用于便捷地使用 Rust 工具链附带的 LLVM 工具.
这些工具包括 LLVM 版本的 `objdump`、`nm` 和 `size`, 用于检视二进制文件.

使用这些工具相比 GNU binutils 的优势在于: (a) 安装 LLVM 工具只需一条命令 (`rustup component add llvm-tools`),
与操作系统无关; (b) 像 `objdump` 这样的工具支持 `rustc` 所支持的全部架构 (从 ARM 到 x86_64),
因为它们共享同一个 LLVM 后端.

## `qemu-system-arm`

QEMU 是一个模拟器. 在本书中我们使用能够完整模拟 ARM 系统的变种.
我们使用 QEMU 在主机上运行嵌入式程序. 有了它, 即使你手头没有硬件, 也能跟随本书的某些部分!

# 嵌入式 Rust 调试工具链

## 总览

在 Rust 中调试嵌入式系统需要专门的工具, 包括用于管理调试过程的软件、用于检查和控制程序执行的调试器,
以及用于主机与嵌入式设备交互的硬件探针 (Probe). 本文档概述了必要的软件工具, 如 Probe-rs 和 OpenOCD,
它们简化并支持了调试过程; 同时还涵盖了主要的调试器, 如 GDB 和 Probe-rs 的 Visual Studio Code 扩展.
此外, 本文档还介绍了关键的硬件探针, 如 Rusty-probe、ST-Link、J-Link 和 MCU-Link,
它们对于嵌入式设备的有效调试和编程是不可或缺的.

## 驱动调试工具的软件

### Probe-rs

Probe-rs 是一款现代的、聚焦 Rust 的软件, 专为嵌入式系统中的调试器而设计.
与 OpenOCD 不同, Probe-rs 在设计时追求简洁, 旨在减少其他调试方案中常见的配置负担.
它支持各种探针和目标, 提供了一个与嵌入式硬件交互的高级接口.
Probe-rs 直接与 Rust 工具链集成, 并通过其 Visual Studio Code 扩展与 VS Code 集成,
让开发者能够简化调试工作流.


### OpenOCD (Open On-Chip Debugger)

OpenOCD 是一个开源软件工具, 用于嵌入式系统的调试、测试和编程.
它在主机系统和嵌入式硬件之间提供一个接口, 支持多种传输层, 如 JTAG 和 SWD (Serial Wire Debug).
OpenOCD 与 GDB (一个调试器) 集成. OpenOCD 拥有广泛的支持、丰富的文档和庞大的社区,
但可能需要复杂的配置, 尤其是在定制化的嵌入式环境中.

## 调试器

调试器允许开发者检查和控制程序的执行, 以识别和修正错误或缺陷 (Bug).
它提供诸如设置断点 (Breakpoint)、逐行单步 (Single Step) 执行代码, 以及检查变量值和内存状态等功能.
调试器对于全面的软件开发和维护至关重要, 使开发者能够确保他们的代码在各种条件下按预期行为运行.

调试器知道如何:
 * 与内存映射 (Memory Mapped) 寄存器 (Register) 交互.
 * 设置断点 / 观察点 (Watchpoint).
 * 读写内存映射的寄存器.
 * 检测 MCU 是否因调试事件而停止.
 * 在遇到调试事件后继续 MCU 的执行.
 * 擦除和写入微控制器的 Flash.

### Probe-rs 的 Visual Studio Code 扩展

Probe-rs 提供了一个 Visual Studio Code 扩展, 无需复杂配置即可提供无缝的调试体验.
通过这种连接, 开发者可以使用 Rust 特有的功能 (如美化打印和详细的错误消息),
确保他们的调试过程与 Rust 生态系统相契合. 

### TRACE32

TRACE32 是 Lauterbach 为嵌入式系统开发的专业的调试和跟踪解决方案.
它支持广泛的处理器架构, 包括 ARM 和 RISC-V, 并通过 JTAG、SWD 和各种跟踪接口连接到目标硬件.
TRACE32 提供高级的调试能力, 例如多核调试、复杂断点以及实时跟踪分析.
它使用标准的 ELF/DWARF 调试信息, 因此与使用常规工具链构建的 Rust 二进制文件兼容.

### GDB (GNU Debugger) 

GDB 是一个多用途的调试工具, 允许开发者检查正在运行或崩溃后的程序状态.
对于嵌入式 Rust, GDB 通过 OpenOCD 或其他调试服务器连接到目标系统, 以与嵌入式代码交互.
GDB 高度可配置, 支持远程调试、变量检查和条件断点等功能.
它可在多种平台上使用, 对 Rust 特有的调试需求 (如美化打印和与 IDE 的集成) 提供了广泛的支持.


## 探针 (Probes)

硬件探针是一种用于嵌入式系统开发和调试的设备, 用于促进主机与目标嵌入式设备之间的通信.
它通常支持 JTAG 或 SWD 等协议, 使其能够对嵌入式系统上的微控制器或微处理器进行编程、调试和分析.
硬件探针对于开发者设置断点、单步执行代码、检查内存和处理器寄存器至关重要, 能够让他们实时诊断和修复问题.

### Rusty-probe

Rusty-probe 是一款开源的基于 USB 的硬件调试探针, 专为与 probe-rs 配合使用而设计.
Rusty-probe 与 probe-rs 的组合为从事嵌入式 Rust 应用程序开发的开发者提供了一个易用且经济实惠的解决方案.

### ST-Link

ST-Link 是 STMicroelectronics 开发的一款流行的调试和编程探针, 主要用于其 STM32 和 STM8 微控制器系列.
它支持通过 JTAG 或 SWD (Serial Wire Debug) 接口进行调试和编程.
ST-Link 因 STMicroelectronics 大量开发板的直接支持以及与主流 IDE 的集成而被广泛使用,
使其成为使用 STM 微控制器的开发者的便捷选择.

### J-Link

J-Link 由 SEGGER Microcontroller 开发, 是一款强大而多用途的调试器, 支持远超 ARM 的各种 CPU 核和设备, 例如 RISC-V.
J-Link 以其高性能和可靠性著称, 支持各种通信接口, 包括 JTAG、SWD 和细间距 JTAG 接口.
它因 Flash 中无限制的断点等高级功能, 以及与众多开发环境的兼容性而受到青睐.

### MCU-Link

MCU-Link 是一款由 NXP Semiconductors 提供的, 兼具调试探针和编程器功能的设备.
它支持多种 ARM Cortex 微控制器, 并能与 MCUXpresso IDE 等开发工具无缝协作.
MCU-Link 因其多功能性和经济实惠而尤为突出, 成为业余爱好者、教育工作者和专业开发者都可获得的选择.
