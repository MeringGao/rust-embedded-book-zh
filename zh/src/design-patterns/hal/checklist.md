# HAL 设计模式清单

- **命名** *(crate 遵循 Rust 命名约定)*
  - [ ] crate 命名合适 ([C-CRATE-NAME])
- **互操作性** *(crate 与其他库的功能交互良好)*
  - [ ] 包装类型提供析构方法 ([C-FREE])
  - [ ] HAL 重新导出它们的寄存器访问 crate ([C-REEXPORT-PAC])
  - [ ] 类型实现 `embedded-hal` 的 trait ([C-HAL-TRAITS])
- **可预测性** *(crate 生成的代码易于阅读, 行为符合其外观)*
  - [ ] 使用构造函数而不是扩展 trait ([C-CTOR])
- **GPIO 接口** *(GPIO 接口遵循通用模式)*
  - [ ] 引脚类型默认为零大小类型 ([C-ZST-PIN])
  - [ ] 引脚类型提供抹除引脚和端口的方法 ([C-ERASED-PIN])
  - [ ] 引脚状态应编码为类型参数 ([C-PIN-STATE])

[C-CRATE-NAME]: naming.html#c-crate-name

[C-FREE]: interoperability.html#c-free
[C-REEXPORT-PAC]: interoperability.html#c-reexport-pac
[C-HAL-TRAITS]: interoperability.html#c-hal-traits

[C-CTOR]: predictability.html#c-ctor

[C-ZST-PIN]: gpio.md#c-zst-pin
[C-ERASED-PIN]: gpio.md#c-erased-pin
[C-PIN-STATE]: gpio.md#c-pin-state
