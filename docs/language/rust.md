# Rust

## bytes

提供了高效的字节缓冲区结构（Bytes）和用于缓冲区实现（Buf、BufMut）的特征。

**Bytes**是用于存储和操作连续内存片的高效容器。Bytes value通过允许多个Bytes对象指向相同的底层内存来促进零复制编程。

**Buf**和**BufMut**提供对缓冲区的读写访问，维护跟踪底层字节存储中当前位置的游标，当读取或写入字节时，光标前进。

Buf:从缓冲区读取字节，最简单的Buf是&[u8]。常用接口包括：**get_u16**(以大端字节顺序从self获取无符号16位整数，当前位置前进2)

BufMut:提供对字节的顺序写入访问的值的特征，最简单的BufMut是Vec<u8>。常用的接口包括：**put**（将字节从src传输到self并将光标前进所写入的字节数），put_u16(以大端字节顺序将无符号16位整数写入self，当前位置前进2)

光标的移动也可以通过advance函数操作

```rust
// Vec -> Bytes Write
let mut buf: Vec<u8> = vec.clone();
buf.put(xxx);
buf.put_u16(xxx);
buf.into()

//Vec ->Bytes Write partially
buf.copy_to_bytes(len)

//Bytes -> Vec Write
buf.put_slice(&b)

// Vec -> Vec Read
let mut entry = &data[offset..]
let key_len = entry.get_u16() as typename;
let key = entry[..key_len].to_vec();
entry.advance(key_len);
```

------

## Convert

用于类型转换的特征

**into**: Converts this type into the (usually inferred) input type.

**from**: [`From`](https://rustwiki.org/zh-CN/std/convert/trait.From.html) trait 允许一种类型定义 “怎么根据另一种类型生成自己”.

------

## Slice

**Chunks**: 以块为单位对切片进行迭代的迭代器，从切片的开头开始

**partition_point**:根据给定谓词返回分区点的索引。可以配合saturating_sub（1）找到相等的点。

------

## Iter

**map**: 迭代器方法，接受一个闭包并创建一个迭代器，该迭代器在每个元素上调用该闭包。

**collect**: 迭代器方法，将迭代器转换为集合。

```rust
// Bytes -> Vec Read data: &[u8]
data_raw = &data[start..end];
data_Vec_u16 = data_raw.chunks(SIZEOF_U16).map(|mut x| x.get_u16()).collect();
data_Vec_u8 = data_raw.to_vec();
```

------

## 模块与crate

(以下内容来自于rustwiki.org)

默认情况下，模块中的项拥有私有的可见性（private visibility），不过可以加上 `pub` 修饰语来重载这一行为。模块中只有公有的（public）项可以从模块外的作用域访问。

结构体的字段也是一个可见性的层次。字段默认拥有私有的可见性，也可以加上 `pub` 修饰语来重载该行为。只有从结构体被定义的模块之外访问其字段时，这个可见性才会起作用，其意义是隐藏信息（即封装，encapsulation）。

`use` 声明可以将一个完整的路径绑定到一个新的名字，从而更容易访问

可以在路径中使用 `super` （父级）和 `self`（自身）关键字，从而在访问项时消除歧义，以及防止不必要的路径硬编码。

crate（中文有 “包，包装箱” 之意）是 Rust 的编译单元。当调用 `rustc some_file.rs` 时，`some_file.rs` 被当作 **crate 文件**。如果 `some_file.rs` 里面含有 `mod` 声明，那么模块文件的内容将在编译之前被插入 crate 文件的相应声明处。换句话说，模块**不会**单独被编译，只有 crate 才会被编译。

crate 可以编译成二进制可执行文件（binary）或库文件（library）。默认情况下，`rustc` 将从 crate 产生二进制可执行文件。这种行为可以通过 `rustc` 的选项 `--crate-type` 重载。

------

## [Result](https://rustwiki.org/zh-CN/rust-by-example/error/result.html)

(以下内容来自于rustwiki.org)

[`Result`](https://rustwiki.org/zh-CN/std/result/enum.Result.html) 是 [`Option`](https://rustwiki.org/zh-CN/std/option/enum.Option.html) 类型的更丰富的版本，描述的是可能的**错误**而不是可能的**不存在**。

也就是说，`Result<T，E>` 可以有两个结果的其中一个：

- `Ok<T>`：找到 `T` 元素
- `Err<E>`：找到 `E` 元素，`E` 即表示错误的类型。

------

## std::mem

处理内存的基本函数

**replace**<T>(dest: &mut T, src: T) -> T: move src -> dest, 返回先前的dest

**take**<T>(dest: &mut T) -> T: 将dest替换成T的默认值，返回之前的dest

------

## [anyhow](https://docs.rs/anyhow/latest/anyhow/)

提供了anyhow::Error，一种基于特征对象的错误类型，用于在Rust应用程序中轻松地进行惯用错误处理。

Details: 使用Result<T, anyhow::Error>或等效的anyhow::Result<T>作为易出错函数的返回类型。Within the function, use `?` to easily propagate any error that implements the `std::error::Error` trait.

```rust
use anyhow::Result;

fn get_cluster_info() -> Result<ClusterMap> {
    let config = std::fs::read_to_string("cluster.json")?;
    let map: ClusterMap = serde_json::from_str(&config)?;
    Ok(map)
}
```

------

**Arc**

线程安全的引用计数指针。代表原子引用计数。

------

## Option

可选值。Type [`Option`](https://doc.rust-lang.org/nightly/core/option/enum.Option.html) represents an optional value: every [`Option`](https://doc.rust-lang.org/nightly/core/option/enum.Option.html) is either [`Some`](https://doc.rust-lang.org/nightly/core/option/enum.Option.html#variant.Some) and contains a value, or [`None`](https://doc.rust-lang.org/nightly/core/option/enum.Option.html#variant.None), and does not. 

map:通过函数应用于包含的值，Maps an `Option<T>` to `Option<U>`

------

## crossbeam_skiplist

skipMap.get:返回一个Entry，可以用于访问键的关联值

------

## std::ops::Bound

键范围的端点。

------

## Box

A pointer type that uniquely owns a heap allocation of type `T`.