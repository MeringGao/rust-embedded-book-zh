# 在 Rust 中使用少量 C 代码

在 Rust 项目中使用 C 或 C++ 主要由两部分组成:

- 将暴露的 C API 包装起来供 Rust 使用
- 构建你的 C 或 C++ 代码, 以便与 Rust 代码集成

由于 C++ 没有供 Rust 编译器面向的稳定 ABI (Application Binary Interface, 应用二进制接口),
建议在将 Rust 与 C 或 C++ 结合时使用 `C` ABI.

## 定义接口

在 Rust 中使用 C 或 C++ 代码之前, 有必要 (在 Rust 中) 定义链接代码中存在的数据类型和函数签名.
在 C 或 C++ 中, 你会包含一个定义这些数据的头文件 (`.h` 或 `.hpp`).
而在 Rust 中, 则需要手动将这些定义翻译为 Rust, 或者使用工具自动生成这些定义.

首先, 我们将介绍如何手动将定义从 C/C++ 翻译到 Rust.

### 包装 C 函数和数据类型

通常, 用 C 或 C++ 编写的库会提供一个头文件, 用于定义公共接口中使用的所有类型和函数. 示例文件可能如下所示:

```C
/* File: cool.h */
typedef struct CoolStruct {
    int x;
    int y;
} CoolStruct;

void cool_function(int i, char c, CoolStruct* cs);
```

当翻译到 Rust 时, 这个接口看起来像这样:

```rust,ignore
/* File: cool_bindings.rs */
#[repr(C)]
pub struct CoolStruct {
    pub x: cty::c_int,
    pub y: cty::c_int,
}

extern "C" {
    pub fn cool_function(
        i: cty::c_int,
        c: cty::c_char,
        cs: *mut CoolStruct
    );
}
```

让我们逐段查看这个定义, 以解释其中的各个部分.

```rust,ignore
#[repr(C)]
pub struct CoolStruct { ... }
```

默认情况下, Rust 不保证 `struct` 中数据的顺序, 填充或大小.
为了保证与 C 代码的兼容性, 我们包含了 `#[repr(C)]` 属性,
它指示 Rust 编译器始终使用与 C 相同的规则来组织结构体内的数据.

```rust,ignore
pub x: cty::c_int,
pub y: cty::c_int,
```

由于 C 或 C++ 在定义 `int` 或 `char` 时有多种方式, 建议使用 `cty` 中定义的原生数据类型, 它会将 C 中的类型映射到 Rust 中的类型.

```rust,ignore
extern "C" { pub fn cool_function( ... ); }
```

此语句定义了一个使用 C ABI 的函数的签名, 该函数名为 `cool_function`.
通过定义签名但不定义函数体, 此函数的定义需要在其他地方提供, 或从静态库链接到最终的库或二进制文件中.

```rust,ignore
    i: cty::c_int,
    c: cty::c_char,
    cs: *mut CoolStruct
```

与上面的数据类型类似, 我们使用 C 兼容的定义来指定函数参数的数据类型.
我们还保留了相同的参数名称, 以增加清晰度.

这里出现了一个新类型, `*mut CoolStruct`. 由于 C 没有 Rust 引用 (Reference) 的概念 (后者看起来像这样: `&mut CoolStruct`),
我们这里使用的是裸指针 (Raw Pointer). 因为解引用该指针是 `unsafe` 的, 而且该指针实际上可能是 `null` 指针, 因此在与 C 或 C++ 代码交互时, 必须小心地维护 Rust 典型的保证 (Guarantee).

### 自动生成接口

与其手动生成这些接口 (这可能繁琐且容易出错), 不如使用一个名为 [bindgen] 的工具来自动执行这些转换.
有关 [bindgen] 的使用说明, 请参阅 [bindgen user's manual], 但是典型过程包括以下步骤:

1. 收集所有你想在 Rust 中使用的 C 或 C++ 头文件, 它们定义了接口或数据类型.
2. 编写一个 `bindings.h` 文件, 该文件 `#include "..."` 第一步中收集的每个文件.
3. 将此 `bindings.h` 文件以及用于编译代码的任何编译标志一起输入到 `bindgen`.
   提示: 使用 `Builder.ctypes_prefix("cty")` / `--ctypes-prefix=cty` 和 `Builder.use_core()` / `--use-core` 以使生成的代码与 `#![no_std]` 兼容.
4. `bindgen` 将生成的 Rust 代码输出到终端窗口. 该输出可以重定向到项目中的一个文件, 例如 `bindings.rs`. 你可以在 Rust 项目中使用此文件与作为外部库编译和链接的 C/C++ 代码进行交互.
   提示: 如果生成的绑定中的类型带有 `cty` 前缀, 请不要忘记使用 [`cty`](https://crates.io/crates/cty) crate.

[bindgen]: https://github.com/rust-lang/rust-bindgen
[bindgen user's manual]: https://rust-lang.github.io/rust-bindgen/

## 构建 C/C++ 代码

由于 Rust 编译器并不直接知道如何编译 C 或 C++ 代码 (或来自任何其他呈现 C 接口的语言的代码), 因此有必要提前编译你的非 Rust 代码.

对于嵌入式项目, 这通常意味着将 C/C++ 代码编译为静态归档文件 (例如 `cool-library.a`), 然后可以在最终链接步骤中与你的 Rust 代码合并.

如果你要使用的库已经作为静态归档文件分发, 则无需重新构建代码. 只需按上述方法转换提供的接口头文件, 并在编译/链接时包含该静态归档文件即可.

如果你的代码以源代码项目形式存在, 则需要将 C/C++ 代码编译为静态库, 可以通过触发你现有的构建系统 (例如 `make`, `CMake` 等), 或者将必要的编译步骤移植到使用一个名为 `cc` 的 crate. 对于这两种情况, 都需要使用 `build.rs` 脚本.

### Rust 的 `build.rs` 构建脚本

`build.rs` 脚本是一个用 Rust 语法编写的文件, 它在项目的依赖项构建完成之后, 项目构建之前, 在你的编译机器上执行.

完整参考可以在 [这里](https://doc.rust-lang.org/cargo/reference/build-scripts.html) 找到. `build.rs` 脚本对于生成代码 (例如通过 [bindgen]), 调用外部构建系统 (例如 `Make`), 或者通过 `cc` crate 直接编译 C/C++ 非常有用.

### 触发外部构建系统

对于具有复杂外部项目或构建系统的项目, 最简单的方法可能是使用 [`std::process::Command`] 通过遍历相对路径, 调用固定的命令 (例如 `make library`), 然后将生成的静态库复制到 `target` 构建目录中的适当位置, 从而 "shell out" 到你其他的构建系统.

虽然你的 crate 可能面向 `no_std` 嵌入式平台, 但你的 `build.rs` 仅在编译 crate 的机器上执行. 这意味着你可以使用任何将在编译主机上运行的 Rust crate.

[`std::process::Command`]: https://doc.rust-lang.org/std/process/struct.Command.html

### 使用 `cc` crate 构建 C/C++ 代码

对于依赖较少或复杂度较低的项目, 或者对于难以修改构建系统以生成静态库 (而不是最终二进制或可执行文件) 的项目, 改用 [`cc` crate] 可能更容易, 它为宿主提供的编译器提供了一套符合 Rust 习惯的接口.

[`cc` crate]: https://github.com/alexcrichton/cc-rs

在最简单的情况下, 将单个 C 文件编译为静态库的依赖, 使用 [`cc` crate] 的 `build.rs` 脚本示例如下:

```rust,ignore
fn main() {
    cc::Build::new()
        .file("src/foo.c")
        .compile("foo");
}
```

`build.rs` 放置在包的根目录. 然后 `cargo build` 会在包的构建之前编译并执行它. 将生成一个名为 `libfoo.a` 的静态归档文件, 并放入 `target` 目录中.
