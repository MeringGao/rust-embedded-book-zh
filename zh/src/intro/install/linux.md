# Linux

以下是几个 Linux 发行版的安装命令.

## 软件包

- Ubuntu 18.04 或更新版本 / Debian stretch 或更新版本

> **注意** `gdb-multiarch` 是用于调试 ARM Cortex-M 程序的 GDB 命令

<!-- Debian stretch -->
<!-- GDB 7.12 -->
<!-- OpenOCD 0.9.0 -->
<!-- QEMU 2.8.1 -->

<!-- Ubuntu 18.04 -->
<!-- GDB 8.1 -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.11.1 -->

``` console
sudo apt install gdb-multiarch openocd qemu-system-arm
```

- Ubuntu 14.04 和 16.04

> **注意** `arm-none-eabi-gdb` 是用于调试 ARM Cortex-M 程序的 GDB 命令

<!-- Ubuntu 14.04 -->
<!-- GDB 7.6 (!) -->
<!-- OpenOCD 0.7.0 (?) -->
<!-- QEMU 2.0.0 (?) -->

``` console
sudo apt install gdb-arm-none-eabi openocd qemu-system-arm
```

- Fedora 27 或更新版本

<!-- Fedora 27 -->
<!-- GDB 7.6 (!) -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.10.2 -->

``` console
sudo dnf install gdb openocd qemu-system-arm
```

- Arch Linux

> **注意** `arm-none-eabi-gdb` 是用于调试 ARM Cortex-M 程序的 GDB 命令

``` console
sudo pacman -S arm-none-eabi-gdb qemu-system-arm openocd
```

## udev 规则 (udev rules)

这条规则让你无需 root 权限即可在 Discovery 开发板上使用 OpenOCD.

创建文件 `/etc/udev/rules.d/70-st-link.rules`, 内容如下所示.

``` text
# STM32F3DISCOVERY rev A/B - ST-LINK/V2
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="3748", TAG+="uaccess"

# STM32F3DISCOVERY rev C+ - ST-LINK/V2-1
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374b", TAG+="uaccess"
```

然后用以下命令重新加载所有 udev 规则:

``` console
sudo udevadm control --reload-rules
```

如果之前已将开发板插入笔记本电脑, 请先拔出再重新插入.

你可以通过运行以下命令检查权限:

``` console
lsusb
```

输出应该类似如下

```text
(..)
Bus 001 Device 018: ID 0483:374b STMicroelectronics ST-LINK/V2.1
(..)
```

请记下总线 (bus) 和设备 (device) 编号. 使用这些编号构造路径 `/dev/bus/usb/<bus>/<device>`. 然后像下面这样使用该路径:

``` console
ls -l /dev/bus/usb/001/018
```

```text
crw-------+ 1 root root 189, 17 Sep 13 12:34 /dev/bus/usb/001/018
```

```console
getfacl /dev/bus/usb/001/018 | grep user
```

```text
user::rw-
user:you:rw-
```

权限末尾的 `+` 表示存在扩展权限. `getfacl` 命令告诉你用户 `you` 可以使用此设备.

现在, 进入[下一节][next section].

[next section]: verify.md