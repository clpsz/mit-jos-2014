#结论
当BIOS进入bootloader时，因为此时工作在实模式，0x10000以上的内存根本无法访问，所以内存中的内容应该为空。
bootloader进入kernel时，程序地址0x100000处加载的操作系统的代码。

#qemu验证
###进入bootloader时
使用断点使程序停止在0x7c00处，执行：
```
(gdb) b *0x7c00
Breakpoint 1 at 0x7c00
(gdb) c
Continuing.
[   0:7c00] => 0x7c00:  cli    

Breakpoint 1, 0x00007c00 in ?? ()
(gdb) x /16xb 0x100000
0x100000:   0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x100008:   0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
```
###即将进入kernel时
进入内核入口的程序地址为0x76d1，将程序停在该处断点分析
```
(gdb) b *0x7d61
Breakpoint 3 at 0x7d61
(gdb) c
Continuing.
=> 0x7d61:  call   *0x10018

Breakpoint 3, 0x00007d61 in ?? ()
(gdb) x /16xb 0x100000
0x100000:   0x02    0xb0    0xad    0x1b    0x00    0x00    0x00    0x00
0x100008:   0xfe    0x4f    0x52    0xe4    0x66    0xc7    0x05    0x72
```
可以看出此时内存地址0x10000处已经有内容了
#ELF文件分析
使用objdump可以查看kernel文件的加载地址：
```
$objdump -x obj/kern/kernel
...
...
Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001907  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
```
可以看出内核的代码段加载到了内存地址0x100000

使用objcopy可以将内核的代码段复制出来：
```
objcopy -S -O binary -j .text kernel kernel.text
```
使用hexdump查看kernel.text的内容如下（只看前面的一小部分）：
```
$hexdump -C kernel.text|less
00000000  02 b0 ad 1b 00 00 00 00  fe 4f 52 e4 66 c7 05 72  |.........OR.f..r|
```
可以发现与我们使用gdb看到0x100000处的内存内容一致，说明确实是内核的代码被加载到了0x100000处

