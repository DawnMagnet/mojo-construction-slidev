---
theme: seriph
background: https://source.unsplash.com/collection/94734566/1920x1080
class: text-center
highlighter: shiki
lineNumbers: false
drawings:
  persist: false
transition: slide-left
title: Mojo中的构造函数-Mojo开发者交流会第三期
mdc: true
---

# Mojo中的构造函数

Mojo开发者交流会第三期

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Press Space for next page <carbon:arrow-right class="inline"/>
  </span>
</div>
---

# Mojo中的Struct

Mojo中的Struct与Python中的class非常类似，都可以用于存储变量，拥有成员函数

使用如下的语法定义一个Struct

```mojo
struct Goods:
    var name: String
    var value: Int
```

为上面的`Goods`结构体实现其最重要的成员函数——构造函数

```mojo
struct Goods:
    var name: String
    var value: Int
    fn __init__(inout self, name: String, value: Int):
        self.name = name
        self.value = value
```

在实现了该构造函数之后，我们就可以使用如下方式来声明并初始化一个`Goods`对象

```mojo
var phone = Goods("mobile phone", 1000)
```

---

# Mojo中的Struct

```mojo {all|2-3}
struct Goods:
    var name: String
    var value: Int
    fn __init__(inout self, name: String, value: Int):
        self.name = name
        self.value = value
```

> 注意！：在Mojo中，`let` 和 `var` 分别对应定义不可变变量和可变变量，但是在当前的版本中，只支持在Struct中使用 `var` 作为定义变量的关键词，如果尝试使用 `let` 则会得到如下错误

```text
main.mojo: error: 'let' fields in structs are not supported yet

    let c: String
    ^

mojo: error: failed to parse the provided Mojo
```

---

# Mojo中的构造函数重载

```mojo
struct Complex:
    var re: Float32
    var im: Float32

    fn __init__(inout self, x: Float32):
        """仅通过实部构建复数"""
        self.re = x
        self.im = 0.0

    fn __init__(inout self, r: Float32, i: Float32):
        """通过实部和虚部构建复数"""
        self.re = r
        self.im = i
```

Mojo中的构造函数是可以重载的，可以通过编写多个构造函数以实现不同方式的构造。

```mojo
var a = Complex(3)
var b = Complex(3, 1)
```

---

# Mojo中的构造函数

我们后续的演示将会基于HeapArray类型，这是一个用于管理连续内存的结构体，类似于动态分配的数组。

- mojo定义

```mojo
struct HeapArray:
    var data: Pointer[Int]
    var size: Int
```

- C++定义

```cpp
class HeapArray{
public:
    int * data;
    int size;
};
```

- Rust定义

```rust
struct HeapArray{
    pub data: *mut i32,
    pub size: i32
}
```

---

# Mojo中的构造函数

1.copy构造函数(`__copyinit__`)

一般我们会使用copy构造函数作为一个对象的深度拷贝函数

```mojo
struct HeapArray:
    var data: Pointer[Int]
    var size: Int
    fn __init__(inout self, size: Int):
        self.size = size
        self.data = Pointer[Int].alloc(self.size)
        for i in range(self.size):
            self.data.store(i, 0)
    fn __copyinit__(inout self, borrowed rhs: Self):
        self.size = rhs.size
        self.data = Pointer[Int].alloc(self.size)
        for i in range(self.size):
            self.data.store(i, rhs.data[i])
fn main():
    var a = HeapArray(10)
    let b = a
```

> 只有定义了 `__copyinit__` 或者 `__takeinit__` 才可以使用`var b = a` 这种语法

> 其中 `borrowed` 表示对于变量的借用，并且不能修改原对象！

---

# Mojo中的构造函数

2.take构造函数(`__takeinit__`)

在C++中存在一类默认实现的构造函数，定义为`HeapArray(HeapArray &g) = default;`，功能是浅拷贝

```cpp {all|5}
class HeapArray {
    int *data;
    int size;
public:
    HeapArray(HeapArray &g) = default; // (有没有这一行都没区别)
    HeapArray() {
        data = new int[10];
        size = 10;
        cout << "HeapArray" << endl;
    }
    ~HeapArray() {
        cout << "~HeapArray:" << (unsigned long) data << endl;
        delete data;
    }
};
int main() {
    HeapArray a;
    HeapArray b = a;
    return 0;
}
```

---

# Mojo中的构造函数

2.take构造函数(`__takeinit__`)

这些函数在值语义下不会出问题，而且很高效，但是一旦在类中使用指针，就会出现严重的内存安全问题。

```cpp {monaco-diff}
class HeapArray {
    int *data;
    int size;
public:
    HeapArray(HeapArray &g) = default;
    HeapArray() {
        data = new int[10];
        size = 10;
        cout << "HeapArray" << endl;
    }
    ~HeapArray() {
        cout << "~HeapArray:" << (unsigned long) data << endl;
        delete data;
    }
};
~~~
class HeapArray {
    int *data;
    int size;
public:
    HeapArray(HeapArray &g) {
        data = g.data;
        size = g.size;
        g.data = nullptr;
        g.size = 0;
    };
    HeapArray() {
        data = new int[10];
        size = 10;
        cout << "HeapArray" << endl;
    }
    ~HeapArray() {
        cout << "~HeapArray:" << (unsigned long) data << endl;
        if(data != nullptr) delete data;
    }
};
```

---

# Mojo中的构造函数

2.take构造函数(`__takeinit__`)

Mojo中也提供了类似的构造函数，`__takeinit__`。C++风格的构造函数有一个特征，即其在资源传输完成之后，仍然会运行一遍原对象的析构函数。对于此行为，可以在Mojo中如此定义。

```mojo
struct HeapArray:
    var data: Pointer[Int]
    var size: Int
    fn __init__(inout self, size: Int):
        self.size = size
        self.data = Pointer[Int].alloc(self.size)
        for i in range(self.size):
            self.data.store(i, 0)
        print("HeapArray")
    fn __takeinit__(inout self, inout rhs: Self):
        self.size = rhs.size
        self.data = rhs.data
        rhs.size = 0
        rhs.data = Pointer[Int].get_null()
    fn __del__(owned self):
        self.data.free()
        print("~HeapArray")
```

---

# Mojo中的构造函数

2.take构造函数(`__takeinit__`)

如果在`main`函数中编写如下代码，则会得到如下输出

```mojo
fn main():
    var a = HeapArray(10)
    let b = a
```

```text
HeapArray
~HeapArray
~HeapArray
```

与C++的工作方式相同

> 其中 `owned` 的含义是传递 `rhs` 的可变引用，这种方式与C++中的处理方式是类似的，都是通过修改原对象的某些值来确保将资源传递给新对象，而且不会二次释放指针。是否可以修改原对象也是 `__copyinit__` 和 `__takeinit__` 的主要区别。

---

# Mojo中的构造函数

3.move构造函数(`__moveinit__`)

此种构造函数主要与Rust中的对象构建较为相似。

```rust
struct HeapArray {
    pub data: *mut i32,
    pub size: i32,
}
fn main() {
    let a = HeapArray {
        data: [1, 2, 3, 4].as_ptr() as *mut i32,
        size: 3,
    };
    let b = a;
    dbg!(a.data);
}
```

编写上述Rust代码会报错，因为Rust中默认使用的是move构造函数，这种构造特别关注变量的所有权。

在上文中，`let b = a`这一行，实际上把`a`的所有权转移给了`b`，所以在此之后，如果需要再次使用或者访问`a`，必须对`a`进行重新初始化。

---

# Mojo中的构造函数

3.move构造函数(`__moveinit__`)

在Mojo中，也有与此类似的构造函数成为move构造函数`__moveinit__`，可以按照下面的方式进行定义：

```mojo
struct HeapArray:
    var data: Pointer[Int]
    var size: Int
    fn __init__(inout self, size: Int):
        self.size = size
        self.data = Pointer[Int].alloc(self.size)
        for i in range(self.size):
            self.data.store(i, 0)
        print("HeapArray")
    fn __moveinit__(inout self, owned rhs: Self):
        self.size = rhs.size
        self.data = rhs.data
fn main():
    let a = HeapArray(10)
    let b = a ^
```

> `owned` 和 `^` 表示获得对象的所有权

---

# Mojo中的构造函数

3.move构造函数(`__moveinit__`)

使用move构造函数有如下优点：

- 保证只有一个变量拥有某个资源的所有权
- 保证析构函数之进行一次
- 不会二次释放资源
- 在一定程度上保证内存安全

---

# Mojo中的构造函数

总结

|                      | `__copyinit__` | `__takeinit__`                 | `__moveinit__`                     |
| -------------------- | -------------- | ------------------------------ | ---------------------------------- |
| 是否可以改变原变量   | ✗              | ✓                              | \*                                 |
| 是否获得原对象所有权 | ✗              | ✗                              | ✓                                  |
| 是否可以拷贝         | ✓              | ✓                              | ✗                                  |
| 是否阻止原对象析构   | ✗              | ✗                              | ✓                                  |
| 适合哪种类型的任务   | 需要拷贝多份   | 更加灵活，但是容易出现内存问题 | 某个资源的所有权只能由一个变量拥有 |


---
layout: center
class: text-center
---
# 欢迎大家的聆听！
