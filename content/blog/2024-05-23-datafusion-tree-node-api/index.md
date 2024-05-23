+++
title = "DataFusion 查询引擎 TreeNode API"
date = 2024-05-23
draft = true
+++

```rust
pub trait TreeNode: Sized {
    // 应用 f 函数到当前节点的所有子节点
    // f 函数接收子节点的不可变引用，对子节点做检查
    // 返回控制指令（Continue 继续遍历、Jump跳过、Stop 停止遍历）
    fn apply_children<F: FnMut(&Self) -> Result<TreeNodeRecursion>>(
        &self,
        f: F,
    ) -> Result<TreeNodeRecursion>;

    // 应用 f 函数到当前节点的所有子节点
    // f 函数接收子节点的所有权，对子节点做变换
    // 返回新的当前节点
    fn map_children<F: FnMut(Self) -> Result<Transformed<Self>>>(
        self,
        f: F,
    ) -> Result<Transformed<Self>>;
}
```