# 作为状态机 (State Machines) 的外设

微控制器的外设可以看作是一组状态机 (State Machines). 例如, 一个简化的 [GPIO pin] 的配置可以表示为以下状态树:

[GPIO pin]: https://en.wikipedia.org/wiki/General-purpose_input/output

* Disabled
* Enabled
    * Configured as Output
        * Output: High
        * Output: Low
    * Configured as Input
        * Input: High Resistance
        * Input: Pulled Low
        * Input: Pulled High

如果外设从 `Disabled` 模式启动, 要移动到 `Input: High Resistance` 模式, 我们必须执行以下步骤:

1. Disabled
2. Enabled
3. Configured as Input
4. Input: High Resistance

如果我们想从 `Input: High Resistance` 移动到 `Input: Pulled Low`, 我们必须执行以下步骤:

1. Input: High Resistance
2. Input: Pulled Low

类似地, 如果我们想将 GPIO 引脚从 `Input: Pulled Low` 配置移动到 `Output: High`, 我们必须执行以下步骤:

1. Input: Pulled Low
2. Configured as Input
3. Configured as Output
4. Output: High

## 硬件表示

通常, 上面列出的状态是通过向映射到 GPIO 外设的给定寄存器写入值来设置的. 让我们定义一个假想的 GPIO Configuration Register 来阐明这一点:

| Name         | Bit Number(s) | Value | Meaning   | Notes |
| ---:         | ------------: | ----: | ------:   | ----: |
| enable       | 0             | 0     | disabled  | 禁用 GPIO |
|              |               | 1     | enabled   | 启用 GPIO |
| direction    | 1             | 0     | input     | 将方向设置为 Input |
|              |               | 1     | output    | 将方向设置为 Output |
| input_mode   | 2..3          | 00    | hi-z      | 将输入设置为高阻态 |
|              |               | 01    | pull-low  | 输入引脚被拉低 |
|              |               | 10    | pull-high | 输入引脚被拉高 |
|              |               | 11    | n/a       | 无效状态, 不要设置 |
| output_mode  | 4             | 0     | set-low   | 输出引脚被驱动为低电平 |
|              |               | 1     | set-high  | 输出引脚被驱动为高电平 |
| input_status | 5             | x     | in-val    | 输入 < 1.5v 时为 0, 输入 >= 1.5v 时为 1 |

我们*可以*在 Rust 中暴露以下结构来控制此 GPIO:

```rust,ignore
/// GPIO 接口
struct GpioConfig {
    /// 由 svd2rust 生成的 GPIO Configuration 结构
    periph: GPIO_CONFIG,
}

impl GpioConfig {
    pub fn set_enable(&mut self, is_enabled: bool) {
        self.periph.modify(|_r, w| {
            w.enable().set_bit(is_enabled)
        });
    }

    pub fn set_direction(&mut self, is_output: bool) {
        self.periph.modify(|_r, w| {
            w.direction().set_bit(is_output)
        });
    }

    pub fn set_input_mode(&mut self, variant: InputMode) {
        self.periph.modify(|_r, w| {
            w.input_mode().variant(variant)
        });
    }

    pub fn set_output_mode(&mut self, is_high: bool) {
        self.periph.modify(|_r, w| {
            w.output_mode.set_bit(is_high)
        });
    }

    pub fn get_input_status(&self) -> bool {
        self.periph.read().input_status().bit_is_set()
    }
}
```

但是, 这将允许我们修改一些没有意义的寄存器. 例如, 如果在 GPIO 配置为输入时设置 `output_mode` 字段会发生什么?

一般而言, 使用此结构将允许我们达到上面状态机未定义的状态: 例如, 被拉低的输出, 或被置高的输入. 对于某些硬件, 这可能无关紧要. 而对于其他硬件, 它可能会导致意外或未定义的行为!

虽然这个接口写起来很方便, 但它并没有强制执行我们硬件实现所规定的设计契约 (Design Contract).
