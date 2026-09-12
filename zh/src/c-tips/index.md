# 给嵌入式 C 开发者的建议

本章收集了可能有用的各种建议, 适合希望开始编写 Rust 的资深嵌入式 C 开发者. 它将特别强调你在 C 中已经习惯的东西在 Rust 中有何不同.

## 预处理器 (Preprocessor)

在嵌入式 C 中, 非常常见的是把预处理器 (Preprocessor) 用于各种目的, 例如:

* 使用 `#ifdef` 进行编译期代码块选择
* 编译期数组大小和计算
* 用宏来简化常见模式 (以避免函数调用开销)

在 Rust 中没有预处理器, 因此这些用例中的许多会以不同的方式处理. 在本节的其余部分, 我们将介绍一些不使用预处理器的替代方案.

### 编译期代码选择

Rust 中最接近 `#ifdef ... #endif` 的是 [Cargo features]. 它们比 C 预处理器更正式一些: 所有可能的特性都在每个 crate 中显式列出, 并且只能是启用或关闭两种状态. 当你将一个 crate 列为依赖项时, 这些特性就会被打开, 并且具有叠加性: 如果你的依赖树中的任何 crate 为另一个 crate 启用了某个特性, 那么该特性将对所有该 crate 的使用者启用.

[Cargo features]: https://doc.rust-lang.org/cargo/reference/manifest.html#the-features-section

例如, 你可能有一个 crate 提供信号处理原语的库. 每个原语可能需要额外的编译时间, 或者声明一个你想要避免的大常量表. 你可以在 `Cargo.toml` 中为每个组件声明一个 Cargo feature:

```toml
[features]
FIR = []
IIR = []
```

然后, 在你的代码中, 使用 `#[cfg(feature="FIR")]` 来控制包含哪些代码.

```rust
/// 在你的顶层 lib.rs 中
/// In your top-level lib.rs

#[cfg(feature="FIR")]
pub mod fir;

#[cfg(feature="IIR")]
pub mod iir;
```

你也可以类似地只在某个 feature *未*被启用时, 或者只在特性的某种组合启用或未启用时, 才包含代码块.

此外, Rust 还提供了许多可以使用的自动设置的条件, 例如 `target_arch`, 它可以根据架构选择不同的代码. 有关条件编译支持的完整细节, 请参阅 Rust 参考手册的 [条件编译] 章节.

[条件编译]: https://doc.rust-lang.org/reference/conditional-compilation.html

条件编译只会应用于下一条语句或代码块. 如果当前作用域中无法使用代码块, 则需要多次使用 `cfg` 属性. 值得注意的是, 在大多数情况下, 更好的做法是直接包含所有代码, 然后让编译器在优化时删除死代码: 这对你和你的用户来说都更简单, 而且一般来说, 编译器会很好地删除未使用的代码.

### 编译期大小和计算

Rust 支持 `const fn`, 即保证可以在编译期 (Compile Time) 求值, 因此可以在需要常量的地方使用, 例如数组的大小. 这可以与上面提到的特性一起使用, 例如:

```rust
const fn array_size() -> usize {
    #[cfg(feature="use_more_ram")]
    { 1024 }
    #[cfg(not(feature="use_more_ram"))]
    { 128 }
}

static BUF: [u32; array_size()] = [0u32; array_size()];
```

这些是自稳定版 Rust 1.31 以来新增的特性, 因此相关文档仍然不多. 在撰写本文时, `const fn` 中可用的功能也非常有限; 预计在未来的 Rust 版本中, 它将在 `const fn` 中允许的内容上进行扩展.

### 宏

Rust 提供了一个非常强大的 [宏系统]. C 预处理器几乎直接作用于源代码的文本, 而 Rust 的宏系统在更高的层次上工作. Rust 有两种宏: *示例宏 (macros by example)* 和 *过程宏 (procedural macros)*. 前者更简单, 也更常见; 它们看起来像函数调用, 可以展开为完整的表达式, 语句, 项 (item) 或模式. 过程宏更复杂, 但允许对 Rust 语言进行极其强大的扩展: 它们可以将任意 Rust 语法转换为新的 Rust 语法.

[宏系统]: https://doc.rust-lang.org/book/ch19-06-macros.html

一般来说, 在你以前可能使用 C 预处理器宏的地方, 你可能想看看是否可以使用示例宏来代替. 它们可以在你的 crate 中定义, 也可以由你的 crate 轻松使用, 或者导出供其他用户使用. 请注意, 由于它们必须展开为完整的表达式, 语句, 项或模式, 因此某些 C 预处理器宏的用例将无法工作, 例如展开为变量名的一部分或列表中不完整项集的宏.

与 Cargo features 一样, 值得考虑的是你是否真的需要这个宏. 在许多情况下, 普通函数更易于理解, 并且会被内联为与宏相同的代码. `#[inline]` 和 `#[inline(always)]` [属性] 让你可以进一步控制这个过程, 但这里也需要谨慎 — 编译器会在适当的时候自动内联同一 crate 中的函数, 因此在不适当的情况下强制内联实际上可能导致性能下降.

[属性]: https://doc.rust-lang.org/reference/attributes.html#inline-attribute

解释整个 Rust 宏系统超出了本提示页面的范围, 因此建议你查阅 Rust 文档以获取完整的细节.

## 构建系统 (Build System)

大多数 Rust crate 使用 Cargo 构建 (尽管这不是必需的). Cargo 处理了传统构建系统中的许多难题. 但是, 你可能希望自定义构建过程. Cargo 为此提供了 [`build.rs` 脚本]. 它们是可以根据需要与 Cargo 构建系统交互的 Rust 脚本.

[`build.rs` 脚本]: https://doc.rust-lang.org/cargo/reference/build-scripts.html

构建脚本的常见用例包括:

* 提供构建时信息, 例如将构建日期或 Git 提交哈希静态嵌入到可执行文件中
* 根据所选特性或其他逻辑在构建时生成链接脚本
* 更改 Cargo 构建配置
* 添加要链接的额外静态库

目前还没有对构建后脚本的支持, 你以前可能用它们来执行从构建对象自动生成二进制文件或打印构建信息等任务.

### 交叉编译 (Cross-Compiling)

使用 Cargo 作为构建系统也简化了交叉编译 (Cross-Compiling). 在大多数情况下, 只需告诉 Cargo `--target thumbv6m-none-eabi` 并在 `target/thumbv6m-none-eabi/debug/myapp` 中找到合适的可执行文件即可.

对于 Rust 本身不原生支持的平台, 你需要自己为该目标构建 `libcore`. 在这样的平台上, [Xargo] 可以用作 Cargo 的替代品, 自动为你构建 `libcore`.

[Xargo]: https://github.com/japaric/xargo

## 迭代器 vs 数组访问

在 C 中, 你可能习惯于直接通过索引访问数组:

```c
int16_t arr[16];
int i;
for(i=0; i<sizeof(arr)/sizeof(arr[0]); i++) {
    process(arr[i]);
}
```

在 Rust 中, 这是一种反模式: 索引访问可能更慢 (因为它需要边界检查), 并且可能妨碍编译器的各种优化. 这是一个重要的区别, 值得重复一遍: Rust 会对手动数组索引进行越界检查, 以保证内存安全, 而 C 则会很乐意地索引到数组之外.

相反, 请使用迭代器:

```rust,ignore
let arr = [0u16; 16];
for element in arr.iter() {
    process(*element);
}
```

迭代器提供了一系列强大的功能, 这些功能你在 C 中必须手动实现, 例如链式调用 (chaining), 拉链 (zipping), 枚举 (enumerating), 查找最小值或最大值, 求和等等. 迭代器方法也可以链式调用, 从而产生非常易读的数据处理代码.

更多详细信息, 请参阅 [Rust Book 中的迭代器] 和 [迭代器文档].

[Rust Book 中的迭代器]: https://doc.rust-lang.org/book/ch13-02-iterators.html
[迭代器文档]: https://doc.rust-lang.org/core/iter/trait.Iterator.html

## 引用 vs 指针

在 Rust 中, 指针 (称为[*裸指针 (raw pointer)*]) 虽然存在, 但仅在特定情况下使用, 因为解引用它们始终被认为是 `unsafe` 的 — Rust 无法提供其关于指针背后可能是什么的常规保证.

[*裸指针*]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer

在大多数情况下, 我们改用*引用 (Reference)* — 用 `&` 符号表示 — 或者*可变引用 (Mutable Reference)* — 用 `&mut` 表示. 引用的行为类似于指针, 因为它们可以被解引用以访问底层值, 但它们是 Rust 所有权系统的关键部分: Rust 将严格执行, 在任何给定时刻, 你对同一个值只能拥有一个可变引用, *或*多个不可变引用.

实际上, 这意味着你必须更加小心是否需要对数据的可变访问: 在 C 中默认值是可变的, 你必须显式声明 `const`, 而在 Rust 中则相反.

仍然可能使用裸指针的一种情况是直接与硬件交互 (例如, 将指向缓冲区的指针写入 DMA 外设的寄存器), 而且所有外设访问 crate 在底层也使用它们来允许你读写内存映射的寄存器.

## 易变访问 (Volatile Access)

在 C 中, 单个变量可以被标记为 `volatile`, 这向编译器表明变量的值可能在两次访问之间发生改变. 在嵌入式环境中, 易变 (Volatile) 变量通常用于内存映射的寄存器.

在 Rust 中, 我们不会将变量标记为 `volatile`, 而是使用特定的方法来执行易变访问: [`core::ptr::read_volatile`] 和 [`core::ptr::write_volatile`]. 这些方法接受 `*const T` 或 `*mut T` (如上所述, 是裸指针), 并执行易变读取或写入.

[`core::ptr::read_volatile`]: https://doc.rust-lang.org/core/ptr/fn.read_volatile.html
[`core::ptr::write_volatile`]: https://doc.rust-lang.org/core/ptr/fn.write_volatile.html

例如, 在 C 中你可能会这样写:

```c
volatile bool signalled = false;

void ISR() {
    // 标记中断已发生
    // Signal that the interrupt has occurred
    signalled = true;
}

void driver() {
    while(true) {
        // 休眠直到收到信号
        // Sleep until signalled
        while(!signalled) { WFI(); }
        // 重置已收到信号指示
        // Reset signalled indicator
        signalled = false;
        // 执行等待该中断的某个任务
        // Perform some task that was waiting for the interrupt
        run_task();
    }
}
```

在 Rust 中, 等价的代码会在每次访问时使用易变方法:

```rust,ignore
static mut SIGNALLED: bool = false;

#[interrupt]
fn ISR() {
    // 标记中断已发生
    // (在实际代码中, 你应该考虑更高级的原语,
    //  例如原子类型).
    // Signal that the interrupt has occurred
    // (In real code, you should consider a higher level primitive,
    //  such as an atomic type).
    unsafe { core::ptr::write_volatile(&mut SIGNALLED, true) };
}

fn driver() {
    loop {
        // 休眠直到收到信号
        // Sleep until signalled
        while unsafe { !core::ptr::read_volatile(&SIGNALLED) } {}
        // 重置已收到信号指示
        // Reset signalled indicator
        unsafe { core::ptr::write_volatile(&mut SIGNALLED, false) };
        // 执行等待该中断的某个任务
        // Perform some task that was waiting for the interrupt
        run_task();
    }
}
```

该代码示例中有几点值得注意:
  * 我们可以将 `&mut SIGNALLED` 传递给要求 `*mut T` 的函数,
    因为 `&mut T` 会自动转换为 `*mut T` (对 `*const T` 也是一样)
  * `read_volatile` / `write_volatile` 方法是 `unsafe` 函数,
    因此我们需要 `unsafe` 块. 确保安全使用是程序员的责任:
    有关更多详细信息, 请参阅这些方法的文档.

在代码中直接需要这些函数的情况很少见, 因为它们通常由更高级的库为你处理. 对于内存映射的外设, 外设访问 crate 会自动实现易变访问; 而对于并发原语, 则有更好的抽象可用 (参见 [并发章节]).

[并发章节]: ../concurrency/index.md

## 紧凑和对齐类型

在嵌入式 C 中, 通常会告诉编译器一个变量必须具有某种对齐方式, 或者一个结构体必须是紧凑 (Packed) 而不是对齐的, 这通常是为了满足特定的硬件或协议要求.

在 Rust 中, 这由结构体或联合体上的 `repr` 属性控制. 默认的表示形式不提供布局保证, 因此不应该用于与硬件或 C 互操作的代码. 编译器可能会重新排序结构体成员或插入填充, 并且这种行为可能会随 Rust 未来的版本而改变.

```rust
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
}

// 0x7ffecb3511d0 0x7ffecb3511d4 0x7ffecb3511d2
// 注意顺序已被更改为 x, z, y 以改善打包效果.
// Note ordering has been changed to x, z, y to improve packing.
```

要确保与 C 互操作的布局, 请使用 `repr(C)`:

```rust
#[repr(C)]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
}

// 0x7fffd0d84c60 0x7fffd0d84c62 0x7fffd0d84c64
// 顺序被保留, 布局也不会随时间改变.
// `z` 是两字节对齐, 因此 `y` 和 `z` 之间存在一个字节的填充.
// Ordering is preserved and the layout will not change over time.
// `z` is two-byte aligned so a byte of padding exists between `y` and `z`.
```

要确保紧凑表示, 请使用 `repr(packed)`:

```rust
#[repr(packed)]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    // 引用必须始终对齐, 因此要检查结构体字段的地址,
    // 我们使用 `std::ptr::addr_of!()` 来获取裸指针,
    // 而不是仅仅打印 `&v.x`.
    // References must always be aligned, so to check the addresses of the
    // struct's fields, we use `std::ptr::addr_of!()` to get a raw pointer
    // instead of just printing `&v.x`.
    let px = std::ptr::addr_of!(v.x);
    let py = std::ptr::addr_of!(v.y);
    let pz = std::ptr::addr_of!(v.z);
    println!("{:p} {:p} {:p}", px, py, pz);
}

// 0x7ffd33598490 0x7ffd33598492 0x7ffd33598493
// `y` 和 `z` 之间没有插入填充, 因此 `z` 现在是未对齐的.
// No padding has been inserted between `y` and `z`, so now `z` is unaligned.
```

请注意, 使用 `repr(packed)` 也会将该类型的对齐方式设置为 `1`.

最后, 要指定特定的对齐方式, 请使用 `repr(align(n))`, 其中 `n` 是要对齐到的字节数 (且必须是 2 的幂):

```rust
#[repr(C)]
#[repr(align(4096))]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    let u = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
    println!("{:p} {:p} {:p}", &u.x, &u.y, &u.z);
}

// 0x7ffec909a000 0x7ffec909a002 0x7ffec909a004
// 0x7ffec909b000 0x7ffec909b002 0x7ffec909b004
// `u` 和 `v` 这两个实例已被放置在 4096 字节的对齐位置上,
// 这可以从地址末尾的 `000` 中看出.
// The two instances `u` and `v` have been placed on 4096-byte alignments,
// evidenced by the `000` at the end of their addresses.
```

请注意, 我们可以将 `repr(C)` 与 `repr(align(n))` 结合使用, 以获得既对齐又与 C 兼容的布局. 不允许将 `repr(align(n))` 与 `repr(packed)` 结合使用, 因为 `repr(packed)` 将对齐方式设置为 `1`. `repr(packed)` 类型中包含 `repr(align(n))` 类型也是不允许的.

有关类型布局的更多详细信息, 请参阅 Rust 参考手册的 [类型布局] 章节.

[类型布局]: https://doc.rust-lang.org/reference/type-layout.html

## 其他资源

* 在本书中:
    * [Rust 配合少量 C](../interoperability/c-with-rust.md)
    * [C 配合少量 Rust](../interoperability/rust-with-c.md)
* [Rust Embedded 常见问题解答](https://docs.rust-embedded.org/faq.html)
* [Rust Pointers for C Programmers](http://blahg.josefsipek.net/?p=580)
* [I used to use pointers - now what?](https://github.com/diwic/reffers-rs/blob/master/docs/Pointers.md)
