# 互操作性 (Interoperability)

Rust 与 C 代码之间的互操作性 (Interoperability) 始终依赖于在两种语言之间转换数据.
为此, 标准库 `stdlib` 中有一个专用模块
[`std::ffi`](https://doc.rust-lang.org/std/ffi/index.html).

`std::ffi` 为 C 原生类型提供类型定义, 例如 `char`, `int` 和 `long`.
它还提供了一些实用工具, 用于转换更复杂的类型 (例如字符串),
将 `&str` 和 `String` 映射为更易于处理且更安全的 C 类型.

自 Rust 1.30 起, `std::ffi` 的功能已根据是否涉及内存分配,
在 `core::ffi` 或 `alloc::ffi` 中可用.
[`cty`] crate 和 [`cstr_core`] crate 也提供类似的功能.

[`cstr_core`]: https://crates.io/crates/cstr_core
[`cty`]: https://crates.io/crates/cty

| Rust 类型      | 中间类型  | C 类型         |
|----------------|----------|----------------|
| `String`       | `CString` | `char *`       |
| `&str`         | `CStr`    | `const char *` |
| `()`           | `c_void`  | `void`         |
| `u32` 或 `u64` | `c_uint`  | `unsigned int` |
| 等等            | ...      | ...            |

C 原生类型的值可以用作对应的 Rust 类型之一, 反之亦然,
因为前者只是后者的类型别名.
例如, 以下代码在 `unsigned int` 为 32 位长的平台上可以编译.

```rust,ignore
fn foo(num: u32) {
    let c_num: c_uint = num;
    let r_num: u32 = c_num;
}
```

## 与其他构建系统的互操作性

在嵌入式项目中引入 Rust, 一个常见的需求是将 Cargo 与现有的构建系统 (例如 make 或 cmake) 结合使用.

我们正在我们的 issue 跟踪器中收集相关的示例和用例,
请见 [issue #61].

[issue #61]: https://github.com/rust-embedded/book/issues/61

## 与 RTOS 的互操作性

将 Rust 与 RTOS (例如 FreeRTOS 或 ChibiOS) 集成仍在进行中, 尤其是从 Rust 调用 RTOS 函数可能会有些棘手.

目前, 以下项目公开支持 Rust <-> RTOS 互操作性:

* [Zephyr Project](https://docs.zephyrproject.org/latest/develop/languages/rust/index.html)

我们正在我们的 issue 跟踪器中收集相关的示例和用例,
请见 [issue #62].

[issue #62]: https://github.com/rust-embedded/book/issues/62
