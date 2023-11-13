---
title: Lua 闭包
date: 2023-11-13 19:20:41
categories: [Lua]
tags:
---

## 函数定义

在了解闭包之前，我们先看一下 function 在 Lua 中的定义：

>Functions as first-class values: Function is a value with the same rights as more conventional values like numbers and strings. A program can store functions in variables (both global and local) and in tables, pass functions as arguments to other functions, and return functions as results.

也就是说：函数作为第一类型值（first-class value）在 lua 中与其他类型值一致， 函数其实就是一个字面值， 他在编译阶段就会被识别并保存，这也是为什么函数可以存在变量中，或者是作为参数传入另一个函数以及作为函数返回值。

在 Lua 中， 所有的函数都是匿名的（也就是说函数没有名字），当我们在谈论一个函数名称的时候（比如说 print）， 我们实际指的是保存这个匿名函数的变量名。比如:

```lua
local function FuncName() print("hello") end

-- 等同于
local FuncName
FuncName = function() print("hello") end
```

函数可以被存储在全局变量中，例子如下：

```lua
function Multiple(a, b)
    return a * b
end

c = Multiple
print(c(2, 3)) -- 6

c = function (a, b) return a + b end
print(c(2, 3)) -- 5
```

函数也可以被存储在局部变量中, 以下方式都可以用于 local 函数的创建

```lua
local Lib1 = {}
Lib1.multiple = function (a, b) return a * b end
Lib1.add      = function (a, b) return a + b end

local Lib2 = {
    multiple = function (a, b) return a * b end,
    add      = function (a, b) return a + b end
}

local Lib3 = {}
function Lib3.multiple (a, b) return a * b end
function Lib3.add (a, b) return a + b end
```

同时函数也可以被嵌套进另一个函数中，实现嵌套函数的功能

```lua
a = 1
function Wrapper()
    local b, c = 2, 3
    local func = function ()
        a = a + 1
        b = b + 1
        c = c + 1
        print("a =", a, "b =", b, "c =", c)
    end
    return func
end

local var = Wrapper()
print(var()) -- a = 2, b = 3, c = 4
print(var()) -- a = 3, b = 4, c = 5
```

## 词法作用域 (Lexical Scoping)

函数作为第一类型值有自己的词法作用域 (Lexical Scoping)。上面的例子使用到了词法作用域的特性： 返回的匿名函数可以捕获封闭函数（Enclosing Function）Wrapper() 中的变量 a, b，以及全局变量 a， 并且修改他们的值。这些被捕获的变量被称为非局部变量（non-local variable/upvalues）。 Lua 在执行代码的时候会为代码嵌套一个主函数的外壳。

```lua
print("something")

-- 在执行的时候 Lua 会嵌套一个主函数
function main() 
    print("something")
end
```

从下面的例子可以看出，不同的函数对于捕获的 upvalues 的修改是独立的。我们可以看到 var 和 var2 对 b 和 c 的修改是独立的， 但是全局变量 a 的修改并不独立， 这是因为 a 并不在此函数的词法作用域内， 词法作用域中的非局部变量既不是局部变量， 也不是全局变量； 非局部变量主要应用在嵌套函数和匿名函数中。

```lua
local var = Wrapper()
print(var()) -- a = 2, b = 3, c = 4
print(var()) -- a = 3, b = 4, c = 5

local var2 = Wrapper()
print(var2()) -- a = 4, b = 3, c = 4
print(var2()) -- a = 5, b = 4, c = 5
print(var2()) -- a = 6, b = 5, c = 6
```

## 闭包 (Closure) 的实现原理

理解了词法作用域的定义， 闭包就很好理解了。闭包 (Closure) 其实就是由一个函数和他捕获到的 upvalues 组成的。

![Alt text](/img/lua-image.png)

一个闭包包含了用于处理垃圾回收的头部信息和一个指向原型（prototype）的指针， 这个原型包含了函数的所有静态信息， 包括了编译后的函数代码， 参数的个数， debug 信息以及其他类似的数据。除此之外， 闭包还包含了 0 或者多个 upvalues。

Upvalue 由两种状态：open 和 closed；当一个 upvalue 在被创建的时候处于 open 状态， 函数闭包中的指针指向了这个变量对应的 Lua 堆栈。 当这个变量离开他的作用域被 GC 回收之后，函数闭包会将这个变量的值复制到 upvalue 结构内部， 并且对闭包对应的指针做出调整。这也就是为什么闭包函数可以访问非局部变量的原因。

![Alt text](/img/lua-image-1.png)

### Upvalues 的共享

Upvalue 提供了一种闭包之间共享数据的方法：

```lua
function Wrapper(shared_value)
    local function func1()
        print(shared_value)
    end
    local function func2()
        shared_value = shared_value + 10
    end
    return func1, func2
end

-- func1 和 func2 两个闭包共享 upvalue -> shared_value
-- 即使 shared_value 的状态从 open 转换到 close， 共享依旧存在

local f1, f2 = Wrapper(10)

f1() -- 10
f2()

f1() -- 20
f2()

f1() -- 30
```

从上面的例子我们可以看出函数闭包 f1 和 f2 共用一个 upvalue (也就是多个闭包共享一个 upvalue)。为了实现函数闭包之间的数据共享， Lua 需要确保 Lua 栈中的每一个变量只能有一个 upvalue 指向它。解释器中维护了一个保存栈中所有 open upvalues 的链表，当解释器需要一个变量的 upvalue 时， 他首先会遍历这个链表看这个变量是否已经被生成了一个 upvalue；如果是的话那么直接复用这个 upvalue， 否则就创建一个新的 upvalue 并将其插入链表中正确的位置。

![Alt text](/img/lua-image-2.png)

具体例子可参考下面的代码：当解释器初始化 f2 的时候， 他会在确认变量 a 没有对应的 upvalue 之前会以 f1 -> d -> b 的顺序遍历 3 个 upvalues。

```lua
function foo ()
  local a, b, c, d
  local f1 = function () return d + b end
  local f2 = function () return f1() + a end
  ...
```

如果同一个变量有多个函数闭包的 upvalue 指针指向它， 那么当这个 open upvalue 关闭成为 close upvalue 的时候， 解释器会将栈中的变量值复制到 upvalue 中， 并将这个变量从链表中删除。

![Alt text](/img/lua-image-3.png)

我们可以用以下例子来描述链表是如何确保 upvalue 的共享：

```lua
local a = {} 
local x = 10
for i = 1, 5 do
    local j = i
    a[i] = function () 
        print("x =", x, "j =", j, "x + j =", x + j)
    end
end
x = 20

for _, func in pairs(a) do 
    func()
end
```

程序运行结果如下：

```console
x =     20      j =     1       x + j = 21
x =     20      j =     2       x + j = 22
x =     20      j =     3       x + j = 23
x =     20      j =     4       x + j = 24
x =     20      j =     5       x + j = 25
```

我们可以看到在代码段开头， open upvalue 链表是空的， 当解释器在循环体中创建第一个闭包的时候， 他会为 j 和 x 创建 upvalue，并且插入到 upvalue 链表中； 变量 j 在循环体的末尾走出作用域， 此时对应的 upvalue 就会从链表中被删除； 变量 j 也就变成了 close upvalue。等到下一个循环， 新的闭包可以复用 x 的 upvalue，但是在链表中找不到 j 的 upvalue， 于是再次重新创建 j 的 upvalue， 循环体的末尾 j 的 upvalue 又会从链表中被删除。因此每一次循环中产生的闭包都会共享 upvalue x， 同时单独捕获 upvalue j (也就是说每一个闭包都有一个单独 j 的拷贝)。

## 总结

综上， 闭包在使用的时候不需要产生对象， 也不需要函数名， 同时闭包可以捕获不同的外部变量形成不同的调用环境。这些特性使得闭包的使用十分简洁， 并且闭包可以作为回调函数使用。

闭包的实现原理：

- 闭包在创建的时候会捕获需要的非局部变量， 非局部变量分为 open upvalue 和 close upvalue。 当 Lua 关闭了一个 upvalue， upvalue 指向的值就会被复制到 upvalue 结构内部， 并且指针也会做出相应调整。
- 闭包捕获的非局部变量会被存储在一个全局的 upvalue 链表中， 如果某个非局部变量已经存在与链表中， 那么闭包会直接复用该变量；否则闭包会创建一个新的 open upvalue 并且插入到链表中。此特性使得一个变量在程序中只会有一个对应的 upvalue， 这有利于不同闭包共享 upvalue。
- 当一个 open upvalue 离开作用域的时候， open value 会转换为 close upvalue， 并且解释器会将栈上的值复制到 close upvalue 中， 此时 close value 的值只会被之前捕获了该变量的闭包共享。

## 参考

[理解Lua的闭包机制](https://zhuanlan.zhihu.com/p/494191824)

[深入理解Lua的闭包](https://blog.csdn.net/MaximusZhou/article/details/44280109)

[Closures in Lua](https://www.cs.tufts.edu/~nr/cs257/archive/roberto-ierusalimschy/closures-draft.pdf)