---
title: Lvalue and Rvalue (左值引用 & 右值引用)
date: 2023-10-27 15:56:26
categories: [C++]
tags:
---


## 1. 左值引用 & 常引用

说起左值引用, 我们需要先讨论一下左值是什么. 左值可以简单概括为可以取地址的, 有名字的非匿名对象, 在一个表达式结束之后还是可以存活; 而右值刚好相反, 右值不能取地址, 也没有名字, 比如立即数或者是函数的返回值 (非引用值). 下面是左值引用的一些例子

```c++
int origin = 10;
int &ref = origin;

ref = 20; // 通过左值引用改变引用对象的值
```

但是如果你想要用左值引用来引用一个立即数或者函数返回值, 程序编译是不会通过的. 因为这些临时变量只存在于寄存器中, 他们在内存中没有地址 (i.e. 不存在于内存). 我们可以通过添加 const 修饰符来使得这些临时变量保存在内存中 (i.e. 常引用)

```c++
int& ref = 10; // 编译不通过

// 常引用
const int& ref = 10;

// 上面常引用的语句等同于
const int temp = 10;  // 此时 temp 在内存中有地址
int& ref = temp;  
```

但是这又带来一个问题, 因为我们使用了常引用, 所以我们只能通过这个引用来读取数据, 而无法对这个数据进行修改, 而右值引用可以很好地解决这个问题

## 2. 右值引用

右值引用和常引用在汇编层面上做的事情其实差不多, 都是将临时变量保存在内存中然后就可以引用他了. 他们唯一的不同点就是: 右值引用可以对引用对象进行读写操作, 而常引用只能读取引用对象的数据. 使用右值引用绑定一个右值, 那么这个右值的生命周期就跟这个引用的生命周期一致了

```c++
// 右值引用格式
type && variable_name = r_value (i.e. temporarily objects)

int&& var = 10; // 编译可通过

// 上面右值引用的语句等同于
int temp = 10;
int&& var = std::move(temp);
```

### 2.1 右值引用实现移动语义 (Move Semantics)

除此之外, 在上文中我们介绍了右值的定义, 右值在使用之后会被销毁. 从资源管理的角度上看, 既然这个资源都要被销毁了, 那么我们为什么不能暂时保留下来利用一下呢? **在之前的拷贝构造函数以及赋值运算符重载中我们已经使用了左值引用来避免了一次额外的拷贝, 但是在函数内部还是无可避免地要进行深拷贝 (e.g. 对象成员的拷贝), 移动语义则实现了将传入的对象中的内部数据移动过来, 被拷贝的对象随后丢弃, 从而避免了函数内部的深拷贝.** 我们可以通过 Move Constructor 以及 Move Assignment 来实现移动语义. 

> 移动语义允许将一个对象的所有权从那个一个对象转移到另一个对象, 而不需要进行数据拷贝

我们可以首先观察一下如果没有充分利用已经生成的资源而导致的资源重复拷贝问题的例子

```c++
class Object {
public:
    // 默认的构造函数
    Object() { std::cout << "Default Constructor" << std::endl; };

    // 带有一个参数的构造函数
    Object(int val): val(val) { std::cout << "Normal Constructor" << std::endl; };

    // 析构函数
    ~Object() { std::cout << "Destructor" << std::endl; };

    // 复制构造函数
    Object(const Object &other) { std::cout << "Copy Constructor" << std::endl; };

    // 赋值运算符重载
    Object &operator=(const Object &other) {
        std::cout << "Copy Assignment" << std::endl;
        return *this;
    }
private:
 int val;
};

Object GetObject(int i) {
    Object temp = Object(i);
    return temp;
}

int main() {
    Object obj1 = GetObject(1);
    Object obj2;
    obj2 = obj1;

    return 0;
}
```

程序的打印结果表示, 有三次额外的对象拷贝. 第一次是将创建的临时对象 Object(i) 作为复制构造函数的参数构建函数的局部对象 temp; 第二次是在 GetObject() 中将在函数中创建的 temp 作为参数调用复制构造函数创建函数返回值; 第三次是将这个函数返回值作为参数调用复制构造函数创建 obj1.

```c++
Normal Constructor     // GetObject(1) 函数中调用 Object(int val)
Copy Constructor       // 将这个临时对象作为复制构造函数的参数构建 temp
Destructor             // 临时对象使用完毕, 调用析构函数回收资源
Copy Constructor       // 将 temp 作为复制构造函数的参数构建 GetObject(1) 的返回值
Destructor             // temp 出了 GetObject(1) 的函数作用域, 被销毁 
Copy Constructor       // GetObject(1) 的返回值作为参数构建 obj1
Destructor             // GetObject(1) 的函数返回值使用完毕, 被销毁
Default Constructor    // 构建 obj2;
Copy Assignment        // obj1 作为参数参与 obj2 的赋值运算符重载
Destructor             // main() 结束, obj1 销毁
Destructor             // main() 结束, obj2 销毁
```

在 Object 的定义中添加移动构造函数以及移动赋值函数来实现移动语义, 减少了三次临时对象的构建和析构

```c++
// 移动构造函数
Object(const Object &&other) { std::cout << "Move Constructor" << std::endl; };

// 移动赋值函数
Object &operator=(const Object &&other) {
    std::cout << "Move Assignment" << std::endl;
    return *this;
}
```

在添加移动语义之后的打印结果, 之前的复制构造函数都被替换成了移动构造函数

```c++
Normal Constructor
Move Constructor  // 优化 1
Destructor
Move Constructor  // 优化 2
Destructor
Move Constructor  // 优化 3
Destructor
Default Constructor
Copy Assignment
Destructor
Destructor
```

当然, 在 C++17 中拷贝优化 (Copy Ellison) 被引进, 由编译器进行优化, 现在的优先顺序是先由编译器进行拷贝优化, 如果编译器不支持拷贝优化, 然后就去检测对象是否支持移动语义 (Move Constructor and Move Assignment), 如果支持就使用移动构造函数, 如果对象没有定义移动语义, 那么最后才会调用复制构造函数或者是赋值运算符重载.

下面是使用了拷贝优化之后的程序运行结果

```console
Normal Constructor   // Object obj1 = GetObject(1);
Default Constructor  // Object obj2;
Copy Assignment      // obj2 = obj1;
Destructor
Destructor
```

> 总结: 右值引用的存在并不是为了取代左值引用, 而是充分利用右值 (特别是临时对象) 来减少对象构造和析构操作以达到提高效率的目的

### 2.2 右值引用实现完美转发 (Perfect Forwarding)

To be done

## 3. Aside 

### 3.1 引用 v.s. 指针

下表做一个引用和指针的简单对比

|              |                             引用                             |                             指针                             |
| :----------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|     定义     |                    引用就是一个对象的别名                    | 指针本身是一个变量, 他存储了一个内存单元的地址 (这个地址上的对象就是他指向的对象) |
|    初始化    | 引用必须要初始化, 且在引用的生命周期内不能成为其他对象的别名 | 指针可以一开始初始化为 nullptr, 之后在指向其他对象, 且指向的对象可以任意更改 |
|  自增/自减   |                 对他引用的对象进行自增/自减                  |              指针指向下一个/上一个同类型的地址               |
|   sizeof()   |                    返回他引用对象的字节数                    |                   返回这个指针本身的字节数                   |
|     层级     |                     引用只能使用一个层级                     |               指针可以有多个层级 (e.g. **ptr)                |
| 作为函数参数 |                           地址传递                           |                  本质上是值传递 (复制地址)                   |

左值引用和指针在汇编层面上面没有什么区别, 程序在编译的时候会创建一个符号表, 程序的变量以及他们的地址都会被填入表中, 引用就是一个他引用对象的别名, 所以引用在符号表存储引用名加上引用对象的地址. 而指针本身就是一个变量 (i.e. 指针本身是有自己的地址的), 所以指针在符号表中存储指针变量名加上本身变量的地址. 

程序的符号表在创建之后不能修改, 这也就可以解释为什么引用必须要初始化, 并且引用在其生命周期内不能在成为其他引用对象的别名 (因为引用对应的地址不能更改); 而指针本身就是一个变量, 他在符号表中存储的地址是自己变量的地址, 但是这个地址上存储的数据才是他指向对象的地址, 所以指针指向的地址是可以随意更换的.

### 3.2 std::move(): 将左值转换为右值

> std::move() 的本质就是帮助编译器选择重载函数, 告诉编译器 “请尽量把此参数当作右值处理”. 它本身不生成任何机器码.

std::move() 做的仅仅是类型转换 (将左值转换为右值), 它本身是不会真正的移动对象的. 真正的移动操作都是在移动构造函数和移动赋值函数中实现的.

```c++
Object obj = Object(1);
Object&& ref = std::move(obj); // 将 obj 转换为右值, 本身没有移动这个对象

Object obj1 = Object(ref); // 这里还是会调用复制构造函数, 因为 ref 本身就是一个左值
Object obj1 = Object(std::move(ref)); // 需要再次调用 move() 才会调用移动构造函数

// 在移动构造函数调用完之后, obj 的资源都移动到了 obj1 上, 但是 obj 本身并没有被销毁
// obj 会在函数作用域之外被调用析构函数被销毁
```

从上面的例子可以看出, 右值引用可以指向右值, 也可以使用 std::move() 指向左值; 而左值引用只能指向左值. 


### 3.3 std::forward(): 将一个值原封不动地转发出去

我们在前面知道 std::move() 的调用时需要确定这个数就是就是一个右值, 但是如果有时候你不确定传入的值是左值还是右值, 你可以使用 std::forward() 将这个值原封不动地传过去 (多用于 template function 的参数传递). 综上, std::forward() 需要被用在一个统一引用 (universal referencing) 的地方.

```c++
// 函数参数为左值引用
void func(std::string &str) { std::cout << "using l-value string: " << str << std::endl; }

// 函数参数为右值引用
void func(std::string &&str) { std::cout << "using r-value string: " << str  << std::endl; }

template <typename T>
void wrapper_func(T&& str) {
    // 这里的 str 是一个左值, 因为在内存中有了名字和位置, 所以永远会调用左值版本的函数
    func(str); 

    // 根据传入的参数是左值还是右值来调用对应版本的函数
    func(std::forward<T>(str));

    // 由于左值 str 被转换为了右值, 这里永远会调用右值版本的函数
    func(std::move(str));
}

int main() {
    std::string name = "value";
    wrapper_func(name);
    std::cout << "=====================" << std::endl;
    wrapper_func(name + "~");

    return 0;
}
```

需要注意的是, wrapper_func() 模版函数中的 T&& 参数是通用引用, 通用引用既可以是左值引用, 也可以是右值引用, 这取决于这个引用的初始化的引用类型. (这也解释了为什么这个模版函数既可以传入左值, 也可以传入右值)

最后程序运行的结果为:

```console
using l-value string: value
using l-value string: value
using r-value string: value
=====================
using l-value string: value~
using r-value string: value~
using r-value string: value~
```

