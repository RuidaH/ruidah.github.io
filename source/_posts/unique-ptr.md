---
title: "智能指针 std::unique_ptr 的定义和应用"
categories: [C++]
date: 2023-10-27 15:55:09
tags: Notes of Effective Modern C++
---


Unique poiter 和原始指针一样快, 而且她不能复制, 只能转移所有权 (move-only).

### unique_ptr 定义和使用

unique_ptr 的创建有两种方法

```c++
#include <stdio.h>
#include <iostream>
#include <memory>
#include <string.h>

struct Person {
    std::string Name;
    Person(std::string name) : Name(name){};
};

int main() {
    // unique_ptr 创建方法 1
    std::unique_ptr<Person> person1(new Person("Jason"));
    std::unique_ptr<int> uniquePtr1(new int);
    std::unique_ptr<int> uniquePtr2; // 空指针, nullptr

    std::cout << "Name of person1: " << person1->Name << std::endl;
    std::cout << "Value of uniquePtr1: " << *uniquePtr1 << std::endl;
    if (!uniquePtr2) {
        std::cout << "uniquePtr2 is a nullptr" << std::endl;
    }

    // unique_ptr 创建方法 2
    auto person2 = std::make_unique<Person>("Rose");
    std::cout << "Name of person2: " << person2->Name << std::endl;

    return 0;
}
```

```console
Name of person1: Jason
Value of uniquePtr1: 0
uniquePtr2 is a nullptr
Name of person2: Rose
```

unique_ptr 可以通过 release() 放弃所有权, 可以通过 std::move() 完成所有权的转移, 或者通过 reset() 放弃所有权或者指向一个全新的对象

```c++
struct Person {
    std::string Name;
    Person(std::string name) : Name(name){};
};

int main() {
    auto person2 = std::make_unique<Person>("Rose");
    std::cout << "Name of person2: " << person2->Name << std::endl;

    // person2 放弃所有权
    person2.release();
    if (!person2) {
        std::cout << "person2 is now a nullptr." << std::endl;
    }

    std::cout << "-----------------------------" << std::endl;

    // person3 通过 move() 将所有权转移给 person4
    auto person3 = std::make_unique<Person>("OJBK");
    std::unique_ptr<Person> person4 = std::move(person3);

    std::cout << "Name of person4: " << person4->Name << std::endl;
    if (!person3) {
        std::cout << "person3 is now a nullptr." << std::endl;
    }

    std::cout << "-----------------------------" << std::endl;

    // reset() 放弃或者转移所有权
    auto person5 = std::make_unique<Person>("Preston");
    auto person6 = std::make_unique<Person>("OJBK");
    person5.reset();
    person6.reset(new Person("NewOne"));

    std::cout << "Name of person6: " << person6->Name << std::endl;
    if (!person5) {
        std::cout << "person5 is now a nullptr." << std::endl;
    }

    return 0;
}
```

```console
Name of person2: Rose
person2 is now a nullptr.
-----------------------------
Name of person4: OJBK
person3 is now a nullptr.
-----------------------------
Name of person6: NewOne
person5 is now a nullptr.
```

unique_ptr 也可以作为函数参数或者函数返回值

```c++
struct Person {
    std::string Name;
    Person(std::string name) : Name(name){};
};

void printName(std::unique_ptr<Person> person) {
    std::cout << "Name of person: " << person->Name << std::endl; 
}

std::unique_ptr<Person> createPerson() {
    auto person = std::make_unique<Person>("Olivia");
    return person;
}

int main() {
    auto person = std::make_unique<Person>("Jason");

    // unique_ptr 传入函数参数应该搭配 move() 使用, 因为他没法复制
    printName(std::move(person));
    
    // 但是 unique_ptr 一旦被传入的话, person 就丧失了对象的所有权
    if (!person) {
        std::cout << "person is now a nullptr." << std::endl;
    }

    // 这里调用的是 move assignment, 转移返回的 unique_ptr 的所有权
    auto newPerson = createPerson();
    std::cout << "Name of new person: " << newPerson->Name << std::endl;

    return 0;
}
```

```console
Name of person: Jason
person is now a nullptr.
Name of new person: Olivia
```

### 应用 1: 控制指向对象的生命周期

当你使用一个 std::unique_ptr 指向一个对象时, 你可以通过这个智能指针来控制这个对象的生命周期, 当指针出了函数的作用域的时候, 这个指针会被销毁 (调用指针的析构函数), 这个时候指针指向的对象也会被销毁.

```C++
// 假设有一个 Animal 的父类和三个子类: Dog, Cat, Tiger

// 定义一个返回指向 Animal 对象的智能指针的函数
template <typename... Ts>
std::unique_ptr<Animal> createAnimal(Ts&&... args);

// 函数的使用
{
    ...
    auto animal_ptr = createAnimal(args);
    ...
} // 销毁 animal_ptr
```




### 应用 2: 自定义删除器 (deleter_type)

```c++
// 这里的工厂函数使用了 C++14 的函数返回类型推导
// auto = std::unique_ptr<Animal, decltype(delAnimal)>
template <typename... Ts>
auto createAnimal(Ts&&... args) {
    // 使用 lambda 创建一个自定义的删除器
    auto delAnimal = [](Animal* animal_ptr) {
        makeLogEntry(animal_ptr); // logging
        delete animal_ptr; // 销毁这个指针及其指向的对象
    };

    // 创建一个空的 std::unique_ptr, 然后根据条件指向对应的对象
    // 这里的删除器类型需要作为第二个参数传递进去
    std::unique_ptr<Animal, decltype(delAnimal)> ptr(nullptr, delAnimal);

    // 根据条件来判断动物的类型
    if (...) {
        // 这里使用 reset 来接管通过 new 创建的对象所有权
        // 使用 std::forward() 将参数完美转发出去
        ptr.reset(new Dog(std::forward<Ts>(args)...));
    } else if (...) {
        ptr.reset(new Cat(std::forward<Ts>(args)...));
    } else if (...) {
        ptr.reset(new Tiger(std::forward<Ts>(args)...));
    }
    
    return ptr;
}
```

这里需要注意的是, 自定义删除器中的指针类型是 Animal*, 这意味着无论 createAnimal() 函数中创建的是哪个子类对象, 他在删除器中都是作为 Animal 被删除的. 这个时候我们要允许基类 Animal 有虚析构函数 (virtual destructor)

```c++
class Animal {
public:
    ...
    virtual ~Animal();
    ...
}
```

在使用默认的删除器的时候, std::unique_ptr 的大小可以假设跟原始指针的大小一样; 但是在使用函数指针形式的的自定义删除器的时候, std::unique_ptr 的体积有可能会增大. 对于函数对象形式的删除器来说, 变化的大小取决于函数对象中存储的状态多少, 无状态函数 (stateless function) 对象对大小没有影响. 如果需要使用到上下文信息, 那么就尽量少使用, 不然 std::unique_ptr 的体积可能会过大. 平时在使用

```c++
// 无状态的 lambda 自定义删除器
auto delAnimal1 = [](Animal* animal_ptr) {
    makeLogEntry(animal_ptr); 
    delete animal_ptr; 
};

// 这里返回类型大小就是 Animal* 的大小
template <typename... Ts>
std::unique_ptr<Animal, decltype(delAnimal1)> createAnimal(Ts&&... args);


// 函数形式的自定义删除器
void delAnimal2(Animal* animal_ptr) {
    makeLogEntry(animal_ptr); 
    delete animal_ptr; 
}

// 返回类型大小是 Animal* 指针 + 一个函数指针的大小
template <typename... Ts>
std::unique_ptr<Animal, void (*)(Animal*)> createAnimal(Ts&&... args);
```

### Aside: 无状态函数 && 有状态函数

> 无状态函数 (stateless function): 通常是指不携带任何额外状态或者上下文信息的函数; 比如不捕获变量的 lambda 表达式
>
> 有状态函数 (stateful function): 有状态函数刚好相反, 他依赖于某种形式的内部状态或者上下文. 最常见的实现方式是使用成员变量的成员函数

```c++
/* 无状态函数例子 */
int add(int a, int b) { return a + b; }



/* 状态函数例子 */
class StatefulCounter {
public:
    StatefulCounter(): count(0) {};
    void increment() { count++; };
    void decrement() { count--; };
    int getCount() const { return count; }
private:
    int count;
}


int main() {
    // add() 是一个无状态函数, 他不依赖任何外部状态
    int (*func_ptr)(int, int) = add;
    int result = func_ptr(3, 5);

    StatefulCounter counter;
    counter.increment();
    // getCount() 是一个状态函数, 他返回的数值依赖类成员 count
    int current_count = counter.getCount();
}
```




### 应用 3: 转移所有权

除了使用 reset() 将所有权转让给另外一个 std::unique_ptr 之外, std::unique_ptr 还可以轻松转换为 std::shared_ptr.

```c++
// 将 std::shared_ptr 转化为 std::unique_ptr
std::shared_ptr<Animal> sp = createAnimal(args); 
```

std::unique_ptr 很适合作为工厂函数的返回类型, 当调用者想要共享所有权的时候, 他可以直接将 std::unique_ptr 转换为 std::shared_ptr







