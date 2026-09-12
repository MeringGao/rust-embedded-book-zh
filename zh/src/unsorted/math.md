# 使用 `#[no_std]` 执行数学功能

如果你想执行与数学相关的功能, 例如计算一个数的平方根或指数, 并且你可以使用完整标准库, 你的代码可能如下所示:

```rs
//! 一些具有标准支持的数学函数

fn main() {
    let float: f32 = 4.82832;
    let floored_float = float.floor();

    let sqrt_of_four = floored_float.sqrt();

    let sinus_of_four = floored_float.sin();

    let exponential_of_four = floored_float.exp();
    println!("Floored test float {} to {}", float, floored_float);
    println!("The square root of {} is {}", floored_float, sqrt_of_four);
    println!("The sinus of four is {}", sinus_of_four);
    println!(
        "The exponential of four to the base e is {}",
        exponential_of_four
    )
}
```

在没有标准库支持的情况下, 这些函数不可用.
可以改用 [`libm`](https://crates.io/crates/libm) 之类的外部 crate. 然后示例代码将如下所示:

```rs
#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::{debug, hprintln};
use libm::{exp, floorf, sin, sqrtf};

#[entry]
fn main() -> ! {
    let float = 4.82832;
    let floored_float = floorf(float);

    let sqrt_of_four = sqrtf(floored_float);

    let sinus_of_four = sin(floored_float.into());

    let exponential_of_four = exp(floored_float.into());
    hprintln!("Floored test float {} to {}", float, floored_float).unwrap();
    hprintln!("The square root of {} is {}", floored_float, sqrt_of_four).unwrap();
    hprintln!("The sinus of four is {}", sinus_of_four).unwrap();
    hprintln!(
        "The exponential of four to the base e is {}",
        exponential_of_four
    )
    .unwrap();
    // 退出 QEMU
    // 注意不要在硬件上运行此命令; 它可能会损坏 OpenOCD 状态
    // debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

如果你需要在 MCU 上执行更复杂的操作, 例如 DSP 信号处理或高级线性代数, 以下的 crate 可能会有所帮助

- [CMSIS DSP 库绑定](https://github.com/jacobrosenthal/cmsis-dsp-sys)
- [`constgebra`](https://crates.io/crates/constgebra)
- [`micromath`](https://github.com/tarcieri/micromath)
- [`microfft`](https://crates.io/crates/microfft)
- [`nalgebra`](https://github.com/dimforge/nalgebra)
