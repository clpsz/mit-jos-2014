#结论
查看kern/kernel.asm的反汇编文件，如下：
```
    test_backtrace(5);
f0100138:   c7 04 24 05 00 00 00    movl   $0x5,(%esp)
f010013f:   e8 fc fe ff ff          call   f0100040 <test_backtrace>
```
test_backtrace的代码在0xf0100138地址处，在该处断点调试：
```
(gdb) b *0xf0100138
Breakpoint 1 at 0xf0100138: file kern/init.c, line 51.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0100138 <i386_init+155>:  movl   $0x5,(%esp)

Breakpoint 1, i386_init () at kern/init.c:51
51              test_backtrace(5);
(gdb) info r
eax            0x0      0
ecx            0x3d4    980
edx            0x0      0
ebx            0x10094  65684
esp            0xf010ffd0       0xf010ffd0
```
在进入test_backtrace函数之前，栈顶位置为0xf010ffd0。

通过源代码分析可以得知，递归的基本条件是x=0，当x=0时会执行到mon_backtrace，调用mon_backtrace函数的代码地址是0xf0100082，在此处设置断点并查看栈的内容如下：
```
(gdb) x /52xw $esp
0xf010ff10:     0x00000000      0x00000000      0x00000000      0x00000000
0xf010ff20:     0xf010093b      0x00000001      0xf010ff48      0xf0100069
0xf010ff30:     0x00000000      0x00000001      0xf010ff68      0x00000000
0xf010ff40:     0xf010093b      0x00000002      0xf010ff68      0xf0100069
0xf010ff50:     0x00000001      0x00000002      0xf010ff88      0x00000000
0xf010ff60:     0xf010093b      0x00000003      0xf010ff88      0xf0100069
0xf010ff70:     0x00000002      0x00000003      0xf010ffa8      0x00000000
0xf010ff80:     0xf010093b      0x00000004      0xf010ffa8      0xf0100069
0xf010ff90:     0x00000003      0x00000004      0x00000000      0x00000000
0xf010ffa0:     0x00000000      0x00000005      0xf010ffc8      0xf0100069
0xf010ffb0:     0x00000004      0x00000005      0x00000000      0x00010094
0xf010ffc0:     0x00010094      0x00010094      0xf010fff8      0xf0100144
0xf010ffd0:     0x00000005      0x0000e110      0xf010ffec      0x00000004
```
因为最后一次test_backtrace返回前，所有的cprintf函数都已经返回，所以栈中看不到cprintf栈的痕迹。从下往上看是函数调用的顺序，可以看出来还是很有规律性的，先来猜测一下，稍后分析：

1. 第隔两行就出现的5,4,3,2,1,0肯定是传入到test_backtrace的参数，按从下到上的函数调用的顺序依次是5，4，3，2，1，0
2. 最后一列出现多次的0xf0100069则是函数递归调用的test_backtrace的返回地址，最右下的那个0xf0100144则是i386_init中test_backtrace函数调用的返回地址
3. 最下面一条的第一个数字是对应的内存地址是0xf010ffd0，内存内容是0x00000005，是第一次调用test_backtrace时传入的参数，是分析的起点

其它的好像看不出来什么有意思的东西了，现在在分析下，直接看反汇编的代码(为清晰起见，将源代码与反汇编后的代码分开)：
```
test_backtrace(int x)
{
        cprintf("entering test_backtrace %d\n", x);
        if (x > 0)
                test_backtrace(x-1);
        else
                mon_backtrace(0, 0, 0);
        cprintf("leaving test_backtrace %d\n", x);
}

f0100040:       55                      push   %ebp                             ;压入调用函数的%ebp
f0100041:       89 e5                   mov    %esp,%ebp                        ;将当前%esp存到%ebp中，作为栈帧
f0100043:       53                      push   %ebx                             ;保存%ebx当前值，防止寄存器状态被破坏
f0100044:       83 ec 14                sub    $0x14,%esp                       ;开辟20字节栈空间用于本函数内使用
f0100047:       8b 5d 08                mov    0x8(%ebp),%ebx                   ;取出调用函数传入的第一个参数
f010004a:       89 5c 24 04             mov    %ebx,0x4(%esp)                   ;压入cprintf的最后一个参数，x的值
f010004e:       c7 04 24 e0 19 10 f0    movl   $0xf01019e0,(%esp)               ;压入cprintf的倒数第二个参数，指向格式化字符串"entering test_backtrace %d\n"
f0100055:       e8 27 09 00 00          call   f0100981 <cprintf>               ;调用cprintf函数，打印entering test_backtrace (x)
f010005a:       85 db                   test   %ebx,%ebx                        ;测试是否小于0
f010005c:       7e 0d                   jle    f010006b <test_backtrace+0x2b>   ;如果小于0，则结束递归，跳转到0xf010006b处执行
f010005e:       8d 43 ff                lea    -0x1(%ebx),%eax                  ;如果不小于0，则将x的值减1，复制到栈上
f0100061:       89 04 24                mov    %eax,(%esp)                      ;接上一行
f0100064:       e8 d7 ff ff ff          call   f0100040 <test_backtrace>        ;递归调用test_backtrace
f0100069:       eb 1c                   jmp    f0100087 <test_backtrace+0x47>   ;跳转到f0100087执行
f010006b:       c7 44 24 08 00 00 00    movl   $0x0,0x8(%esp)                   ;如果x小于等于0，则跳到这里执行，压入mon_backtrace的最后一个参数
f0100072:       00 
f0100073:       c7 44 24 04 00 00 00    movl   $0x0,0x4(%esp)                   ;压入mon_backtrace的倒数第二个参数
f010007a:       00 
f010007b:       c7 04 24 00 00 00 00    movl   $0x0,(%esp)                      ;压入mon_backtrace的倒数第三个参数
f0100082:       e8 68 07 00 00          call   f01007ef <mon_backtrace>         ;调用mon_backtrace，这是这个练习需要实现的函数
f0100087:       89 5c 24 04             mov    %ebx,0x4(%esp)                   ;压入cprintf的最后一个参数，x的值
f010008b:       c7 04 24 fc 19 10 f0    movl   $0xf01019fc,(%esp)               ;压入cprintf的倒数第二个参数，指向格式化字符串"leaving test_backtrace %d\n"
f0100092:       e8 ea 08 00 00          call   f0100981 <cprintf>               ;调用cprintf函数，打印leaving test_backtrace (x)
f0100097:       83 c4 14                add    $0x14,%esp                       ;回收开辟的栈空间
f010009a:       5b                      pop    %ebx                             ;恢复寄存器%ebx的值
f010009b:       5d                      pop    %ebp                             ;恢复寄存器%ebp的值
f010009c:       c3                      ret                                     ;函数返回
```
一个栈帧(stack frame)的大小计算如下：
1. 在执行call test_backtrace时有一个副作用就是压入这条指令下一条指令的地址，压入4字节返回地址
2. push %ebp，将上一个栈帧的地址压入，增加4字节
3. push %ebx，保存ebx寄存器的值，增加4字节
4. sub  $0x14, %esp，开辟20字节的栈空间，后面的函数调用传参直接操作这个栈空间中的数，而不是用pu
sh的方式压入栈中

加起来一共是32字节，也就是8个int。因此上面打印出来的栈内容，每两行表示一个栈帧，看起来还算清晰。

#第一次调用分析
以第一调用栈为例分析，32个字节代码的含义如下图所示：
```
0xf010ffb0:     0x00000004      0x00000005      0x00000000      0x00010094
0xf010ffc0:     0x00010094      0x00010094      0xf010fff8      0xf0100144
             +--------------------------------------------------------------+
             |    next x    |     this x     |  don't know   |  don't know  |
             +--------------+----------------+---------------+--------------+
             |  don't know  |    last ebx    |  last ebp     | return addr  |
             +------ -------------------------------------------------------+
```
中间的两字节不知道是干嘛用的(靠近this x的那一个在调用mon_backtrace时会用到)，按照理论分析，一个完整的调用栈最少需要的字节数等于4+4+4+4*3=24字节，即返回地址，上一个函数的ebp，保存的ebx，函数内没有分配局部变量，需要再加12个字节用来调用mon_backtrace时传参数。

有一个说法是，因为x86的栈大小必须是16的整数倍，所以才分配了32个字节的栈大小。


