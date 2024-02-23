---
title: C++ 并发编程总结
date: 2024-02-23 14:46:29
categories: ["C++"]
tags:
---

本篇文章对 C++ 并发编程中多线程的同步方式做一个总结.

## 互斥量 (Mutex)

- 保证在多线程环境下的同一个时间段内只会有一个线程访问共享资源/临界区 (critical section)
- `<mutex>` 头文件定义了四种主要的 mutex
  - `std::mutex`
  - `std::recursive_mutex`
  - `std::time_mutex`
  - `std::recursive_timed_mutex`

### std::mutex

- `std::mutex` 是 C++ 标准库中的一个类, 用于提供互斥锁的功能
- 需要注意的是 `std::mutex` 如果使用不当 (比如说一个线程在释放这个锁之前尝试再次锁定它) 就会造成死锁, 因此建议使用`std::lock_guard` 或者 `std::unique_lock`, 因为他们提供了 RAII (Resource Acquisition is Initialisation; 资源获取即初始化) 的管理方式, 可以确保锁会被自动释放

```c++
#include <iostream>
#include <mutex>
#include <thread>

// global mutex and variable
std::mutex mu;
int counter = 0;

void add_counter() {
    for (int i = 0; i < 100; ++i) {
        mu.lock(); // acquire the mutex
        ++counter;
        mu.unlock(); // release the mutex
    }
}

int main()
{
    std::thread t1(add_counter);
    std::thread t2(add_counter);
    
    t1.join();
    t2.join();
    
    std::cout << "counter = " << counter <<std::endl; // 200

    return 0;
}
```

### std::recursive_mutex

- `std::recursive_mutex` 是 C++ 标准库中的一种互斥锁，它允许同一个线程多次锁定互斥锁而不会导致死锁; recursive_mutex 可以解决基础 mutex 一个线程尝试对已经被它锁定的互斥锁再次加锁而导致死锁的情况
- `std::recursive_mutex` 允许一个线程多次获得互斥锁, 当然它必须释放相同数量的锁才能最终解锁互斥锁
- 相比于普通的 mutex, recursive_mutex 有着更高的性能开销 (追踪锁的持有者 + 记录互斥量的锁定次数), 但是 recursive_mutex 适用于需要在同一线程的多个函数调用中持有锁的情况, 特别是在递归函数中

```c++
#include <iostream>
#include <mutex>
#include <thread>

std::recursive_mutex rmu;

void recursive_func(int level) {
    if (level > 0) {
        rmu.lock();
        std::cout << "level " << level << " acquires lock." << std::endl;
        recursive_func(level - 1);
        std::cout << "level " << level << " releases lock." << std::endl;
        rmu.unlock();
    }
}

int main()
{
    std::thread t(recursive_func, 3);
    t.join();

    return 0;
}
```

- 上述例子得出的结果如下

```console
level 3 acquires lock.
level 2 acquires lock.
level 1 acquires lock.
level 1 releases lock.
level 2 releases lock.
level 3 releases lock.
```

### std::timed_mutex

- `std::timed_mutex` 提供了基于时间的锁定功能, 它允许线程尝试在指定的时间内获取锁, 如果获取失败, 那么线程可以选择放弃或者执行其他操作
- 主要成员函数
  - `lock()`
  - `unlock()`
  - `try_lock()`: 尝试获取锁, 返回 true 表示获取成功; 反之返回 false
  - `try_lock_for(duration)`: 在指定时间内尝试获取锁, 获取成功返回 true, 否则返回 false
  - `try_lock_until(timeout)`: 尝试获取锁直到锁定的时间点, 在这个时间点之前如果获取成功, 返回 true, 否则返回 false

```c++
#include <iostream>
#include <mutex>
#include <thread>
#include <chrono>

std::timed_mutex tmu;

void try_lock_for(int id, std::chrono::milliseconds timeout) {
    if (tmu.try_lock_for(timeout)) {
        std::cout << "Thread " << id << " acquires the lock" << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(100)); // sleep for 1s
        tmu.unlock();
    } else {
        std::cout << "Thread " << id << " couldn't acquire the lock" << std::endl;
    }
}

int main()
{
    std::thread t1(try_lock_for, 1, std::chrono::milliseconds(300));
    std::thread t2(try_lock_for, 2, std::chrono::milliseconds(90));
    
    t1.join();
    t2.join();

    return 0;
}
```

我们可以看到因为线程 t2 的等待时间较短, 所以当线程 t1 获取锁并且花费 1s 的时间“处理工作”, 那么 t2 则无法在 timeout 之前获取锁

```console
Thread 1 acquires the lock
Thread 2 couldn't acquire the lock
```

## Mutex with RAII

### std::lock_guard

- `std::lock_guard` 是一个作用域锁, 在构造时自动获取锁, 在析构时自动释放锁
- 互斥锁的持有时间和 `std::lock_guard` 的生命周期一致
- 他不能显式地解锁或者重新锁定, 因此只适用于简单的锁定场景

```c++
#include <iostream>
#include <mutex>
#include <thread>

// global mutex and variable
std::mutex mu;
int counter = 0;

void add_counter() {
    std::lock_guard<std::mutex> guard(mu); // acquire the lock when guard is created
    for (int i = 0; i < 100; ++i) {
        ++counter;
    }
} // release the lock when guard is out of scope

int main()
{
    std::thread t1(add_counter);
    std::thread t2(add_counter);
    
    t1.join();
    t2.join();
    
    std::cout << "counter = " << counter <<std::endl;

    return 0;
}
```

### std::unique_lock

- `std::unique_lock` 要比 `std::lock_guard` 更加灵活, 他可以在任何时候被显式地加锁和解锁.
- `std::unique_lock` 的一些应用场景
  - 延迟锁定
  - 所有权转移: 搭配 std::move() 使用
  - 支持条件变量

```c++
#include <iostream>
#include <mutex>
#include <thread>
#include <chrono>

std::mutex mu;

void delayed_locking(int id) {
    std::unique_lock<std::mutex> ulock(mu, std::defer_lock); // 延迟锁定
    
    // 执行一些不需要锁的操作
    std::cout << "Thread " << id << " doing some work without lock\n";
    
    // 执行需要互斥锁的操作
    ulock.lock();
    std::cout << "Thread " << id << " has acquired the lock\n";
} // ulock 超出作用域会自动解锁

int main()
{
    std::thread t1(delayed_locking, 1);
    std::thread t2(delayed_locking, 2);
    
    t1.join();
    t2.join();

    return 0;
}
```

上面延迟锁定例子打印出的结果如下

```console
Thread 2 doing some work without lock
Thread 2 has acquired the lock
Thread 1 doing some work without lock
Thread 1 has acquired the lock
```

## 条件变量 (Condition Variable)

- 条件变量用于阻塞一个或者多个线程, 直到某个线程修改线程间的共享变量 (或者满足某些条件), 并且通过 condition_variable 通知其余阻塞线程, 从而使得已阻塞线程可以继续执行后续的操作
- 条件变量的两个作用
    1. 用于通知已阻塞线程, 共享变量已经改变
        - 首先获取 `std::mutex`
        - 在持有锁期间, 在条件变量 `std::condition_variable` 上执行 `notify_one()` / `notify_all()` 来唤醒阻塞线程
    2. 用来阻塞某一线程, 知道该线程被唤醒
        - 使用 `std::unique_lock<std::mutex>` 来实现锁定操作 (unique_lock 为 wait 系列函数提供了解锁和重新锁定的能力)
        - 执行 wait 相关函数, 该操作能够原子性的释放互斥量 mutex, 并且阻塞这个线程
        - 当条件变量 `std::condition_variable` 被通知或者是被虚假唤醒 (与 OS 的线程调度和想能优化有关), 该线程结束阻塞状态, 并且自动获取到互斥量 mutex (获取到 mutex 之后还需要检查一遍条件判断是否成立, 以此避免虚假唤醒)
- std::condition_variable 可以搭配 std::unique_lock 使用来同步线程间的操作

### wait()

```c++
void wait (unique_lock<mutex>& lck);

template <class Pred> 
void wait(unique_lock<mutex>& lock, Pred pred);
```

- 调用 `wait()` 之前, 先获取一个 unique_lock 并将其作为参数传递给该函数
- 如果当前条件变量不满足, 线程会被阻塞, 其持有的锁也会被释放掉, 其他线程可以去获取这个 mutex 继续执行他们的操作
- 当另一个线程调用 `notify_one()` / `notify_all()` 来通知条件变量时, 被阻塞的线程将会被唤醒, 并在此尝试获取锁
- `wait()` 返回时, 锁会被再次持有
  
以下是该函数的 2 种使用方法:

- 当 ready_ == false 时, 当前线程会释放互斥锁, 在 wait() 处保持阻塞, 等待其他线程调用 notify 相关函数唤醒该线程
- 线程苏醒之后遍历多一遍 while loop 来避免虚假唤醒

```c++
std::condition_variable cv;
std::mutex mutex;
bool ready_ = false;

std::unique_lock<std::mutex> lock(mutex);
while (!ready_) {
  cv_.wait(lock);
}
```

- 使用谓词 (predicate): 一个返回 boolean 值的函数或者是函数对象;
- 当程序首次执行到 wait() 这一行代码时, 互斥量是被持有的; 此时线程判断谓词返回的结果
  - `predicate == true`: 条件成立, 线程继续工作
  - `predicate == false`: 条件不成立, 线程将在这一代码行保持阻塞状态等待唤醒, 释放互斥锁
- 当另一个线程调用 notify() 相关函数唤醒该阻塞线程时
  - 线程会重新获得互斥锁, 并且从阻塞状态转换为运行状态
  - 再次检查谓词
    - `true`: 线程继续工作
    - `false`: 线程将释放互斥锁, 重新回到阻塞状态, 等待唤醒

```c++
cv_.wait(lock, []() { return ready_; });
```

### wait_for()

```c++
template<class Rep, class Preiod>
cv_status wait_for(unique_lock<mutex>& lock, const chrono::duration<Rep, Period>& rel_time);

template<class Rep, class Preiod, class Pred>
bool wait_for(unique_lock<mutex>& lock, const chrono::duration<Rep, Period>& rel_time, Pred pred);
```

- `std::cv_status` 是 C++ 标准库中的一个枚举类型, 用于表示条件变量等待操作的状态; 他有两个枚举值:
  - `std::cv_status::no_timeout`: 表示等待操作在超时之前就已经被通知; 即条件已经满足或者条件变量收到了通知
  - `std::cv_status::timeout`: 表示等待操作因为超时而结束, 即在指定的时间内没有被满足, 也没有收到通知
- `wait_for()` 函数相比于 `wait()` 函数多了一个时间段 (`std::chrono::duration`) 限制; 如果在指定时间段内没有被唤醒, 那么就会返回超时的枚举类型

```c++
void worker_thread_without_predicate() {
    std::unique_lock<std::mutex> ulock(mu);
    auto status = cv.wait_for(ulock, std::chrono::seconds(5));

    if (status == std::cv_status::no_timeout) {
        // 被通知
    } else {
        // 超时
    }
}

void worker_thread_with_predicate() {
    std::unique_lock<std::mutex> ulock(mu);
    if (cv.wait_for(ulock, std::chrono::seconds(5), [] {return ready; })) {
        // 被通知, 且 ready 为 true
    } else {
        // 超时
    }
}
```

### wait_until()

```c++
template<class Clock, class Duration> 
cv_status wait_until(unique_lock<mutex>& lock, const chrono::time_point<Clock, Duration>& abs_time);

template<class Clock, class Duration, class Pred> 
bool wait_until(unique_lock<mutex>& lock, const chrono::time_point<Clock, Duration>& abs_time, Pred pred);
```

- `wait_until()` 用于阻塞当前线程, 直到达到指定的时间点或者被通知.

```c++
void worker_thread() {
    std::unique_lock<std::mutex> ulock(mu);
    auto timeout_time = std::chrono::steady_clock::now() + std::chrono::seconds(5);
    if (cv.wait_until(ulock, timeout_time, [] {return ready; })) {
        // 被通知, 且 ready ==  true
    } else {
        // 超时
    }
}
```

### 例子

```c++
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <chrono>

struct SharedZone {
    std::mutex mu_;
    std::condition_variable cv_;
    bool data_ready_ = false;  // 条件变量相关联的条件
};

void PrintId(SharedZone& shared_zone, int id) {
    std::unique_lock<std::mutex> lck(shared_zone.mu_);  // 锁定互斥体
    std::cout << "thread " << id << " is waiting for the flag" << std::endl;
    while (!shared_zone.data_ready_) {  // 条件不满足, 则等待
        // std::unique_lock 会自动释放互斥锁并使线程阻塞
        // 当 cv 收到通知时并且通知线程被唤醒时, std::unique_lock 会在 cv.wait() 返回之前自动重新获取互斥锁
        shared_zone.cv_.wait(lck); 
    }
    std::cout << "thread " << id << " accesses the shared zone" << std::endl;
}

void Go(SharedZone& shared_zone) {
    std::unique_lock<std::mutex> lck(shared_zone.mu_);
    shared_zone.data_ready_ = true;  // 改变共享状态
    shared_zone.cv_.notify_all();  // 唤醒所有等待线程
}

int main() {
    std::thread threads[5];
    SharedZone shared_zone;
    // 启动 5 个线程
    for (int i = 0; i < 5; ++i) {
        threads[i] = std::thread(PrintId, std::ref(shared_zone), i); // 使用 std::ref() 传递引用
    }

    std::this_thread::sleep_for(std::chrono::seconds(3));
    std::cout << "\nthread race \n\n";
    Go(shared_zone);
    
    for (auto &th : threads) th.join();

    return 0;
}
```

- 上述条件变量搭配 unique_lock 使用的例子打印的结果如下

```console
thread 0 is waiting for the flag
thread 1 is waiting for the flag
thread 3 is waiting for the flag
thread 4 is waiting for the flag
thread 2 is waiting for the flag

thread race 

thread 2 accesses the shared zone
thread 3 accesses the shared zone
thread 1 accesses the shared zone
thread 4 accesses the shared zone
thread 0 accesses the shared zone
```

<!-- ## 读写锁

## 信号量 (Semaphore)

- 信号量是一个整型计数器, 表示资源的数量, 用来控制对共享资源的访问
- 信号量初始值
  - 1: 表示互斥信号量 (保证同一时间段内只有一个进程可以访问共享内存)
  - 0: 表示同步信号量 (使进程同步完成同一任务)
- `P 操作 = 信号量 - 1`: 获取资源
  - `< 0`: 无资源可用 => 申请信号量的线程保持阻塞
  - `>= 0`: 有资源可用 => 线程继续执行
- `V 操作 = 信号量 + 1`: 释放资源
  - `<= 0`: 有阻塞进程 => 唤醒阻塞进程获取资源
  - `> 0`: 无阻塞进程 => 继续执行 -->

# 参考

https://www.cnblogs.com/fenghualong/p/13855360.html

https://www.cnblogs.com/haippy/p/3284540.html