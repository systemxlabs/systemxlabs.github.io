+++
title = "DataFusion 查询引擎 TreeNode API"
date = 2024-05-23
draft = true
+++

数据库系统中有很多树形数据结构（如逻辑计划、表达式、物理计划），经常需要对这些树形结构遍历来进行检查（Inspecting）或者变换（Transforming），因此设计一个好的 API 可以事半功倍。

## 底层 API
```rust
pub trait TreeNode: Sized {
    fn apply_children<'n, F: FnMut(&'n Self) -> Result<TreeNodeRecursion>>(
        &'n self,
        f: F,
    ) -> Result<TreeNodeRecursion>;

    fn map_children<F: FnMut(Self) -> Result<Transformed<Self>>>(
        self,
        f: F,
    ) -> Result<Transformed<Self>>;

    ...
}

pub enum TreeNodeRecursion {
    Continue,
    Jump,
    Stop,
}

pub struct Transformed<T> {
    pub data: T,
    pub transformed: bool,
    pub tnr: TreeNodeRecursion,
}
```

这是两个底层 API，主要用来实现其他更高级的 API。而且它没有默认实现，因此必须手动实现。
- `apply_children` 接收一个 f 函数，将 f 函数应用到其每个子节点上，对每个子节点进行检查
- `map_children` 接收一个 f 函数，将 f 函数应用到其每个子节点上，对每个子节点进行变换，产生新的子节点，最后更新并返回新的父节点

## 默认检查 API
```rust
pub trait TreeNode: Sized {
    fn apply<'n, F: FnMut(&'n Self) -> Result<TreeNodeRecursion>>(
        &'n self,
        mut f: F,
    ) -> Result<TreeNodeRecursion> {
        fn apply_impl<'n, N: TreeNode, F: FnMut(&'n N) -> Result<TreeNodeRecursion>>(
            node: &'n N,
            f: &mut F,
        ) -> Result<TreeNodeRecursion> {
            f(node)?.visit_children(|| node.apply_children(|c| apply_impl(c, f)))
        }

        apply_impl(self, &mut f)
    }
    
    ...
}

impl TreeNodeRecursion {
    pub fn visit_children<F: FnOnce() -> Result<TreeNodeRecursion>>(
        self,
        f: F,
    ) -> Result<TreeNodeRecursion> {
        match self {
            TreeNodeRecursion::Continue => f(),
            TreeNodeRecursion::Jump => Ok(TreeNodeRecursion::Continue),
            TreeNodeRecursion::Stop => Ok(self),
        }
    }
}
```
`apply` API 接收一个 f 函数，从当前节点（作为根）开始自上而下前序遍历整个树，对每个节点应用 f 函数进行检查。

## 默认变换 API
```rust
pub trait TreeNode: Sized {
    fn transform_down<F: FnMut(Self) -> Result<Transformed<Self>>>(
        self,
        mut f: F,
    ) -> Result<Transformed<Self>> {
        fn transform_down_impl<N: TreeNode, F: FnMut(N) -> Result<Transformed<N>>>(
            node: N,
            f: &mut F,
        ) -> Result<Transformed<N>> {
            f(node)?.transform_children(|n| n.map_children(|c| transform_down_impl(c, f)))
        }

        transform_down_impl(self, &mut f)
    }

    fn transform_up<F: FnMut(Self) -> Result<Transformed<Self>>>(
        self,
        mut f: F,
    ) -> Result<Transformed<Self>> {
        fn transform_up_impl<N: TreeNode, F: FnMut(N) -> Result<Transformed<N>>>(
            node: N,
            f: &mut F,
        ) -> Result<Transformed<N>> {
            node.map_children(|c| transform_up_impl(c, f))?
                .transform_parent(f)
        }

        transform_up_impl(self, &mut f)
    }

    ...
}
```
- `transform_down` API 接收一个 f 函数，从当前节点（作为根）开始自上而下前序遍历整个树，对每个节点应用 f 函数进行变换，返回新的树。
- `transform_up` API 接收一个 f 函数，从当前节点开始自下而上后序遍历整个树，对每个节点应用 f 函数进行变换，返回新的树。