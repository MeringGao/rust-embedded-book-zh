# 优化: 速度与大小的权衡

每个人都希望他们的程序既超级快又超级小, 但通常不可能同时具有这两个特性.
本节讨论 `rustc` 提供的不同优化级别, 以及它们如何影响程序的执行时间和二进制大小.

## 不进行优化

这是默认设置. 当你调用 `cargo build` 时, 你使用的是开发 (也叫 `dev`) profile. 此 profile 针对调试进行了优化, 因此它启用调试信息并且*不*启用任何优化, 也就是说, 它使用 `-C opt-level=0`.

至少对于裸机 (bare metal) 开发来说, 调试信息 (debuginfo) 在某种意义上是零成本的, 因为它不会占用 Flash / ROM 中的空间, 因此我们实际上建议你在 release profile 中启用 debuginfo -- 它默认是禁用的. 这将使你在调试 release 构建时能够使用断点.

``` toml
[profile.release]
# 符号信息很好, 而且不会增加 Flash 上的大小
debug = true
```

不优化非常适合调试, 因为单步执行代码时感觉像是逐条语句执行程序, 并且你可以在 GDB 中 `print` 栈变量和函数参数. 当代码被优化时, 尝试打印变量会导致打印出 `$0 = <value optimized out>`.

`dev` profile 最大的缺点是生成的二进制文件会很大且运行缓慢. 大小通常是一个更大的问题, 因为未优化的二进制文件可能占用几十 KiB 的 Flash, 而你的目标设备可能没有这么多 -- 结果: 你的未优化二进制文件无法装入你的设备!

我们能得到更小的, 方便调试的二进制文件吗? 可以, 有个技巧.

### 优化依赖项

Cargo 有一个名为 [`profile-overrides`] 的功能, 可以让你覆盖依赖项的优化级别. 你可以使用该功能在优化所有依赖项大小的同时, 保持顶层 crate 未优化且方便调试.

请注意, 泛型代码有时可以与其实例化的 crate 一起优化, 而不是在定义它的 crate 中优化. 如果你在应用程序中创建了一个泛型结构体的实例, 并发现它引入了占用空间很大的代码, 那可能是因为提高相关依赖项的优化级别没有效果.

[`profile-overrides`]: https://doc.rust-lang.org/cargo/reference/profiles.html#overrides

这是一个示例:

``` toml
# Cargo.toml
[package]
name = "app"
# ..

[profile.dev.package."*"] # +
opt-level = "z" # +
```

未使用 override 时:

``` text
$ cargo size --bin app -- -A
app  :
section               size        addr
.vector_table         1024   0x8000000
.text                 9060   0x8000400
.rodata               1708   0x8002780
.data                    0  0x20000000
.bss                     4  0x20000000
```

使用 override 时:

``` text
$ cargo size --bin app -- -A
app  :
section               size        addr
.vector_table         1024   0x8000000
.text                 3490   0x8000400
.rodata               1100   0x80011c0
.data                    0  0x20000000
.bss                     4  0x20000000
```

这样 Flash 使用量减少了 6 KiB, 而且顶层 crate 的可调试性没有任何损失. 如果你单步执行到依赖项中, 那么你将再次开始看到那些 `<value optimized out>` 消息, 但通常情况下, 你要调试的是顶层 crate, 而不是依赖项. 而如果你*确实*需要调试某个依赖项, 那么你可以使用 `profile-overrides` 功能将特定依赖项从优化中排除. 参见下面的示例:

``` toml
# ..

# 不优化 `cortex-m-rt` crate
[profile.dev.package.cortex-m-rt] # +
opt-level = 0 # +

# 但优化所有其他依赖项
[profile.dev.package."*"]
codegen-units = 1 # 更好的优化
opt-level = "z"
```

现在, 顶层 crate 和 `cortex-m-rt` 都是方便调试的了!

## 针对速度进行优化

截至 2018-09-18, `rustc` 支持三个 "针对速度优化" 级别: `opt-level=1`, `2` 和 `3`.
当你运行 `cargo build --release` 时, 你使用的是 release profile, 它默认为 `opt-level=3`.

`opt-level=2` 和 `3` 都以牺牲二进制大小为代价针对速度进行了优化, 但级别 `3` 比级别 `2` 进行更多的向量化 (Vectorization) 和内联 (Inlining). 特别是, 你会看到, 当 `opt-level` 大于等于 `2` 时, LLVM 会展开循环 (Loop Unrolling). 循环展开在 Flash / ROM 方面成本相当高 (例如, 对于一个清零数组的循环, 从 26 字节增加到 194 字节), 但在合适的条件下 (例如, 迭代次数足够大) 也可以将执行时间减半.

目前在 `opt-level=2` 和 `3` 中无法禁用循环展开, 因此, 如果你负担不起其成本, 则应针对大小优化你的程序.

## 针对大小进行优化

截至 2018-09-18, `rustc` 支持两个 "针对大小优化" 级别: `opt-level="s"` 和 `"z"`. 这些名称是从 clang / LLVM 继承而来的, 描述性不是很强, 但 `"z"` 意在传达它生成的二进制文件比 `"s"` 更小的概念.

如果你希望 release 二进制文件针对大小进行了优化, 请更改 `Cargo.toml` 中的 `profile.release.opt-level` 设置, 如下所示.

``` toml
[profile.release]
# 或 "z"
opt-level = "s"
```

这两个优化级别大大降低了 LLVM 的内联阈值 (Inline Threshold), 这是用于决定是否内联函数的指标.
Rust 的原则之一是零成本抽象 (Zero Cost Abstraction); 这些抽象往往使用大量的 newtype 和小函数来保持不变式 (Invariant) (例如, 借用内部值的函数, 如 `deref`, `as_ref`), 因此较低的内联阈值可能会让 LLVM 错失优化机会 (例如, 消除死分支, 内联对闭包的调用).

在针对大小进行优化时, 你可能希望尝试提高内联阈值, 以查看是否对二进制大小有任何影响. 更改内联阈值的推荐方法是将 `-C inline-threshold` 标志附加到 `.cargo/config.toml` 中的其他 rustflags 中.

``` toml
# .cargo/config.toml
# 这假设你正在使用 cortex-m-quickstart 模板
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
rustflags = [
  # ..
  "-C", "inline-threshold=123", # +
]
```

使用什么值? [截至 1.29.0, 这些是不同优化级别使用的内联阈值][inline-threshold]:

[inline-threshold]: https://github.com/rust-lang/rust/blob/1.29.0/src/librustc_codegen_llvm/back/write.rs#L2105-L2122

- `opt-level=3` 使用 275
- `opt-level=2` 使用 225
- `opt-level="s"` 使用 75
- `opt-level="z"` 使用 25

在针对大小进行优化时, 你应该尝试 `225` 和 `275`.
