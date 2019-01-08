title: Shellcode 手册
date: 2014-6-13
---

# 介绍
说到 shellcode 大多数的脚本小子都常常会避而远之，有的也只是听闻罢了，私底下就 copy and paste shellcode 那些 *bytes*。
这是安全时代，那个脚本小子没听过 metasploit ？ Metasploit 帮我们解决了 shellcode 的生成，至少我们不必愁被 IDS 拦截我们的 payload 了。
shellcode 真的只是个 exploit payload (漏洞承载) 吗？ 真的只是如其名，帮我们取得个黑漆漆的 shell terminal 吗？ shellcode 是*什么*？

如果还停留在用 metasploit 生成 shellcode 或是在 copy-and-paste 的话，那么你接触的永远都是表面而已。自定义的 shellcode 不止能
删改日志，还能 add 个 admin account 等等你能想象到的东西。

## Shellcode 背景
shellcode 的名字由来是由于早期是用来取得 shell 的一段小代码而来的。可当作一段数据写到其他进程或者网络中去，并执行它的任务。
例如当我们发现到 PROGRAM_A 有缓冲溢出时，我们可以写一段小代码数据写入 ((覆盖)) 该数据然后执行。因此而对数据大小十分挑剔，尽量在最小的空间
完成它的任务。

* shellcode 是个可注入的 16 进制指令
* shellcode 用来注入进入软件的缓冲区，所以对大小挑剔
* shellcode 对大小挑剔，且需要控制内存操作与 register。
* shellcode 要欺骗指针，自行执行。

以上种种的特征都注定了 shellcode 是用 机器码，assembly 汇编来编写的了。

## 高级语言与底层语言的分别
shellcode 控制 register, 必须得是参考处理器规格才对。因为不同的处理器有不同的 machine intruction ((机器指令)) 与 register 规格。
所以在语言上有很大作用，语言的分别很大，但是其中的道理不变。

讲到这里有点乱了，我们拿 C 语言来调用函数来看看。

```c
/* helloworld.c */
#include <stdio.h>
int main() {
  printf ("hello,world\n");
  return 0;
}
```

编译了之后我们再来看看他的系统调用与指令长短。

```bash
$ gcc helloworld.c
$ strace ./a.out
execve("./a.out", ["./a.out"], [/* 27 vars */]) = 0
brk(0)                                  = 0x808410
open("/opt/gnome2/lib/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/opt/gnome2/lib", {st_mode=S_IFDIR|0755, st_size=20480, ...}) = 0
open("/etc/ld.so.cache", O_RDONLY)      = 3
fstat64(3, {st_mode=S_IFREG|0644, st_size=65928, ...}) = 0
old_mmap(NULL, 65928, PROT_READ, MAP_PRIVATE, 3, 0) = 0xbf596000
close(3)                                = 0
open("/lib/tls/libc.so.6", O_RDONLY)    = 3
read(3, "\177ELF\1\1\1\0\0\0\0\0\0\0\0\0\3\0\3\0\1\0\0\0`ht\000"..., 512) = 512
....................................
mmap2(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xbf7ef5000
write(1, "hello,world!\n", 12hello,world!
)          = 12
exit_group(0)                         = ?
Process 478910 detached
```

```bash
08048370 <main>:
 8048370:       55               push   %ebp
 8048371:       89 e5            mov    %esp,%ebp
 8048373:       83 ec 08         sub    $0x8,%esp
 8048376:       83 e4 f0         and    $0xfffffff0,%esp
 8048379:       b8 00 00 00 00   mov    $0x0,%eax
 804837e:       29 c4            sub    %eax,%esp
 8048380:       83 ec 04         sub    $0x4,%esp
 8048383:       6a 0e            push   $0xe
 8048385:       68 9c 84 04 08   push   $0x804849c
 804838a:       6a 01            push   $0x1
 804838c:       e8 07 ff ff ff   call   8048298 <write@plt>
 8048391:       83 c4 10         add    $0x10,%esp
 8048394:       b8 00 00 00 00   mov    $0x0,%eax
 8048399:       c9               leave  
 804839a:       c3               ret    
 804839b:       90               nop    
 804839c:       90               nop    
 804839d:       90               nop    
 804839e:       90               nop    
 804839f:       90               nop
```

*strace* 命令检查了软件的系统调用，而 *objdump* 检查软件的汇编命令。

我们好像发现了什么。编译了的 C 代码貌似*不止是*打印个"hello,world"那么简单，内部经过编译器编译，优化后，会增加许多不必要
((也不是说不必要，是编译器自动铺垫内存，内存映射一番的过程。所以说电脑始终不及人脑)) 的系统调用，函数。这 printf() 其实也是一系列的系统调用组成。其中包括 write()。

## 系统调用
系统调用是代码直接进入保护模式运行，意味着我们将会将一堆的指令和配置交给我们的操作系统来运行。而我们也需要更高的权限，对
内核通信，这一串的调用就是 System Call ((也被叫成是 syscall))。

Unix manual page 有相当详细的 syscall 说明。我们可以 man 一下看看。*man 2 write*
我们可以得到 prototype 和相关的讯息。

```c
  ssize_t write (int fd, const void *buf, size_t count);
```

fd 就是 *file descriptor*, 这是 UNIX 的 POSIX 用来标记读写，access，网络socket 的标记。((例如整数 0 是 STDIN, 1 是STDOUT, 2 是 STDERR.... 也就是说 0，1，2 分别表示 input, output, error))
这些都能在 /usr/include/unistd.h 文件参考得到。

第二个配置就是将指针 pass 到我们的 string 命令串, 第三个就是标记我们 string 的长度。

我们就来重写我们的 helloworld 使它更为简短，小巧

```c
/* helloworld_write.c */
int main() {
  write (1,"hello,world!\n", 13);
  return 0;
}
```

看是不是更小了？

```bash
gcc helloworld_write.c
./a.out
hello,world!
```

## linux 内核下的 syscall
Linux 下的 libc 自动链接到各类的头文件，其中有不少都是与 linux 内核通信的函数。为了简化开发者的开发程序，就将每个 syscall 定义成一个整数。可以在 */usr/include/asm* 下参考。

32 bit 系统的就参考 */usr/include/asm/unistd_32.h*
64 bit 系统就参考 */usr/include/asm/unistd_64.h*

就拿 unistd_32.h 来看吧。

```c
#ifndef _ASM_X86_UNISTD_H_
#define _ASM_X86_UNISTD_H_ 1

#define __NR_restart_syscall 0
#define __NR_exit 1
#define __NR_fork 2
#define __NR_read 3
#define __NR_write 4
#define __NR_open 5

...
```

从头文件我们可以得知 write 被定义成 syscall 4 了。所以只需要在 register 上推栈一个 "4" 就算定义 write 了。
Linux 在系统调用这一点上与 Unix 不同，使用的是 fastcall。因此只需要在 EAX 上推一个 syscall number，依次在 EBX,ECX,EDX,ESU
EDI,EPB里，那么我们就能容纳六个的配置了。超过六个的我们就用 EAX 所定义的数据结构来传递。

好吧，我已经铺垫了相关的 syscall 知识了。那么我就开始汇编。

## 汇编是何物？

汇编由简写码 (mnemonic code) 和操作码 (opcode) 组成。专用与一种处理器架构，可以最大幅度的控制我们的 register 和内存操作。
当然，近代的编译器都是用汇编器来生成我们的目标文件。汇编语言要通过汇编器才能编译成目标文件。目标文件才是电脑读得懂的语言，通过加载器 (loader) 来加载进内存，由 pointer 所读取。

编写汇编是不容易的事，但是要掌握它却莫名的容易。现在我们将会选择 NASM 来做汇编实验。当然，intel 语法，AT&T 太复杂了。

```asm
section .data
msg db "hello, world!", 0x0a

section .text
global _start

_start:
; 这是我们的注释...
; write (1, msg, 14)
mov eax, 4
mov ebx, 1
mov ecx, msg
mov edx, 14
int 80h

; exit(0)
mov eax, 1
mov ebx, 0
int 80h
```

*section* 语句分区代码在编译了后的内存段
*db* 是在 .data 内部压栈数据，在这里我们弄 "hello, world!" 和 newline char "0x0a“ (( 0x0a 就是 '\n' 的 16 进制编码 ))

.text 存储我们的代码，是不可改写的。之后我们就定义入口函数为 *_start* (( _start 就是linux ELF 的默认链接函数 ))
*mov* opcode 代表 move 之意， intel 语法是继 opcode 之后是 dest, src。意思是说我们*将 4 mov 到 eax register 里*。

*int* 是内核呼叫, interrupt 的简写，一旦读取，内核就会读取之后的序号然后做出指令。 (( 80h 就是 0x80 的意思，那 'h' 是 16 进制的标记 )) 0x80 就是叫 linux 内核读取 register 的数据然后做出命令。

我们如此编译…

```bash
nasm -f elf helloworld.asm
ld helloworld.0
./a.out
Hello, world!
```

我们的小小软件运行得到。但是它还不是 shellcode。为什么？

* Shellcode 是不能链接其他的函数库的。
* Shellcode 是独立代码位置的。

## 栈的艺术
![Stack Push and Pop]({{ site.url }}/images/data_stack.svg)
stack ((栈))是数据存储的其中一种方式。是很特别的是…他是向下生长的。我们来研究研究。

* 栈由编译器自动分配释放，存放函数的参数值，局部变量的值等。
* 栈由高位址到低位址增长
* 栈就像叠加盘子那样，最先放置的盘子在底层，最后一个被取出 ((放置为 push, 取出为 pop))

*push* [source] 压栈配置到栈段
*pop* [dest] 将栈取出，然后存储在目的地

*call* [location] 呼叫函数，将下一个位址压栈，然后 EIP 跳转到目的函数的位址然后执行。直到 *ret* 然后 pop 先前的位址继续运行，就像函数运行完后回到先前函数那样。

简单的实验讲解的比我好。

```asm
[BITS 32]
section .text
global _start

_start:
  push byte 0x0a
  push "rld!"
  push "o wo"
  push "hell"

  push byte 0x04
  pop eax
  xor ebx, ebx
  mov ecx, esp
  push byte 0xd
  pop edx
  int 80h

  ; exit(0)
  xor eax, eax
  mov eax, 1
  mov ebx, 0
  int 80h
```

我们依次将 4 bytes 的字符压栈然后传递 esp 给我们的 ecx 供做 string 来使。由于栈是向下增长，我们就将"helloworld" 调转来压。然后传递栈的位址 ((就是 esp 所 hold 着的))

熟知栈，堆这些结构能使我们更加理解对电脑的理解，对高级语言编程也有一定的帮助。

## 独立位址，这是什么？
我们的 shellcode 是常常注入在一些有限的空间，除此之外，我们还需要独立位址，意味这不管在什么位址都要保证我们的 shellcode 能够运行。

有个经典的 hacks 能使我们的 shellcode 免于受到 EIP 跳转的影响，那就是利用 stack 的特性。

![Stack JMP]({{ site.url }}/images/jmp_stack.png)

在代码的开头放一个 *call* 这样在跳转的过程中会将下一个位址压栈，然后跳到底部的函数 ((也就是我们的编写的 footer))，然后再来 *call* 上面的函数。这样我们不但弄成了一个独立的 shellcode 身体，还可以利用相对地址来存储我们的代码。

```asm
  call goto

shellcode:
  pop esi
  ............
  ............
  ............

goto:
  call shellcode
  db "/bin/sh"
```
这样跳转两次是不是非常有创意呢？

* 第一行 call 跳转到 goto 将下个位址压栈
* goto 函数接着跳转到 shellcode 将 "/bin/sh" 压栈
* shellcode 函数的第一行就 pop 将 "/bin/sh" 存储到 esi 去
* 继续我们的 shellcode 内容

## 我们来下手写个 Shellcode 吧？
shellcode 就是要来注入才是真用途，我们就来写个 shell spawner 吧。
写 shellcode 我们先从 C 高级语言起。然后再从编译后的目标文件开始起。
我们这次用的是 *execve* 来运行 shell。
参考 *man 2 execve* 我们得到

```c
int execve (const char *filename, char *const argv[], char *const envp[])
```

得到之后我们着手写吧！
```c
/* execve.c */
int main() {
  char *cmd[1];

  cmd[0] = "/bin/sh";
  cmd[1] = 0;

  execve (cmd [0], cmd, NULL);
}
```

我们在上面的 C 语言用 execve 来调用函数 */bin/sh*, 由于我们没有任何的 keyvalue 和配置，就放个 NULL。
编译了后我们用 gdb 来看看

```bash
$ gcc -static execve.c -o execve
$ ./execve
sh-4.2$ exit
$ objdump -D | grep -A25 '<main>'
08048e94 <main>:
 8048e94:	55                   	push   %ebp
 8048e95:	89 e5                	mov    %esp,%ebp
 8048e97:	83 e4 f0             	and    $0xfffffff0,%esp
 8048e9a:	83 ec 20             	sub    $0x20,%esp
 8048e9d:	c7 44 24 1c 88 7a 0c 	movl   $0x80c7a88,0x1c(%esp)
 8048ea4:	08
 8048ea5:	c7 44 24 20 00 00 00 	movl   $0x0,0x20(%esp)
 8048eac:	00
 8048ead:	8b 44 24 1c          	mov    0x1c(%esp),%eax
 8048eb1:	c7 44 24 08 00 00 00 	movl   $0x0,0x8(%esp)
 8048eb8:	00
 8048eb9:	8d 54 24 1c          	lea    0x1c(%esp),%edx
 8048ebd:	89 54 24 04          	mov    %edx,0x4(%esp)
 8048ec1:	89 04 24             	mov    %eax,(%esp)
 8048ec4:	e8 07 a8 00 00       	call   80536d0 <__execve>
 8048ec9:	c9                   	leave  
 8048eca:	c3                   	ret    
 8048ecb:	66 90                	xchg   %ax,%ax
 8048ecd:	66 90                	xchg   %ax,%ax
 8048ecf:	90                   	nop
08048ed0 <__libc_start_main>:
 8048ed0:	55                   	push   %ebp
 8048ed1:	b8 00 00 00 00       	mov    $0x0,%eax
 8048ed6:	57                   	push   %edi
```
我们用 static 来编译 C 代码，原因就是我们为了避免动态库链接。
编译了后我们用 objdump 浏览其中的 assembly。我们得到了它的机器码，我们又离 shellcode 进了一步。
前提是我们要了解软件的运作过程，从 assembly 中。

*第 5 行* string '/bin/sh' 的位址被 mov 去 esp

> 8048e9d:	c7 44 24 1c 88 7a 0c 	movl   $0x80c7a88,0x1c(%esp)

*第 7 行* 然后我们的 string 被 mov 到了 eax 去，原因是要将 NULL 压栈需要清空 esp

> 8048ead:	8b 44 24 1c          	mov    0x1c(%esp),%eax

*第 6 行* 我们的 NULL 被 mov 去 esp

> 8048ea5:	c7 44 24 20 00 00 00 	movl   $0x0,0x20(%esp)

*第 11 行* 我们 call 了 *<__execve>* ，往下寻找…

```bash
080536d0 <__execve>:
 80536d0:	53                   	push   %ebx
 80536d1:	8b 54 24 10          	mov    0x10(%esp),%edx
 80536d5:	8b 4c 24 0c          	mov    0xc(%esp),%ecx
 80536d9:	8b 5c 24 08          	mov    0x8(%esp),%ebx
 80536dd:	b8 0b 00 00 00       	mov    $0xb,%eax
 80536e2: cd 80                 int    0x80
 80536e8: 90                    nop
 80536ed:	3d 00 f0 ff ff       	cmp    $0xfffff000,%eax
 80536ef:	5b                   	pop    %ebx
 80536f0:	c3                   	ret    
 80536f1:	c7 c2 e8 ff ff ff    	mov    $0xffffffe8,%edx
 80536f7:	f7 d8                	neg    %eax
 80536f9:	65 89 02             	mov    %eax,%gs:(%edx)
 80536fc:	83 c8 ff             	or     $0xffffffff,%eax
 80536ff:	5b                   	pop    %ebx
 8053700:	c3                   	ret    
 8053701:	66 90                	xchg   %ax,%ax
 8053703:	90                   	nop
 8053704:	66 90                	xchg   %ax,%ax
 8053706:	66 90                	xchg   %ax,%ax
 8053708:	66 90                	xchg   %ax,%ax
 805370a:	66 90                	xchg   %ax,%ax
 805370c:	66 90                	xchg   %ax,%ax
 805370e:	66 90                	xchg   %ax,%ax
08053710 <__exit_thread>:
 8053710:	89 da                	mov    %ebx,%edx
 8053712:	8b 5c 24 04          	mov    0x4(%esp),%ebx
 8053716:	b8 01 00 00 00       	mov    $0x1,%eax
 805371b:	ff 15 a8 45 0f 08    	call   *0x80f45a8
```

来到了 *execve* 我们再来看 assembly 如何 system calling 吧！

将我们的配置的栈传到 ecx ((其实我们没有任何配置。配置为 NULL)) 这样做只是将 pointer 传到 array 里，而 array 为 'NULL'

> 80536d5:	8b 4c 24 0c          	mov    0xc(%esp),%ecx

将我们之前压栈的 '/bin/sh' mov 到 ebx

> 80536d9:	8b 5c 24 08          	mov    0x8(%esp),%ebx

execve() 的 syscall 序号为 11, 把 11 ((0xb))放置到 eax

> 80536dd:	b8 0b 00 00 00       	mov    $0xb,%eax

*第 6 行* int 80 内核调用。

接下来我们就 NASM 构架出我们的 shellcode 来，通过反编译出来的代码 :-)
我们就来

```asm
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
```

解释下相关的语法。
*lea* ((Load Effective Address)) 与 mov 不一样他是将数据写入该位址偏移 ((memory offset))，就像 C 语言的 address-of 那样。
我们通过它来定位我们的数据写入口，stack 位置，还有许多许多 VMA。位址偏移就用 *[]* 来表示。

我们将 eax ((内含'0')) 分别写入 *esi 后 7 byte 和 12 byte 的位址* ((为的是在 /bin/sh 后加上 NULL, 还有为envp 做 NULL))然后将我们的 array 地址写入 esi + 8。

编译看，
```bash
nasm -f elf shellcode.asm
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
```

好吧写完了NASM，编译出了机器码。那么我们的 shellcode 为

```c
\xeb\x18\x5e\xb8\x00\x00\x00\x00\x89\x46\x07\x89\x76\x08\x89\x46\x0c\xb0\x0b\x00\x00\x00
\x8d\x1e\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\xe8\xe3\xff\xff\xff\x2f\x62\x69\x6e\x2f\x73\x68
```

完成了。你发觉到错误吗？

## 编写个 NULL-free 的 shellcode
shellcoder 们，玩 shellcode 记得先测试啊！

```c
#include <stdio.h>
#include <string.h>

char shellcode[] = " /*[这是shellcode]*/ ";

int main() {
  void *exec = shellcode;
  memcpy (exec, shellcode, strlen(shellcode));
  printf ("[*] Bad shellcode if occur segmentation fault\n");
  printf ("[+] Executing!\n");
  return 0;
}
```

把我们的 shellcode 放进去，
```bash
gcc asm-tester.c
./a.out
[*] Bad shellcode if occur segmentation fault
[+] Executing!
 11978 segmentation fault ./a.out
```

出错误了。原因是什么？
我们下回再来探讨，加工我们的 shellcode 使之更完整。

< 篇1 完>
