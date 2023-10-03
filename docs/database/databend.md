# Databend

## contexts:

主要用于加载选项设置，数据源与查询统计信息。

- Settings: 设置某个变量的类型，支持的类型包括：u64, i6, f64和string。支持的操作包括：set(设置value, default value和description), update(更新value), get(获取value)
- Options:设置了log_level, num_cpus, mysql_listrn_host, mysql_handler_port, mysql_handler_thread_num的变量类型，value以及description。
- FuseQueryContext: 设置了max_threads, max_block_size, default_db的变量类型，value以及description。

```rust
pub struct FuseQueryContext {
    datasource: Arc<Mutex<dyn IDataSource>>,
    statistics: Mutex<Statistics>,
    settings: Settings,
}

pub type FuseQueryContextRef = Arc<FuseQueryContext>;
```

------

## datasources:

这一层抽象是为了更好实现不同类型文件的读取。

- DataSource:实现IDataSource特征（add_database, check_database, add_table, get_table）。初始化时会注册一个叫system的database。
- ITable: 特征。需要实现name, schema, read_plan和read的函数。
- Statistics：统计读取的函数和字节信息。

```rust
pub struct DataSource {
    databases: HashMap<String, HashMap<String, Arc<dyn ITable>>>,
}

#[async_trait]
pub trait ITable: Sync + Send {
    fn name(&self) -> &str;

    fn schema(&self) -> FuseQueryResult<DataSchemaRef>;

    fn read_plan(
        &self,
        ctx: FuseQueryContextRef,
        push_down_plan: PlanNode,
    ) -> FuseQueryResult<ReadDataSourcePlan>;

    async fn read(
        &self,
        ctx: FuseQueryContextRef,
        parts: Vec<Partition>,
    ) -> FuseQueryResult<SendableDataBlockStream>;
}

#[derive(Clone, Debug)]
pub struct Partition {
    pub name: String,
    pub version: u64,
}
```

------

## datavalues

对列向量进行抽象。

- DataSchema为arrow::datatypes::Schema, DataSchemaRef为arrow::datatypes::SchemaRef,
- DataType用的是arrow::datatypes::DataType。实现了数字类型变量的强制转换。
- DataField为arrow::datatypes::Field
- DataArray: 向量。DataArrayRef为arrow::array::ArrayRef。
- DataValue: 标量。 能与DataArray, rust标准类型，DataType相互转换。实现了算术运算与聚合运算。
- DataColumnarValue:实现了And与Or的逻辑运算，比较运算，算术运算，聚合运算。（如果操作对象是标量则转换为向量）

```rust
#[derive(Clone, Debug)]
pub enum DataColumnarValue {
    // Array of values.
    Array(DataArrayRef),
    // A Single value.
    Scalar(DataValue),
}
```

------

## datablocks:

对多行列向量进行抽象。

​	能与arrow::record_batch::RecordBatch相互转换。

```rust
#[derive(Debug, Clone)]
pub struct DataBlock {
    schema: DataSchemaRef,
    columns: Vec<DataArrayRef>,
}
```

------

## servers:

- Session: 目前实现了on_query，首先根据输入的sql语句与contexts通过build_from_sql方法生成查询计划。然后由ExecutorFactory根据查询计划与contexts生成物理查询计划executor。然后调用executor.execute()得到最后结果。
- MySQLHandler: 启动服务。
- MySQLStream: 输出格式。

```rust
struct Session {
    ctx: FuseQueryContextRef,
}

pub struct MySQLHandler {
    opts: Options,
    datasource: Arc<Mutex<dyn IDataSource>>,
}

pub struct MySQLStream {
    blocks: Vec<DataBlock>,
}
```

------

## planners:

主要用于生成逻辑查询计划。

- plan_parser: build_from_sql首先将sql语句转换为statements(暂时只支持Query与SetVariable, 通过DFParser实现)。对于Query statements，按having -> from-> filter-> projection -> aggr -> limit生成计划(主要通过PlanBuilder生成PlanNode, 在处理from时，会进一步调用table.read_plan生成ReadSource的PlanNode)。对于表达式，通过sql_to_rex生成ExpressionPlan。

```rust
#[derive(Clone)]
pub enum PlanNode {
    Empty(EmptyPlan),
    Projection(ProjectionPlan),
    Aggregate(AggregatePlan),
    Filter(FilterPlan),
    Limit(LimitPlan),
    Scan(ScanPlan),
    ReadSource(ReadDataSourcePlan),
    Explain(ExplainPlan),
    Select(SelectPlan),
    SetVariable(SettingPlan),
}

#[derive(Clone)]
pub enum ExpressionPlan {
    Alias(String, Box<ExpressionPlan>),
    Field(String),
    Constant(DataValue),
    BinaryExpression {
        left: Box<ExpressionPlan>,
        op: String,
        right: Box<ExpressionPlan>,
    },
    Function {
        op: String,
        args: Vec<ExpressionPlan>,
    },
    Wildcard,
}
```

------

## executors:

由于查询计划可能需要优化，而其他执行计划可能不需要，因此需要这样一层。

- ExecutorFactory: 将PlanNode转换为IExecutor。对于SelectExecutor来说，会首先调用Optimizer::create().optimize得到优化后的PlanNode。然后调用PipelineBuilder执行计划。

------

## Optimizers:

主要用于对逻辑查询计划的优化。

- Optimizer: 调用create添加需要应用的优化规则。再调用optimize按顺序优化查询计划。目前实现的filter_push_down只完成了projection算子中的别名表达式替换。

```rust
pub trait IOptimizer {
    fn name(&self) -> &str;
    fn optimize(&mut self, plan: &PlanNode) -> FuseQueryResult<PlanNode>;
}

pub struct Optimizer {
    optimizers: Vec<Box<dyn IOptimizer>>,
}
```

------

## processors:

主要用于构建进程之间的执行顺序。

- 对于ReadSource的PlanNode, 会将输入进行分区，对于每个分区产生一个物理执行计划，之后可能会依次进行filter, projection, aggregate和limit。其中aggregate和Limit时pipeline breaker，通过调用merge_processor来进行等待之前所有进程的完成。

```rust
#[async_trait]
pub trait IProcessor: Sync + Send {
    /// Processor name.
    fn name(&self) -> &str;

    /// Connect to the input processor, add an edge on the DAG.
    fn connect_to(&mut self, input: Arc<dyn IProcessor>) -> FuseQueryResult<()>;

    /// Execute the processor.
    async fn execute(&self) -> FuseQueryResult<SendableDataBlockStream>;

    /// Format the processor.
    fn format(
        &self,
        f: &mut std::fmt::Formatter,
        setting: &mut FormatterSettings,
    ) -> std::fmt::Result {
        if setting.indent > 0 {
            writeln!(f)?;
            for _ in 0..setting.indent {
                write!(f, "{}", setting.indent_char)?;
            }
        }
        write!(
            f,
            "{} {} × {} {}",
            setting.prefix,
            self.name(),
            setting.ways,
            if setting.ways == 1 {
                "processor"
            } else {
                "processors"
            },
        )
    }
}

pub type Pipe = Vec<Arc<dyn IProcessor>>;

pub struct Pipeline {
    processors: Vec<Pipe>,
}
```

------

## functions:

主要用于对向量真正进行运算。

每个functions通过eval方法得到DataColumnarValue。其中聚合函数需要实现accumulate方法将结果存储在self.state中，merge方法会把同一深度的self.state进行聚合。

- FunctionFactory: 通过get方法产生算数，比较，逻辑，聚合以及UDF Function

------

## transforms:

主要用于实现对每个DataBlock的物理查询计划执行。

- source: 调用table.read进行分区的读取。
- 每个进程通过transforms实现executor方法，transforms会将表达式转换为functions，并实现expression_executor方法(对每个datablock执行functions的eval操作并生成新的DataBlock）。然后将functions和expression_executor方法传给datastream进行执行。

------

## datastreams:

将数据流划分成数据块进行执行。

```rust
pub type SendableDataBlockStream = std::pin::Pin<
    Box<dyn futures::stream::Stream<Item = FuseQueryResult<DataBlock>> + Sync + Send>,
>;
```

