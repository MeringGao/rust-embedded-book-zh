# 在 C 中使用少量 Rust 代码

在 C 或 C++ 项目中使用 Rust 代码主要由两部分组成.

- 在 Rust 中创建对 C 友好的 API
- 将你的 Rust 项目嵌入到外部构建系统中

除了 `cargo` 和 `meson` 之外, 大多数构建系统都没有原生的 Rust 支持.
因此, 你最有可能的最佳选择是仅使用 `cargo` 来编译你的 crate 以及任何依赖项.

## 搭建项目

像往常一样创建一个新的 `cargo` 项目.

有一些标志可以告诉 `cargo` 输出一个系统库, 而不是它通常的 Rust 目标.
如果你希望库的输出名称与 crate 的其余部分不同, 这也允许你为库设置不同的输出名称.

```toml
[lib]
name = "your_crate"
crate-type = ["cdylib"]      # 创建动态库
# crate-type = ["staticlib"] # 创建静态库
```

## 构建一个 `C` API

由于 C++ 没有供 Rust 编译器面向的稳定 ABI, 我们使用 `C` 作为任何不同语言之间互操作性的基础.
在 C 和 C++ 代码中使用 Rust 时也不例外.

### `#[no_mangle]`

Rust 编译器对符号名 (Symbol Name) 的修饰 (Name Mangling) 方式与原生代码链接器期望的不同.
因此, 任何 Rust 导出供 Rust 之外使用的函数都需要被告知不要被编译器修饰.

### `extern "C"`

默认情况下, 你在 Rust 中编写的任何函数都将使用 Rust ABI (同样也未稳定).
相反, 在构建对外的 FFI (Foreign Function Interface, 外部函数接口) API 时, 我们需要告诉编译器使用系统 ABI.

根据你的平台, 你可能希望面向特定的 ABI 版本, 这些版本在 [这里](https://doc.rust-lang.org/reference/items/external-blocks.html) 有文档说明.

---

将这些部分组合在一起, 你会得到一个大致如下所示的函数.

```rust,ignore
#[no_mangle]
pub extern "C" fn rust_function() {

}
```

就像在 Rust 项目中使用 `C` 代码一样, 你现在需要在数据之间进行转换, 使其能被应用程序的其余部分理解.

## 链接及更大的项目上下文.

那么, 这就是问题的一半解决了.
那么该如何使用它呢?

**这在很大程度上取决于你的项目和/或构建系统**

`cargo` 将创建一个 `my_lib.so`/`my_lib.dll` 或 `my_lib.a` 文件,
具体取决于你的平台和设置. 此库可以简单地由你的构建系统链接.

但是, 从 C 调用 Rust 函数需要头文件来声明函数签名.

你在 Rust-ffi API 中的每个函数都需要有相应的头文件函数.

```rust,ignore
#[no_mangle]
pub extern "C" fn rust_function() {}
```

那么它将变成

```C
void rust_function();
```

等等.

有一个工具可以自动执行此过程, 称为 [cbindgen], 它会分析你的 Rust 代码, 然后从中为你的 C 和 C++ 项目生成头文件.

[cbindgen]: https://github.com/eqrion/cbindgen

此时, 从 C 中使用 Rust 函数就像包含头文件并调用它们一样简单!

```C
#include "my-rust-project.h"
rust_function();
```
