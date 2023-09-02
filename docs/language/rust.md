# Rust

source from [https://github.com/wubx/rust-in-databend](https://github.com/wubx/rust-in-databend)

背景：很难编写内存/线程安全的代码。Rust的期望是性能可以和 C/C++ 媲美，还能保证安全性，同时可以提供⾼ 效的开发效率，代码还得容易维护。

设计思想：安全；并发；高效；零成本抽象。

------

**所有权**

所有程序在运行时都必须管理它们使用计算机内存的方式。

```rust
fn main() {
    let name = "Pascal".to_string();
    let a = name;
    let b = name;
}
```

<center>
  ![rustowner1](../img/rustowner1.png)
  <br>
</center>

如图所示，当把name赋值给a时会将所有权也交给a，导致let b = name报错。

```rust
fn main() {
    let name = "Pascal".to_string();
    let a = name;
    let b = a.clone();
}
```

<center>
  ![rustowner2](../img/rustowner2.png)
  <br>
</center>

​	如图所示，使用clone会进行深拷贝。

```rust
fn main() {
    let name = "Pascal".to_string();
    let a = &name;
    let b = a; // or let b = &name;
}
```

​	可以通过&引用，而使name不丢失所有权。

------

**宏**
