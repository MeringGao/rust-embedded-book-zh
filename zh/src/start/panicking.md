# 异常处理 (Panicking)

异常处理 (Panicking) 是 Rust 语言的核心组成部分. 像索引这样的内置操作会在运行时检查内存安全. 当尝试越界索引时, 这将导致一个 panic.

在标准库中, panic 有一个定义明确的行为: 它会展开 (unwind) 触发 panic 的线程的栈 (Stack), 除非用户选择在 panic 时中止程序.

然而, 在没有标准库的 (即 `no_std`) 程序中, panic 行为是未定义的. 可以通过声明一个 `#[panic_handler]` 函数来选择一种行为. 这个函数必须在程序的依赖图中**恰好**出现一次, 并且必须具有以下签名: `fn(&PanicInfo) -> !`, 其中 [`PanicInfo`] 是一个包含 panic 位置信息的结构体.

[`PanicInfo`]: https://doc.rust-lang.org/core/panic/struct.PanicInfo.html

鉴于嵌入式系统的范围从面向用户到安全关键 (不能崩溃) 不等, 没有一种一刀切的 panic 行为, 但有许多常用的行为. 这些常见的行为已经被打包成一些 crate, 它们定义了 `#[panic_handler]` 函数. 一些例子包括:

- [`panic-abort`]. 一个 panic 会导致执行 abort 指令.
- [`panic-halt`]. 一个 panic 会使程序 (或当前线程) 通过进入死循环而停止 (halt).
- [`panic-itm`]. panic 消息通过 ITM 记录, ITM 是 ARM Cortex-M 特有的一个外设.
- [`panic-semihosting`]. panic 消息通过半主机 (Semihosting) 技术记录到主机.

[`panic-abort`]: https://crates.io/crates/panic-abort
[`panic-halt`]: https://crates.io/crates/panic-halt
[`panic-itm`]: https://crates.io/crates/panic-itm
[`panic-semihosting`]: https://crates.io/crates/panic-semihosting

你可以通过在 crates.io 上搜索 [`panic-handler`] 关键字找到更多 crate.

[`panic-handler`]: https://crates.io/keywords/panic-handler

程序可以通过简单地链接到相应的 crate 来选择这些行为之一. panic 行为在应用程序源码中以一行代码表示, 这一事实不仅作为文档很有用, 而且还可以根据编译配置 (profile) 改变 panic 行为. 例如:

``` rust,ignore
#![no_main]
#![no_std]

// dev profile: 更容易调试 panic; 可以在 `rust_begin_unwind` 上打断点
// dev profile: easier to debug panics; can put a breakpoint on `rust_begin_unwind`
#[cfg(debug_assertions)]
use panic_halt as _;

// release profile: 最小化应用程序的二进制大小
// release profile: minimize the binary size of the application
#[cfg(not(debug_assertions))]
use panic_abort as _;

// ..
```

在这个例子中, 当使用 dev profile (`cargo build`) 构建时, crate 链接到 `panic-halt`, 而当使用 release profile (`cargo build --release`) 构建时, 链接到 `panic-abort`.

> `use panic_abort as _;` 这种形式的 `use` 语句用于确保 `panic_abort` panic handler 被包含在我们最终的可执行文件中, 同时向编译器表明我们不会显式使用该 crate 中的任何东西. 如果没有 `as _` 重命名, 编译器会警告我们有一个未使用的导入.
> 有时你可能会看到 `extern crate panic_abort`, 这是 Rust 2018 edition 之前使用的旧风格, 现在只应用于 "sysroot" crate (那些随 Rust 本身分发的), 例如 `proc_macro`, `alloc`, `std` 和 `test`.

## 一个示例

下面是一个尝试对数组进行越界索引的示例. 该操作会导致一个 panic.

```rust,ignore
#![no_main]
#![no_std]

use panic_semihosting as _;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    let xs = [0, 1, 2];
    let i = xs.len();
    let _y = xs[i]; // 越界访问
                    // out of bounds access

    loop {}
}
```

这个示例选择了 `panic-semihosting` 行为, 它使用半主机将 panic 消息打印到主机控制台.

``` text
$ cargo run
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb (..)
panicked at 'index out of bounds: the len is 3 but the index is 4', src/main.rs:12:13
```

你可以试着把行为改为 `panic-halt`, 并确认在这种情况下不会打印任何消息.
