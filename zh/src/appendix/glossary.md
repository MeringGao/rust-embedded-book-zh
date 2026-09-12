# 附录 A: 术语表

嵌入式生态系统充满了使用自己术语和缩写的不同协议, 硬件组件和供应商特定的事物. 本术语表尝试列出它们并提供理解它们的参考.

### BSP

板级支持 crate (Board Support Crate) 提供针对特定开发板配置的高级接口. 它通常依赖于 [HAL](#hal) crate.
有关更详细的描述, 请参阅 [内存映射寄存器页面](../start/registers.md), 或查看 [此视频](https://youtu.be/vLYit_HHPaY) 以获得更广泛的概览.

### FPU

浮点单元 (Floating-point Unit). 一个仅对浮点数执行运算的 "数学处理器".

### HAL

硬件抽象层 (Hardware Abstraction Layer) crate 为微控制器的功能和外设 (Peripheral) 提供了对开发者友好的接口. 它通常在 [外设访问 crate (PAC)](#pac) 之上实现. 它也可能实现 [`embedded-hal`](https://crates.io/crates/embedded-hal) crate 中的 trait.
有关更详细的描述, 请参阅 [内存映射寄存器页面](../start/registers.md), 或查看 [此视频](https://youtu.be/vLYit_HHPaY) 以获得更广泛的概览.

### I2C

有时称为 `I²C` 或 Inter-IC. 它是一种用于单个集成电路内硬件通信的协议. 有关更多详细信息, 请参阅 [这里][i2c]

[i2c]: https://en.wikipedia.org/wiki/I2c

### PAC

外设访问 crate (Peripheral Access Crate) 提供对微控制器外设的访问. 它是较低级别的 crate 之一, 通常直接由提供的 [SVD](#svd) 生成, 通常使用 [svd2rust](https://github.com/rust-embedded/svd2rust/). [硬件抽象层](#hal) 通常依赖于这个 crate.
有关更详细的描述, 请参阅 [内存映射寄存器页面](../start/registers.md), 或查看 [此视频](https://youtu.be/vLYit_HHPaY) 以获得更广泛的概览.

### SPI

串行外设接口 (Serial Peripheral Interface). 有关更多信息, 请参阅 [这里][spi].

[spi]: https://en.wikipedia.org/wiki/Serial_peripheral_interface

### SVD

系统视图描述 (System View Description) 是一种 XML 文件格式, 用于描述微控制器设备的程序员视图. 你可以在 [ARM CMSIS 文档站点](https://www.keil.com/pack/doc/CMSIS/SVD/html/index.html) 上阅读更多相关信息.

### UART

通用异步收发器 (Universal asynchronous receiver-transmitter). 有关更多信息, 请参阅 [这里][uart].

[uart]: https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter

### USART

通用同步和异步收发器 (Universal synchronous and asynchronous receiver-transmitter). 有关更多信息, 请参阅 [这里][usart].

[usart]: https://en.wikipedia.org/wiki/Universal_synchronous_and_asynchronous_receiver-transmitter
