# 静态保证 (Static Guarantees)

Rust 的类型系统在编译时防止数据竞争 (见 [`Send`] 和 [`Sync`] trait). 该类型系统还可以用于在编译时检查其他属性; 在某些情况下减少对运行时检查的需要.

[`Send`]: https://doc.rust-lang.org/core/marker/trait.Send.html
[`Sync`]: https://doc.rust-lang.org/core/marker/trait.Sync.html

当应用于嵌入式程序时, 这些*静态检查 (Static Checks)* 可用于例如强制 I/O 接口的正确配置. 例如, 可以设计这样一个 API: 只有先配置好接口将要使用的引脚, 才能初始化串行接口.

我们还可以静态地检查某些操作 (例如将引脚拉低) 只能在正确配置的外设上执行. 例如, 试图更改处于浮空输入模式的引脚的输出状态, 将会引发编译错误.

并且, 如前一章所述, 所有权的概念可以应用于外设, 以确保程序的某些特定部分才能修改某个外设. 这种*访问控制 (Access Control)* 比将外设视为全局可变状态的替代方案更易于推理.

[embedded-hal]: https://crates.io/crates/embedded-hal
