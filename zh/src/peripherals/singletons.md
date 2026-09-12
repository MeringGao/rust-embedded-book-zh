# 单例 (Singletons)

> 在软件工程中, 单例模式 (Singleton Pattern) 是一种软件设计模式, 它将类的实例化限制为一个对象.
>
> *Wikipedia: [Singleton Pattern]*

[Singleton Pattern]: https://en.wikipedia.org/wiki/Singleton_pattern


## 但为什么我们不能直接使用全局变量呢?

我们可以将所有东西都设为公共的 static, 像这样

```rust,ignore
static mut THE_SERIAL_PORT: SerialPort = SerialPort;

fn main() {
    let _ = unsafe {
        THE_SERIAL_PORT.read_speed();
    };
}
```

但这有几个问题. 它是一个可变全局变量, 在 Rust 中, 与它们交互始终是 unsafe 的. 这些变量在整个程序中都可见, 这意味着借用检查器无法帮助你跟踪这些变量的引用和所有权.

## 在 Rust 中我们怎么做?

我们可以决定将外设放入一个结构 (在这里称为 `PERIPHERALS`) 中, 该结构为每个外设包含一个 `Option<T>`, 而不是仅仅把外设作为全局变量.

```rust,ignore
struct Peripherals {
    serial: Option<SerialPort>,
}
impl Peripherals {
    fn take_serial(&mut self) -> SerialPort {
        let p = replace(&mut self.serial, None);
        p.unwrap()
    }
}
static mut PERIPHERALS: Peripherals = Peripherals {
    serial: Some(SerialPort),
};
```

这个结构允许我们获得外设的单个实例. 如果我们尝试多次调用 `take_serial()`, 代码将会 panic!

```rust,ignore
fn main() {
    let serial_1 = unsafe { PERIPHERALS.take_serial() };
    // 这里会 panic!
    // let serial_2 = unsafe { PERIPHERALS.take_serial() };
}
```

虽然与该结构交互是 `unsafe`, 但一旦我们拥有了它包含的 `SerialPort`, 就不再需要使用 `unsafe` 或 `PERIPHERALS` 结构了.

这有一点运行时开销, 因为我们必须将 `SerialPort` 结构包装在 option 中, 并且需要调用一次 `take_serial()`, 然而, 这种较小的前期成本让我们能够在程序的其他部分利用借用检查器.

## 现成的库支持

虽然我们在上面创建了自己的 `Peripherals` 结构, 但你不必为你的代码这样做. `cortex_m` crate 包含一个名为 `singleton!()` 的宏, 它会为你执行此操作.

```rust,ignore
use cortex_m::singleton;

fn main() {
    // 仅当 `main` 只执行一次时 OK
    let x: &'static mut bool =
        singleton!(: bool = false).unwrap();
}
```

[cortex_m docs](https://docs.rs/cortex-m/latest/cortex_m/macro.singleton.html)

此外, 如果你使用 [`cortex-m-rtic`](https://github.com/rtic-rs/cortex-m-rtic), 定义和获取这些外设的整个过程都会为你抽象出来, 转而你会得到一个 `Peripherals` 结构, 其中包含你所定义的所有项目的非 `Option<T>` 版本.

```rust,ignore
// cortex-m-rtic v0.5.x
#[rtic::app(device = lm3s6965, peripherals = true)]
const APP: () = {
    #[init]
    fn init(cx: init::Context) {
        static mut X: u32 = 0;
         
        // Cortex-M 外设
        let core: cortex_m::Peripherals = cx.core;
        
        // 设备特定外设
        let device: lm3s6965::Peripherals = cx.device;
    }
}
```

## 但为什么呢?

但是这些单例 (Singletons) 如何在我们的 Rust 代码工作方式上产生明显的差异呢?

```rust,ignore
impl SerialPort {
    const SER_PORT_SPEED_REG: *mut u32 = 0x4000_1000 as _;

    fn read_speed(
        &self // <------ 这非常重要
    ) -> u32 {
        unsafe {
            ptr::read_volatile(Self::SER_PORT_SPEED_REG)
        }
    }
}
```

这里有两个重要因素:

* 因为我们使用的是单例, 所以只有一种方式或一个地方可以获得 `SerialPort` 结构
* 要调用 `read_speed()` 方法, 我们必须拥有 `SerialPort` 结构的所有权或引用

这两个因素合在一起意味着, 只有当我们适当地满足借用检查器时, 才可能访问硬件, 这意味着在任何一个时刻, 我们都不会有对同一硬件的多个可变引用!

```rust,ignore
fn main() {
    // 缺少 `self` 的引用! 无法工作.
    // SerialPort::read_speed();

    let serial_1 = unsafe { PERIPHERALS.take_serial() };

    // 你只能读你有权限读的内容
    let _ = serial_1.read_speed();
}
```

## 将硬件视为数据

此外, 因为有些引用是可变的, 有些是不可变的, 我们就可以看出函数或方法是否可能修改硬件的状态. 例如,

这是允许修改硬件设置的:

```rust,ignore
fn setup_spi_port(
    spi: &mut SpiPort,
    cs_pin: &mut GpioPin
) -> Result<()> {
    // ...
}
```

这是不允许的:

```rust,ignore
fn read_button(gpio: &GpioPin) -> bool {
    // ...
}
```

这允许我们在**编译时**强制约束代码是否应该修改硬件, 而不是在运行时. 需要注意的是, 这通常只在一个应用程序内有效, 但对于裸机 (bare metal) 系统, 我们的软件将编译成一个单一的应用程序, 所以这通常不是一个限制.
