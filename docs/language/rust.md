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

**错误处理**

分类：可恢复（例如文件未找到，可再次尝试）/不可恢复（bug）

不可恢复的错误与panic!:

- 打印错误信息；展开并清理调用栈；退出程序
- 也可以在Cargo.toml中设置panic = 'abort'，这样不会展开清理调用栈，直接退出程序。
- 可以通过设置环境变量RUST_BACKTRACE得到回溯信息

可恢复错误与Result枚举：

- 可以通过unwrap获取ok里的值，错误时会触发panic!。也可以使用expect自定义错误信息。
- 可以通过?进行错误的传播。被?应用的错误，会隐式的被from函数处理。
- Box\<dyn Error\>是trait对象，简单理解:任何可能的错误类型。

------

**生命周期**

引用保持有效的作用域。大部分情况生命周期是隐式的，可被推断的。

目的：避免悬垂指针。

生命周期的标注：描述了多个引用生命周期的关系，但不影响生命周期。标注的生命周期的实际生命周期是所有标注的引用生命周期中最小的引用的生命周期。

什么时候需要生命周期标注：引用分为输入引用和输出引用。输入引用会自动的标注成不同的生命周期，如果只有一个输入引用或者存在self的输入引用那么输出引用的生命周期和单个的输入引用/self的输入引用相同。否则无法判断输出引用，需要自行标注输出引用和输入引用生命周期的关系。

------

**智能指针**

- Box\<T\>:在heap内存上分配值。使用场景：当编译时类型大小无法确定，而使用该类型时，上下文却需要知道其大小；有大量数据移交所有权且不要求复制；使用某个变量时，只关心是否实现了某个trait，而不关心具体类型。
- Rc\<T\>:启用多重所有权的引用计数类型。使用场景：需要在heap上分配数据，这些数据被多个部分读取（read-only），但在编译时无法确定哪个部分最后使用完这些数据；只适用于单线程。
- Arc\<T\>:与Rc有相同的API，并且可以安全的用于并发环境。
- Cow\<'a, B'\>:The type `Cow` is a smart pointer providing clone-on-write functionality: it can enclose and provide immutable access to borrowed data, and clone the data lazily when mutation or ownership is required.

------

**宏**
