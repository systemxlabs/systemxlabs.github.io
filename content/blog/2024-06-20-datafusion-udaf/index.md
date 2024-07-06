+++
title = "DataFusion 查询引擎 UDAF 设计"
date = 2024-06-20
draft = true
+++

UDAF 是指用户自定义聚合函数（User Defined Aggregate Functions），函数为输入的多行聚合生成一行输出，它可以用于常规的聚合函数（Group By）也可用于窗口函数（Over）。DataFusion 中使用 [AggregateUDFImpl] trait 对 UDAF 进行抽象，用户 UDAF 只需实现该 trait 并将其注册到 SessionContext 即可使用。DataFusion 内置聚合函数和用户 UDAF 均使用同一套 API [AggregateUDFImpl] 实现。

```rust
pub trait AggregateUDFImpl: Debug + Send + Sync {
    fn as_any(&self) -> &dyn Any;

    fn name(&self) -> &str;

    fn signature(&self) -> &Signature;

    fn return_type(&self, arg_types: &[DataType]) -> Result<DataType>;

    fn accumulator(&self, acc_args: AccumulatorArgs) -> Result<Box<dyn Accumulator>>;

    fn state_fields(&self, args: StateFieldsArgs) -> Result<Vec<Field>> {
        let fields = vec![Field::new(
            format_state_name(args.name, "value"),
            args.return_type.clone(),
            true,
        )];

        Ok(fields
            .into_iter()
            .chain(args.ordering_fields.to_vec())
            .collect())
    }

    fn groups_accumulator_supported(&self, _args: AccumulatorArgs) -> bool {
        false
    }

    fn create_groups_accumulator(
        &self,
        _args: AccumulatorArgs,
    ) -> Result<Box<dyn GroupsAccumulator>> {
        not_impl_err!("GroupsAccumulator hasn't been implemented for {self:?} yet")
    }

    fn aliases(&self) -> &[String] {
        &[]
    }

    fn create_sliding_accumulator(
        &self,
        args: AccumulatorArgs,
    ) -> Result<Box<dyn Accumulator>> {
        self.accumulator(args)
    }

    fn with_beneficial_ordering(
        self: Arc<Self>,
        _beneficial_ordering: bool,
    ) -> Result<Option<Arc<dyn AggregateUDFImpl>>> {
        if self.order_sensitivity().is_beneficial() {
            return exec_err!(
                "Should implement with satisfied for aggregator :{:?}",
                self.name()
            );
        }
        Ok(None)
    }

    fn order_sensitivity(&self) -> AggregateOrderSensitivity {
        AggregateOrderSensitivity::HardRequirement
    }

    fn simplify(&self) -> Option<AggregateFunctionSimplification> {
        None
    }

    fn reverse_expr(&self) -> ReversedUDAF {
        ReversedUDAF::NotSupported
    }

    fn coerce_types(&self, _arg_types: &[DataType]) -> Result<Vec<DataType>> {
        not_impl_err!("Function {} does not implement coerce_types", self.name())
    }
}
```

[AggregateUDFImpl]: https://docs.rs/datafusion/38.0.0/datafusion/logical_expr/trait.AggregateUDFImpl.html