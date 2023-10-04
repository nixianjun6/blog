# Databend

## common

### datavalues

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

### datablocks:

对多行列向量进行抽象。

- 能与arrow::record_batch::RecordBatch相互转换。
- 通过block_take_by_indices方法按行取索引值

```rust
#[derive(Debug, Clone)]
pub struct DataBlock {
    schema: DataSchemaRef,
    columns: Vec<DataArrayRef>,
}
```

### planners:

主要用于生成逻辑查询计划。(之前的partition和statistics也移到这里来了)

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

### functions:

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

### streams:

将数据流划分成数据块进行执行。

```rust
pub type SendableDataBlockStream = std::pin::Pin<
    Box<dyn futures::stream::Stream<Item = FuseQueryResult<DataBlock>> + Sync + Send>,
>;
```

### infallible:

​	互斥锁和读写锁的封装

------

## fusestore

### api:

​	实现了rpc的通信服务

### configs:

主要用于加载选项设置。

```rust
#[derive(Debug, StructOpt, Clone)]
pub struct Config {
    #[structopt(env = "FUSE_STORE_VERSION", default_value = "Unknown")]
    pub version: String,

    #[structopt(long, env = "FUSE_STORE_LOG_LEVEL", default_value = "info")]
    pub log_level: String,

    #[structopt(
        long,
        env = "FUSE_STORE_METRIC_API_ADDRESS",
        default_value = "127.0.0.1:7171"
    )]
    pub metric_api_address: String,

    #[structopt(
        long,
        env = "FUSE_QUERY_RPC_API_ADDRESS",
        default_value = "127.0.0.1:9191"
    )]
    pub rpc_api_address: String,
}
```

### engine:

​	MemEngine is a prototype storage that is primarily used for testing purposes.

```rust
pub struct MemEngine {
    pub dbs: HashMap<String, Db>,
    pub next_id: i64,
    pub next_ver: i64,
}
```

### executor:

​	存储服务的执行器。目前支持的服务包括:CreateDatabase, CreateTable和GetTable。

```rust
pub struct ActionHandler {
    meta: Arc<Mutex<MemEngine>>,
}
```

------

## fusequery

### api:

​	实现了http与rpc的通信服务。

### configs:

主要用于加载选项设置。

```rust
#[derive(Debug, StructOpt, Clone)]
pub struct Config {
    #[structopt(env = "FUSE_QUERY_VERSION", default_value = "Unknown")]
    pub version: String,

    #[structopt(long, env = "FUSE_QUERY_LOG_LEVEL", default_value = "info")]
    pub log_level: String,

    #[structopt(long, env = "FUSE_QUERY_NUM_CPUS", default_value = "0")]
    pub num_cpus: u64,

    #[structopt(
        long,
        env = "FUSE_QUERY_MYSQL_HANDLER_HOST",
        default_value = "127.0.0.1"
    )]
    pub mysql_handler_host: String,

    #[structopt(long, env = "FUSE_QUERY_MYSQL_HANDLER_PORT", default_value = "3307")]
    pub mysql_handler_port: u64,

    #[structopt(
        long,
        env = "FUSE_QUERY_MYSQL_HANDLER_THREAD_NUM",
        default_value = "256"
    )]
    pub mysql_handler_thread_num: u64,

    #[structopt(
        long,
        env = "FUSE_QUERY_CLICKHOUSE_HANDLER_HOST",
        default_value = "127.0.0.1"
    )]
    pub clickhouse_handler_host: String,

    #[structopt(
        long,
        env = "FUSE_QUERY_CLICKHOUSE_HANDLER_PORT",
        default_value = "9000"
    )]
    pub clickhouse_handler_port: u64,

    #[structopt(
        long,
        env = "FUSE_QUERY_CLICKHOUSE_HANDLER_THREAD_NUM",
        default_value = "256"
    )]
    pub clickhouse_handler_thread_num: u64,

    #[structopt(
        long,
        env = "FUSE_QUERY_RPC_API_ADDRESS",
        default_value = "127.0.0.1:9090"
    )]
    pub rpc_api_address: String,

    #[structopt(
        long,
        env = "FUSE_QUERY_HTTP_API_ADDRESS",
        default_value = "127.0.0.1:8080"
    )]
    pub http_api_address: String,

    #[structopt(
        long,
        env = "FUSE_QUERY_METRIC_API_ADDRESS",
        default_value = "127.0.0.1:7070"
    )]
    pub metric_api_address: String,

    #[structopt(long, env = "STORE_API_ADDRESS", default_value = "127.0.0.1:9191")]
    pub store_api_address: String,

    #[structopt(long, env = "STORE_API_USERNAME", default_value = "root")]
    pub store_api_username: String,

    #[structopt(long, env = "STORE_API_PASSWORD", default_value = "root")]
    pub store_api_password: String,
}
```

### clusters:

集群的抽象，记录了节点名字与节点相关信息的映射。

```rust
#[derive(serde::Serialize, serde::Deserialize, Clone, Debug, PartialEq)]
pub struct Node {
    pub name: String,
    pub cpus: usize,
    pub priority: u8,
    pub address: String,
    pub local: bool,
}

pub struct Cluster {
    cfg: Config,
    nodes: Mutex<HashMap<String, Node>>,
}
```

### sessions:

原来的contexts把options功能解耦。记录context uuid与context的映射关系。

- Settings: 设置某个变量的类型，支持的类型包括：u64, i6, f64和string。支持的操作包括：set(设置value, default value和description), update(更新value), get(获取value)
- FuseQueryContext: 设置了max_threads, max_block_size, default_db的变量类型，value以及description。把原来的partition_queue从iTable中解耦到context中，并新增了uuid和cluster属性以支持分布式查询。

```rust
#[derive(Clone)]
pub struct FuseQueryContext {
    uuid: Arc<RwLock<String>>,
    settings: Settings,
    cluster: Arc<RwLock<ClusterRef>>,
    datasource: Arc<dyn IDataSource>,
    statistics: Arc<RwLock<Statistics>>,
    partition_queue: Arc<RwLock<VecDeque<Partition>>>,
}

pub struct Session {
    sessions: RwLock<HashMap<String, FuseQueryContextRef>>,
}
```

### servers:

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

### datasources:

这一层抽象是为了更好实现不同类型文件的读取。

- DataSource:初始化时会注册system, local, default,  remote四个database。

```rust
#[async_trait::async_trait]
pub trait IDataSource: Sync + Send {
    fn get_database(&self, db_name: &str) -> Result<Arc<dyn IDatabase>>;
    fn get_table(&self, db_name: &str, table_name: &str) -> Result<Arc<dyn ITable>>;
    fn get_all_tables(&self) -> Result<Vec<(String, Arc<dyn ITable>)>>;
    fn get_table_function(&self, name: &str) -> Result<Arc<dyn ITableFunction>>;
    async fn create_database(&self, plan: CreateDatabasePlan) -> Result<()>;
}

pub struct DataSource {
    conf: Config,
    databases: RwLock<HashMap<String, Arc<dyn IDatabase>>>,
    table_functions: RwLock<HashMap<String, Arc<dyn ITableFunction>>>,
    store_client: RwLock<Option<StoreClient>>,
}

#[async_trait::async_trait]
pub trait IDatabase: Sync + Send {
    // Database name.
    fn name(&self) -> &str;

    // Database engine.
    fn engine(&self) -> &str;

    // Get one table by name.
    fn get_table(&self, table_name: &str) -> Result<Arc<dyn ITable>>;

    // Get all tables.
    fn get_tables(&self) -> Result<Vec<Arc<dyn ITable>>>;

    // Get database table functions.
    fn get_table_functions(&self) -> Result<Vec<Arc<dyn ITableFunction>>>;

    // DDL
    async fn create_table(&self, plan: CreateTablePlan) -> Result<()>;
}

pub trait ITableFunction: Sync + Send + ITable {
    fn function_name(&self) -> &str;
    fn db(&self) -> &str;

    fn as_table<'a>(self: Arc<Self>) -> Arc<dyn ITable + 'a>
    where
        Self: 'a;
}


#[async_trait::async_trait]
pub trait ITable: Sync + Send {
    fn name(&self) -> &str;
    fn engine(&self) -> &str;
    fn as_any(&self) -> &dyn Any;
    fn schema(&self) -> Result<DataSchemaRef>;

    // Get the read source plan.
    fn read_plan(
        &self,
        ctx: FuseQueryContextRef,
        push_down_plan: PlanNode,
    ) -> Result<ReadDataSourcePlan>;

    // Read block datas from the underfling.
    async fn read(&self, ctx: FuseQueryContextRef) -> Result<SendableDataBlockStream>;
}
```

### sql:

- PlanParse: 从原来的planners中解耦出来。build_from_sql首先将sql语句转换为DFStatements(通过DFParser实现)。对于Statement::Query，按having -> from-> filter-> projection -> aggr -> limit生成计划(主要通过PlanBuilder生成PlanNode)
- 在处理from时，会进一步调用table.read_plan生成ReadSource的PlanNode)。
- 在生成projection计划时，会先调用planRewrite来重写别名。
- 在生成aggr计划时，会生成aggregate_partial, stage(类似于mapreduce中的shuffle), aggregate_final三个计划。
- 对于表达式，通过sql_to_rex生成ExpressionPlan。

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum DfStatement {
    /// ANSI SQL AST node
    Statement(SQLStatement),
    Explain(DfExplain),
    ShowTables(DfShowTables),
    ShowSettings(DfShowSettings),
    CreateDatabase(DfCreateDatabase),
    CreateTable(DfCreateTable),
}
```

### interpreters:

由于查询计划可能需要优化，而其他执行计划可能不需要，因此需要这样一层。

- InterpreterFactory: 将PlanNode转换为IInterpreter。对于SelectInterpreter来说，会首先调用Optimizer::create().optimize得到优化后的PlanNode。然后调用PipelineBuilder执行计划。

```rust
#[async_trait]
pub trait IInterpreter: Sync + Send {
    fn name(&self) -> &str;
    async fn execute(&self) -> FuseQueryResult<SendableDataBlockStream>;
}
```

### Optimizers:

主要用于对逻辑查询计划的优化。

- Optimizer: 调用create添加需要应用的优化规则。再调用optimize按顺序优化查询计划。
- filter_push_down: 完成了filter算子中的别名表达式替换。
- limit_push_down: 

```rust
pub trait IOptimizer {
    fn name(&self) -> &str;
    fn optimize(&mut self, plan: &PlanNode) -> FuseQueryResult<PlanNode>;
}

pub struct Optimizer {
    optimizers: Vec<Box<dyn IOptimizer>>,
}
```

### planners:

​	对Stage查询计划进行节点间的调度。

### pipelines:

**processors**:

​	主要用于构建进程之间的执行顺序

- 对于ReadSource的PlanNode, 会将输入进行分区，对于每个分区产生一个物理执行计划。
- 对于stage的PlanNode，会重构之前的Pipeline。按照planner中的调度方案产生RemoteTransform的物理查询计划，加入到pipeline中。
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

#[derive(Clone)]
pub struct Pipe {
    processors: Vec<Arc<dyn IProcessor>>,
}

pub struct Pipeline {
    processors: Vec<Pipe>,
}
```

**transforms:**

​	主要用于实现对每个DataBlock的物理查询计划执行。

- source: 调用table.read进行分区的读取。
- remote: 通过rpc调用其他节点执行。
- 每个进程通过transforms实现executor方法，transforms会将表达式转换为functions，并实现expression_executor方法(对每个datablock执行functions的eval操作并生成新的DataBlock）。然后将functions和expression_executor方法传给datastream进行执行。

