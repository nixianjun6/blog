# leveldb

​	在阅读源码之前，我们应该去关注以下问题:

- 为什么会有NoSQL数据库的出现, 这样的数据库解决了关系型数据库哪方面的不足

------

[source code](https://github.com/google/leveldb)

[build && simple use in windows 11](https://zhuanlan.zhihu.com/p/558559654)

------

## Overview:

LevelDB is a fast key-value storage library written at Google that provides an ordered mapping from string keys to string values.

------

## Features:

- 键值为任意字节数组
- 数据会按键的顺序存储
- 用户可以自定义比较函数
- 基本的操作为增，删，查
- 在一个原子批次中对数据库进行一系列的编辑，并且保证这些编辑按顺序应用。除此之外还能将大量的修改放入一个批次中进行加速(WriteBatch)
- 用户可以创建一个瞬时快照去获得一致的数据视图
- 提供向前和向后的迭代器
- 使用Snappy压缩库自动压缩数据，但也支持Zstd压缩
- 外部活动通过虚拟界面进行中继，因此用户可以自定义操作系统交互

------

## Test Features:

my test code is [here](https://github.com/nixianjun6/leveldb_test).

------

## Data Structure:

​	**Slice**: 由于string类在返回时，会进行一个拷贝操作，而Slice则只需要返回长度和指针，这样保证了对于任意字节数组的键值都有较好的性能。除此之外，Slice不以'\0'作为字符的终止符，可以存储值为'\0'的数据。

​	**Comparator:** 纯虚类，用于用户自定义比较函数。主要需要实现四个接口比较函数，比较器的名字，以及用于两个压缩字符串存储空间的方法: FindShortestSeparator,  FindShortSuccessor。思考:

- 为什么比较器需要实现两个用于压缩字符串的方法?
- 这两个方法的使用场景有什么区别?

------

## Code Reading:

​	首先，我们一起阅读数据库的读写部分。

<center>
  ![leveldb read and write](../img/leveldb_read_write.png)
  <br>
    <div>How Leveldb read && write</div>
</center>

​	读操作: 首先会进行上锁，并读出mem_, imm_, versions->current()。然后进行解锁，并依次在mem_, imm_, current中根据look up key进行查找。找到之后进行上锁，如果是从current中找到的，并且current更新了状态，会触发一个可能的压缩操作。之后，维护mem_, imm_, current的引用计数，并返回值。读这段代码我有三个问题:

- mutex_控制的是线程间什么样的并发过程?
- mem_, imm_, current分别是什么?
- 这里的状态更新是什么，为什么会触发一个可能的压缩操作?

