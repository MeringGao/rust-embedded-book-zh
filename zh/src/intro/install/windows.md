# Windows

## `arm-none-eabi-gdb`

ARM 提供了 `.exe` 安装程序用于 Windows. 从[这里][gcc]下载一个, 并按照说明操作.
在安装过程即将结束时, 勾选/选择 "Add path to environment variable" (添加到环境变量) 选项. 然后验证工具是否已加入你的 `%PATH%`:

``` text
$ arm-none-eabi-gdb -v
GNU gdb (GNU Tools for Arm Embedded Processors 7-2018-q2-update) 8.1.0.20180315-git
(..)
```

[gcc]: https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads

## OpenOCD

Windows 上没有 OpenOCD 的官方二进制发布版本, 但如果你不想自己编译, xPack 项目提供了一个二进制发行版, 见[这里][openocd]. 按照提供的安装说明进行操作. 然后将二进制文件所在的路径加入你的 `%PATH%` 环境变量.
(如果你使用的是简易安装方式, 路径类似 `C:\Users\USERNAME\AppData\Roaming\xPacks\@xpack-dev-tools\openocd\0.10.0-13.1\.content\bin\`)

[openocd]: https://xpack.github.io/openocd/

使用以下命令验证 OpenOCD 是否已加入你的 `%PATH%`:

``` text
$ openocd -v
Open On-Chip Debugger 0.10.0
(..)
```

## QEMU

从[官方网站][qemu]下载 QEMU.

[qemu]: https://www.qemu.org/download/#windows

## ST-LINK USB 驱动

你还需要安装[此 USB 驱动][this USB driver], 否则 OpenOCD 无法工作. 按照安装程序的说明操作, 并确保安装了正确版本 (32 位或 64 位) 的驱动.

[this USB driver]: http://www.st.com/en/embedded-software/stsw-link009.html

完成! 进入[下一节][next section].

[next section]: verify.md