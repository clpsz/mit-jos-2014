#结论
经过测试，把link address改成0x7c10的话，会导致循环执行bootloader。第一条出问题的语句是:
```
lgdt gdtdesc
```
第一条直接出错的语句是:
```
ljmp    $PROT_MODE_CSEG, $protcseg
```
#简单分析
bootloader是由BIOS加载的，不论连接的地址是什么，最终都会被拷贝到物理内存的0x7c00处，但是由于连接时指定的地址是0x7c10，那么所有的标号都会错乱，但是段时的短跳转指令因为使用的是相对地址，所以并不会出问题。

第一条直接使用绝对地址的指令是lgdt gdtdesc，但是这条指令只是加载GDTR寄存器，虽然这个寄存器的内容是有问题的，但是由于还没有访存，所以还没有直接导致出错，后面的ljmp需要访存了，立马就出错了。

#对比分析
###正常情况分析
正常情况分析，正常情况下执行到lgdt时：
```
# Switch from real to protected mode, using a bootstrap GDT
# and segment translation that makes virtual addresses 
# identical to their physical addresses, so that the 
# effective memory map does not change during the switch.
lgdt    gdtdesc
    7c1e:   0f 01 16                lgdtl  (%esi)
    7c21:   64                      fs
    7c22:   7c 0f                   jl     7c33 <protcseg+0x1>
```
这里加载的gdt描述符在0x7c64地址处，在反汇编文件boot.asm中看到的该地址处的内容是：
```
00007c4c <gdt>:
    ...
    7c54:   ff                      (bad)
    7c55:   ff 00                   incl   (%eax)
    7c57:   00 00                   add    %al,(%eax)
    7c59:   9a cf 00 ff ff 00 00    lcall  $0x0,$0xffff00cf
    7c60:   00 92 cf 00 17 00       add    %dl,0x1700cf(%edx)（这里的17 00是从下面借过来的）

00007c64 <gdtdesc>: 

    7c64:   17                      pop    %ss
    7c65:   00 4c 7c 00             add    %cl,0x0(%esp,%edi,2)
    ...
```
gdt全局描述符指针和gdb描述符表本来应该在数据段的，但是这里为了编译方便，直接放在代码段里面了，所以也被反汇编出来了，所以反汇编出来的结果有的是非法指令，且把数据显示成了指令，<gdt>和<gdtdesc>两个标号处本来应该是这样的：
```
00007c4c <gdt>:
    7c4c: 00 00 00 00 00 00 00 00
    7c54: ff ff 00 00 00 9a cf 00
    7c5c: ff ff 00 00 00 92 cf 00

00007c64 <gdtdesc>:
    7c64: 17 00 4c 7c 00 00
```
使用gdb验证下：
```
(gdb) x /24xb 0x7c4c
0x7c4c: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7c54: 0xff    0xff    0x00    0x00    0x00    0x9a    0xcf    0x00
0x7c5c: 0xff    0xff    0x00    0x00    0x00    0x92    0xcf    0x00
(gdb) x /6xb 0x7c64
0x7c64: 0x17    0x00    0x4c    0x7c    0x00    0x00
```
注释如下：
```
00007c4c <gdt>:
    7c4c: 00 00 00 00 00 00 00 00 ;第一项没有使用
    7c54: ff ff 00 00 00 9a cf 00 ;可读代码段，基地址为0x00000000，段限长为0x0fffff
    7c5c: ff ff 00 00 00 92 cf 00 ;可写数据段，基地址为0x00000000，段限长为0x0fffff

00007c64 <gdtdesc>:
    7c64: 17 00 4c 7c 00 00 ;全局描述符表基地址为0x00007c00，gdt描述表长度为0017=24-1，共可以保存3个gdt描述符
```
gdb中看到的内容也是如此：
```
(gdb) x /6xb 0x7c64
0x7c64: 0x17    0x00    0x4c    0x7c    0x00    0x00
```
执行ljmp指令时:
```
    ljmp    $PROT_MODE_CSEG, $protcseg
```
因为代码段基地址为0，所以直接跳转到$protcseg处执行，即当前指令的下一条指令
gdb描述符中指出全局描述符表的基地址是0x7c4c，当调用ljmp时，PROT_MODE_CSEG使用的是第一个段GDT描述符，而第一个GDT描述符会把段基地址设为0，因些会跳转到下一条指令即protcseg的位置继续执行:
```
    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp    $PROT_MODE_CSEG, $protcseg   

    .code32                     # Assemble for 32-bit mode
protcseg:
```
###错误情况分析
下面分析将连接地址改为0x7c10的情况，此时gdtdesc被连接到了0x7c74，而gdt被连接到了0x7c5c
```
00007c5c <gdt>:
    ... 
    7c64:   ff                      (bad)  
    7c65:   ff 00                   incl   (%eax)
    7c67:   00 00                   add    %al,(%eax)
    7c69:   9a cf 00 ff ff 00 00    lcall  $0x0,$0xffff00cf
    7c70:   00 92 cf 00 17 00       add    %dl,0x1700cf(%edx)

00007c74 <gdtdesc>:
    7c74:   17                      pop    %ss 
    7c75:   00 5c 7c 00             add    %bl,0x0(%esp,%edi,2)
    ...
```
整理后得到以下代码(与正常代码下比，所以的内存地址加0x10)：
```
00007c5c <gdt>:
    7c5c: 00 00 00 00 00 00 00 00 ;第一项没有使用
    7c64: ff ff 00 00 00 9a cf 00 ;可读代码段，基地址为0x00000000，段限长为0x0fffff
    7c6c: ff ff 00 00 00 92 cf 00 ;可写数据段，基地址为0x00000000，段限长为0x0fffff

00007c74 <gdtdesc>:
    7c4: 17 00 4c 7c 00 00 ;全局描述符表基地址为0x00007c00，gdt描述表长度为0x0017=24-1，共可以保存3个gdt描述符
```
但是最终bootloader代码会被BIOS加载到0x7c00处，那么实际的加载地址将比原来的地址小0x10，则实际访问的地址比原来的地址大0x10，即原来访问gdtdesc(0x7c74)的指令，现在访问的实际地址是反汇编代码0x7c84处的内存。
0x7c84处的内存为：
```
    7c83:   83 e0 c0                and    $0xffffffc0,%eax
    7c86:   3c 40                   cmp    $0x40,%al
    7c88:   75 f8                   jne    7c82 <waitdisk+0x8>
        /* do nothing */;
}
    7c8a:   5d
```
整理后可得到：
```
00007c84:
    7c84: e0 c0 3c 40 75 f8
```
使用gdb验证下(gdb中0x7c74等于反汇编代码中的0x7c84)：
```
(gdb) x /6xb 0x7c74
0x7c74: 0xe0    0xc0    0x3c    0x40    0x75    0xf8
```
注释如下：
```
    7c84: e0 c0 3c 40 75 f8 ;全局描述符表的基地址为0xf875403c，长度为0xc0e0
```
下面来看下0xf875403c处的内存是啥:
```
(gdb) x /24xb 0xf875403c
0xf875403c: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0xf8754044: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0xf875404c: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
```
好吧，这个没法分析了，ljmp时会使用第二组gdt描述符，因为段限长是0，所以一访问肯定出错，跳到了一个莫名其妙的地址，然后就死循环了。而且gdb因为运行在特权级0，看不了GDTR。
