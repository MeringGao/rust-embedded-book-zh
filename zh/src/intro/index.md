# 介绍 (Introduction)

欢迎阅读 *The Embedded Rust Book*: 这是一本关于如何在"裸机 (Bare Metal)" 嵌入式系统 (例如微控制器 (Microcontroller)) 上使用 Rust 编程语言的入门书籍.

## 嵌入式 Rust 适用人群
嵌入式 Rust 适合每一位希望在嵌入式编程中利用 Rust 语言提供的更高级概念与安全保证的开发者.
(另见 [Rust 适用人群](https://doc.rust-lang.org/book/ch00-00-introduction.html))

## 本书范围

本书的目标包括:

* 帮助开发者快速上手嵌入式 Rust 开发, 即如何搭建开发环境.

* 分享使用 Rust 进行嵌入式开发的 *当前* 最佳实践, 即如何最佳地使用 Rust 语言功能来编写更正确的嵌入式软件.

* 在某些情况下充当 *cookbook*, 例如, 如何在单个项目中混合使用 C 与 Rust?

本书力求尽可能通用, 但为了方便读者和作者, 在所有示例中都使用 ARM Cortex-M 架构. 但本书并不假设读者熟悉该特定架构, 并在需要时解释该架构特有的细节.

## 本书的目标读者
本书面向具有一定嵌入式背景或一定 Rust 背景的读者, 然而我们相信每一位对嵌入式 Rust 编程感到好奇的人都能从本书中获得收获. 对于没有任何先验知识的读者, 我们建议先阅读 "前提与假设" 一节, 补齐缺失的知识, 以便从本书中获得更多收获并改善阅读体验. 你可以查看 "其他资源" 一节, 找到你想要补齐的主题的资源.

### 前提与假设

* 你能够熟练使用 Rust 编程语言, 并且已经在桌面环境中编写, 运行与调试过 Rust 应用程序. 你还应该熟悉 [2018 edition] 的惯用法, 因为本书以 Rust 2018 为目标.

[2018 edition]: https://doc.rust-lang.org/edition-guide/

* 你能够熟练地使用另一种语言 (如 C, C++ 或 Ada) 开发与调试嵌入式系统, 并且熟悉以下概念:
    * 交叉编译 (Cross Compilation)
    * 内存映射外设 (Memory Mapped Peripherals)
    * 中断 (Interrupts)
    * 常见接口, 如 I2C, SPI, Serial 等.

### 其他资源
如果你对上述任何内容不熟悉, 或者想要获取本书中特定主题的更多信息, 以下几个资源可能会对你有所帮助.

| 主题        | 资源 | 描述 |
|--------------|----------|-------------|
| Rust         | [Rust Book](https://doc.rust-lang.org/book/) | 如果你还不熟悉 Rust, 我们强烈建议先阅读本书. |
| Rust, 嵌入式 | [Discovery Book](https://docs.rust-embedded.org/discovery/) | 如果你从未做过任何嵌入式编程, 本书可能是更好的起点 |
| Rust, 嵌入式 | [Embedded Rust Bookshelf](https://docs.rust-embedded.org) | 你可以在这里找到 Rust 嵌入式工作组提供的其他若干资源. |
| Rust, 嵌入式 | [Embedonomicon](https://docs.rust-embedded.org/embedonomicon/) | Rust 嵌入式编程的细致入微的细节. |
| Rust, 嵌入式 | [embedded FAQ](https://docs.rust-embedded.org/faq.html) | 关于 Rust 嵌入式的常见问题. |
| Rust, 嵌入式 | [Comprehensive Rust 🦀: Bare Metal](https://google.github.io/comprehensive-rust/bare-metal.html) | 关于裸机 Rust 开发的 4 天课程的教学材料 |
| 中断 | [Interrupt](https://en.wikipedia.org/wiki/Interrupt) | - |
| 内存映射 I/O 与外设 | [Memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O) | - |
| SPI, UART, RS232, USB, I2C, TTL | [Stack Exchange about SPI, UART, and other interfaces](https://electronics.stackexchange.com/questions/37814/usart-uart-rs232-usb-spi-i2c-ttl-etc-what-are-all-of-these-and-how-do-th) | - |

### 翻译版本

本书由热心的志愿者翻译. 如果你希望你的翻译版本列在这里, 请提交 PR 来添加。

- [日语版](https://tomoyuki-nakabayashi.github.io/book/)
  ([仓库](https://github.com/tomoyuki-nakabayashi/book))

- [中文版](https://meringgao.github.io/rust-embedded-book-zh/)
  ([仓库](https://github.com/MeringGao/rust-embedded-book-zh))

## 如何使用本书

本书通常假设你会从前到后依次阅读. 后面的章节建立在前面章节的概念之上, 而前面的章节可能不会深入某个主题的细节, 而是在后续章节中重新讨论该主题.

本书将使用 STMicroelectronics 的 [STM32F3DISCOVERY] 开发板来完成大部分示例. 该开发板基于 ARM Cortex-M 架构, 虽然基于该架构的大多数 CPU 的基本功能相同, 但不同厂商的微控制器之间的外设与其他实现细节存在差异, 即使是同一厂商的不同微控制器系列之间也常常存在差异.

因此, 我们建议你购买 [STM32F3DISCOVERY] 开发板来跟随本书的示例.

[STM32F3DISCOVERY]: http://www.st.com/en/evaluation-tools/stm32f3discovery.html

## 为本书做贡献

本书的工作在 [该仓库] 中进行协调, 主要由 [资源团队 (resources team)] 开发.

[该仓库]: https://github.com/rust-embedded/book
[资源团队 (resources team)]: https://github.com/rust-embedded/wg#the-resources-team

如果你在跟随本书的过程中遇到困难, 或发现本书中某些章节不够清晰或难以理解, 那么这就是一个 bug, 应该在本书的 [issue 跟踪器] 中报告.

[issue 跟踪器]: https://github.com/rust-embedded/book/issues/

非常欢迎提交修复错别字或新增内容的 PR!

## 复用本书内容

本书按以下许可证发布:

* 本书中包含的代码示例和独立 Cargo 项目均按 [MIT License] 与 [Apache License v2.0] 双重许可.
* 本书中包含的书面文字, 图片与图表均按 Creative Commons [CC-BY-SA v4.0] 许可证发布.

[MIT License]: https://opensource.org/licenses/MIT
[Apache License v2.0]: http://www.apache.org/licenses/LICENSE-2.0
[CC-BY-SA v4.0]: https://creativecommons.org/licenses/by-sa/4.0/legalcode

简而言之: 如果你想在你的作品中复用我们的文字或图片, 你需要:

* 给出恰当的署名 (即在你的幻灯片中提及本书, 并提供相关页面的链接)
* 提供 [CC-BY-SA v4.0] 许可证的链接
* 注明你是否对材料进行了任何修改, 并允许以相同许可证发布你对材料所做的修改

另外, 如果你认为本书有用, 请告诉我们!