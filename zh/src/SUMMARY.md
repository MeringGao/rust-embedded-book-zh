# 目录

<!--

本书章节的组织结构仍在不断完善中.

更多信息和协调请参阅 https://github.com/rust-embedded/book/issues

-->

- [介绍 (Introduction)](./intro/index.md)
    - [硬件 (Hardware)](./intro/hardware.md)
    - [`no_std`](./intro/no-std.md)
    - [工具链 (Tooling)](./intro/tooling.md)
    - [安装 (Installation)](./intro/install.md)
        - [Linux](./intro/install/linux.md)
        - [macOS](./intro/install/macos.md)
        - [Windows](./intro/install/windows.md)
        - [验证安装 (Verify Installation)](./intro/install/verify.md)
- [入门 (Getting started)](./start/index.md)
  - [QEMU](./start/qemu.md)
  - [硬件 (Hardware)](./start/hardware.md)
  - [内存映射寄存器 (Memory-mapped Registers)](./start/registers.md)
  - [半主机 (Semihosting)](./start/semihosting.md)
  - [异常处理 (Panicking)](./start/panicking.md)
  - [异常 (Exceptions)](./start/exceptions.md)
  - [中断 (Interrupts)](./start/interrupts.md)
- [外设 (Peripherals)](./peripherals/index.md)
    - [Rust 中的初次尝试 (A first attempt in Rust)](./peripherals/a-first-attempt.md)
    - [借用检查器 (The Borrow Checker)](./peripherals/borrowck.md)
    - [单例 (Singletons)](./peripherals/singletons.md)
- [静态保证 (Static Guarantees)](./static-guarantees/index.md)
    - [类型状态编程 (Typestate Programming)](./static-guarantees/typestate-programming.md)
    - [作为状态机的外设 (Peripherals as State Machines)](./static-guarantees/state-machines.md)
    - [设计契约 (Design Contracts)](./static-guarantees/design-contracts.md)
    - [零成本抽象 (Zero Cost Abstractions)](./static-guarantees/zero-cost-abstractions.md)
- [可移植性 (Portability)](./portability/index.md)
- [并发 (Concurrency)](./concurrency/index.md)
- [集合 (Collections)](./collections/index.md)
- [设计模式 (Design Patterns)](./design-patterns/index.md)
    - [HAL](./design-patterns/hal/index.md)
        - [清单 (Checklist)](./design-patterns/hal/checklist.md)
        - [命名 (Naming)](./design-patterns/hal/naming.md)
        - [互操作性 (Interoperability)](./design-patterns/hal/interoperability.md)
        - [可预测性 (Predictability)](./design-patterns/hal/predictability.md)
        - [GPIO](./design-patterns/hal/gpio.md)
- [面向嵌入式 C 开发者的提示 (Tips for embedded C developers)](./c-tips/index.md)
    <!-- TODO: 定义各小节 -->
- [互操作性 (Interoperability)](./interoperability/index.md)
    - [Rust 搭配 C (A little C with your Rust)](./interoperability/c-with-rust.md)
    - [C 搭配 Rust (A little Rust with your C)](./interoperability/rust-with-c.md)
- [未分类主题 (Unsorted topics)](./unsorted/index.md)
  - [优化: 速度与大小的权衡 (Optimizations: The speed size tradeoff)](./unsorted/speed-vs-size.md)
  - [执行数学运算 (Performing Math Functionality)](./unsorted/math.md)

---

[附录 A: 术语表 (Appendix A: Glossary)](./appendix/glossary.md)