---
title: Python 的优化技巧
date: 2016-01-23
---

上两个月，也就是 2015 年 11 月头的时候个人参与，并为一个高技术创业公司做架构设计。转眼间，在这我写这篇文章的时候已经过去两个半月了。写了不少的代码，也对优化有了大体的认识，因此在空余时间我就来扯淡扯淡。

首先我这几个月写的都是 Python 与 C 代码，而 Python 与 C 之间没有互相呼叫 (calling)，原因放在之后。所以我的用 Python 来做优化例子。

## Python实现

> "Premature Optimization is the root of all evil" —— Donald Knuth

在谈 Python 的技巧前请让我说说 Python 实现的选择。个人选择 PyPy 实现是因为它所用的 JIT (Just-In-Time) 动态编译对 Python 代码而言提速很大。考虑 Numba 与 Pyston 的成熟度，PyPy 是支持度最高的一个 JIT 实现了，经过测试能和 Go 代码媲美。

> 如果选择核心用其他语言代替 Python 内部实现的话，不考虑使用 JIT Python 实现。考虑到很多包没有对 C 扩展进行优化，这样做的话有可能跑下去比 CPython 还来的慢！

解释器和编译器都会按照关键字来处理源码，所以了解编译器底层知识在这里扮演着很重要的角色。

## 善用 generator 
```bash
$ pypy -mtimeit -n 100 "a = (i for i in range(10000))"
100 loops, best of 3: 34.7 usec per loop
$ pypy -mtimeit -n 100 "a = [i for i in range(10000)]"
100 loops, best of 3: 0.441 usec per loop
```
使用 '()' 的是 generator 对象，所需要的内存与列表无关，效率大了许多，但是遇到遍历 for loop 的时候
```bash
$ pypy -mtimeit -n 10 "for x in [i for i in range(10000)]: pass"
10 loops, best of 3: 203 usec per loop
$ pypy -mtimeit -n 10 "for x in (i for i in range(10000)): pass"
10 loops, best of 3: 277 usec per loop
```

> 在 Python 2.x 里面内置的 generator 为 xrange 和 itertools

## 使用关键字会比函数高效
有 ** 和 pow() 的对比：
```bash
$ pypy -mtimeit -n 10000 "c = 2**22"
10000 loops, best of 3: 0.0473 usec per loop
$ pypy -mtimeit -n 10000 "c = pow(2, 22)"
10000 loops, best of 3: 0.185 usec per loop
```
还有 if is ：
``` bash
$ pypy -mtimeit -n 100 "a = range(10000); [i for i in a if i is True]"
100 loops, best of 3: 32.3 usec per loop
$ pypy -mtimeit -n 100 "a = range(10000); [i for i in a if i == True]"
100 loops, best of 3: 36.9 usec per loop
```

## Python 运行函数会比全局代码来的高效
```bash
$ pypy -mtimeit -n 10 \
"
def main():
  a = range(10000); [i for i in a if i == True]
main()
"
10 loops, best of 3: 86.8 usec per loop
$ pypy -mtimeit -n 10 "a = range(10000); [i for i in a if i == True]"
10 loops, best of 3: 87 usec per loop
```
因为在函数中的代码块有着自己的局部作用域，有固定的矩阵大小 (因为考虑你无法动态增加变量进入函数里)，变量的名字存储成索引 (indexes)，从作用域外取得局部变量的流程就是用指针 (pointer) 指向这个列表跳转得到的。

反观全局代码要查询全局变量用的是 bytecode 的 *LOAD_GLOBAL*, 全局变量就是一个 dict() 结构。加上全局变量的查询是比局部变量查询来得耗时，函数会比全局代码快得多。

## 在循环里面缓存全局变量
```python
g = range(10000)
def main(g):
  p = g
  for a in range(10):
    [i for i in p if i == True]
```
在循环里面要查询全局变量每次需要动用一次全局查询，不如在循环里面做一个缓存 _p_, 这样就会动用 bytecode 的 *STORE_FAST* 了。

> 属性的查询是非常耗时的

#### 参考资料
[Python: Faster Way](http://pythonfasterway.cf/) - 这个有一系列的对比，可以得知差别
[Python性能优化的20条建议](http://segmentfault.com/a/1190000000666603)
[Why does Python code runs fast in function](http://stackoverflow.com/questions/11241523/why-does-python-code-run-faster-in-a-function)
