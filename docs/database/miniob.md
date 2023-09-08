# Miniob

## BufferPool

<center>
  ![miniobbufferpool](../img/miniobbufferpool.png)
  <br>
    <div>miniob bufferpool</div>
</center>

BufferPoolManager: 管理全局所有的BufferPool。全局仅有一个实例default_bpm。主要向外提供的接口包括：create_file, open_file, close_file和flush_page。当调用create_file接口时，会返回一个DiskBufferPool，并且会把文件的元信息（存储在BPFileHeader 中）加载到内存。

DiskBufferPool: 每个文件对应一个DiskBufferPool，主要向外提供接口包括get_this_page(取页)，allocate_page(新页)，dispose_page(删页)，purge_page(驱逐页面)，unpin_page(解除绑定)。取页时会先在内存中查找，如果找不多再从磁盘中加载。另外可以通过BufferPoolIterator进行遍历。

BPFrameManager ：向外透明的模块，主要在BufferPool内部调用。主要的目的包括：

- 进行内存管理。一次性申请多个页面，减少频繁申请释放内存的开销。
- 替换策略。当缓存池满时，对驱逐页面进行判定。

------

## **Table and Index**

<center>
  ![miniobtableandindex](../img/miniobtableandindex.png)
  <br>
</center>

Table主要提供make, insert, delete, get, visit_record接口进行对行数据的增删查。也可以通过get_record_scanner得到迭代器进行遍历操作。索引提供了insert, delete, get_entry接口进行增删查。也可以通过create_scanner接口得到索引的迭代器进行遍历操作。

------

## SQL流程

session stage: 收到客户端的string, 然后进入plan cache stage

plan cache stage: 啥也没做，直接进入parse stage

parse stage: 词法分析 + 语法分析得到Query，然后进入resolve stage

resolve stage: 得到query fields或检查query fields信息。然后进入query cache stage

query cache stage:啥也没做，直接进入optimize stage

optimize stage:啥也没错，直接进入exeute stage

execute stage:构建执行计划，使用火山模型。每个算子都需要实现open, next两个方法。



