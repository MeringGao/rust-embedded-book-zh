# 验证安装 (Verify Installation)

在本节中, 我们将检查一些必需的工具 / 驱动是否已正确安装和配置.

使用 Mini-USB 数据线将笔记本电脑 / 台式机连接到 discovery 开发板. Discovery 开发板有两个 USB 接口; 请使用标有 "USB ST-LINK" 的那个, 它位于板子边缘的中央.

同时检查 ST-LINK 排针是否已焊接. 参见下图; ST-LINK 排针已在图中高亮标出.

<p align="center">
<img title="已连接的 discovery 开发板" src="../../assets/verify.jpeg">
</p>

现在运行以下命令:

``` console
openocd -f interface/stlink.cfg -f target/stm32f3x.cfg
```

> **注意**: 旧版本的 openocd, 包括 2017 年的 0.10.0 发布版, 并不包含新的 (也是更推荐的) `interface/stlink.cfg` 文件; 你可能需要改用 `interface/stlink-v2.cfg` 或 `interface/stlink-v2-1.cfg`.

你应该会看到以下输出, 并且程序会阻塞控制台:

``` text
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
none separate
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v27 API v2 SWIM v15 VID 0x0483 PID 0x374B
Info : using stlink api v2
Info : Target voltage: 2.919881
Info : stm32f3x.cpu: hardware has 6 breakpoints, 4 watchpoints
```

内容可能不完全一致, 但你应该会看到关于断点 (breakpoints) 和观察点 (watchpoints) 的最后一行. 如果看到了, 那么终止 OpenOCD 进程并进入[下一节][next section].

[next section]: ../../start/index.md

如果你没有看到 "breakpoints" 这一行, 请尝试以下命令之一.

``` console
openocd -f interface/stlink-v2.cfg -f target/stm32f3x.cfg
```

``` console
openocd -f interface/stlink-v2-1.cfg -f target/stm32f3x.cfg
```

如果这些命令之一可以工作, 说明你的 discovery 开发板是旧硬件版本. 这不会造成问题, 但请记住这一点, 因为稍后你需要用略有不同的方式配置. 你可以进入[下一节][next section].

如果以普通用户身份运行这些命令都不工作, 请尝试以 root 权限运行 (例如 `sudo openocd ..`). 如果使用 root 权限后命令可以工作, 那么请检查 [udev 规则][udev rules] 是否已正确设置.

[udev rules]: linux.md#udev-rules

如果你已经走到这一步, 但 OpenOCD 仍然无法工作, 请[提交一个 issue][an issue], 我们会帮你解决!

[an issue]: https://github.com/rust-embedded/book/issues