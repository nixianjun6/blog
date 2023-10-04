# Databend

## configs:

主要用于加载选项设置，重构了之前的Options。

```rust
#[derive(Debug, StructOpt, Clone)]
pub struct Config {
    #[structopt(default_value = "Unknown")]
    pub version: String,

    #[structopt(long, default_value = "debug")]
    pub log_level: String,

    #[structopt(long, default_value = "0")]
    pub num_cpus: u64,

    #[structopt(long, default_value = "127.0.0.1")]
    pub mysql_handler_host: String,

    #[structopt(long, default_value = "3307")]
    pub mysql_handler_port: u64,

    #[structopt(long, default_value = "256")]
    pub mysql_handler_thread_num: u64,

    #[structopt(long, default_value = "127.0.0.1:9090")]
    pub rpc_api_address: String,

    #[structopt(long, default_value = "127.0.0.1:8080")]
    pub http_api_address: String,

    #[structopt(long, default_value = "127.0.0.1:7070")]
    pub metric_api_address: String,
}
```

------

## clusters:

集群的抽象，记录了节点名字与节点相关信息的映射。

```rust
#[derive(serde::Serialize, serde::Deserialize, Clone, Debug, PartialEq)]
pub struct Node {
    pub name: String,
    pub cpus: usize,
    pub address: String,
}
```

------

## sessions:

原来的contexts把options功能解耦。记录context uuid与context的映射关系。

- Settings: 设置某个变量的类型，支持的类型包括：u64, i6, f64和string。支持的操作包括：set(设置value, default value和description), update(更新value), get(获取value)
- FuseQueryContext: 设置了max_threads, max_block_size, default_db的变量类型，value以及description。把原来的partition_queue从iTable中解耦到context中，并新增了uuid和cluster属性以支持分布式查询。

```rust
#[derive(Clone)]
pub struct FuseQueryContext {
    uuid: Arc<Mutex<String>>,
    settings: Settings,
    cluster: Arc<Mutex<ClusterRef>>,
    datasource: Arc<Mutex<dyn IDataSource>>,
    statistics: Arc<Mutex<Statistics>>,
    partition_queue: Arc<Mutex<VecDeque<Partition>>>,
}

pub struct Session {
    sessions: Mutex<HashMap<String, FuseQueryContextRef>>,
}
```

------

## servers:

- Session（与上一部分的Session不同，与上一版本的Session相同）: 目前实现了on_query，首先根据输入的sql语句与contexts通过build_from_sql方法生成查询计划。然后由InterpreterFactory根据查询计划与contexts生成物理查询计划executor。然后调用executor.execute()得到最后结果。并通过mysql stream发送给客户端。
- MySQLHandler: 启动服务。
- MySQLStream: 输出格式。

```rust
struct Session {
    ctx: FuseQueryContextRef,
}

pub struct MySQLHandler {
    conf: Config,
    cluster: ClusterRef,
    session_manager: SessionRef,
}

pub struct MySQLStream {
    blocks: Vec<DataBlock>,
}
```

------

## datasources:

这一层抽象是为了更好实现不同类型文件的读取。

- DataSource:实现IDataSource特征（add_database, check_database, add_table, get_table, list_database_tables）。初始化时会注册system， local,  remote三个database。
- ITable: 特征。需要实现name, schema, read_plan和read的函数。
- Statistics：统计读取的函数和字节信息。
- TableFactory: 创建Table的抽象。

```rust
pub struct DataSource {
    databases: HashMap<String, HashMap<String, Arc<dyn ITable>>>,
}

#[async_trait]
pub trait ITable: Sync + Send {
    fn name(&self) -> &str;
    fn engine(&self) -> &str;

    fn schema(&self) -> FuseQueryResult<DataSchemaRef>;

    fn read_plan(
        &self,
        ctx: FuseQueryContextRef,
        push_down_plan: PlanNode,
    ) -> FuseQueryResult<ReadDataSourcePlan>;

    async fn read(&self, ctx: FuseQueryContextRef) -> FuseQueryResult<SendableDataBlockStream>;
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

## sql:

- PlanParse: 从原来的planners中解耦出来。build_from_sql首先将sql语句转换为DFStatements(通过DFParser实现)。对于Statement::Query，按having -> from-> filter-> projection -> aggr -> limit生成计划(主要通过PlanBuilder生成PlanNode)
- 在处理from时，会进一步调用table.read_plan生成ReadSource的PlanNode)。
- 在生成projection计划时，会先调用planRewrite来重写别名。
- 在生成aggr计划时，会生成aggregate_partial, stage(类似于mapreduce中的shuffle), aggregate_final三个计划。
- 对于表达式，通过sql_to_rex生成ExpressionPlan。

```rust
pub enum DFStatement {
    /// ANSI SQL AST node
    Statement(SQLStatement),
    /// Extension: `EXPLAIN <SQL>`
    Explain(DFExplainPlan),
    Create(FuseCreateTable),
    ShowTables(FuseShowTables),
    ShowSettings(FuseShowSettings),
}
```

------

## planners:

主要用于生成逻辑查询计划。

```rust
#[derive(Clone)]
pub enum PlanNode {
    Empty(EmptyPlan),
    Stage(StagePlan),
    Projection(ProjectionPlan),
    AggregatorPartial(AggregatorPartialPlan),
    AggregatorFinal(AggregatorFinalPlan),
    Filter(FilterPlan),
    Limit(LimitPlan),
    Scan(ScanPlan),
    ReadSource(ReadDataSourcePlan),
    Explain(ExplainPlan),
    Select(SelectPlan),
    Create(CreatePlan),
    SetVariable(SettingPlan),
}

#[derive(Clone)]
pub enum ExpressionPlan {
    /// An expression with a alias name.
    Alias(String, Box<ExpressionPlan>),
    /// Column name.
    Column(String),
    /// Constant value.
    Literal(DataValue),
    /// A binary expression such as "age > 40"
    BinaryExpression {
        left: Box<ExpressionPlan>,
        op: String,
        right: Box<ExpressionPlan>,
    },
    /// Functions with a set of arguments.
    Function {
        op: String,
        args: Vec<ExpressionPlan>,
    },
    /// All fields(*) in a schema.
    Wildcard,
}
```

------

## interpreters:

由于查询计划可能需要优化，而其他执行计划可能不需要，因此需要这样一层。

- InterpreterFactory: 将PlanNode转换为IInterpreter。对于SelectInterpreter来说，会首先调用Optimizer::create().optimize得到优化后的PlanNode。然后调用PipelineBuilder执行计划。

```rust
#[async_trait]
pub trait IInterpreter: Sync + Send {
    fn name(&self) -> &str;
    async fn execute(&self) -> FuseQueryResult<SendableDataBlockStream>;
}
```

------

## Optimizers:

主要用于对逻辑查询计划的优化。

- Optimizer: 调用create添加需要应用的优化规则。再调用optimize按顺序优化查询计划。目前实现的filter_push_down只完成了filter算子中的别名表达式替换。

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

- 对于ReadSource的PlanNode, 会将输入进行分区，对于每个分区产生一个物理执行计划。
- 对于stage的PlanNode，会重构之前的Pipeline。如果当前文件读取的分区数小于等于当前节点的cpu核数量，则只在当前单节点上运行。否则每个节点处理ceil(partitions.len()  / cluster_nums) + 1个分区（一个chunk）。之后对每个chunk产生一个叫RemoteTransform的物理查询计划，加入到pipeline中。
- 对于pipeline breaker的算子需要调用merge_processor等待某个节点前所有进程的完成。

```rust
#[async_trait]
pub trait IProcessor: Sync + Send {
    /// Processor name.
    fn name(&self) -> &str;

    /// Connect to the input processor, add an edge on the DAG.
    fn connect_to(&mut self, input: Arc<dyn IProcessor>) -> FuseQueryResult<()>;

    /// Inputs.
    fn inputs(&self) -> Vec<Arc<dyn IProcessor>>;

    /// Reference used for downcast.
    fn as_any(&self) -> &dyn Any;

    /// Execute the processor.
    async fn execute(&self) -> FuseQueryResult<SendableDataBlockStream>;
}

pub struct Pipeline {
    pipes: Vec<Pipe>,
}


pub struct Pipeline {
    processors: Vec<Pipe>,
}
```

------

## functions:

主要用于对向量真正进行运算。

每个functions通过eval方法得到DataColumnarValue。其中聚合函数需要实现accumulate方法将结果存储在self.state中，merge方法会把同一深度的self.state进行聚合。

- FunctionFactory: 通过get方法产生算数，比较，逻辑，聚合以及UDF Function

```rust
pub trait IFunction: fmt::Display + Sync + Send + DynClone {
    fn return_type(&self, input_schema: &DataSchema) -> FuseQueryResult<DataType>;
    fn nullable(&self, input_schema: &DataSchema) -> FuseQueryResult<bool>;
    fn eval(&self, block: &DataBlock) -> FuseQueryResult<DataColumnarValue>;
    fn set_depth(&mut self, depth: usize);
    fn accumulate(&mut self, block: &DataBlock) -> FuseQueryResult<()>;
    fn accumulate_result(&self) -> FuseQueryResult<Vec<DataValue>>;
    fn merge(&mut self, states: &[DataValue]) -> FuseQueryResult<()>;
    fn merge_result(&self) -> FuseQueryResult<DataValue>;
    fn is_aggregator(&self) -> bool {
        false
    }
}
```

------

## transforms:

主要用于实现对每个DataBlock的物理查询计划执行。

- source: 调用table.read进行分区的读取。
- remote: 通过rpc调用其他节点执行。
- 每个进程通过transforms实现executor方法，transforms会将表达式转换为functions，并实现expression_executor方法(对每个datablock执行functions的eval操作并生成新的DataBlock）。然后将functions和expression_executor方法传给datastream进行执行。

------

## datastreams:

将数据流划分成数据块进行执行。

```rust
pub type SendableDataBlockStream = std::pin::Pin<
    Box<dyn futures::stream::Stream<Item = FuseQueryResult<DataBlock>> + Sync + Send>,
>;
```

