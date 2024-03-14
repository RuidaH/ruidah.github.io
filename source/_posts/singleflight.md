---
title: Singleflight 机制
date: 2024-03-14 16:12:02
categories: ["C++", "Golang"]
tags: “缓存优化”
---

# 介绍

一句话介绍 singleflight: 当有多个 go-routine 同时执行相同的请求时 (e.g. 调用相同的函数), 只有第一个到达的 go-routine 能够去执行这个请求, 其他 go-routine 会保持阻塞, 等待执行请求的协程共享请求返回的结果.

singleflight 可以用来避免缓存击穿 (当某一个热点数据 (key) 过期之后, 大量的请求在缓存无法得到结果, 从而全部打到数据库中, 造成数据库的压力骤增); singleflight 只会允许第一个访问热点数据的请求打到数据库上, 其他相同的请求则会阻塞在缓存服务器中等待请求结果的共享 (后续的数据访问发生在缓存服务器中, 因为第一个请求已经将数据从数据库加载回缓存中了)

# 实现原理

以下是 singleflight 初始化部分的源码 (来源: https://github.com/golang/sync/blob/master/singleflight/singleflight.go)

```go
// call is an in-flight or completed singleflight.Do call
type call struct {
    wg sync.WaitGroup

    // These fields are written once before the WaitGroup is done
    // and are only read after the WaitGroup is done.
    val interface{}
    err error

    // These fields are read and written with the singleflight
    // mutex held before the WaitGroup is done, and are read but
    // not written after the WaitGroup is done.
    dups  int
    chans []chan<- Result
}

// Group represents a class of work and forms a namespace in
// which units of work can be executed with duplicate suppression.
type Group struct {
    mu sync.Mutex       // protects m
    m  map[string]*call // lazily initialized
}

// Result holds the results of Do, so they can be passed
// on a channel.
type Result struct {
    Val    interface{}
    Err    error
    Shared bool
}
```

- singleflight package 通过 `group` 的数据结构来管理不同的请求.
  - `m`: 管理请求的字典, 其中 key 为请求名称, value 为 `call`, 也就是正在进行的请求
  - `mu`: 互斥锁用来保护结构体中有关于请求的字典, 防止不同协程并发写入
- `call`: 表示正在进行的请求
  - `wg`: call 通过 `sync.WaitGroup` 锁来避免多个 go-routine 重入
  - `val`: 请求执行结束后返回的结果
  - `dups`: 记录相同并发请求数量
  - `chans`: 保存所有相同并发请求结果的通道; 当使用异步方式处理请求时, 返回的结果会通过这些通道发送给其他等待的协程

```go
// 当有多个相同的并发请求产生时, Do() 确保了只会有一个 go-routine 来执行正在进行的请求
// 第一个调用 Do() 的 go-routine 执行完毕, 会讲结果保存在请求结构体 call 的 val 和 err 中, 
// 其他并发的 go-routine 可以直接通过 group 的全局 map 来访问到返回的结果
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
    g.mu.Lock()  // 全局锁保护字典
    if g.m == nil {
        g.m = make(map[string]*call)  // lazy initialisation
    }
    // 如果这个 key 对应的请求已经产生了 (也就是说已经有协程去执行执行这个请求了), 
    // 那么当前的协程可以阻塞等待结果的返回
    if c, ok := g.m[key]; ok {   
        c.dups++    // 记录执行相同请求的协程数量
        g.mu.Unlock()  // 解开全局锁
        c.wg.Wait()    // 保持阻塞, 直到执行请求的协程返回结果 (i.e. 调用 c.wg.Done())

        // ......

        return c.val, c.err, true // 直接返回 call 请求返回的结果
    }

    // 当前请求还没有被协程处理, 那么就执行以下逻辑
    c := new(call)
    c.wg.Add(1)   // 协程发起请求之前加锁
    g.m[key] = c  // 添加到 group 的字典中, 表示当前 key 对应的请求已经被处理了
    g.mu.Unlock() // 解开全局锁

    g.doCall(c, key, fn)  // 执行请求
    return c.val, c.err, c.dups > 0
}
```

singlefligt package 除了 Do() 之外还支持只返回一个通道的 DoChan(); 这个 channel 在结果准备好的时候接收结果, 这使得调用者可以以异步的方式等待结果

```go
// 返回的 channel 不会被关闭
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
    // 协程创建一个缓冲大小为 1 的通道, 用来接受请求返回的结果
    // 缓冲大小为 1 是为了让发送结果的操作不会阻塞
    ch := make(chan Result, 1)
    g.mu.Lock()
    if g.m == nil {
        g.m = make(map[string]*call)
    }
    if c, ok := g.m[key]; ok {  // 已经有协程去执行这个相同的请求了
        c.dups++
        c.chans = append(c.chans, ch)  // 将接收结果的 channel 加入到这个请求对应的 channel 列表中
        g.mu.Unlock()
        return ch  // 返回接受到结果的通道
    }

    // 当前请求还没有协程处理, 创建一个新的 call, 并且把接收通道加入到通道列表中去
    c := &call{chans: []chan<- Result{ch}}
    c.wg.Add(1)  // 当前协程加锁, 其他协程会被阻塞
    g.m[key] = c
    g.mu.Unlock()

    go g.doCall(c, key, fn)  // 异步方式处理请求

    return ch
}
```

下面是 key 对应的请求真正被处理的逻辑代码 -- `doCall()`;  我们当前只关心当请求执行可以被正常返回的情况

```go
// doCall handles the single call for a key.
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
    // ......

    defer func() {
        // ......

        g.mu.Lock()  // 请求已经处理完成, 加上全局锁来修改 map
        defer g.mu.Unlock()
        c.wg.Done() // 处理请求的协程已经结束任务了, 调用 Done() 通知其余阻塞的协程继续执行操作 (也就是直接去读取返回结果即可)
        if g.m[key] == c {
            delete(g.m, key)  // 删除全局字典的 call 请求映射
        }

        if e, ok := c.err.(*panicError); ok {
            // panic error
        } else if c.err == errGoexit {
            // ......
        } else {
            // 这里逻辑的处理对应的是 DoChan() 而不是 Do()
            // 正常返回, 把结果发送到所有正在等待的 channels 上
            for _, ch := range c.chans {
                ch <- Result{c.val, c.err, c.dups > 0}
            }
        }
    }()

    func() {
        // ......

        c.val, c.err = fn()  // 请求逻辑的执行, 返回结果或者错误
        normalReturn = true
    }()
}
```

## Q&A

> 如果第一个调用下游函数的协程被阻塞住了, 那么其他执行相同并发请求的协程是否也会保持阻塞?

是的; 所以为了防止第一次调用阻塞所有调用的情况, 我们可以调用 Context 来设置超时时间来解决第一次请求阻塞问题; 如果在超时时间内没有得到结果, 那么就`尽快失败, 释放我们的资源, 从而避免请求堆积`

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"

	"golang.org/x/sync/singleflight"
)

var group singleflight.Group

func fetchData(ctx context.Context, key string, id int) (string, error) {
	fmt.Printf("(Request %d) fetchData called for key: %s\n", id, key)
	select {
	case <-time.After(2 * time.Second): // 模拟数据获取延迟
		return "data for " + key, nil
	case <-ctx.Done():
		return "", ctx.Err()  // 返回超时错误
	}
}

// 协程创建请求
func request(key string, timeout time.Duration, id int) {
	ctx, cancel := context.WithTimeout(context.Background(), timeout)
	defer cancel()

	result, err, _ := group.Do(key, func() (interface{}, error) {
		return fetchData(ctx, key, id)
	})

	if err != nil {
		fmt.Printf("Request %d error: %s\n", id, err)
	} else {
		fmt.Printf("Request %d result: %s\n", id, result)
	}
}

func main() {
	var wg sync.WaitGroup

	// 第一个请求
	wg.Add(1)
	go func() {
		defer wg.Done()
		request("key1", 1 * time.Second, 1)  // 设置超时时间为 1s
	}()

	// time.Sleep(3 * time.Second) // 等待第一个请求完成

	// 第二个请求
	wg.Add(1)
	go func() {
		defer wg.Done()
		request("key1", 3 * time.Second, 2)  // 设置超时时间为 3s
	}()

	wg.Wait()  // 主协程阻塞等待
}
```

当前例子代码中第一个请求 (超时时间设置为 1s) 和第二个请求 (超时时间设置为 3s) 并发进行, request() 调用模拟会花费 2s, 并发请求后的结果如下

```console
(Request 2) fetchData called for key: key1
Request 2 result: data for key1  // 第二个请求获得结果
Request 1 result: data for key1  // 第一个结果分享结果
```

如果在第一个和第二个请求中间增加 3s 的等待时间, 那么两个请求就算不上并发了, 执行程序后的结果如下

```console
(Request 1) fetchData called for key: key1
Request 1 error: context deadline exceeded  // 超时时间 1s  < 请求执行时间 2s, 超时
(Request 2) fetchData called for key: key1
Request 2 result: data for key1  // 请求成功
```

> 什么是相同的并发请求?

在某个协程还没有处理完当前请求之前到来的相同请求才会称为并发请求; 当前请求处理完毕后到达的相同请求会被单独处理, 而不是等待分享的结果.

> singleflight 可以用来作为全局的分布式优化吗?

singleflight 并不是一个合适分布式优化方法, 但是他作为一个单节点的负载均衡的优化方法用在分布式系统也还是可以接受;

在分布式系统中, 可能相同的请求会被打到不同的缓存服务器上 (每一个缓存服务器上都带有 singleflight 优化), 由于分布式的服务器数量不会过多 (最多几十个), 所以最坏情况下会有几十个相同的请求同时打到数据库上, 这个数量级在实际场景中还是可以接受的.

## C++ 实现 singleflight 机制

```c++
#include <iostream>
#include <unordered_map>
#include <mutex>
#include <condition_variable>
#include <string>
#include <thread>
#include <vector>
#include <functional>

class SingleFlight {
public:
    // 函数调用的结果
    struct Result {
        std::string value;
        std::string error;
        bool shared;
    };
    
    Result Do(const std::string& key, std::function<std::string()> fn) {
        std::unique_lock<std::mutex> ulock(mutex_);
        
        // 检查 map 看这个 key 是否已经有对应的请求正在执行了
        auto& call = calls_[key];
        if (call.inFlight) { // 当前请求已经被执行了
            call.cond.wait(ulock, [&call] { return !call.inFlight; });
            return {call.value, call.error, true};
        }
        
        // 现在还没有线程执行当前请求
        call.inFlight = true;
        ulock.unlock();
        
        // 执行函数
        try {
            call.value = fn();
        } catch (const std::exception& e) {
            call.error = e.what();
        }
        
        // 请求执行完毕, 返回结果
        ulock.lock();  // 重新上锁
        call.inFlight = false;
        std::this_thread::sleep_for(std::chrono::seconds(1));
        call.cond.notify_all();
        
        return {call.value, call.error, false};
    }
    
private:
    struct Call {
        bool inFlight = false; // 检查是否有调用进行
        std::condition_variable cond; 
        std::string value;  // 调用结果
        std::string error;  // 错误信息
    };
    
    std::mutex mutex_;  // 用来保护下面 map 的并发写入
    std::unordered_map<std::string, Call> calls_;  // 存储每一个 key 对应的请求信息
};

void Worker(SingleFlight& sf, const std::string& key, int id) {
    auto res = sf.Do(key, [&] {
       std::this_thread::sleep_for(std::chrono::seconds(1)); // 模拟耗时操作
       return "[result from worker" + std::to_string(id) + "]";
    });
    if (!res.shared) {
        std::cout << "First worker " << id << " got result: " << res.value << "\n";
    } else {
        std::cout << "Worker " << id << " got shared result: " << res.value << "\n";
    }
}

int main() {
    SingleFlight sf;
    
    std::vector<std::thread> workers;
    for (int i = 0; i < 5; ++i) {
        workers.emplace_back(Worker, std::ref(sf), "key", i);
    }
    
    for (auto& worker : workers) {
        worker.join();
    }
    
    return 0;
}
```

最后打印出来的结果: 

```console
First worker 2 got result: [result from worker2]
Worker 3 got shared result: [result from worker2]
Worker 4 got shared result: [result from worker2]
Worker 1 got shared result: [result from worker2]
Worker 0 got shared result: [result from worker2]
```

# 参考

https://www.bilibili.com/video/BV1dB421z7JT/?spm_id_from=333.337.search-card.all.click&vd_source=ceae61c3392491cff551f9e868d292bd

https://geektutu.com/post/quick-go-context.html