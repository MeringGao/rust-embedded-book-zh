# `no_std` Rust 环境

"嵌入式编程" 这个术语涵盖了范围非常广泛的不同编程类别.
从仅有几 KB RAM 和 ROM 的 8 位 MCU (例如 [ST72325xx](https://www.st.com/resource/en/datasheet/st72325j6.pdf)),
到像 Raspberry Pi ([Model B 3+](https://en.wikipedia.org/wiki/Raspberry_Pi#Specifications)) 这样的系统:
它拥有 32/64 位 4 核 Cortex-A53 @ 1.4 GHz 和 1 GB RAM.
在编写代码时, 根据你的目标硬件和使用场景, 会受到不同的限制.

嵌入式编程大致分为两类:

## 有操作系统环境 (Hosted Environments)
这类环境接近普通 PC 环境.
也就是说, 你会获得一个系统接口 (例如 [POSIX](https://en.wikipedia.org/wiki/POSIX)),
它提供与各种系统交互的原语, 例如文件系统、网络、内存管理、线程等.
标准库通常依赖于这些原语来实现其功能.
你还可能拥有某种 sysroot 以及对 RAM/ROM 使用的限制, 也可能存在一些特殊的硬件或 I/O.
总体上感觉像是在一台专用 PC 环境下编码.

## 裸机环境 (Bare Metal Environments)
在裸机环境中, 你的程序之前没有任何代码被加载.
没有操作系统提供的软件, 我们无法加载标准库.
相反, 程序以及它所使用的 crate 只能使用硬件 (裸机) 来运行.
为了防止 Rust 加载标准库, 使用 `no_std`.
标准库中与平台无关的部分可以通过 [libcore](https://doc.rust-lang.org/core/) 获得.
libcore 还排除了一些在嵌入式环境中并非总是需要的东西.
其中之一就是用于动态内存分配的内存分配器.
如果你需要这个功能或任何其他功能, 通常都有相应的 crate 来提供它们.

### libstd 运行时
如前所述, 使用 [libstd](https://doc.rust-lang.org/std/) 需要某种系统集成, 但这不仅仅是因为
[libstd](https://doc.rust-lang.org/std/) 只是提供了一种访问操作系统抽象的通用方式, 它还提供了一个运行时.
该运行时除了其他职责外, 还负责设置栈溢出保护 (Stack Overflow Protection)、处理命令行参数,
以及在程序的主函数被调用之前派生出主线程. 该运行时在 `no_std` 环境中同样不可用.

## 小结
`#![no_std]` 是一个 crate 级别的属性 (Attribute), 表明该 crate 将链接到 core-crate 而不是 std-crate.
[libcore](https://doc.rust-lang.org/core/) crate 则是 std crate 的一个与平台无关的子集,
它对程序将要运行的系统不做任何假设.
因此, 它为语言原语 (如浮点数、字符串和切片) 提供了 API, 也为暴露处理器特性的 API
(如原子操作 (Atomic Operation) 和 SIMD 指令) 提供了 API. 然而, 它缺少任何涉及平台集成的 API.
由于这些特性, `no_std` 和 [libcore](https://doc.rust-lang.org/core/) 代码可以用于任何种类的
引导 (stage 0) 代码, 例如引导程序 (Bootloader)、固件 (Firmware) 或内核 (Kernel).

### 总览

| feature                                                   | no_std | std |
|-----------------------------------------------------------|--------|-----|
| heap (dynamic memory)                                     |   *    |  ✓  |
| collections (Vec, BTreeMap, etc)                          |  **    |  ✓  |
| stack overflow protection                                 |   ✘    |  ✓  |
| runs init code before main                                |   ✘    |  ✓  |
| libstd available                                          |   ✘    |  ✓  |
| libcore available                                         |   ✓    |  ✓  |
| writing firmware, kernel, or bootloader code              |   ✓    |  ✘  |

\* 只有当你使用 `alloc` crate 并搭配一个合适的分配器, 如 [alloc-cortex-m] 时才可用.

\** 只有当你使用 `collections` crate 并配置一个全局默认分配器时才可用.

\** HashMap 和 HashSet 不可用, 因为缺少安全的随机数生成器.

[alloc-cortex-m]: https://github.com/rust-embedded/alloc-cortex-m

## 参见
* [RFC-1184](https://github.com/rust-lang/rfcs/blob/master/text/1184-stabilize-no_std.md)
