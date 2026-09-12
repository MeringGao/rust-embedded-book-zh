# 零成本抽象 (Zero Cost Abstraction)

类型状态 (Typestate) 也是零成本抽象 (Zero Cost Abstraction) 的一个很好的例子 — 零成本抽象指的是把某些行为移到编译期执行或分析的能力. 这些类型状态不包含任何实际数据, 而是作为标记 (Marker) 使用. 由于它们不包含数据, 因此在运行时内存中没有实际的表示:

```rust,ignore
use core::mem::size_of;

let _ = size_of::<Enabled>();    // == 0
let _ = size_of::<Input>();      // == 0
let _ = size_of::<PulledHigh>(); // == 0
let _ = size_of::<GpioConfig<Enabled, Input, PulledHigh>>(); // == 0
```

## 零大小类型 (Zero Sized Type)

```rust,ignore
struct Enabled;
```

像这样定义的结构体被称为零大小类型 (Zero Sized Type, ZST), 因为它们不包含任何实际数据. 虽然这些类型在编译期表现得"真实存在" — 你可以复制它们, 移动它们, 获取对它们的引用等等 — 但是优化器会彻底把它们抹掉.

在这段代码中:

```rust,ignore
pub fn into_input_high_z(self) -> GpioConfig<Enabled, Input, HighZ> {
    self.periph.modify(|_r, w| w.input_mode().high_z());
    GpioConfig {
        periph: self.periph,
        enabled: Enabled,
        direction: Input,
        mode: HighZ,
    }
}
```

我们返回的 `GpioConfig` 在运行时根本不存在. 调用此函数通常会简化为一条汇编指令 — 把一个常量寄存器值存储到寄存器位置. 这意味着我们开发的类型状态接口是一种零成本抽象 — 跟踪 `GpioConfig` 状态不会消耗更多的 CPU, RAM 或代码空间, 编译出的机器码与直接访问寄存器完全相同.

## 嵌套 (Nesting)

一般来说, 这些抽象可以按你希望的任何深度进行嵌套. 只要所有使用的组件都是零大小类型, 整个结构在运行时就不会存在.

对于复杂或深层嵌套的结构, 定义所有可能的状态组合可能很繁琐. 在这些情况下, 可以使用宏来生成所有的实现.
