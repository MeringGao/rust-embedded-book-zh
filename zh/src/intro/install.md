# 安装工具

本页面包含一些与操作系统无关的工具安装说明:

### Rust 工具链 (Toolchain)

按照 [https://rustup.rs](https://rustup.rs) 上的说明安装 rustup.

**注意** 请确保你的编译器版本等于或高于 `1.31`. `rustc -V` 应返回比下面显示的日期更新的版本.

``` text
$ rustc -V
rustc 1.31.1 (b6c32da9b 2018-12-18)
```

出于带宽和磁盘占用的考虑, 默认安装仅支持本地编译. 要添加对 ARM Cortex-M 架构的交叉编译 (Cross Compilation) 支持, 请从以下编译目标中选择一个. 对于本书示例中使用的 STM32F3DISCOVERY 板子, 请使用 `thumbv7em-none-eabihf` 目标.
[为你寻找最合适的 Cortex-M.](https://developer.arm.com/ip-products/processors/cortex-m#c-7d3b69ce-5b17-4c9e-8f06-59b605713133) 

Cortex-M0、M0+ 和 M1 (ARMv6-M 架构):
``` console
rustup target add thumbv6m-none-eabi
```

Cortex-M3 (ARMv7-M 架构):
``` console
rustup target add thumbv7m-none-eabi
```

Cortex-M4 和 M7 (无硬件浮点) (ARMv7E-M 架构):
``` console
rustup target add thumbv7em-none-eabi
```

Cortex-M4F 和 M7F (带硬件浮点) (ARMv7E-M 架构):
``` console
rustup target add thumbv7em-none-eabihf
```

Cortex-M23 (ARMv8-M 架构):
``` console
rustup target add thumbv8m.base-none-eabi
```

Cortex-M33 和 M35P (ARMv8-M 架构):
``` console
rustup target add thumbv8m.main-none-eabi
```

Cortex-M33F 和 M35PF (带硬件浮点) (ARMv8-M 架构):
``` console
rustup target add thumbv8m.main-none-eabihf
```


### `cargo-binutils`

``` text
cargo install cargo-binutils

rustup component add llvm-tools
```
WINDOWS: 前置条件: 已安装 Visual Studio 2019 的 C++ 生成工具. https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=BuildTools&rel=16 
### `cargo-generate`

我们稍后会使用它来从模板生成项目.

``` console
cargo install cargo-generate
```

注意: 在某些 Linux 发行版 (例如 Ubuntu) 上, 你可能需要先安装 `libssl-dev` 和 `pkg-config` 软件包, 然后再安装 cargo-generate.

### 操作系统特定说明

现在, 请按照你所使用操作系统的特定说明进行操作:

- [Linux](install/linux.md)
- [Windows](install/windows.md)
- [macOS](install/macos.md)
