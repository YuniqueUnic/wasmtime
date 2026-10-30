# Wasmtime 49.0.1：epoch 回调前后同步 fuel

上游：[bytecodealliance/wasmtime v49.0.1](https://github.com/bytecodealliance/wasmtime/tree/v49.0.1)，
提交 `46c23a87dac1465986a8ad53ba6a7ae49372857b`。
维护仓库：`https://github.com/YuniqueUnic/wasmtime`，分支 `syzygy/epoch-fuel-v49.0.1`。

## 问题与最小修复

Cranelift 将 fuel 缓存在函数局部变量。无函数调用的循环会更新局部余额，
但 `epoch_check_full` 原先直接调用 `new_epoch`。回调中的 `Store::get_fuel()`
因此可能仍读到进入循环前的余额；回调中断后，宿主也无法追回未写回的局部消费。
局部 fuel 上限仍然有效，缺口是 epoch 边界的可见性，不能解释成循环没有计费。

仅在实际调用 `new_epoch` 的冷分支、且 `consume_fuel` 开启时，先调用既有
`fuel_save_from_var`，正常返回后调用 `fuel_load_into_var`。
前者保证回调和中断后的结算能读到已消耗余额；后者保留回调、重入或恢复期间
对 Store 余额的修改。未过期的循环路径不增加 Store 写入。
不增加预算，不改变应用的重试、超时、取消或插件权限策略。

这项修复只保证实际 epoch 回调的余额可见性。上游对其他动态 trap 仍采用
近似计费；例如未经过下一次写回就发生除零，函数局部的尾段消费可能不可见。
局部 fuel 上限仍有效，但不能把本补丁写成“所有 trap 都精确结算”。

## Cargo 接入

只替换 `wasmtime-internal-cranelift`。该目录的 `Cargo.toml` 使用同版本
crates.io 发布包的规范化 manifest，并添加独立 `[workspace]`。
这项 manifest 调整是必要边界：原 manifest 从上游 workspace 继承本地 path
依赖，仅替换 leaf 会引入另一份 `wasmtime-environ`、core 等类型身份。
规范化 manifest 保持原发布依赖、版本、features 和 lints，其他 Wasmtime 包仍用
crates.io 的 **49.0.1**。源文件保留完整上游仓库结构和历史。

Syzygy 主 workspace 与归档中的独立业务驱动分别声明 `[patch.crates-io]`。
Cargo 不会把依赖 workspace 的 patch 传给调用者；新增独立 Host 时必须在其
workspace 根声明同一 patch，并核对 lockfile 中只有这一份 compiler。

## 验证与删除条件

消费方定向检查：`mbx check -p syzygy-plugin-runtime --features wasm`。
业务验证入口是 Syzygy 的 `prompt_plugin_business` example 和归档的
`prompt-compute-business`，覆盖活跃计算的父预算结算、停用、取消、直接 Drop，
以及普通文档发布、CAS、Git 回滚和停用后的恢复。实际结果归档在 Syzygy 的
`docs/plugins/implementation/`，不能用编译通过代替业务通过。

上游发布等价 fuel 同步修复后，先重跑上述旅程，再删除该 patch 和精确 pin。
不要在升级时静默回到未修复 compiler，也不要为绕过此边界添加第二套应用计数器。
