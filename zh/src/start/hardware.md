# 硬件 (Hardware)

到目前为止, 你应该已经对工具链和开发流程比较熟悉了. 在这一节, 我们将切换到真实硬件上; 流程基本保持不变. 让我们开始吧.

## 了解你的硬件

在开始之前, 你需要识别目标设备的一些特性, 因为这些特性将用于配置项目:

- ARM 内核. 例如 Cortex-M3.

- ARM 内核是否包含 FPU? Cortex-M4**F** 和 Cortex-M7**F** 内核包含.

- 目标设备有多少 Flash 内存和 RAM? 例如 256 KiB Flash 和 32 KiB RAM.

- Flash 内存和 RAM 在地址空间中的映射位置在哪里? 例如 RAM 通常位于地址 `0x2000_0000`.

你可以在设备的数据手册 (data sheet) 或参考手册 (reference manual) 中找到这些信息.

在本节中, 我们将使用我们的参考硬件 STM32F3DISCOVERY. 该开发板上有一颗 STM32F303VCT6 微控制器. 该微控制器具有:

- 一个 Cortex-M4F 内核, 包含单精度 FPU

- 256 KiB Flash, 位于地址 0x0800_0000.

- 40 KiB RAM, 位于地址 0x2000_0000. (还有另一块 RAM 区域, 为了简单起见我们忽略它).

## 配置

我们将从零开始, 用一个新的模板实例. 关于如何在没有 `cargo-generate` 的情况下做这件事, 请参阅 [上一节关于 QEMU 的内容] 以复习.

[上一节关于 QEMU 的内容]: qemu.md

``` text
$ cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app

$ cd app
```

第一步是在 `.cargo/config.toml` 中设置默认编译目标.

``` console
tail -n5 .cargo/config.toml
```

``` toml
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
# target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```

我们将使用 `thumbv7em-none-eabihf`, 因为它能覆盖 Cortex-M4F 内核.
> **注意 (NOTE)**: 正如你可能从上一章记得的那样, 我们必须安装所有目标, 而这是一个新目标. 所以别忘了为这个目标运行安装过程 `rustup target add thumbv7em-none-eabihf`.

第二步是将内存区域信息填入 `memory.x` 文件.

``` text
$ cat memory.x
/* STM32F303VCT6 的链接器脚本 (Linker Script) */
/* Linker script for the STM32F303VCT6 */
MEMORY
{
  /* NOTE 1 K = 1 KiBi = 1024 bytes */
  FLASH : ORIGIN = 0x08000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 40K
}
```
> **注意 (NOTE)**: 如果你在对某个特定构建目标做了第一次构建之后, 因为某些原因修改了 `memory.x` 文件, 那么在 `cargo build` 之前请执行 `cargo clean`, 因为 `cargo build` 可能不会跟踪 `memory.x` 的更新.

我们还是从 hello 示例开始, 但首先需要做一个小的改动.

在 `examples/hello.rs` 中, 确保 `debug::exit()` 调用被注释掉或删除. 它只在 QEMU 中运行时才需要.

```rust,ignore
#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    // 退出 QEMU
    // exit QEMU
    // 注意不要在硬件上运行这个, 它会破坏 OpenOCD 状态
    // NOTE do not run this on hardware; it can corrupt OpenOCD state
    // debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

现在你可以使用 `cargo build` 进行交叉编译, 并像之前一样用 `cargo-binutils` 检查二进制文件. `cortex-m-rt` crate 处理了让你的芯片运行起来所需的所有魔法, 因为很方便地, 几乎所有 Cortex-M CPU 都以相同的方式启动.

``` console
cargo build --example hello
```

## 调试

调试的过程会看起来有点不同. 实际上, 第一步在不同的目标设备上可能看起来都不一样. 在本节中, 我们将展示调试运行在 STM32F3DISCOVERY 上的程序所需的步骤. 这旨在作为参考; 关于调试的设备特定信息, 请查阅 [the Debugonomicon](https://github.com/rust-embedded/debugonomicon).

和之前一样, 我们将进行远程调试, 客户端将是一个 GDB 进程. 不过这次, 服务器将是 OpenOCD.

如同在 [verify] 一节中所做的那样, 把 discovery 板连接到你的笔记本或 PC 上, 并检查 ST-LINK 排针是否被焊接上.

[verify]: ../intro/install/verify.md

在一个终端中运行 `openocd`, 以连接到 discovery 板上的 ST-LINK. 从模板的根目录运行该命令; `openocd` 会读取 `openocd.cfg` 文件, 其中指明要使用的 interface 文件和 target 文件.

``` console
cat openocd.cfg
```

``` text
# STM32F3DISCOVERY 开发板的 OpenOCD 配置示例
# Sample OpenOCD configuration for the STM32F3DISCOVERY development board

# 根据你拿到的硬件版本, 你需要从这些接口中选择一个.
# 任何时候, 只有一个接口应该被取消注释.
# Depending on the hardware revision you got you'll have to pick ONE of these
# interfaces. At any time only one interface should be commented out.

# 版本 C (较新版本)
# Revision C (newer revision)
source [find interface/stlink.cfg]

# 版本 A 和 B (较旧版本)
# Revision A and B (older revisions)
# source [find interface/stlink-v2.cfg]

source [find target/stm32f3x.cfg]
```

> **注意 (NOTE)** 如果你在 [verify] 一节中发现你拿到的是旧版本的 discovery 板, 那么你应该在这个时候修改 `openocd.cfg` 文件, 改用 `interface/stlink-v2.cfg`.

``` text
$ openocd
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
none separate
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v27 API v2 SWIM v15 VID 0x0483 PID 0x374B
Info : using stlink api v2
Info : Target voltage: 2.913879
Info : stm32f3x.cpu: hardware has 6 breakpoints, 4 watchpoints
```

在另一个终端中运行 GDB, 也从模板的根目录运行.

``` text
gdb-multiarch -q target/thumbv7em-none-eabihf/debug/examples/hello
```

**注意 (NOTE)**: 和之前一样, 你可能需要使用其他版本的 gdb 而不是 `gdb-multiarch`, 这取决于你在安装章节中安装的是哪个版本. 它也可能是 `arm-none-eabi-gdb` 或者就是 `gdb`.

接下来把 GDB 连接到 OpenOCD, OpenOCD 正在 3333 端口上等待 TCP 连接.

``` console
(gdb) target remote :3333
Remote debugging using :3333
0x00000000 in ?? ()
```

现在用 `load` 命令**烧录 (Flash)** (加载) 程序到微控制器上.

``` console
(gdb) load
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1518 lma 0x8000400
Loading section .rodata, size 0x414 lma 0x8001918
Start address 0x08000400, load size 7468
Transfer rate: 13 KB/sec, 2489 bytes/write.
```

程序现在已加载. 这个程序使用了半主机, 所以在我们进行任何半主机调用之前, 必须告诉 OpenOCD 启用半主机. 你可以使用 `monitor` 命令向 OpenOCD 发送命令.

``` console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

> 你可以通过调用 `monitor help` 命令查看所有 OpenOCD 命令.

和之前一样, 我们可以用断点和 `continue` 命令一路跳到 `main`.

``` console
(gdb) break main
Breakpoint 1 at 0x8000490: file examples/hello.rs, line 11.
Note: automatically using hardware breakpoints for read-only addresses.

(gdb) continue
Continuing.

Breakpoint 1, hello::__cortex_m_rt_main_trampoline () at examples/hello.rs:11
11      #[entry]
```

> **注意 (NOTE)** 如果在执行上面的 `continue` 命令之后, GDB 阻塞了终端, 而不是命中断点, 那么你可能需要再次确认 `memory.x` 文件中的内存区域信息 (起始地址**和**长度) 是否针对你的设备正确设置.

用 `step` 步入 main 函数.

``` console
(gdb) step
halted: PC: 0x08000496
hello::__cortex_m_rt_main () at examples/hello.rs:13
13          hprintln!("Hello, world!").unwrap();
```

在用 `next` 让程序前进之后, 你应该能在 OpenOCD 控制台上看到 "Hello, world!" 被打印出来, 以及其他一些输出.

``` console
$ openocd
(..)
Info : halted: PC: 0x08000502
Hello, world!
Info : halted: PC: 0x080004ac
Info : halted: PC: 0x080004ae
Info : halted: PC: 0x080004b0
Info : halted: PC: 0x080004b4
Info : halted: PC: 0x080004b8
Info : halted: PC: 0x080004bc
```
消息只被显示一次, 因为程序即将进入第 19 行定义的死循环: `loop {}`

现在你可以使用 `quit` 命令退出 GDB.

``` console
(gdb) quit
A debugging session is active.

        Inferior 1 [Remote target] will be detached.

Quit anyway? (y or n)
```

调试现在还需要几个步骤, 因此我们把所有这些步骤打包到一个名为 `openocd.gdb` 的 GDB 脚本中. 该文件是在 `cargo generate` 步骤中创建的, 应当无需任何修改即可工作. 让我们来看一下:

``` console
cat openocd.gdb
```

``` text
target extended-remote :3333

# print demangled symbols
set print asm-demangle on

# detect unhandled exceptions, hard faults and panics
break DefaultHandler
break HardFault
break rust_begin_unwind

monitor arm semihosting enable

load

# start the process but immediately halt the processor
stepi
```

现在运行 `<gdb> -x openocd.gdb target/thumbv7em-none-eabihf/debug/examples/hello` 将立即把 GDB 连接到 OpenOCD, 启用半主机, 加载程序并启动进程.

或者, 你也可以把 `<gdb> -x openocd.gdb` 转成自定义 runner, 让 `cargo run` 构建程序**并**启动 GDB 会话. 这个 runner 包含在 `.cargo/config.toml` 中, 但被注释掉了.

``` console
head -n10 .cargo/config.toml
```

``` toml
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
# runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"

[target.'cfg(all(target_arch = "arm", target_os = "none"))']
# 取消注释这三个选项之一, 让 `cargo run` 启动一个 GDB 会话
# uncomment ONE of these three option to make `cargo run` start a GDB session
# 选择哪一个取决于你的系统
# which option to pick depends on your system
runner = "arm-none-eabi-gdb -x openocd.gdb"
# runner = "gdb-multiarch -x openocd.gdb"
# runner = "gdb -x openocd.gdb"
```

``` text
$ cargo run --example hello
(..)
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1e70 lma 0x8000400
Loading section .rodata, size 0x61c lma 0x8002270
Start address 0x800144e, load size 10380
Transfer rate: 17 KB/sec, 3460 bytes/write.
(gdb)
```
