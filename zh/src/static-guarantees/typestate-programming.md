# 类型状态编程 (Typestate Programming)

[类型状态 (typestates)] 的概念描述了将对象的当前状态信息编码到对象的类型中. 虽然这听起来可能有点深奥, 但如果你在 Rust 中使用过 [构建器模式 (Builder Pattern)], 你就已经开始使用类型状态编程 (Typestate Programming) 了!

[typestates]: https://en.wikipedia.org/wiki/Typestate_analysis
[Builder Pattern]: https://doc.rust-lang.org/1.0.0/style/ownership/builders.html

```rust
pub mod foo_module {
    #[derive(Debug)]
    pub struct Foo {
        inner: u32,
    }

    pub struct FooBuilder {
        a: u32,
        b: u32,
    }

    impl FooBuilder {
        pub fn new(starter: u32) -> Self {
            Self {
                a: starter,
                b: starter,
            }
        }

        pub fn double_a(self) -> Self {
            Self {
                a: self.a * 2,
                b: self.b,
            }
        }

        pub fn into_foo(self) -> Foo {
            Foo {
                inner: self.a + self.b,
            }
        }
    }
}

fn main() {
    let x = foo_module::FooBuilder::new(10)
        .double_a()
        .into_foo();

    println!("{:#?}", x);
}
```

在这个例子中, 没有直接的方法创建 `Foo` 对象. 我们必须创建一个 `FooBuilder`, 并在能够获得我们想要的 `Foo` 对象之前对其进行适当的初始化.

这个最小示例编码了两种状态:

* `FooBuilder`, 表示"未配置"或"正在配置"的状态
* `Foo`, 表示"已配置"或"可以使用"的状态.

## 强类型 (Strong Types)

因为 Rust 有一个 [强类型系统 (Strong Type System)], 所以没有简单的方法可以神奇地创建 `Foo` 的实例, 或者在不调用 `into_foo()` 方法的情况下将 `FooBuilder` 转换为 `Foo`. 此外, 调用 `into_foo()` 方法会消耗原始的 `FooBuilder` 结构, 这意味着它不能在不创建新实例的情况下被重用.

[Strong Type System]: https://en.wikipedia.org/wiki/Strong_and_weak_typing

这允许我们将系统的状态表示为类型, 并将状态转换所需的操作纳入交换一个类型为另一个类型的方法中. 通过创建一个 `FooBuilder`, 并将其换为一个 `Foo` 对象, 我们已经走完了一个基本状态机 (State Machine) 的步骤.
