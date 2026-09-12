# 集合 (Collections)

最终, 你会希望在程序中使用动态数据结构 (也叫集合 (Collection)). `std` 提供了一组常见的集合: [`Vec`], [`String`], [`HashMap`] 等等. `std` 中实现的所有集合都使用全局动态内存分配器 (也叫堆 (Heap)).

[`Vec`]: https://doc.rust-lang.org/std/vec/struct.Vec.html
[`String`]: https://doc.rust-lang.org/std/string/struct.String.html
[`HashMap`]: https://doc.rust-lang.org/std/collections/struct.HashMap.html

由于 `core` 按定义不进行内存分配, 所以这些实现在那里不可用, 但可以在编译器自带的 `alloc` crate 中找到它们.

如果你需要使用集合, 堆分配的实现并不是你唯一的选择. 你也可以使用*固定容量 (Fixed Capacity)* 的集合; 在 [`heapless`] crate 中可以找到这样一种实现.

[`heapless`]: https://crates.io/crates/heapless

在本节中, 我们将探讨和比较这两种实现.

## 使用 `alloc`

`alloc` crate 随标准 Rust 发行版一起提供. 要导入该 crate, 你可以直接 `use` 它, 而*无需*在你的 `Cargo.toml` 文件中将其声明为依赖项.

``` rust,ignore
#![feature(alloc)]

extern crate alloc;

use alloc::vec::Vec;
```

要能够使用任何集合, 首先需要使用 `global_allocator` 属性声明你的程序将要使用的全局分配器. 要求你所选择的分配器必须实现 [`GlobalAlloc`] trait.

[`GlobalAlloc`]: https://doc.rust-lang.org/core/alloc/trait.GlobalAlloc.html

为了完整起见, 并尽可能让本节自成一体, 我们将实现一个简单的指针碰撞分配器 (Bump Pointer Allocator), 并将其用作全局分配器. 但是, 我们*强烈*建议你使用 crates.io 上经过实战检验的分配器, 而不是这个分配器.

``` rust,ignore
// 指针碰撞分配器实现
// Bump pointer allocator implementation

use core::alloc::{GlobalAlloc, Layout};
use core::cell::UnsafeCell;
use core::ptr;

use cortex_m::interrupt;

// 单核系统的指针碰撞分配器
// Bump pointer allocator for *single* core systems
struct BumpPointerAlloc {
    head: UnsafeCell<usize>,
    end: usize,
}

unsafe impl Sync for BumpPointerAlloc {}

unsafe impl GlobalAlloc for BumpPointerAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // `interrupt::free` 是一个临界区, 使得我们的分配器
        // 可以在中断内部安全使用.
        // `interrupt::free` is a critical section that makes our allocator safe
        // to use from within interrupts
        interrupt::free(|_| {
            let head = self.head.get();
            let size = layout.size();
            let align = layout.align();
            let align_mask = !(align - 1);

            // 把 start 上移到下一个对齐边界
            // move start up to the next alignment boundary
            let start = (*head + align - 1) & align_mask;

            if start + size > self.end {
                // 空指针表示内存不足 (Out Of Memory) 情况
                // a null pointer signal an Out Of Memory condition
                ptr::null_mut()
            } else {
                *head = start + size;
                start as *mut u8
            }
        })
    }

    unsafe fn dealloc(&self, _: *mut u8, _: Layout) {
        // 这个分配器从不释放内存
        // this allocator never deallocates memory
    }
}

// 声明全局内存分配器
// NOTE 用户必须确保内存区域 `[0x2000_0100, 0x2000_0200]`
// 没有被程序的其他部分使用
// Declaration of the global memory allocator
// NOTE the user must ensure that the memory region `[0x2000_0100, 0x2000_0200]`
// is not used by other parts of the program
#[global_allocator]
static HEAP: BumpPointerAlloc = BumpPointerAlloc {
    head: UnsafeCell::new(0x2000_0100),
    end: 0x2000_0200,
};
```

除了选择全局分配器之外, 用户还必须使用*不稳定 (unstable)* 的 `alloc_error_handler` 属性来定义如何处理内存不足 (Out Of Memory, OOM) 错误.

``` rust,ignore
#![feature(alloc_error_handler)]

use cortex_m::asm;

#[alloc_error_handler]
fn on_oom(_layout: Layout) -> ! {
    asm::bkpt();

    loop {}
}
```

一切就绪后, 用户终于可以在 `alloc` 中使用集合了.

```rust,ignore
#[entry]
fn main() -> ! {
    let mut xs = Vec::new();

    xs.push(42);
    assert!(xs.pop(), Some(42));

    loop {
        // ..
    }
}
```

如果你使用过 `std` crate 中的集合, 那么这些应该会让你感到熟悉, 因为它们是完全相同的实现.

## 使用 `heapless`

`heapless` 不需要任何设置, 因为它的集合不依赖于全局内存分配器. 只需 `use` 它的集合, 然后直接实例化即可:

```rust,ignore
// heapless 版本: v0.4.x
// heapless version: v0.4.x
use heapless::Vec;
use heapless::consts::*;

#[entry]
fn main() -> ! {
    let mut xs: Vec<_, U8> = Vec::new();

    xs.push(42).unwrap();
    assert_eq!(xs.pop(), Some(42));
    loop {}
}
```

你会注意到这些集合与 `alloc` 中的集合之间有两点不同.

首先, 你必须预先声明集合的容量. `heapless` 集合永远不会重新分配, 并且具有固定的容量; 该容量是集合类型签名的一部分. 在本例中, 我们声明 `xs` 的容量为 8 个元素, 也就是说该向量最多可以容纳 8 个元素. 这通过类型签名中的 `U8` 表示 (参见 [`typenum`]).

[`typenum`]: https://crates.io/crates/typenum

其次, `push` 方法以及许多其他方法都会返回一个 `Result`. 由于 `heapless` 集合具有固定容量, 所有将元素插入集合的操作都有可能失败. API 通过返回 `Result` 来反映这个问题, 以指示操作是否成功. 相反, `alloc` 集合会在堆上重新分配自己以增加容量.

从 v0.4.x 版本开始, 所有 `heapless` 集合都会内联存储其所有元素. 这意味着像 `let x = heapless::Vec::new();` 这样的操作会在栈 (Stack) 上分配集合, 但也可以将集合分配到 `static` 变量上, 甚至可以分配到堆上 (`Box<Vec<_, _>>`).

## 取舍

在堆分配的可重定位集合与固定容量集合之间进行选择时, 请牢记以下几点.

### 内存不足与错误处理

使用堆分配时, 内存不足始终是可能的, 并且可能发生在集合需要增长的任何地方: 例如, 所有 `alloc::Vec.push` 调用都有可能产生 OOM 情况. 因此, 某些操作可能会*隐式*失败. 一些 `alloc` 集合暴露了 `try_reserve` 方法, 让你可以在增长集合时检查潜在的 OOM 情况, 但你需要主动使用它们.

如果你只使用 `heapless` 集合, 并且没有将内存分配器用于其他任何用途, 那么 OOM 情况就不可能发生. 相反, 你将不得不根据具体情况处理集合容量耗尽的情况. 也就是说, 你必须处理由 `Vec.push` 等方法返回的*所有* `Result`.

OOM 失败可能比直接对 `heapless::Vec.push` 返回的所有 `Result` 调用 `unwrap` 更加难以调试, 因为观察到的失败位置可能*不*匹配于问题原因的位置. 例如, 即使是 `vec.reserve(1)` 也会在分配器几乎耗尽时触发 OOM, 因为其他某个集合正在泄漏内存 (内存泄漏在 safe Rust 中是可能发生的).

### 内存使用

对堆分配集合的内存使用进行推理是很困难的, 因为长寿命集合的容量可以在运行时改变. 某些操作可能会隐式地重新分配集合, 从而增加其内存使用量; 一些集合还暴露了 `shrink_to_fit` 之类的方法, 这些方法可能会减少集合所使用的内存 — 但最终, 是否真正减少内存分配取决于分配器. 此外, 分配器可能不得不处理内存碎片, 这会增加*表观*内存使用.

另一方面, 如果你只使用固定容量集合, 将它们大部分存储在 `static` 变量中, 并为调用栈设置一个最大大小, 那么当你试图使用超出物理可用内存时, 链接器 (Linker) 就会检测到.

此外, 栈上分配的固定容量集合会被 [`-Z emit-stack-sizes`] 标志报告, 这意味着栈使用情况分析工具 (例如 [`stack-sizes`]) 会将它们纳入分析范围.

[`-Z emit-stack-sizes`]: https://doc.rust-lang.org/beta/unstable-book/compiler-flags/emit-stack-sizes.html
[`stack-sizes`]: https://crates.io/crates/stack-sizes

但是, 固定容量集合*不能*缩小, 这可能导致比可重定位集合更低的负载因子 (集合大小与容量的比率).

### 最坏情况执行时间 (WCET)

如果你正在构建时间敏感的应用或强实时 (Hard Real-Time) 应用, 那么你会关心 — 甚至非常关心 — 程序不同部分的最坏情况执行时间 (Worst Case Execution Time, WCET).

`alloc` 集合可能会重新分配, 因此可能使集合增长的操作的 WCET 也会包括重新分配集合所花费的时间, 而这本身又取决于集合的*运行时*容量. 这使得确定例如 `alloc::Vec.push` 操作的 WCET 很困难, 因为它既取决于所使用的分配器, 也取决于其运行时容量.

另一方面, 固定容量集合永远不会重新分配, 因此所有操作都具有可预测的执行时间. 例如, `heapless::Vec.push` 以常数时间执行.

### 易用性

`alloc` 需要设置全局分配器, 而 `heapless` 不需要. 但是, `heapless` 要求你为每个要实例化的集合选择容量.

`alloc` API 对于几乎每个 Rust 开发者来说都很熟悉. `heapless` API 试图紧密模仿 `alloc` API, 但由于其显式的错误处理, 永远无法做到完全相同 — 一些开发者可能会觉得显式的错误处理过度或过于繁琐.
