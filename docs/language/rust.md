# Rust

## bytes

提供了高效的字节缓冲区结构（Bytes）和用于缓冲区实现（Buf、BufMut）的特征。

Bytes是用于存储和操作连续内存片的高效容器。Bytes value通过允许多个Bytes对象指向相同的底层内存来促进零复制编程。

Buf和BufMut提供对缓冲区的读写访问，维护跟踪底层字节存储中当前位置的游标，当读取或写入字节时，光标前进。

Buf:从缓冲区读取字节，最简单的Buf是&[u8]。常用接口包括：get_u16(以大端字节顺序从self获取无符号16位整数，当前位置前进2)

BufMut:提供对字节的顺序写入访问的值的特征，最简单的BufMut是Vec<u8>。常用的接口包括：put（将字节从src传输到self并将光标前进所写入的字节数），put_u16(以大端字节顺序将无符号16位整数写入self，当前位置前进2)

光标的移动也可以通过advance函数操作

------

## Convert

用于类型转换的特征

into: Converts this type into the (usually inferred) input type.

------

## Chunks

以块为单位对切片进行迭代的迭代器，从切片的开头开始

------

## map

迭代器方法，接受一个闭包并创建一个迭代器，该迭代器在每个元素上调用该闭包。

------

## collect

迭代器方法，将迭代器转换为集合。