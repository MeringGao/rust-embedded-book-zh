# 第一次尝试

## 寄存器 (Registers)

让我们看看 `SysTick` 外设 — 一个简单的定时器, 每个 Cortex-M 处理器内核都自带. 通常你会在芯片制造商的数据手册或*技术参考手册 (Technical Reference Manual)* 中查找这些内容, 但本例对所有 ARM Cortex-M 内核都是通用的, 我们来查看 [ARM reference manual]. 我们看到有四个寄存器:

[ARM reference manual]: http://infocenter.arm.com/help/topic/com.arm.doc.dui0553a/Babieigh.html

| Offset | Name        | Description                 | Width  |
|--------|-------------|-----------------------------|--------|
| 0x00   | SYST_CSR    | Control and Status Register | 32 bits|
| 0x04   | SYST_RVR    | Reload Value Register       | 32 bits|
| 0x08   | SYST_CVR    | Current Value Register      | 32 bits|
| 0x0C   | SYST_CALIB  | Calibration Value Register  | 32 bits|

## C 语言的写法

在 Rust 中, 我们可以用与 C 完全相同的方式来表示寄存器的集合 — 用 `struct`.

```rust,ignore
#[repr(C)]
struct SysTick {
    pub csr: u32,
    pub rvr: u32,
    pub cvr: u32,
    pub calib: u32,
}
```

`#[repr(C)]` 这个限定符告诉 Rust 编译器像 C 编译器那样排列这个结构体的布局. 这非常重要, 因为 Rust 允许重新排序结构体字段, 而 C 不允许. 你可以想象一下, 如果这些字段被编译器悄悄重新排列, 我们要进行怎样的调试! 有了这个限定符, 我们就有了对应上表的四个 32 位字段. 当然, 这个 `struct` 本身没什么用 — 我们需要一个变量.

```rust,ignore
let systick = 0xE000_E010 as *mut SysTick;
let time = unsafe { (*systick).cvr };
```

## Volatile 访问

现在, 上面的方法存在一些问题.

1. 每次想要访问外设时, 我们都必须使用 unsafe.
2. 我们没有办法指定哪些寄存器是只读的, 哪些是可读写的.
3. 程序中任何地方的任何代码都可以通过这个结构访问硬件.
4. 最重要的是, 它实际上并不能工作...

现在, 问题在于编译器是很聪明的. 如果你对同一块 RAM 进行两次连续写入, 编译器可以注意到这一点并完全跳过第一次写入. 在 C 中, 我们可以将变量标记为 `volatile` 以确保每次读或写都按预期执行. 在 Rust 中, 我们改为将*访问*标记为 volatile, 而非变量.

```rust,ignore
let systick = unsafe { &mut *(0xE000_E010 as *mut SysTick) };
let time = unsafe { core::ptr::read_volatile(&mut systick.cvr) };
```

所以, 我们解决了四个问题中的一个, 但现在我们有更多的 `unsafe` 代码! 幸运的是, 有一个第三方 crate 可以帮忙 — [`volatile_register`].

[`volatile_register`]: https://crates.io/crates/volatile_register

```rust,ignore
use volatile_register::{RW, RO};

#[repr(C)]
struct SysTick {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

fn get_systick() -> &'static mut SysTick {
    unsafe { &mut *(0xE000_E010 as *mut SysTick) }
}

fn get_time() -> u32 {
    let systick = get_systick();
    systick.cvr.read()
}
```

现在, volatile 访问通过 `read` 和 `write` 方法自动执行. 执行写入仍然是 `unsafe` 的, 但公平地说, 硬件就是一堆可变状态, 编译器没有办法知道这些写入是否真的安全, 所以这是一个合理的默认立场.

## Rust 风格的封装

我们需要把这个 `struct` 包装成一个更高级的 API, 让用户可以安全地调用. 作为驱动作者, 我们手动验证 unsafe 代码的正确性, 然后为用户提供一个安全的 API, 这样他们就不必担心这些 (前提是他们相信我们能写对!).

一个例子可能是:

```rust,ignore
use volatile_register::{RW, RO};

pub struct SystemTimer {
    p: &'static mut RegisterBlock
}

#[repr(C)]
struct RegisterBlock {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

impl SystemTimer {
    pub fn new() -> SystemTimer {
        SystemTimer {
            p: unsafe { &mut *(0xE000_E010 as *mut RegisterBlock) }
        }
    }

    pub fn get_time(&self) -> u32 {
        self.p.cvr.read()
    }

    pub fn set_reload(&mut self, reload_value: u32) {
        unsafe { self.p.rvr.write(reload_value) }
    }
}

pub fn example_usage() -> String {
    let mut st = SystemTimer::new();
    st.set_reload(0x00FF_FFFF);
    format!("Time is now 0x{:08x}", st.get_time())
}
```

现在, 这种方法的问题在于, 以下代码对编译器来说完全是可以接受的:

```rust,ignore
fn thread1() {
    let mut st = SystemTimer::new();
    st.set_reload(2000);
}

fn thread2() {
    let mut st = SystemTimer::new();
    st.set_reload(1000);
}
```

我们对 `set_reload` 函数的 `&mut self` 参数会检查是否没有对该*特定* `SystemTimer` 结构体的其他引用, 但它们并不能阻止用户创建指向完全相同外设的第二个 `SystemTimer`! 如果作者足够细心以发现所有这些"重复"的驱动实例, 以这种方式编写的代码可以工作, 但一旦代码分散在多个模块、驱动、开发者和日子里, 犯这类错误就会变得越来越容易.
