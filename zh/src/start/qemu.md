# QEMU

我们将开始为 [LM3S6965] 编写程序, 这是一款 Cortex-M3 微控制器 (Microcontroller, MCU). 我们之所以把它作为初始目标, 是因为它 [可以被 QEMU 模拟](https://wiki.qemu.org/Documentation/Platforms/ARM#Supported_in_qemu-system-arm), 这样在这一节你就不必摆弄硬件, 可以专注于工具链和开发流程.

[LM3S6965]: http://www.ti.com/product/LM3S6965

**重要 (IMPORTANT)**
在本教程中, 我们将使用 "app" 作为项目名. 每当你看到 "app" 这个词时, 都应该把它替换成你为自己项目选择的名称. 当然, 你也可以直接把自己的项目命名为 "app", 这样就避免了替换.

## 创建一个非标准的 Rust 程序

我们将使用 [`cortex-m-quickstart`] 项目模板来生成一个新项目. 生成的项目会包含一个骨架应用: 它是一个很好的嵌入式 Rust 应用起点. 此外, 项目中还会包含一个 `examples` 目录, 其中有若干独立的示例程序, 突出展示了嵌入式 Rust 的一些关键功能.

[`cortex-m-quickstart`]: https://github.com/rust-embedded/cortex-m-quickstart

### 使用 `cargo-generate`
首先安装 cargo-generate
```console
cargo install cargo-generate
```
然后生成一个新项目
```console
cargo generate --git https://github.com/knurling-rs/app-template
```

```text
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app
```

```console
cd app
```

### 使用 `git`

克隆仓库

```console
git clone https://github.com/rust-embedded/cortex-m-quickstart app
cd app
```

然后填充 `Cargo.toml` 文件中的占位符

```toml
[package]
authors = ["{{authors}}"] # "{{authors}}" -> "John Smith"
edition = "2018"
name = "{{project-name}}" # "{{project-name}}" -> "app"
version = "0.1.0"

# ..

[[bin]]
name = "{{project-name}}" # "{{project-name}}" -> "app"
test = false
bench = false
```

### 两种都不用

抓取 `cortex-m-quickstart` 模板的最新快照并解压.

```console
curl -LO https://github.com/rust-embedded/cortex-m-quickstart/archive/master.zip
unzip master.zip
mv cortex-m-quickstart-master app
cd app
```

或者你也可以浏览 [`cortex-m-quickstart`], 点击绿色的 "Clone or download" 按钮, 然后点击 "Download ZIP".

然后按照 "使用 `git`" 部分中的第二步填充 `Cargo.toml` 文件中的占位符.

## 程序概览

为方便起见, 这里给出 `src/main.rs` 中最重要的几部分源码:

```rust,ignore
#![no_std]
#![no_main]

use panic_halt as _;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    loop {
        // 你的代码写在这里
        // your code goes here
    }
}
```

这个程序和标准 Rust 程序有一点不同, 让我们仔细看看.

`#![no_std]` 表示这个程序**不会**链接到标准 crate, 即 `std`. 取而代之, 它会链接到 `std` 的子集: `core` crate.

`#![no_main]` 表示这个程序不会使用大多数 Rust 程序所使用的标准 `main` 接口. 使用 `no_main` 的主要原因是: 在 `no_std` 环境下使用 `main` 接口需要 nightly 工具链.

`use panic_halt as _;`. 这个 crate 提供了一个 `panic_handler`, 用于定义程序的异常处理 (Panicking) 行为. 我们将在本书的 [Panicking](panicking.md) 一章中详细介绍这一点.

[`#[entry]`][entry] 是 [`cortex-m-rt`] crate 提供的一个属性 (attribute), 用来标记程序的入口点. 由于我们不使用标准的 `main` 接口, 所以需要另一种方式来指示程序的入口点, 那就是 `#[entry]`.

[entry]: https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.entry.html
[`cortex-m-rt`]: https://crates.io/crates/cortex-m-rt

`fn main() -> !`. 我们的程序将是目标硬件上运行的**唯一**进程, 所以我们不希望它结束! 我们使用一个 [发散函数 (divergent function)](https://doc.rust-lang.org/rust-by-example/fn/diverging.html) (即函数签名中的 `-> !`) 来在编译期保证这一点.

## 交叉编译 (Cross Compilation)

首先, 我们需要目标微控制器 (本例中为 LM3S6965) 的内存布局信息. 否则构建过程将无法链接镜像. 在项目根目录创建一个名为 `memory.x` 的文件, 并粘贴以下内容:

```text
MEMORY
{
  /* NOTE 1 K = 1 KiBi = 1024 bytes */
  /* TODO 根据你的设备内存布局调整这些内存区域 */
  /* TODO Adjust these memory regions to match your device memory layout */
  /* 这些数值对应 LM3S6965, 是 QEMU 能够模拟的少数设备之一 */
  /* These values correspond to the LM3S6965, one of the few devices QEMU can emulate */
  FLASH : ORIGIN = 0x00000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 64K
}

/* 调用栈将在此处分配. */
/* This is where the call stack will be allocated. */
/* 栈是 full descending 类型. */
/* The stack is of the full descending type. */
/* 你可能想使用这个变量来将调用栈和静态变量放在不同的内存区域中. 下面是默认值 */
/* You may want to use this variable to locate the call stack and static
   variables in different memory regions. Below is shown the default value */
/* _stack_start = ORIGIN(RAM) + LENGTH(RAM); */

/* 你可以使用这个符号来自定义 .text 段的位置 */
/* You can use this symbol to customize the location of the .text section */
/* 如果省略, .text 段将被放在 .vector_table 段的紧后面 */
/* If omitted the .text section will be placed right after the .vector_table
   section */
/* 这只在那些在向量表后存储了一些配置的微控制器上是必需的 */
/* This is required only on microcontrollers that store some configuration right
   after the vector table */
/* _stext = ORIGIN(FLASH) + 0x400; */

/* 把未初始化变量放到自定义 RAM 区域的示例. */
/* Example of putting non-initialized variables into custom RAM locations. */
/* 这假设你在上面定义了 RAM2 区域, 并且在 Rust 源码中给数据项 */
/* This assumes you have defined a region RAM2 above, and in the Rust
   sources added the attribute `#[link_section = ".ram2bss"]` to the data
   you want to place there. */
/* 添加了 `#[link_section = ".ram2bss"]` 属性. */
/* Note that the section will not be zero-initialized by the runtime! */
/* 注意运行时不会将该段清零! */
/* SECTIONS {
     .ram2bss (NOLOAD) : ALIGN(4) {
       *(.ram2bss);
       . = ALIGN(4);
     } > RAM2
   } INSERT AFTER .bss;
*/
```

下一步是为 Cortex-M3 架构进行**交叉 (cross)** 编译. 如果你知道编译目标 (`$TRIPLE`) 是什么, 那么只要运行 `cargo build --target $TRIPLE` 就可以了. 幸运的是, 模板中的 `.cargo/config.toml` 已经给出了答案:

```console
tail -n6 .cargo/config.toml
```

```toml
[build]
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
# target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```

要为 Cortex-M3 架构进行交叉编译, 我们必须使用 `thumbv7m-none-eabi`. 安装 Rust 工具链 (Toolchain) 时该目标不会自动安装, 如果你之前还没有添加过这个目标, 那么现在是把它加入工具链的好时机:

``` console
rustup target add thumbv7m-none-eabi
```

 由于 `thumbv7m-none-eabi` 编译目标已经被设置为 `.cargo/config.toml` 中的默认值, 下面两条命令的作用是相同的:

```console
cargo build --target thumbv7m-none-eabi
cargo build
```

## 检查

现在我们在 `target/thumbv7m-none-eabi/debug/app` 得到了一个非原生 ELF (Executable and Linkable Format) 二进制文件 (Binary). 我们可以使用 `cargo-binutils` 来检查它.

使用 `cargo-readobj`, 我们可以打印 ELF 头来确认这是一个 ARM 二进制.

``` console
cargo readobj --bin app -- --file-headers
```

注意:
* `--bin app` 是检查 `target/$TRIPLE/debug/app` 处二进制文件的语法糖
* `--bin app` 在必要的情况下也会 (重新) 编译二进制文件


``` text
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0x0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x405
  Start of program headers:          52 (bytes into file)
  Start of section headers:          153204 (bytes into file)
  Flags:                             0x5000200
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         19
  Section header string table index:  18
```

`cargo-size` 可以打印二进制文件的链接段 (Linker Section) 的大小.


```console
cargo size --bin app --release -- -A
```
我们使用 `--release` 来检查优化后的版本

``` text
app  :
section             size        addr
.vector_table       1024         0x0
.text                 92       0x400
.rodata                0       0x45c
.data                  0  0x20000000
.bss                   0  0x20000000
.debug_str          2958         0x0
.debug_loc            19         0x0
.debug_abbrev        567         0x0
.debug_info         4929         0x0
.debug_ranges         40         0x0
.debug_macinfo         1         0x0
.debug_pubnames     2035         0x0
.debug_pubtypes     1892         0x0
.ARM.attributes       46         0x0
.debug_frame         100         0x0
.debug_line          867         0x0
Total              14570
```

> ELF 链接段复习
>
> - `.text` 包含程序指令
> - `.rodata` 包含字符串之类的常量值
> - `.data` 包含初始值**不**为零的静态分配变量
> - `.bss` 也包含静态分配变量, 但这些变量的初始值**是**零
> - `.vector_table` 是一个**非**标准段, 我们用它来存储向量 (中断) 表
> - `.ARM.attributes` 和 `.debug_*` 段包含元数据, 在烧录二进制时**不会**被加载到目标设备上.

**重要 (IMPORTANT)**: ELF 文件包含调试信息等元数据, 所以它们**在磁盘上的大小**并**不**能准确反映烧录到设备上时程序实际占用的空间. 请**始终**使用 `cargo-size` 来检查二进制文件的真实大小.

`cargo-objdump` 可以用来反汇编二进制文件.

```console
cargo objdump --bin app --release -- --disassemble --no-show-raw-insn --print-imm-hex
```

> **注意 (NOTE)** 如果上面的命令报 `Unknown command line argument` 错误, 请参阅以下 bug 报告: https://github.com/rust-embedded/book/issues/269

> **注意 (NOTE)** 此输出在你的系统上可能有所不同. 新版本的 rustc, LLVM 和库可能生成不同的汇编. 我们截断了一部分指令以保持片段简短.

```text
app:  file format ELF32-arm-little

Disassembly of section .text:
main:
     400: bl  #0x256
     404: b #-0x4 <main+0x4>

Reset:
     406: bl  #0x24e
     40a: movw  r0, #0x0
     < .. truncated any more instructions .. >

DefaultHandler_:
     656: b #-0x4 <DefaultHandler_>

UsageFault:
     657: strb  r7, [r4, #0x3]

DefaultPreInit:
     658: bx  lr

__pre_init:
     659: strb  r7, [r0, #0x1]

__nop:
     65a: bx  lr

HardFaultTrampoline:
     65c: mrs r0, msp
     660: b #-0x2 <HardFault_>

HardFault_:
     662: b #-0x4 <HardFault_>

HardFault:
     663: <unknown>
```

## 运行

接下来, 让我们看看如何在 QEMU 上运行嵌入式程序! 这次我们将使用 `hello` 示例, 它实际上会做一些事情. 默认情况下, 此示例使用 `[defmt]` 和 RTT 来打印文本.

[defmt]: https://defmt.ferrous-systems.com/

> **注意 (NOTE)** `defmt` 是一个第三方依赖 (即非核心依赖), 在嵌入式 Rust 生态中被广泛使用.

为了在主机端读取并解码 `defmt` 生成的消息, 我们需要将 RTT 传输输出切换为半主机 (Semihosting). 在真实硬件上这需要一个调试会话, 但在 QEMU 中直接就能工作.

让我们切换依赖:

```console
cargo remove defmt-rtt
cargo add defmt-semihosting
```

打开 `src/lib.rs`, 把 `use defmt_rtt as _;` 替换为 `use defmt_semihosting as _;`

现在我们可以构建示例:

```console
cargo build --bin hello
```

输出二进制文件位于
`target/thumbv7m-none-eabi/debug/hello`.

要在 QEMU 上运行这个二进制文件, 通常使用以下命令就够了:

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -kernel target/thumbv7m-none-eabi/debug/hello
```

在我们这里, 由于使用了 `defmt`, 主机将无法解码输出. 相反, 我们需要使用 Ferrous Systems 提供的一个工具 [`qemu-run`]:

[`qemu-run`]: https://github.com/knurling-rs/defmt/tree/main/qemu-run/

```console
git clone git@github.com:knurling-rs/defmt.git
cd defmt/qemu-run/
cargo run -- --machine lm3s6965evb ../qemu-rs/target/thumbv7m-none-eabi/debug/hello
```

```text
Hello, world!
```

该命令应该在打印文本后成功退出 (退出码 = 0). 在 *nix 系统上, 你可以用以下命令检查:

```console
echo $?
```

```text
0
```

我们来分解一下那条 QEMU 命令:

- `qemu-system-arm`. 这就是 QEMU 模拟器. QEMU 二进制有几种变体; 这个变体对 **ARM** 机器进行完整的**系统**模拟, 因此得名.

- `-cpu cortex-m3`. 这告诉 QEMU 模拟一个 Cortex-M3 CPU. 指定 CPU 型号能让我们捕获到一些编译错误: 例如, 运行一个为 Cortex-M4F (它带有硬件 FPU (Floating Point Unit, 浮点单元)) 编译的程序, QEMU 在执行时会报错.

- `-machine lm3s6965evb`. 这告诉 QEMU 模拟 LM3S6965EVB, 一块包含 LM3S6965 微控制器的评估板.

- `-nographic`. 这告诉 QEMU 不要启动它的图形界面.

- `-semihosting-config (..)`. 这告诉 QEMU 启用半主机 (Semihosting). 半主机让被模拟的设备 (除其他功能外) 能够使用主机的 stdout, stderr 和 stdin, 并在主机上创建文件.

- `-kernel $file`. 这告诉 QEMU 在被模拟的机器上加载并运行哪个二进制文件.

每次都输入这么长的 QEMU 命令太麻烦了! 我们可以设置一个自定义 runner 来简化这个流程. `.cargo/config.toml` 中有一个被注释掉的 runner, 它会调用 QEMU; 让我们把它取消注释:

```console
head -n3 .cargo/config.toml
```

```toml
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"
```

这个 runner 只对 `thumbv7m-none-eabi` 目标生效, 它正是我们默认的编译目标. 现在 `cargo run` 将编译程序并在 QEMU 上运行它:

```console
cargo run --example hello --release
```

```text
   Compiling app v0.1.0 (file:///tmp/app)
    Finished release [optimized + debuginfo] target(s) in 0.26s
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target/thumbv7m-none-eabi/release/examples/hello`
Hello, world!
```

## 调试

调试 (Debug) 对嵌入式开发至关重要. 让我们看看具体该怎么做.

调试嵌入式设备涉及**远程**调试, 因为我们要调试的程序并不会运行在运行调试器程序 (GDB 或 LLDB) 的那台机器上.

远程调试涉及一个客户端和一个服务器. 在 QEMU 配置中, 客户端将是一个 GDB (或 LLDB) 进程, 而服务器将是同样在运行嵌入式程序的 QEMU 进程.

在本节中, 我们将使用之前已经编译过的 `hello` 示例.

调试的第一步是以调试模式启动 QEMU:

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -gdb tcp::3333 \
  -S \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

这条命令不会向控制台打印任何东西, 并且会阻塞终端. 这次我们多传了两个标志:

- `-gdb tcp::3333`. 这告诉 QEMU 在 TCP 端口 3333 上等待 GDB 连接.

- `-S`. 这告诉 QEMU 在启动时冻结机器. 没有这个标志的话, 在我们启动调试器之前程序就已经执行完 main 了!

接下来我们在另一个终端中启动 GDB, 并告诉它加载示例的调试符号:

```console
gdb-multiarch -q target/thumbv7m-none-eabi/debug/examples/hello
```

**注意 (NOTE)**: 你可能需要使用其他版本的 gdb, 而不是 `gdb-multiarch`, 这取决于你在安装章节中安装的是哪个版本. 它也可能是 `arm-none-eabi-gdb` 或者就是 `gdb`.

然后在 GDB shell 中连接到正在 TCP 端口 3333 上等待连接的 QEMU.

```console
target remote :3333
```

```text
Remote debugging using :3333
Reset () at $REGISTRY/cortex-m-rt-0.6.1/src/lib.rs:473
473     pub unsafe extern "C" fn Reset() -> ! {
```


你会看到进程被挂起, 程序计数器指向一个名为 `Reset` 的函数. 那是复位处理函数 (reset handler): Cortex-M 内核在启动时执行的就是它.

> 注意, 在某些环境下, gdb 不会显示上面那一行 `Reset () at $REGISTRY/cortex-m-rt-0.6.1/src/lib.rs:473`, 而是可能打印一些警告, 例如:
>
> `core::num::bignum::Big32x40::mul_small () at src/libcore/num/bignum.rs:254`
> `    src/libcore/num/bignum.rs: No such file or directory.`
>
> 这是一个已知的 bug, 你可以安全地忽略这些警告, 你大概率就停在 `Reset()` 那里了.


这个复位处理函数最终会调用我们的 main 函数. 让我们用断点和 `continue` 命令一路跳到那里. 要设置断点, 先用 `list` 命令看看我们想在哪里打断.

```console
list main
```
这会显示来自文件 examples/hello.rs 的源码.

```text
6       use panic_halt as _;
7
8       use cortex_m_rt::entry;
9       use cortex_m_semihosting::{debug, hprintln};
10
11      #[entry]
12      fn main() -> ! {
13          hprintln!("Hello, world!").unwrap();
14
15          // exit QEMU
```
我们想在 "Hello, world!" 之前加一个断点, 也就是第 13 行. 用 `break` 命令来做这件事:

```console
break 13
```
现在我们可以用 `continue` 命令让 gdb 一直运行到我们的 main 函数:

```console
continue
```

```text
Continuing.

Breakpoint 1, hello::__cortex_m_rt_main () at examples\hello.rs:13
13          hprintln!("Hello, world!").unwrap();
```

现在我们离打印 "Hello, world!" 的代码很近了. 让我们用 `next` 命令前进.

``` console
next
```

```text
16          debug::exit(debug::EXIT_SUCCESS);
```

此时, 你应该能在运行 `qemu-system-arm` 的那个终端上看到 "Hello, world!" 被打印出来.

```text
$ qemu-system-arm (..)
Hello, world!
```

再次调用 `next` 会终止 QEMU 进程.

```console
next
```

```text
[Inferior 1 (Remote target) exited normally]
```

现在你可以退出 GDB 会话了.

``` console
quit
```
