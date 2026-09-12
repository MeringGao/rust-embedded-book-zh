# GPIO 接口建议

<a id="c-zst-pin"></a>
## 引脚类型默认为零大小类型 (C-ZST-PIN)

HAL 暴露的 GPIO 接口应该为每个接口或端口上的每个引脚提供专用的零大小类型, 从而在所有引脚分配都是静态已知的情况下实现零成本的 GPIO 抽象.

每个 GPIO 接口或端口都应该实现一个 `split` 方法, 返回一个包含所有引脚的结构体.

示例:

```rust
pub struct PA0;
pub struct PA1;
// ...

pub struct PortA;

impl PortA {
    pub fn split(self) -> PortAPins {
        PortAPins {
            pa0: PA0,
            pa1: PA1,
            // ...
        }
    }
}

pub struct PortAPins {
    pub pa0: PA0,
    pub pa1: PA1,
    // ...
}
```

<a id="c-erased-pin"></a>
## 引脚类型提供抹除引脚和端口的方法 (C-ERASED-PIN)

引脚应该提供类型抹除方法, 把它们的属性从编译期转移到运行时, 并允许应用程序具有更大的灵活性.

示例:

```rust
/// 端口 A, 引脚 0.
/// Port A, pin 0.
pub struct PA0;

impl PA0 {
    pub fn erase_pin(self) -> PA {
        PA { pin: 0 }
    }
}

/// 端口 A 上的一个引脚.
/// A pin on port A.
pub struct PA {
    /// 引脚编号
    /// The pin number.
    pin: u8,
}

impl PA {
    pub fn erase_port(self) -> Pin {
        Pin {
            port: Port::A,
            pin: self.pin,
        }
    }
}

pub struct Pin {
    port: Port,
    pin: u8,
    // (这些字段可以被打包以减少内存占用)
    // (these fields can be packed to reduce the memory footprint)
}

enum Port {
    A,
    B,
    C,
    D,
}
```

<a id="c-pin-state"></a>
## 引脚状态应编码为类型参数 (C-PIN-STATE)

引脚可以根据芯片或系列的不同被配置为输入或输出, 并具有不同的特性. 这种状态应该被编码到类型系统中, 以防止在错误的状态下使用引脚.

此外, 特定于芯片的状态 (例如驱动强度) 也可以通过这种方式编码, 使用额外的类型参数.

应该提供 `into_input` 和 `into_output` 方法来更改引脚状态.

此外, 还应该提供 `with_{input,output}_state` 方法, 用于在不移动引脚的情况下临时将引脚重新配置为另一种状态.

每个引脚类型 (也就是说, 被抹除和未被抹除的引脚类型都应该提供相同的 API) 都应提供以下方法:

* `pub fn into_input<N: InputState>(self, input: N) -> Pin<N>`
* `pub fn into_output<N: OutputState>(self, output: N) -> Pin<N>`
* ```ignore
  pub fn with_input_state<N: InputState, R>(
      &mut self,
      input: N,
      f: impl FnOnce(&mut PA1<N>) -> R,
  ) -> R
  ```
* ```ignore
  pub fn with_output_state<N: OutputState, R>(
      &mut self,
      output: N,
      f: impl FnOnce(&mut PA1<N>) -> R,
  ) -> R
  ```


引脚状态应该受到封闭 trait (Sealed Trait) 的限制. HAL 的用户应该不需要添加自己的状态. 这些 trait 可以提供实现引脚状态 API 所需的 HAL 特定方法.

示例:

```rust
# use std::marker::PhantomData;
mod sealed {
    pub trait Sealed {}
}

pub trait PinState: sealed::Sealed {}
pub trait OutputState: sealed::Sealed {}
pub trait InputState: sealed::Sealed {
    // ...
}

pub struct Output<S: OutputState> {
    _p: PhantomData<S>,
}

impl<S: OutputState> PinState for Output<S> {}
impl<S: OutputState> sealed::Sealed for Output<S> {}

pub struct PushPull;
pub struct OpenDrain;

impl OutputState for PushPull {}
impl OutputState for OpenDrain {}
impl sealed::Sealed for PushPull {}
impl sealed::Sealed for OpenDrain {}

pub struct Input<S: InputState> {
    _p: PhantomData<S>,
}

impl<S: InputState> PinState for Input<S> {}
impl<S: InputState> sealed::Sealed for Input<S> {}

pub struct Floating;
pub struct PullUp;
pub struct PullDown;

impl InputState for Floating {}
impl InputState for PullUp {}
impl InputState for PullDown {}
impl sealed::Sealed for Floating {}
impl sealed::Sealed for PullUp {}
impl sealed::Sealed for PullDown {}

pub struct PA1<S: PinState> {
    _p: PhantomData<S>,
}

impl<S: PinState> PA1<S> {
    pub fn into_input<N: InputState>(self, input: N) -> PA1<Input<N>> {
        todo!()
    }

    pub fn into_output<N: OutputState>(self, output: N) -> PA1<Output<N>> {
        todo!()
    }

    pub fn with_input_state<N: InputState, R>(
        &mut self,
        input: N,
        f: impl FnOnce(&mut PA1<N>) -> R,
    ) -> R {
        todo!()
    }

    pub fn with_output_state<N: OutputState, R>(
        &mut self,
        output: N,
        f: impl FnOnce(&mut PA1<N>) -> R,
    ) -> R {
        todo!()
    }
}

// 同样适用于 `PA`, `Pin` 和其他引脚类型.
// Same for `PA` and `Pin`, and other pin types.
```
