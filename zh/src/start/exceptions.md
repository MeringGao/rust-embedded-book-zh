# 异常 (Exception)

异常 (Exception) 和中断 (Interrupt) 是处理器用来处理异步事件和致命错误 (例如执行了无效指令) 的硬件机制. 异常意味着抢占 (Preemption), 并涉及异常处理程序 (exception handler), 即响应触发事件的信号而执行的子例程.

`cortex-m-rt` crate 提供了一个 [`exception`] 属性, 用于声明异常处理程序.

[`exception`]: https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.exception.html

``` rust,ignore
// SysTick (系统定时器) 异常的异常处理程序
// Exception handler for the SysTick (System Timer) exception
#[exception]
fn SysTick() {
    // ..
}
```

除了 `exception` 属性, 异常处理程序看起来就像普通函数, 但还有一个区别: `exception` 处理程序**不能**被软件调用. 沿用前面的例子, 语句 `SysTick();` 将导致编译错误.

这种行为基本上是有意为之的, 并且它是提供下面这个特性的必要条件: 在 `exception` 处理程序**内部**声明的 `static mut` 变量可以**安全地**使用.

``` rust,ignore
#[exception]
fn SysTick() {
    static mut COUNT: u32 = 0;

    // `COUNT` 已经被转换为 `&mut u32` 类型, 并且可以安全地使用
    // `COUNT` has transformed to type `&mut u32` and it's safe to use
    *COUNT += 1;
}
```

正如你可能知道的, 在函数中使用 `static mut` 变量会使该函数变为 [不可重入 (non-reentrant)](https://en.wikipedia.org/wiki/Reentrancy_(computing)). 从多个异常或中断处理程序, 或者从 `main` 与一个或多个异常/中断处理程序, 直接或间接地调用一个不可重入的函数, 是未定义行为.

安全的 Rust 绝不能导致未定义行为, 所以不可重入函数必须被标记为 `unsafe`. 但我刚才说过 `exception` 处理程序可以安全地使用 `static mut` 变量. 这是怎么做到的呢? 这是可能的, 因为 `exception` 处理程序**不能**被软件调用, 因此不可能发生重入. 这些处理程序由硬件本身调用, 假设硬件在物理上是不可并发的.

因此, 在嵌入式系统的异常处理程序上下文中, 缺少对同一处理程序的并发调用确保了即使处理程序使用了静态可变变量, 也不会有重入问题.

在多核系统中, 多个处理器内核并发执行代码, 即使在异常处理程序内, 重入问题的可能性也变得相关. 虽然每个内核可能有自己的一组异常处理程序, 但仍然可能出现多个内核试图同时执行同一异常处理程序的情况.
为了在多核环境中解决这个问题, 必须在异常处理程序内采用适当的同步机制, 以确保对共享资源的访问在内核之间得到正确协调. 这通常涉及使用锁, 信号量 (Semaphore) 或原子操作 (Atomic Operation) 等技术, 以防止数据竞争并维护数据完整性.

> 注意, `exception` 属性通过将函数内部静态变量的定义包装到 `unsafe` 块中, 并向我们提供同名但类型为 `&mut` 的新适当变量, 来转换这些定义.
> 因此, 我们可以通过 `*` 解引用该引用来访问变量的值, 而无需将其包装在 `unsafe` 块中.

## 一个完整的示例

下面是一个示例, 它使用系统定时器大约每秒触发一次 `SysTick` 异常. `SysTick` 异常处理程序在 `COUNT` 变量中跟踪它被调用的次数, 然后使用半主机将 `COUNT` 的值打印到主机控制台.

> **注意 (NOTE)**: 你可以在任何 Cortex-M 设备上运行这个示例; 你也可以在 QEMU 上运行它

```rust,ignore
#![deny(unsafe_code)]
#![no_main]
#![no_std]

use panic_halt as _;

use core::fmt::Write;

use cortex_m::peripheral::syst::SystClkSource;
use cortex_m_rt::{entry, exception};
use cortex_m_semihosting::{
    debug,
    hio::{self, HostStream},
};

#[entry]
fn main() -> ! {
    let p = cortex_m::Peripherals::take().unwrap();
    let mut syst = p.SYST;

    // 配置系统定时器, 使其每秒触发一次 SysTick 异常
    // configures the system timer to trigger a SysTick exception every second
    syst.set_clock_source(SystClkSource::Core);
    // 这是为 LM3S6965 配置的, 其默认 CPU 时钟为 12 MHz
    // this is configured for the LM3S6965 which has a default CPU clock of 12 MHz
    syst.set_reload(12_000_000);
    syst.clear_current();
    syst.enable_counter();
    syst.enable_interrupt();

    loop {}
}

#[exception]
fn SysTick() {
    static mut COUNT: u32 = 0;
    static mut STDOUT: Option<HostStream> = None;

    *COUNT += 1;

    // 延迟初始化
    // Lazy initialization
    if STDOUT.is_none() {
        *STDOUT = hio::hstdout().ok();
    }

    if let Some(hstdout) = STDOUT.as_mut() {
        write!(hstdout, "{}", *COUNT).ok();
    }

    // 重要提示: 如果在真实硬件上运行, 请省略这个 `if` 块,
    // 否则你的调试器将以不一致的状态结束
    // IMPORTANT omit this `if` block if running on real hardware or your
    // debugger will end in an inconsistent state
    if *COUNT == 9 {
        // 这将终止 QEMU 进程
        // This will terminate the QEMU process
        debug::exit(debug::EXIT_SUCCESS);
    }
}
```

``` console
tail -n5 Cargo.toml
```

``` toml
[dependencies]
cortex-m = "0.5.7"
cortex-m-rt = "0.6.3"
panic-halt = "0.2.0"
cortex-m-semihosting = "0.3.1"
```

``` text
$ cargo run --release
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb (..)
123456789
```

如果在 Discovery 板上运行它, 你将在 OpenOCD 控制台上看到输出. 此外, 程序在计数达到 9 时**不会**停止.

## 默认异常处理程序

`exception` 属性实际做的事情是**覆盖**特定异常的默认异常处理程序. 如果你没有覆盖某个特定异常的处理程序, 它将由 `DefaultHandler` 函数处理, 默认实现是:

``` rust,ignore
fn DefaultHandler() {
    loop {}
}
```

这个函数由 `cortex-m-rt` crate 提供, 并被标记为 `#[no_mangle]`, 因此你可以把断点放在 "DefaultHandler" 上, 来捕获**未处理**的异常.

可以使用 `exception` 属性覆盖这个 `DefaultHandler`:

``` rust,ignore
#[exception]
fn DefaultHandler(irqn: i16) {
    // 自定义默认处理程序
    // custom default handler
}
```

`irqn` 参数指示正在处理哪个异常. 负值表示正在处理一个 Cortex-M 异常; 而零或正值表示正在处理一个设备特定的异常, 也即中断.

## 硬 fault 处理程序

`HardFault` 异常有点特殊. 当程序进入无效状态时会触发该异常, 所以它的处理程序**不能**返回, 因为返回可能导致未定义行为. 此外, 运行时的 crate 在调用用户定义的 `HardFault` 处理程序之前会做一些工作, 以提高可调试性.

结果就是, `HardFault` 处理程序必须具有以下签名: `fn(&ExceptionFrame) -> !`. 处理程序的参数是指向异常时压入栈中的寄存器的指针. 这些寄存器是异常触发时刻处理器状态的一个快照, 可用于诊断硬 fault.

下面是一个执行非法操作的示例: 读取一个不存在的内存位置.

> **注意 (NOTE)**: 这个程序在 QEMU 上**不会**工作, 也就是说, 它**不会**崩溃, 因为 `qemu-system-arm -machine lm3s6965evb` 不检查内存加载, 会愉快地对无效内存读取返回 `0`.

```rust,ignore
#![no_main]
#![no_std]

use panic_halt as _;

use core::fmt::Write;
use core::ptr;

use cortex_m_rt::{entry, exception, ExceptionFrame};
use cortex_m_semihosting::hio;

#[entry]
fn main() -> ! {
    // 读取一个不存在的内存位置
    // read a nonexistent memory location
    unsafe {
        ptr::read_volatile(0x3FFF_0000 as *const u32);
    }

    loop {}
}

#[exception]
fn HardFault(ef: &ExceptionFrame) -> ! {
    if let Ok(mut hstdout) = hio::hstdout() {
        writeln!(hstdout, "{:#?}", ef).ok();
    }

    loop {}
}
```

`HardFault` 处理程序打印 `ExceptionFrame` 的值. 如果运行它, 你将在 OpenOCD 控制台上看到类似下面的内容.

``` text
$ openocd
(..)
ExceptionFrame {
    r0: 0x3fff0000,
    r1: 0x00000003,
    r2: 0x080032e8,
    r3: 0x00000000,
    r12: 0x00000000,
    lr: 0x080016df,
    pc: 0x080016e2,
    xpsr: 0x61000000,
}
```

`pc` 的值是异常发生时刻程序计数器的值, 它指向触发该异常的指令.

如果你查看程序的反汇编:

``` text
$ cargo objdump --bin app --release -- -d --no-show-raw-insn --print-imm-hex
(..)
ResetTrampoline:
 8000942:       movw    r0, #0xfffe
 8000946:       movt    r0, #0x3fff
 800094a:       ldr     r0, [r0]
 800094c:       b       #-0x4 <ResetTrampoline+0xa>
```

你可以在反汇编中查找程序计数器 `0x0800094a` 这个值. 你会看到是一次加载操作 (`ldr r0, [r0]`) 导致了该异常. `ExceptionFrame` 的 `r0` 字段会告诉你, 当时寄存器 `r0` 的值是 `0x3fff_fffe`.
