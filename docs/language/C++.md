# C++

## 关键字：

**explicit**: 对于类的构造函数都可以使用explicit来修饰，这样可以避免隐式类型转换的发生。

**=delete**: 显示删除编译器默认生成的函数

**const**: 

- 成员函数后加const: 表示成员函数不修改成员变量
- const int *a == int const *a，a所指的值不能改但是a可以指向别的值
- int const* const a: a所指的值和a都不能改

**new与malloc**（[参考](https://zhuanlan.zhihu.com/p/550217439)）:

- new/delete是C++关键字，需要编译器支持。malloc/free是库函数，需要头文件支持
- new操作符内存分配成功时，返回的是对象类型的指针，类型严格与对象匹配，无须进行类型转换，故new是符合类型安全性的操作符。而malloc内存分配成功则是返回void * ，需要通过强制类型转换将void*指针转换成我们需要的类型。
- new内存分配失败时，会抛出bac_alloc异常。malloc分配内存失败时返回NULL。

​	**inline**（[参考](https://zhuanlan.zhihu.com/p/550217439)）：内联函数。编译器不一定会满足要求，取决于函数大小或者函数是否调用自身。

​	**template**:

模板的声明和定义都放在头文件时，并且在多个源文件中被包含和使用，那么编译器会为每个源文件生成相同的模板实例化代码，会导致重复的代码段。

如果只有定义放在源文件中，可能导致链接错误，因为其他源文件无法找到源文件中定义的模板。

模板的声明和定义分开写方法：

```c++
// template.h
template <typename T>
void swap(T& a, T& b);

// template.cpp
template <typename T>
void swap(T& a, T& b) {
    T temp = a;
    a = b;
    b = temp;
}

template void swap<int>(int&, int&);
template void swap<double>(double&, double&);
```

​	**类型转换：**static_cast执行非多态类型转换，dynamic_cast基类转换为派生类，reinterpret_cast不用指针类型转换。

​	**Virtual:**父类型的指针指向其子类的实例，然后通过父类的指针调用实际子类的成员函数。

------

## 多线程：

​	**std::atomic**:

- 互斥量保护一段共享代码段，当共享数据为一个变量时，原子操作std::atomic效率更高。
- 常用接口：load(T arg, std::memory_order order), store, fetch_add。
- memory_order_relaxed: 仅保证原子性，不保证操作之间的顺序。memory_order_release: 保证释放操作之前，所有写入都已完成，并且对其他线程可见。

​	**mutex:**

- 基本用法：mutex.lock(), mutex.unlock()。也可以通过std::lock_guard进行锁保护。
- std::shared_mutex：读写锁，独占性锁：lock, unlock。共享锁：lock_share, unlock_share。
- std::condition_variable。
- lock_guard解锁功能通过析构函数实现，并不能如unique_lock那样，提供unlock接口，导致其并不能被std::condition_variable使用。std::scoped_lock可以对多个不同类型的mutex进行Scoped Locking。

------

## STL：

​	**vector:** 

- 避免频繁扩容，可以通过reserve预取空间。
- emplace_back性能比push_back更高，因为减少了拷贝开销。
- erase删除多个元素时可以考虑先排序，再一次性删除，避免多次扩容操作。
- 可以通过std::move避免扩容时引发自定义类型挨个复制构造。

​	**set, map:** 底层是红黑树。

​	**umap, uset**: 底层是哈希表，适用于小数据，查找插入删除最优情况都是O(1)，最坏O(n)。

------

**新特性：**

​	**智能指针([参考](https://blog.csdn.net/m0_50816320/article/details/129326768))：**

- std::shared_ptr是一种共享式智能指针，它可以让多个shared_ptr实例同时拥有同一个内存资源。shared_ptr内部维护了一个计数器，记录当前有多少个shared_ptr实例共享同一块内存。只有当计数器变为0时，才会自动释放内存。
- std::unique_ptr是一种独占式智能指针，它可以保证指向的内存只被一个unique_ptr实例所拥有。当unique_ptr被销毁时，它所拥有的内存也会被自动释放。unique_ptr还支持移动语义，因此可以通过std::move来转移拥有权。

​	**移动语义：**它允许将一个对象的资源所有权从一个对象转移到另一个对象，而不需要进行昂贵的复制操作。