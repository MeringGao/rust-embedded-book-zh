# 中断 (Interrupts)

中断 (Interrupt) 在许多方面与异常 (Exception) 不同, 但它们的操作和使用大体相似, 并且也由同一个中断控制器处理. 异常由 Cortex-M 架构定义, 而中断始终是供应商 (甚至常常是特定芯片) 特定的实现, 在命名和功能上都是如此.

中断确实允许很大的灵活性, 在以高级方式尝试使用它们时需要加以考虑. 我们不会在本书中介绍那些用法, 然而牢记以下几点是个好主意:

* 中断具有可编程的优先级 (Priority), 优先级决定了它们处理程序的执行顺序
* 中断可以嵌套和抢占 (Preempt), 即一个中断处理程序的执行可能会被另一个具有更高优先级的中断打断
* 通常, 必须清除触发中断的原因, 以防止无休止地重新进入中断处理程序

运行时的一般初始化步骤总是相同的:
* 设置外设, 以在期望的时机生成中断请求
* 在中断控制器中设置中断处理程序的期望优先级
* 在中断控制器中启用该中断处理程序

与异常类似, cortex-m-rt crate 暴露了一个 [`interrupt`] 属性, 用于声明中断处理程序. 然而, 只有当 device feature 被启用时, 这个属性才可用. 话虽如此, 这个属性并不是打算被直接使用的——直接使用会导致编译错误.

相反, 你应该使用由设备 crate (通常使用 svd2rust 生成) 重新导出的 interrupt 属性版本. 这可以确保编译器能够验证该中断确实存在于目标设备上. 可用中断的列表——以及它们在中断向量表 (Vector Table) 中的位置——通常由 svd2rust 从 SVD 文件自动生成.

[`interrupt`]: https://docs.rs/cortex-m-rt-macros/0.1.5/cortex_m_rt_macros/attr.interrupt.html

``` rust,ignore
use lm3s6965::interrupt; // 从设备 crate 重新导入的属性
                          // Re-exported attribute from the device crate

// Timer2 中断的中断处理程序
// Interrupt handler for the Timer2 interrupt
#[interrupt]
fn TIMER2A() {
    // ..
    // 清除生成中断请求的原因
    // Clear reason for the generated interrupt request
}
```

中断处理程序看起来就像普通函数 (只是没有参数), 类似于异常处理程序. 然而, 由于特殊的调用约定, 它们不能被固件 (Firmware) 的其他部分直接调用. 然而, 可以用软件生成中断请求来触发向中断处理程序的转向.

与异常处理程序类似, 也可以在中断处理程序内部声明 `static mut` 变量, 用于**安全地**维护状态.

``` rust,ignore
#[interrupt]
fn TIMER2A() {
    static mut COUNT: u32 = 0;

    // `COUNT` 的类型是 `&mut u32`, 并且可以安全地使用
    // `COUNT` has type `&mut u32` and it's safe to use
    *COUNT += 1;
}
```

关于这里展示的机制的更详细描述, 请参阅 [异常一节][exceptions section].

[exceptions section]: ./exceptions.md
