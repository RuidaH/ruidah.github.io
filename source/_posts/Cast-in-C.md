---
title: C++ 类型转换
date: 2023-11-18 14:56:40
categories: [C++]
tags:
---

## static_cast

```cpp
int main() {
    double d = 3.124;
    int i = static_cast<int>(d);
    std::cout << i << std::endl;  // 打印 3, 转换涉及到值的截断
    
    Base* base = new Derived();
    std::cout << static_cast<int>(base) << std::endl;  // 转换不合法
    /* error: invalid 'static_cast' from type 'Base*' to type 'int' */
}
```

static_cast 用于非多态类型转换 (i.e. 普通类型转换); 他在编译时会检查转换的有效性, 如果转换不合法, 会产生编译错误; 需要注意的是基本类型之间的转换可能会涉及值的截断或者扩展

## dynamic_cast

dynamic_cast 用于多态类型转换 (检查两者是否有继承关系)， 运行时会检查对象的类型信息. 比如说在游戏中为一把枪创建一个基类, 然后这个基类枪可以派生出不同类型的枪 (i.e. 不同的子类). 在游戏中可以通过 ‘GetSpear(int type, int uid)’ 中的 type 和 uid 返回不同的子类枪对象, 同时函数的返回类型可以是父类枪的指针, 然后再调用函数之后使用 dynamic_cast 对其进行转换. 这个例子保证了接口调用的统一.

这种转换有运行时的开销, 因为他需要检查对象的实际类型. 除此之外, 动态转换也分为指针和引用的转换

```cpp
class Base {
public:
    virtual void Print() {
        std::cout << "Base::Print()" << std::endl;
    }
};

class Derived: public Base {
public:
    void Print() override {  // 虚函数
        std::cout << "Derived::Print()" << std::endl;
    }
    void NonVirtualPrint() { // 非虚函数
        std::cout << "Derived::NonVirtualPrint()" << std::endl;
    }
};

int main() {
    /* 动态转换指针 */
    Base* base = new Derived();
    Derived* derived = dynamic_cast<Derived*>(base);
    
    if (derived != nullptr) {
        derived->Print(); // Derived::Print()
    }
}
```

转换合法的话就返回转换类型的指针, 失败直接返回空指针 (nullptr)

```cpp
/* 复用上面代码对于 Base, Derived 类的定义 */

void PrintDerived(Base& base) {
    try {
        /* 动态转换引用 */
        Derived& derived = dynamic_cast<Derived&>(base);
        derived.Print();
    } catch (const std::bad_cast& e) {
        std::cout << "std::bad_cast" << std::endl;
    }
}

int main() {
    Derived derived;
    Base& base_ref = derived; // base_ref 实际上引用的是 Derived 对象
    
    PrintDerived(base_ref);  // Derived::Print()
    
    Base real_base;
    PrintDerived(real_base); // std::bad_cast
}
```

转换成功的话直接返回准换类型的引用, 失败的话抛出一个 ‘std::bad_cast’ 的异常. 在这个例子中, PrintDerived 函数会尝试将传入的 Base 引用类型转换为 Derived 类型引用**.** 第一次传入的 base_ref 是一个指向 Derived 类型对象的引用, 所以转换成功; 但是第二次传入的 real_base 实际上是一个 Base 类型对象, 所以转换失败.
  
dynamic_cast 又分为上行转换 (upcasting) 和 下行转换 (downcasting)

### dynamic_cast 上行转换 (Upcasting)

向上转换是指将派生类的指针或引用转换为基类的指针或引用。向上转换通常是安全的，因为派生类对象 Derived 可以被看作是基类对象 Base. 上行转换有显性和隐性两种转换方式

```cpp
int main() {
    /* 隐性转换 */
    Derived d;
    Base* derived = &d;
    derived->Print();  // Derived::Print()
    
    /* 显性转换 */
    Derived* dp = new Derived();
    Base* derived1 = dynamic_cast<Base*>(dp);
    derived1->Print();  // Derived::Print()
    
    return 0;
}
```

### dynamic_cast 下行转换 (Downcasting)

向下转换是指将基类的指针或引用转换为派生类的指针或引用。向下转换是不安全的，因为基类指针可能并不真正指向派生类对象. dynamic_cast 会在运行时检查基类指针是否真的指向一个派生类的对象. 如果转换失败, 那么就会返回空指针(转换指针时) 或者异常 std::bad_cast (转换引用时)

```cpp
int main() {
    /* 安全的向下转换 */
    Base* base1 = new Derived();  // 指向派生类对象
    Derived* derived1 = dynamic_cast<Derived*>(base1);
    derived1->Print();  // Derived::Print()
    
    /* 不安全的向下转换 */
    Base* base2 = new Base(); // 指向基类对象
    Derived* derived2 = dynamic_cast<Derived*>(base2);
    derived2->Print();  // Segmentation fault 
    
    return 0;
}
```

当进行不安全的下行转换的时候, dynamic_cast 直接返回空指针 nullptr, 空指针调用虚函数会直接报错 (segment fault).

## const_cast

const_cast 用于修改类型的 const 或者 volatile 属性. 最常见的用途是去掉对象的 const 性质, 以便可以修改他.

```cpp
int main() {
    const int num = 10;
    int* ref = const_cast<int*>(&num);
    *ref = 20; // 未定义行为， 原始变量 num 是 const
    
    std::cout << *ref << std::endl;  // 20
}
```

const_cast 同时也可以去除类对象的 const 属性, 从而使得该对象调用非 const 的函数

```cpp
class TempClass {
public:
    void func() const {
        std::cout << "const function" << std::endl;
    }
    void func() {
        std::cout << "non-const function" << std::endl;
    }
};

int main() {
    const TempClass obj;
    obj.func();  // const function
    const_cast<TempClass&>(obj).func();  // non-const function

    return 0;
}
```

需要注意的是, 这并不是真正的去除, 变量本身的 const/volatile 属性是不会发生改变的. const_cast 只是使得非 const/volatile 类型的引用或指针能够实际取代 const 对象.

const_cast 需要非常谨慎地使用, 因为通过非 const 访问路径修改 const/volatile 对象会破坏常量性的保证, 可能会导致未定义行为.

## reinterpret_cast

reinterpret_cast 作为一种类型转换运算符, 可以将任何一种指针类型转换为任何其他的指针类型方式. reinterpret_cast 不检查转换的安全性, 因此使用者在调用的时候需要完全确定自己在做什么, 并且理解其潜在的风险, 因为该类型转换有可能会导致程序崩溃或者是其他意外的行为.

我们以 CMU 15445 proj 2 构建数据库中的 B+ 树为例子, 当我们从 buffer pool 中读取一个树中的页 (page), 我们需要判断这个页是属于数的叶子节点还是内部节点, 然后调用 reinterpret_cast 进行相应的类型转换.

```cpp
Page* fetched_page = ...;
if (fetched_page->IsLeafPage()) {
    LeafPage* leaf_page = reinterpret_cast<LeafPage*>(page);
} else {
    InternalPage* internal_page = reinterpret_cast<InternalPage*>(page);
}
```

reinterpret_cast 常见于系统的底层编程, 例如数据库底层或者操作系统内核的研发. 在日常的应用程序开发中, 我们很少用到 reinterpret_cast.

## 总结

综上, static_cast 作为最普遍的类型转换, 多用于基本类型之间的转换, 在编译的时候会进行检查; dynamic_cast 作用于多态类型转换, 在运行时会使用 RTTI 确保类型是否可以正确转换; 上述两种转换较为安全. const_cast 用于去除对象的 const/volatile 的属性; reinterpret_cast 用于系统底层, 基本上是对位模式进行直接解释. const_cast 与 reinterpret_cast 的不正确使用会导致未定义行为, 因此应当谨慎使用.

## 参考

https://zh.cppreference.com/w/cpp/language/const_cast

https://lancern.xyz/2022/11/04/dynamic-cast-benchmark/
