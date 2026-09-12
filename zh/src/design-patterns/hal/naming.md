# 命名 (Naming)


<a id="c-crate-name"></a>
## crate 命名合适 (C-CRATE-NAME)

HAL crate 应该以它们要支持的芯片或芯片系列来命名. 它们的名称应该以 `-hal` 结尾, 以便与寄存器访问 crate 区分开. 名称不应包含下划线 (请改用短横线).
