# leveldb

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

