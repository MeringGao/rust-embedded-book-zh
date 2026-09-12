# macOS

所有工具都可以通过 [Homebrew] 或 [MacPorts] 安装:

[Homebrew]: http://brew.sh/
[MacPorts]: https://www.macports.org/

## 使用 [Homebrew] 安装工具

``` text
$ # GDB
$ brew install arm-none-eabi-gdb

$ # OpenOCD
$ brew install openocd

$ # QEMU
$ brew install qemu
```

> **注意** 如果 OpenOCD 崩溃, 你可能需要使用以下命令安装最新版本:
```text
$ brew install --HEAD openocd
```

## 使用 [MacPorts] 安装工具

``` text
$ # GDB
$ sudo port install arm-none-eabi-gcc

$ # OpenOCD
$ sudo port install openocd

$ # QEMU
$ sudo port install qemu
```



完成! 进入[下一节][next section].

[next section]: verify.md