---
title: Grep 常见命令选项
date: 2023-12-09 14:07:01
categories: [Linux]
tags: Tool
---

## grep 命令常用选项

- `-i`: 忽略大小写进行匹配, 然后显示符合的行

```console
$ cat test.txt
This is a random text file* 
AAA()*
bbb
aaa
BBB
aabbcc
ccc**&
CCC
hello world?/*

$ grep -i aaa test.txt 
AAA()*
aaa
```

- `-v`: 反向查找， 只打印不匹配的行 (下面例子打印除了 aaa 和 AAA 之外的所有行)

```console
$ grep -v -i aaa test.txt
This is a random text file* 
bbb
BBB
aabbcc
ccc**&
CCC
hello world?/*
```

- `-n`: 显示匹配的行号

```console
$ grep -n aaa test.txt
4:aaa
```

- `-c (--count)`: 只打印匹配的行数 (下面例子只有 aaa 和 AAA 符合条件)

```console
$ grep -c -i aaa test.txt   
2
```

- `-n`: 显示匹配的行号

```console
$ grep -n -i aaa test.txt 
2:AAA()*
4:aaa
```

- `-F`: -F 会将提供的模式当作是固定的字符串 (而非正则表达式); 如果你搜索的字符中包含有用于正则表达式的特殊字符, 那可以添加此参数
- 在下面例子中, *, ?, 以及 / 都会被当成普通字符看待

```console
$ grep -F "hello world?/*" test.txt 
hello world?/*

$ grep -F "*" test.txt              
This is a random text file* 
AAA()*
ccc**&
hello world?/*
```

- `-f (--file)`: -F 参数允许你从文件中读取模式, 而非在命令行中直接指定他们
- 下面例子中 search_list.txt 中的每一行都会被当成模式 在 test.txt 中搜索

```console
$ cat search_list.txt 
ccc
hello

$ grep -f search_list.txt test.txt 
ccc**&
hello world?/*
```

- `-G (--basic-regexp)`: 表示模式会被当成正则表达式; 当然 grep 默认的就是使用正则表达式, 所以 -G 一般都是隐含的

- `-A + 显示行数`: 显示符合模式的那一行, 外加该行之后的内容

```console
$ grep -A2 aaa test.txt 
aaa
BBB
aabbcc

$ grep -A2 -i aaa test.txt 
AAA()*
bbb
aaa
BBB
aabbcc
```

- `-B + 显示行数`: 显示符合模式的那一行, 外加该行之前的内容

```console
$ grep -B2 aaa test.txt 
AAA()*
bbb
aaa
```

- `-C + 显示行数`: 显示符合模式的那一行, 外加该行前后的内容

```console
$ grep -C1 ccc test.txt  
aabbcc
ccc**&
CCC
```

- `-m`: 在找到指定数量的匹配行之后停止读取文件

```console
$ cat test2.txt 
aaa&&
ikvbc
haven65
link65

aaa**&
another one 
find

aaa()*
nil

// 找到两行匹配 aaa 模式直接结束搜索
$ grep aaa -m2 test2.txt 
aaa&&
aaa**&
```

### NOTE: 利用 grep 只打印最后一部分符合条件的信息

```console
$ grep aaa -A2 test2.txt
aaa&&
ikvbc
haven65
--
aaa**&
another one 
find
--
aaa()*
nil
```

- 如果我想要打印最后一部分 aaa 的内容呢
  - 可以使用 grep 与 tac 命令将文件的内容以行为单位反向输出 (e.g. 最后一行变成第一行)

```console
$ tac test2.txt 
nil
aaa()*

find
another one 
aaa**&

link65
haven65
ikvbc
aaa&&

// 此时的结果依然是翻转的, 需要再次调用一个 tac 进行恢复
$ tac test2.txt | grep -m1 -B2 aaa
nil
aaa()*

$ tac test2.txt | grep -m1 -B2 aaa | tac
aaa()*
nil

```

- 但是在使用 grep + tac 的时候必须要直到最后一部分匹配的内容的行数; 因为该方法不具有通用性
- 可以考虑使用 `awk`, `sed` 等其他文本处理工具来进行提取

## grep + tail/head

- 如果你只需要打印 grep 后结果的前面几行或者是后面几行, 可以使用 `head` 或者 `tail`
- `head` 和 `tail` 的默认打印长度都是 10 行

## grep + head

- `head -n num`: 显示前面 num 行的结果

```console
$ grep aaa -A2 test2.txt
aaa&&
ikvbc
haven65
--
aaa**&
another one 
find
--
aaa()*
nil

$ grep aaa -A2 test2.txt | head -n3
aaa&&
ikvbc
haven65
```

- `head -n -num`: 显示除了最后 num 行外的所有结果

```console
// 显示除了最后 3 行之外的所有内容
$ grep aaa -A2 test2.txt | head -n -3
aaa&&
ikvbc
haven65
--
aaa**&
another one 
find
```

## grep + tail

- `tail -n num`: 显示后面 num 行的结果

```console
$ grep aaa -A2 test2.txt
aaa&&
ikvbc
haven65
--
aaa**&
another one 
find
--
aaa()*
nil

$ grep aaa -A2 test2.txt | tail -n3
--
aaa()*
nil
```

- `tail -n +num`: 显示从第 num 行开始的结果

```console
// 显示从第 3 行开始的结果
$ grep aaa -A2 test2.txt | tail -n +3
haven65
--
aaa**&
another one 
find
--
aaa()*
nil
```

## grep 搭配正则表达式

下面列举了一些常用的正则表达式

- `^{word}`: 匹配所有以 {word} 开头的行

```console
$ cat test3.txt 
Line 1 - aa****()
line 2 - abc&&%%()
link
linnk - random test ^%&*
Linnnk - random test2 #
linnnnk
3ink
love the feeling, end
love C++, end
Love coding, end
love training, end

// 匹配以 line 开头的所有行
$ grep '^line' test3.txt                                                         
line 2 - abc&&%%()
```

- `{word}$`: 匹配所有以 {word} 结尾的行

```console
// 匹配以 nk 为结尾的所有行
$ grep 'nk$' test3.txt                                                                       
link
linnnnk
3ink
```

- `.`: 匹配 `一个` 除了非换行符的任意字符

```console
// 匹配 li 后接一个任意字符, 然后是 k
$ grep 'li.k' test3.txt                                                           
link

// 匹配 li 后接 2 个任意字符, 然后是 k
$ grep 'li..k' test3.txt  
linnk - random test ^%&*
```

- `*`: 匹配 `0 个` 或者 `多个` 先前字符

```console
// 匹配 li 后接一个或者多个字符 n, 然后是 k
$ grep 'lin*k' test3.txt               
link
linnk - random test ^%&*
linnnnk
```

- `{word}\{a\}`: 重复字符 `word` a 次

```console
// 匹配包含 2 个 '+' 字符的行
$ grep '+\{2\}' test3.txt                                                     
love C++, end
```

- `{word}\{a,\}`: 重复字符 `word` 至少a 次

```console
// 匹配 lin, 且至少包含 2 个 'n' 字符的行, 后接 k
$ grep 'lin\{2,\}k' test3.txt 
linnk - random test ^%&*
linnnnk
```

- `{word}\{a,b\}`: 重复字符 `word` a 到 b 次

```console
// 中间不能有空格, 否则会报错
$ grep 'n\{2, 3\}' test3.txt                                                       
grep: Invalid content of \{\}

// 匹配包含 2 - 3 个 'n' 字符的行
$ grep 'n\{2,3\}' test3.txt  
linnk - random test ^%&*
Linnnk - random test2 #
linnnnk
```

- `[]`: 匹配一个在指定范围内的字符

```console
$ grep '[Ll]in' test3.txt                                                              
Line 1 - aa****()
line 2 - abc&&%%()
link
linnk - random test ^%&*
Linnnk - random test2 #
linnnnk
love the feeling, end
```

- `[^]`: 匹配一个不在在指定范围内的字符

```console
// 匹配一个不是小写字母, 且后接 in 字符的行
$ grep '[^a-z]in' test3.txt  
Line 1 - aa****()
Linnnk - random test2 #
3ink
```

- `\w`: 匹配 `一个` 文字/数字字符

```console
$ grep '\wink' test3.txt 
link
3ink
```

- `\w`: 匹配 `一个` 非文字/数字字符, 如空格或者是句号等

```console
// 匹配最后以一个或者多个非文字字符结尾的行
$ grep '\W\+$' test3.txt
Line 1 - aa****()
line 2 - abc&&%%()
linnk - random test ^%&*
Linnnk - random test2 #

// 匹配包含 '@' 字符后面跟着任何非文字字符的行
$ grep '@\W'  test3.txt 
love@#
```

- `\<`: 左单词边界
- `\>`: 右单词边界

```console
$ cat test4.txt 
testing
For the test A
experiment test
tttest A B C

// 匹配包含以 test 作为单词开头的行
$ grep '\<test' test4.txt 
testing
For the test A
experiment test
```

```console
// 匹配包含以 test 作为单词结尾的行
$ grep 'test\>' test4.txt 
For the test A
experiment test
tttest A B C

$ grep '\<test\>' test4.txt 
For the test A
experiment test
```

- `-E`: 使用扩展正则表达式 (Extended Regular Expressions, ERE), 他可以支持 +, ?, |, () 等无需转义的元字符

```console
$ grep -E 'For|exper' test4.txt 
For the test A
experiment test

// 如果不使用 -E 的话就需要在这些元字符面前加上 \
$ grep 'For\|exper' test4.txt 
For the test A
experiment test
```

- `-e`: 实现多个选项之间的逻辑 OR 关系

```console
// 找到以 ing 字符结尾或者是以 ex 字符开头的行
$ grep -e 'ing$' -e '^ex' test4.txt 
testing
experiment test
```

The end.
