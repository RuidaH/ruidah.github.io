---
title: "论 std::weak_ptr 打破 std::shared_ptr 的循环引用"
date: 2023-10-27 15:46:25
categories: ["C++"]
tags:
---


根据 weak_ptr 的定义, 他可以作为观察者 (Observer) 来检测 shared_ptr 指向的对象是否存在 (也就是指针悬空); 除此之外, weak_ptr 还可以用于打破 shared_ptr 的环状结构. 这篇文章会通过一个例子展示为什么 weak_ptr 在这个场景下的应用.

### shared_ptr 的环状结构

```c++
struct Person;

struct Team {
    std::shared_ptr<Person> Person;
    ~Team() { std::cout << "Team is destructed." << std::endl; }
};

struct Person {
    std::shared_ptr<Team> Team;
    ~Person() { std::cout << "Person is destructed." << std::endl; }
};
```

在这个例子中, Team 和 Person 类都存在一个 shared_ptr 的成员指向对象. 我们创建一段执行代码, 并且在代码中分别创建指向 Team 和 Person 类的对象.

```c++
int main() {

    auto team = std::make_shared<Team>();

    {
        auto person1 = std::make_shared<Person>();

        person1->Team = team;
        team->Person = person1;
    }

    // person1 指针出了这个作用域之后, 指针和被指向的对象都被销毁
    if (team->Person.expired()) {
        std::cout << "team senses the destruction of person1." << std::endl;
    }

    std::cout << "------------------" << std::endl;

    return 0;
}
```

最后从执行上述代码得出的结果可以看出 Person 和 Team 的析构函数都没有被调用 (也就是说发生了两个对象都没有被销毁, 发生了内存泄露)
```console
------------------
```

具体原因如下图所示, 简单来说就是到最后这两个对象的引用计数都不为 0, 因此他们都没有被删除.

![1](/img/example_shared_ptr.jpg)




### 使用 weak_ptr 打破环状结构

```c++
struct Person;

struct Team {
    std::weak_ptr<Person> Person; // 将 shared_ptr 改成 weak_ptr
    ~Team() { std::cout << "Team is destructed." << std::endl; }
};

struct Person {
    std::shared_ptr<Team> Team;
    ~Person() { std::cout << "Person is destructed." << std::endl; }
};
```

然后再次执行相同的 main() 代码, 程序可以跑出以下结果: 我们可以看到 Person 和 Team 都可以被安全销毁, 而且在 Person 销毁的时候, Team 可以通过他的 weak_ptr 来检测 Person 对象是否存在

```console
Person is destructed.
team senses the destruction of person1.
------------------
Team is destructed.
```

程序执行过程由下图所示, 这两个对象的引用计数最后都变为 0

![1](/img/example_weak_ptr.jpg)


### 题外话

这个 weak_ptr 和 shared_ptr 一起使用的方法并不常见, 因为在严格分层的数据结构中, 子结点只被父节点持有, 当父节点被销毁的时候, 子节点就被销毁; 所以从父节点指向子结点的方向可以用 std::unique_ptr 表示, 从子到父的方向连接可以使用原始指针安全实现, 因为子结点的生命周期肯定短于父节点, 因此这个原始指针不会变成悬空指针. 但总的来说, 可以把 weak_ptr 和 shared_ptr 结合的方法当成是备用的选项.

### 完整代码

```c++
#include <stdio.h>
#include <iostream>
#include <memory>
#include <string.h>

struct Person;

struct Team {
    std::weak_ptr<Person> Person;
    ~Team() { std::cout << "Team is destructed." << std::endl; }
};

struct Person {
    std::shared_ptr<Team> Team;
    ~Person() { std::cout << "Person is destructed." << std::endl; }
};

int main() {

    auto team = std::make_shared<Team>();

    {
        auto person1 = std::make_shared<Person>();

        person1->Team = team;
        team->Person = person1;
    }

    // person1 指针出了这个作用域之后, 指针和被指向的对象都被销毁
    if (team->Person.expired()) {
        std::cout << "team senses the destruction of person1." << std::endl;
    }

    std::cout << "------------------" << std::endl;

    return 0;
}
```