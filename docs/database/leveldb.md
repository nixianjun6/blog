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

## 功能测试:

```c++
#include <iostream>

#include "leveldb/db.h"

auto PutData(leveldb::Status mystatus, leveldb::DB* mydb, std::string key, std::string value) -> bool {
  if (mystatus.ok()) {
    mydb->Put(leveldb::WriteOptions(), key, value);
    return true;
  }
  std::cerr << mystatus.ToString() << std::endl;
  return false;
}

auto DeleteData(leveldb::Status mystatus, leveldb::DB* mydb, std::string key) -> bool {
  if (mystatus.ok()) {
    mydb->Delete(leveldb::WriteOptions(), key);
    return true;
  }
  std::cerr << mystatus.ToString() << std::endl;
  return false;
}

void GetData(leveldb::DB* mydb, std::string key, std::string& value) {
  mydb->Get(leveldb::ReadOptions(), key, &value);
}

int main() {
  // 声明
  leveldb::DB* mydb;
  leveldb::Options myoptions;
  leveldb::Status mystatus;

  // 创建
  myoptions.create_if_missing = true;
  mystatus = leveldb::DB::Open(myoptions, "testdb", &mydb);
  std::string result;
  
  // 对增删查进行测试
  PutData(mystatus, mydb, "nixianjun", "a handsome man");
  GetData(mydb, "nixianjun", result);
  std::cout << "nixianjun : " << result << std::endl;

  PutData(mystatus, mydb, "zhaokangkang", "a beautiful girl");
  GetData(mydb, "zhaokangkang", result);
  std::cout << "zhaokangkang : " << result << std::endl;

  std::cout << "delete nixianjun" << std::endl;
  DeleteData(mystatus, mydb, "nixianjun");
  GetData(mydb, "nixianjun", result);
  std::cout << "nixianjun : " << result << std::endl;
  GetData(mydb, "zhaokangkang", result);
  std::cout << "zhaokangkang : " << result << std::endl;

  std::cout << "delete zhaokangkang" << std::endl;
  DeleteData(mystatus, mydb, "zhaokangkang");
  GetData(mydb, "nixianjun", result);
  std::cout << "nixianjun : " << result << std::endl;
  GetData(mydb, "zhaokangkang", result);
  std::cout << "zhaokangkang : " << result << std::endl;

  std::cout << "put xxx" << std::endl;
  PutData(mystatus, mydb, "xxx", "Others");
  GetData(mydb, "nixianjun", result);
  std::cout << "nixianjun : " << result << std::endl;
  GetData(mydb, "zhaokangkang", result);
  std::cout << "zhaokangkang : " << result << std::endl;

  //nixianjun:
  //a handsome man zhaokangkang : a beautiful girl
  //delete nixianjun
  //nixianjun : a beautiful girl
  //zhaokangkang : a beautiful girl
  //delete zhaokangkang 
  //nixianjun : a beautiful girl
  //zhaokangkang : a beautiful girl
  //put xxx
  //nixianjun : a beautiful girl zhaokangkang : a beautiful girl

  std::cout << "------------------------------" << std::endl;
  
  // 测试不用WriteBatch，会出BUG的情况，然而并没有BUG
  PutData(mystatus, mydb, "nixianjun", "a handsome man");
  PutData(mystatus, mydb, "zhaokangkang", "a beautiful girl");
  if (mystatus.ok()) {
    mydb->Delete(leveldb::WriteOptions(), "nixianjun");
    mydb->Put(leveldb::WriteOptions(), "nixianjun", "a funny man");
  }
  GetData(mydb, "nixianjun", result);
  std::cout << "nixianjun : " << result << std::endl;

  delete mydb;
}
```

