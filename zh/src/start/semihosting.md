# 半主机 (Semihosting)

半主机 (Semihosting) 是一种让嵌入式设备在主机上进行 I/O 的机制, 主要用于将消息记录到主机控制台. 半主机需要一个调试 (Debug) 会话, 几乎不需要其他任何东西 (没有额外的接线!), 所以用起来非常方便. 缺点是它非常慢: 每次写操作可能需要几毫秒, 具体取决于你使用的硬件调试器 (例如 ST-Link).

[`cortex-m-semihosting`] crate 提供了一个 API, 用于在 Cortex-M 设备上执行半主机操作. 下面的程序是半主机版本的 "Hello, world!":

[`cortex-m-semihosting`]: https://crates.io/crates/cortex-m-semihosting

```rust,ignore
#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::hprintln;

#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    loop {}
}
```

如果在硬件上运行这个程序, 你将在 OpenOCD 日志中看到 "Hello, world!" 消息.

``` text
$ openocd
(..)
Hello, world!
(..)
```

你确实需要先在 GDB 中为 OpenOCD 启用半主机:
``` console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

QEMU 理解半主机操作, 所以上面的程序也可以与 `qemu-system-arm` 一起工作, 而无需启动调试会话. 注意, 你需要给 QEMU 传 `-semihosting-config` 标志来启用半主机支持; 这些标志已经包含在模板的 `.cargo/config.toml` 文件中了.

``` text
$ # 这个程序会阻塞终端
$ # this program will block the terminal
$ cargo run
     Running `qemu-system-arm (..)
Hello, world!
```

还有一个 `exit` 半主机操作, 可以用来终止 QEMU 进程. 重要提示: **不要**在硬件上使用 `debug::exit`; 这个函数会破坏你的 OpenOCD 会话, 在你重启它之前, 你将无法再调试更多程序.

```rust,ignore
#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    if roses == "red" {
        debug::exit(debug::EXIT_SUCCESS);
    } else {
        debug::exit(debug::EXIT_FAILURE);
    }

    loop {}
}
```

``` text
$ cargo run
     Running `qemu-system-arm (..)

$ echo $?
1
```

最后一个提示: 你可以将异常处理 (Panicking) 行为设置为 `exit(EXIT_FAILURE)`. 这将允许你编写可以在 QEMU 上运行的 `no_std` 通过型 (run-pass) 测试.

为方便起见, `panic-semihosting` crate 提供了一个 "exit" feature (特性), 启用后会在将 panic 消息记录到主机 stderr 之后调用 `exit(EXIT_FAILURE)`.

```rust,ignore
#![no_main]
#![no_std]

use panic_semihosting as _; // features = ["exit"]

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    assert_eq!(roses, "red");

    loop {}
}
```

``` text
$ cargo run
     Running `qemu-system-arm (..)
panicked at 'assertion failed: `(left == right)`
  left: `"blue"`,
 right: `"red"`', examples/hello.rs:15:5

$ echo $?
1
```

**注意 (NOTE)**: 要在 `panic-semihosting` 上启用这个 feature, 请编辑你的 `Cargo.toml` 依赖部分, 在其中按如下方式指定 `panic-semihosting`:

``` toml
panic-semihosting = { version = "VERSION", features = ["exit"] }
```

其中 `VERSION` 是你想要的版本. 关于依赖特性的更多信息, 请参阅 Cargo 手册的 [`specifying dependencies`] 一节.

[`specifying dependencies`]:
https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html
