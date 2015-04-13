#结论
1. entry.S 77行初始化栈
2. 栈的位置是0xf0108000-0xf0110000
3. 设置栈的方法是在kernel的数据段预留32KB空间(entry.S 92行)
4. 栈顶的初始化位置是0xf0110000

#分析
bootloader最后一条语句进入内核，进入内核后的几件事情顺序如下：

1. 开启分页(entry.S 62行)
2. 设置栈指针(entry.S 77行)
3. 调用i386_init(entry.S 80行)

设置栈指针的代码如下：
```
    # Set the stack pointer
    movl    $(bootstacktop),%esp
```
可以从kern/kernel文件中找出符号bootstacktop的位置：
```
zzzz@ubuntu:~/workspace/github/xv6-note/Lab1/2014-jos-Lab1$ objdump -D obj/kern/kernel | grep -3 bootstacktop
f0108000 <bootstack>:
        ...

f0110000 <bootstacktop>:
f0110000:       01 10                   add    %edx,(%eax)
f0110002:       11 00                   adc    %eax,(%eax)
        ...
```
因为没有设置CS，因此CS还是指向之前在bootloader阶段设置的数据段描述符，该描述符指定的基地址为0x0，因此%esp的值就是栈顶的位置。因此栈顶的位置就是0xf0110000。
堆栈的大小由下面的指令设置(entry.S 92行):
```
.data
###################################################################
# boot stack
###################################################################
    .p2align    PGSHIFT     # force page alignment
    .globl      bootstack
bootstack:
    .space      KSTKSIZE
    .globl      bootstacktop   
bootstacktop:
```
可以看出，栈的设置方法是在数据段中预留出一些空间来用作栈空间。memlayout.h 97行定义的栈的大小:
```
#define PGSIZE      4096        // bytes mapped by a page
...
#define KSTKSIZE    (8*PGSIZE)          // size of a kernel stack
```
因此栈大小为32KB，栈的位置为0xf0108000-0xf0110000

#gdb验证
调用call i386_init函数的位置为0xf0100039，在该位置设置断点，查看寄存器内容：
```
(gdb) b *0xf0100039
Breakpoint 1 at 0xf0100039: file kern/entry.S, line 80.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0100039 <relocated+10>:   call   0xf010009d <i386_init>

Breakpoint 1, relocated () at kern/entry.S:80
80              call    i386_init
(gdb) info r
eax            0xf010002f       -267386833
ecx            0x0      0
edx            0x9d     157
ebx            0x10094  65684
esp            0xf0110000       0xf0110000 <entry_pgdir>
```
可以看到，调用函数i386_init之前，栈的位置确实是在0xff010000(%esp)。查看栈的内容如下：
```
(gdb) x /8xw 0xf010fff0
0xf010fff0:     0x00000000      0x00000000      0x00000000      0x00000000
0xf0110000 <entry_pgdir>:       0x00111021      0x00000000      0x00000000      0x0000000
```
第一个压入栈的数据应该是call i386_init的返回地址，即这条指令的下一条指令的地址，stepi单步之后再查看：
```
(gdb) x /8xw 0xf010fff0
0xf010fff0:     0x00000000      0x00000000      0x00000000      0xf010003e
0xf0110000 <entry_pgdir>:       0x00111021      0x00000000      0x00000000      0x00000000
(gdb) x /2i 0xf0100039
   0xf0100039 <relocated+10>:   call   0xf010009d <i386_init>
   0xf010003e <spin>:   jmp    0xf010003e <spin>
```
再次查看栈中的内容验证了之前的猜测。

