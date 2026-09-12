# 并发 (Concurrency)

当程序的不同部分可能在不同时间或乱序执行时, 就会发生并发 (Concurrency). 在嵌入式环境中, 这包括:

* 中断处理程序 (Interrupt Handler) — 在关联的中断 (Interrupt) 发生时执行
* 各种形式的多线程 (Multithreading) — 微处理器定期在程序的不同部分之间切换
* 在某些系统中, 多核微处理器 — 每个核心可以同时独立运行程序的不同部分

由于许多嵌入式程序需要处理中断, 并发迟早会出现, 这也是许多微妙且棘手的 bug 容易发生的地方. 幸运的是, Rust 提供了一系列抽象和安全保证来帮助我们编写正确的代码.

## 无并发

嵌入式程序最简单的并发形式就是无并发: 软件由一个不断运行的主循环 (Main Loop) 组成, 完全没有中断. 有时这种方案非常适合手头的问题! 通常, 你的循环会读取一些输入, 进行一些处理, 然后写入一些输出.

```rust,ignore
#[entry]
fn main() {
    let peripherals = setup_peripherals();
    loop {
        let inputs = read_inputs(&peripherals);
        let outputs = process(inputs);
        write_outputs(&peripherals, outputs);
    }
}
```

因为没有并发, 所以无需担心在程序的不同部分之间共享数据, 也无需同步对外设 (Peripheral) 的访问. 如果这种简单的方案能够满足需求, 那么它就是一个很好的选择.

## 全局可变数据

与非嵌入式 Rust 不同, 我们通常无法奢侈地在堆 (Heap) 上分配内存并把对该数据的引用传递给新创建的线程. 相反, 我们的中断处理程序可能在任何时刻被调用, 并且必须知道如何访问我们使用的任何共享内存. 在最底层, 这意味着我们必须有*静态分配 (Statically Allocated)* 的可变内存, 这样中断处理程序和主代码都可以引用它.

在 Rust 中, 这种 [`static mut`] 变量的读写总是 `unsafe` 的, 因为如果不特别小心, 可能会触发竞态条件 (Race Condition) — 在这种情况下, 你对变量的访问可能会被一个同样访问该变量的中断中途打断.

[`static mut`]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable

举例说明这种行为如何在代码中引起微妙的错误, 考虑一个嵌入式程序: 它在一秒钟内对某个输入信号的上升沿进行计数 (一个频率计数器):

```rust,ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // 危险 - 实际上并不安全! 可能导致数据竞争
            // DANGER - Not actually safe! Could cause data races.
            unsafe { COUNTER += 1 };
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

每秒, 定时器中断把计数器重置为 0. 与此同时, 主循环持续测量信号, 当检测到从低到高的变化时, 把计数器加 1. 我们不得不使用 `unsafe` 来访问 `COUNTER`, 因为它是 `static mut`, 这意味着我们在向编译器承诺不会引发任何未定义行为 (Undefined Behaviour). 你能发现竞态条件吗? 对 `COUNTER` 的自增操作*不*保证是原子的 (Atomic) — 事实上, 在大多数嵌入式平台上, 它会被拆分为加载 (load), 自增, 然后存储 (store). 如果中断在加载之后, 存储之前触发, 那么重置为 0 的操作将在中断返回后被忽略 — 这意味着我们会在该周期内多计一倍数量的跳变.

## 临界区 (Critical Section)

那么, 对于数据竞争我们能做些什么呢? 一种简单的方法是使用*临界区* (Critical Section) — 一种禁用中断的上下文. 通过把主循环中对 `COUNTER` 的访问包裹在临界区里, 我们可以确保定时器中断不会在 `COUNTER` 自增完成之前触发:

```rust,ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // 新的临界区确保对 COUNTER 的同步访问
            // New critical section ensures synchronised access to COUNTER
            cortex_m::interrupt::free(|_| {
                unsafe { COUNTER += 1 };
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

在本例中, 我们使用了 `cortex_m::interrupt::free`, 但其他平台也会有类似的在临界区中执行代码的机制. 这等价于: 禁用中断, 运行一些代码, 然后重新启用中断.

注意我们不需要在定时器中断内部放置临界区, 原因有二:

  * 将 0 写入 `COUNTER` 不会受到竞态的影响, 因为我们没有读取它
  * 它也不会被 `main` 线程中断

如果 `COUNTER` 被多个可能相互*抢占 (Preempt)* 的中断处理程序共享, 那么每个处理程序也都可能需要临界区.

这解决了眼前的直接问题, 但我们仍然不得不编写大量需要仔细推理的 unsafe 代码, 而且我们可能没有必要地使用了临界区. 由于每个临界区都会暂时暂停中断处理, 因此会带来一些额外的代码大小以及更高的中断延迟和抖动 (中断处理可能需要更长时间, 处理的等待时间也会更不确定) 等成本. 这是否会成为问题取决于你的系统, 但通常情况下, 我们希望避免它.

值得注意的是, 虽然临界区保证不会有中断触发, 但在多核系统上它并不提供互斥保证! 另一个核心可能正在访问与本核心相同的内存, 即使没有中断. 如果你使用多核, 则需要更强的同步原语.

## 原子访问 (Atomic Access)

在某些平台上, 可以使用特殊的原子指令, 它们为读-改-写操作提供保证. 具体到 Cortex-M: `thumbv6` (Cortex-M0, Cortex-M0+) 仅提供原子的加载和存储指令, 而 `thumbv7` (Cortex-M3 及以上) 提供完整的比较并交换 (Compare and Swap, CAS) 指令. 这些 CAS 指令提供了一种替代粗暴地禁用所有中断的方案: 我们可以尝试自增, 大多数时候会成功, 但如果被打断, 它会自动重试整个自增操作. 这些原子操作即使在多个核心之间也是安全的.

```rust,ignore
use core::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // 使用 `fetch_add` 原子地将 1 加到 COUNTER
            // Use `fetch_add` to atomically add 1 to COUNTER
            COUNTER.fetch_add(1, Ordering::Relaxed);
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // 使用 `store` 直接将 0 写入 COUNTER
    // Use `store` to write 0 directly to COUNTER
    COUNTER.store(0, Ordering::Relaxed)
}
```

这一次 `COUNTER` 是一个安全的 `static` 变量. 多亏了 `AtomicUsize` 类型, `COUNTER` 可以安全地从中断处理程序和主线程中被修改, 而无需禁用中断. 在可能的情况下, 这是更好的解决方案 — 但你的平台可能并不支持它.

关于 [`Ordering`] 的说明: 它会影响编译器和硬件如何重排指令, 并且对缓存可见性也有影响. 假设目标是单核平台, `Relaxed` 已经足够, 并且在这种情况下是最高效的选择. 更严格的排序会导致编译器在原子操作周围发出内存屏障; 至于是否需要, 这取决于你使用原子操作的目的! 原子模型的精确细节比较复杂, 最好在其他地方详细描述.

有关原子操作和排序的更多细节, 请参见 [nomicon].

[`Ordering`]: https://doc.rust-lang.org/core/sync/atomic/enum.Ordering.html
[nomicon]: https://doc.rust-lang.org/nomicon/atomics.html


## 抽象, Send 和 Sync

上面的解决方案都不太理想. 它们需要非常仔细地检查 `unsafe` 块, 而且不够符合人体工程学. 在 Rust 中我们一定能做得更好!

我们可以把计数器抽象为一个安全的接口, 可以在代码的任何其他地方安全地使用. 在这个例子中, 我们将使用基于临界区的计数器, 但你也可以用原子操作做一些非常类似的事情.

```rust,ignore
use core::cell::UnsafeCell;
use cortex_m::interrupt;

// 我们的计数器只是对 UnsafeCell<u32> 的包装, 它是 Rust 内部可变性
// (Interior Mutability) 的核心. 通过使用内部可变性, 我们可以让
// COUNTER 成为 `static` 而不是 `static mut`, 同时仍然能够修改其计数值.
// Our counter is just a wrapper around UnsafeCell<u32>, which is the heart
// of interior mutability in Rust. By using interior mutability, we can have
// COUNTER be `static` instead of `static mut`, but still able to mutate
// its counter value.
struct CSCounter(UnsafeCell<u32>);

const CS_COUNTER_INIT: CSCounter = CSCounter(UnsafeCell::new(0));

impl CSCounter {
    pub fn reset(&self, _cs: &interrupt::CriticalSection) {
        // 由于要求传入一个 CriticalSection, 我们就知道
        // 当前一定是在临界区内运行, 因此可以放心地
        // 使用这个 unsafe 块 (调用 UnsafeCell::get 所需).
        // By requiring a CriticalSection be passed in, we know we must
        // be operating inside a CriticalSection, and so can confidently
        // use this unsafe block (required to call UnsafeCell::get).
        unsafe { *self.0.get() = 0 };
    }

    pub fn increment(&self, _cs: &interrupt::CriticalSection) {
        unsafe { *self.0.get() += 1 };
    }
}

// 允许静态 CSCounter, 详见下文.
// Required to allow static CSCounter. See explanation below.
unsafe impl Sync for CSCounter {}

// COUNTER 不再是 `mut`, 因为它使用了内部可变性;
// 因此访问它也不再需要 unsafe 块.
// COUNTER is no longer `mut` as it uses interior mutability;
// therefore it also no longer requires unsafe blocks to access.
static COUNTER: CSCounter = CS_COUNTER_INIT;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // 这里没有 unsafe!
            // No unsafe here!
            interrupt::free(|cs| COUNTER.increment(cs));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // 我们确实需要在这里进入一个临界区,
    // 只是为了获得一个有效的 cs 令牌,
    // 即使我们知道没有其他中断会抢占它.
    // We do need to enter a critical section here just to obtain a valid
    // cs token, even though we know no other interrupt could pre-empt
    // this one.
    interrupt::free(|cs| COUNTER.reset(cs));

    // 如果你真的想避免开销, 可以用 unsafe 代码生成一个伪造的
    // CriticalSection:
    // We could use unsafe code to generate a fake CriticalSection if we
    // really wanted to, avoiding the overhead:
    // let cs = unsafe { interrupt::CriticalSection::new() };
}
```

我们已经把 `unsafe` 代码移到了精心设计的抽象内部, 现在的应用代码已经不包含任何 `unsafe` 块了.

这个设计要求应用代码传入一个 `CriticalSection` 令牌: 这些令牌只能由 `interrupt::free` 安全地生成, 因此通过要求传入令牌, 我们就确保了当前是在临界区内部运行, 而无需自己实际加锁. 这种保证由编译器在静态时提供: `cs` 不会带来任何运行时开销. 如果我们有多个计数器, 它们都可以被传入同一个 `cs`, 而不需要嵌套多个临界区.

这也引出了 Rust 中关于并发的一个重要主题: [`Send` 和 `Sync`] trait. 概括地说, 当一个类型可以安全地移动到另一个线程时, 它就是 `Send`; 而当一个类型可以安全地在多个线程之间共享时, 它就是 `Sync`. 在嵌入式环境中, 我们认为中断是在与应用代码不同的线程中执行的, 因此被中断和主代码同时访问的变量必须是 `Sync` 的.

[`Send` 和 `Sync`]: https://doc.rust-lang.org/nomicon/send-and-sync.html

对于 Rust 中的大多数类型, 这两个 trait 都会由编译器自动派生. 然而, 由于 `CSCounter` 包含一个 [`UnsafeCell`], 它不是 `Sync` 的, 因此我们不能创建 `static CSCounter`: `static` 变量*必须*是 `Sync` 的, 因为它们可以被多个线程访问.

[`UnsafeCell`]: https://doc.rust-lang.org/core/cell/struct.UnsafeCell.html

为了告诉编译器我们已经确保 `CSCounter` 在线程间共享确实是安全的, 我们显式地实现了 `Sync` trait. 与之前使用临界区的情况一样, 这仅在单核平台上才是安全的: 在多核平台上, 你需要采取更多的措施来确保安全性.

## 互斥锁 (Mutex)

我们已经为计数器问题创建了一个有用的特定抽象, 但并发中还有许多常见的抽象.

*同步原语 (Synchronisation Primitive)* 之一就是互斥锁 (Mutex, mutual exclusion 的缩写). 这些结构确保对某个变量 (例如我们的计数器) 的独占访问. 一个线程可以尝试*加锁 (lock)* (或*获取 acquire*) 互斥锁, 然后要么立即成功, 要么阻塞等待加锁成功, 要么返回一个错误, 表示无法加锁. 在该线程持有锁的期间, 它被授予对被保护数据的访问权. 当该线程使用完毕后, 它*解锁 (unlock)* (或*释放 release*) 互斥锁, 允许其他线程再加锁. 在 Rust 中, 我们通常会使用 [`Drop`] trait 来实现 unlock, 以确保互斥锁在超出作用域时一定会被释放.

[`Drop`]: https://doc.rust-lang.org/core/ops/trait.Drop.html

将互斥锁与中断处理程序一起使用可能会有些棘手: 通常情况下, 中断处理程序不能被阻塞 (Blocking), 而如果它阻塞等待主线程释放锁, 那将是特别灾难性的, 因为我们会因此*死锁 (Deadlock)* (由于执行停留在中断处理程序中, 主线程永远不会释放锁). 死锁本身并不被视为 unsafe: 即使在 safe Rust 中, 死锁也可能发生.

为了完全避免这种行为, 我们可以实现一个需要临界区才能加锁的互斥锁, 就像我们的计数器示例那样. 只要临界区的持续时间和锁的持续时间一致, 我们就能确保对被包装变量的独占访问, 甚至无需跟踪互斥锁的加锁/解锁状态.

事实上, `cortex_m` crate 已经为我们做了这件事! 我们本可以使用它来写计数器:

```rust,ignore
use core::cell::Cell;
use cortex_m::interrupt::Mutex;

static COUNTER: Mutex<Cell<u32>> = Mutex::new(Cell::new(0));

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            interrupt::free(|cs|
                COUNTER.borrow(cs).set(COUNTER.borrow(cs).get() + 1));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // 我们仍然需要在这里进入临界区以满足 Mutex 的要求.
    // We still need to enter a critical section here to satisfy the Mutex.
    interrupt::free(|cs| COUNTER.borrow(cs).set(0));
}
```

我们现在使用了 [`Cell`], 它和它的兄弟类型 `RefCell` 用于提供安全的内部可变性. 我们已经见过 `UnsafeCell`, 它是 Rust 内部可变性的最底层: 它允许你获得其值的多个可变引用, 但只能通过 unsafe 代码实现. `Cell` 类似于 `UnsafeCell`, 但它提供了一个安全的接口: 它只允许获取当前值的副本或替换当前值, 而不允许获取引用, 并且由于它不是 `Sync` 的, 所以它不能在多个线程之间共享. 这些限制使得它可以安全地使用, 但我们不能直接在 `static` 变量中使用它, 因为 `static` 必须是 `Sync` 的.

[`Cell`]: https://doc.rust-lang.org/core/cell/struct.Cell.html

那为什么上面的例子能工作呢? `Mutex<T>` 为任何是 `Send` 的 `T` 实现 `Sync` — 例如 `Cell`. 它之所以能安全地做到这一点, 是因为它只在临界区期间才提供对其内容的访问. 因此, 我们能够得到一个完全没有 unsafe 代码的安全计数器!

这对于像我们的计数器 `u32` 这样简单的类型来说很棒, 但对于那些不是 `Copy` 的更复杂的类型呢? 嵌入式环境中一个非常常见的例子就是外设结构体, 它通常不是 `Copy` 的. 对于这种情况, 我们可以使用 `RefCell`.

## 共享外设

使用 `svd2rust` 和类似抽象生成的设备 crate 通过强制要求某一外设结构体一次只能存在一个实例来提供对外设的安全访问. 这确保了安全性, 但使得从主线程和中断处理程序同时访问一个外设变得困难.

为了安全地共享外设访问, 我们可以使用之前见到的 `Mutex`. 我们还需要使用 [`RefCell`], 它使用运行时检查来确保一次只给出一个外设的引用. 这比普通的 `Cell` 开销更大, 但由于我们给出的是引用而不是副本, 所以必须确保一次只存在一个引用.

[`RefCell`]: https://doc.rust-lang.org/core/cell/struct.RefCell.html

最后, 我们还必须考虑如何在外设初始化完成之后把它移入共享变量中. 为此, 我们可以使用 `Option` 类型, 初始化为 `None`, 之后再设置为外设实例.

```rust,ignore
use core::cell::RefCell;
use cortex_m::interrupt::{self, Mutex};
use stm32f4::stm32f405;

static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    // 获取外设单例 (Singleton) 并进行配置.
    // 这个例子来自一个 svd2rust 生成的 crate,
    // 但大多数嵌入式设备 crate 都与此类似.
    // Obtain the peripheral singletons and configure it.
    // This example is from an svd2rust-generated crate, but
    // most embedded device crates will be similar.
    let dp = stm32f405::Peripherals::take().unwrap();
    let gpioa = &dp.GPIOA;

    // 某种配置函数.
    // 假设它将 PA0 设置为输入, PA1 设置为输出.
    // Some sort of configuration function.
    // Assume it sets PA0 to an input and PA1 to an output.
    configure_gpio(gpioa);

    // 将 GPIOA 存储到互斥锁中, 完成移动.
    // Store the GPIOA in the mutex, moving it.
    interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
    // 此后不能再使用 `gpioa` 或 `dp.GPIOA`,
    // 而必须通过互斥锁来访问它.
    // We can no longer use `gpioa` or `dp.GPIOA`, and instead have to
    // access it via the mutex.

    // 注意要在设置 MY_GPIO 之后才使能中断:
    // 否则中断可能在它仍然包含 None 的时候触发,
    // 而按当前写法 (带 `unwrap()`) 会发生 panic.
    // Be careful to enable the interrupt only after setting MY_GPIO:
    // otherwise the interrupt might fire while it still contains None,
    // and as-written (with `unwrap()`), it would panic.
    set_timer_1hz();
    let mut last_state = false;
    loop {
        // 现在我们通过互斥锁以数字输入的方式读取状态
        // We'll now read state as a digital input, via the mutex
        let state = interrupt::free(|cs| {
            let gpioa = MY_GPIO.borrow(cs).borrow();
            gpioa.as_ref().unwrap().idr.read().idr0().bit_is_set()
        });

        if state && !last_state {
            // 当 PA0 检测到上升沿时, 把 PA1 置高.
            // Set PA1 high if we've seen a rising edge on PA0.
            interrupt::free(|cs| {
                let gpioa = MY_GPIO.borrow(cs).borrow();
                gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // 这次在中断中我们只是清零 PA0.
    // This time in the interrupt we'll just clear PA0.
    interrupt::free(|cs| {
        // 我们可以使用 `unwrap()`, 因为我们知道在设置 MY_GPIO 之前
        // 中断不会被使能; 否则应该处理 None 值的可能性.
        // We can use `unwrap()` because we know the interrupt wasn't enabled
        // until after MY_GPIO was set; otherwise we should handle the potential
        // for a None value.
        let gpioa = MY_GPIO.borrow(cs).borrow();
        gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().clear_bit());
    });
}
```

内容有点多, 让我们把重要的几行拆解一下.

```rust,ignore
static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));
```

我们的共享变量现在是 `Mutex` 包装的 `RefCell`, 而 `RefCell` 中又包含一个 `Option`. `Mutex` 确保我们只能在临界区内访问它, 因此即使普通的 `RefCell` 不是 `Sync` 的, 它也能让该变量变成 `Sync`. `RefCell` 为我们提供了带引用的内部可变性, 这正是访问 `GPIOA` 所需要的. `Option` 允许我们先把变量初始化为空, 然后再把实际的值移入其中. 我们无法在静态时访问外设单例, 只能在运行时访问, 所以这是必需的.

```rust,ignore
interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
```

在临界区内部, 我们可以调用互斥锁的 `borrow()`, 它会给我们一个 `RefCell` 的引用. 然后我们调用 `replace()` 把新值移入 `RefCell`.

```rust,ignore
interrupt::free(|cs| {
    let gpioa = MY_GPIO.borrow(cs).borrow();
    gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
});
```

最后, 我们以一种安全且并发的方式使用了 `MY_GPIO`. 临界区照常阻止中断触发, 并允许我们借用互斥锁. 然后 `RefCell` 给我们一个 `&Option<GPIOA>`, 并跟踪它被借用了多长时间 — 一旦该引用超出作用域, `RefCell` 就会更新以表明它不再被借用.

由于我们无法从 `&Option` 中移出 `GPIOA`, 所以需要使用 `as_ref()` 将其转换为 `&Option<&GPIOA>`, 然后我们才能对它调用 `unwrap()` 得到 `&GPIOA`, 进而修改外设.

如果我们需要一个对共享资源的可变引用, 那么应该改用 `borrow_mut` 和 `deref_mut`. 下面的代码展示了一个使用 TIM2 定时器的例子.

```rust,ignore
use core::cell::RefCell;
use core::ops::DerefMut;
use cortex_m::interrupt::{self, Mutex};
use cortex_m::asm::wfi;
use stm32f4::stm32f405;

static G_TIM: Mutex<RefCell<Option<Timer<stm32::TIM2>>>> =
	Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    let mut cp = cm::Peripherals::take().unwrap();
    let dp = stm32f405::Peripherals::take().unwrap();

    // 某种定时器配置函数.
    // 假设它配置了 TIM2 定时器及其 NVIC 中断,
    // 最后启动定时器.
    // Some sort of timer configuration function.
    // Assume it configures the TIM2 timer, its NVIC interrupt,
    // and finally starts the timer.
    let tim = configure_timer_interrupt(&mut cp, dp);

    interrupt::free(|cs| {
        G_TIM.borrow(cs).replace(Some(tim));
    });

    loop {
        wfi();
    }
}

#[interrupt]
fn timer() {
    interrupt::free(|cs| {
        if let Some(ref mut tim)) =  G_TIM.borrow(cs).borrow_mut().deref_mut() {
            tim.start(1.hz());
        }
    });
}

```

呼! 这虽然安全, 但也有点笨拙. 还有别的办法吗?

## RTIC

另一个选择是 [RTIC 框架], 全称是 Real Time Interrupt-driven Concurrency (实时中断驱动的并发). 它强制执行静态优先级, 并跟踪对 `static mut` 变量 (称为"资源") 的访问, 以静态地确保共享资源始终被安全地访问, 而无需始终进入临界区以及像 `RefCell` 那样使用引用计数. 这有很多优点, 例如保证不会发生死锁, 并且在时间和内存开销上都极低.

[RTIC 框架]: https://rtic.rs/2/book/en

RTIC 自带一个异步执行器, 因此你的软件任务是 `async` 函数, 除了常规的同步 API 之外, 你还可以使用 `async` API.

该框架还包括其他特性, 例如消息传递, 它减少了对显式共享状态的需求, 以及在指定时间调度任务的能力, 这可以用于实现周期性任务. 查看 [相关文档] 获取更多信息!

[相关文档]: https://rtic.rs

## Embassy

Embassy 是一个专注于使用 Rust 中的 `async` / `await` 语法进行并发的库生态系统. embassy 的核心是它的 [异步执行器] (Asynchronous Executor), 它支持大多数常见的 MCU 架构.

embassy 还采用了一站式 (battery-included) 方案, 提供了许多其他组件, 例如:

- [时间库](https://docs.rs/embassy-time/latest/embassy_time/)
- 多个还提供时间库支持的 HAL 库.
- 用于同步原语的 [embassy-sync](https://docs.embassy.dev/embassy-sync/git/default/index.html)

你可以查看 [官网](https://embassy.dev/) 和 [书籍](https://embassy.dev/book/) 获取更多信息.

## 实时操作系统 (RTOS)

嵌入式并发的另一种常见模型是实时操作系统 (Real-Time Operating System, RTOS). 虽然目前在 Rust 中探索得还不多, 但它们在传统嵌入式开发中被广泛使用. 开源示例包括 [FreeRTOS] 和 [ChibiOS]. 这些 RTOS 支持运行多个应用程序线程, CPU 在这些线程之间切换, 既可以在线程主动让出控制权时切换 (称为协作式多任务 (Cooperative Multitasking)), 也可以基于周期性定时器或中断进行切换 (抢占式多任务 (Preemptive Multitasking)). RTOS 通常提供互斥锁和其他同步原语, 并且经常与 DMA 引擎等硬件特性配合使用.

[FreeRTOS]: https://freertos.org/
[ChibiOS]: http://chibios.org/

在撰写本文时, Rust 中的 RTOS 示例还不多, 但这是一个有趣的领域, 让我们拭目以待!

## 多核

在嵌入式处理器中拥有两个或更多核心正变得越来越常见, 这给并发增加了额外的复杂性. 包括 `cortex_m::interrupt::Mutex` 在内的所有使用临界区的示例都假设另一个执行线程是中断线程, 但在多核系统上情况就不再是这样了. 相反, 我们需要为多核设计的同步原语 (也称为 SMP, 即对称多处理 (Symmetric Multi-Processing)).

这些原语通常使用我们前面见过的原子指令, 因为处理系统会确保在所有核心之间都保持原子性.

详细讨论这些主题超出了本书的范围, 但总体模式与单核情况相同.
