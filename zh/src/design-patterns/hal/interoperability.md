# 互操作性 (Interoperability)


<a id="c-free"></a>
## 包装类型提供析构方法 (C-FREE)

HAL 提供的任何非 `Copy` 包装类型都应该提供一个 `free` 方法, 该方法消耗包装器并返回原始外设 (以及可能的其他对象), 原始外设正是创建该包装器所用的对象.

如果必要, 该方法应该关闭并复位外设. 用 `free` 返回的原始外设再次调用 `new` 时, 不应该因为外设处于意外状态而失败.

如果 HAL 类型需要其他非 `Copy` 对象才能构造 (例如 I/O 引脚), 那么任何这样的对象也应该由 `free` 释放并返回. 在这种情况下, `free` 应该返回一个元组.

例如:

```rust
# pub struct TIMER0;
pub struct Timer(TIMER0);

impl Timer {
    pub fn new(periph: TIMER0) -> Self {
        Self(periph)
    }

    pub fn free(self) -> TIMER0 {
        self.0
    }
}
```

<a id="c-reexport-pac"></a>
## HAL 重新导出它们的寄存器访问 crate (C-REEXPORT-PAC)

HAL 既可以基于 [svd2rust] 生成的 PAC 编写, 也可以基于提供原始寄存器访问的其他 crate 编写. HAL 应该始终在 crate 根目录重新导出它们所基于的寄存器访问 crate.

PAC 应该以 `pac` 的名称被重新导出, 而与 crate 的实际名称无关, 因为 HAL 的名称本身应该已经清楚地表明正在访问哪个 PAC.

[svd2rust]: https://github.com/rust-embedded/svd2rust

<a id="c-hal-traits"></a>
## 类型实现 `embedded-hal` 的 trait (C-HAL-TRAITS)

HAL 提供的类型应该实现 [`embedded-hal`] crate 提供的所有适用 trait.

可以为同一个类型实现多个 trait.

[`embedded-hal`]: https://github.com/rust-embedded/embedded-hal
