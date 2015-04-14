#结论
```
movl    %eax, %cr0
```
这条语句的作用就是开启分页模式，在执行这条语句之前，所有的线性地址直接等于物理地址，执行之后，线性地址需要经过MMU的映射才对应到物理地址。

所以执行之前0x100000处为内核代码，0xf0100000处为空。执行之后0x100000和0xf0100000都映射到物理内存0x100000，所以他们的内容相同。

如果不执行这条语句的话，第一条出错的语句将是后面的绝对跳转语句(kern/entry.S 59)：
```
# Turn on paging.
movl    %cr0, %eax
orl $(CR0_PE|CR0_PG|CR0_WP), %eax
movl    %eax, %cr0

# Now paging is enabled, but we're still running at a low EIP
# (why is this okay?).  Jump up above KERNBASE before entering
# C code.
mov $relocated, %eax
jmp *%eax
```
即第一条出错的语句是jmp *%eax，程序将跳到一个错误的地址，该地址内容为空。

#gdb分析
执行mov %eax, %cr0之前，0xf0100000处内容为空
```
(gdb) b *0x100025
Breakpoint 1 at 0x100025
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x100025:    mov    %eax,%cr0

Breakpoint 1, 0x00100025 in ?? ()
(gdb) x /16xb 0x100000
0x100000:   0x02    0xb0    0xad    0x1b    0x00    0x00    0x00    0x00
0x100008:   0xfe    0x4f    0x52    0xe4    0x66    0xc7    0x05    0x72
(gdb) x /16xb 0xf0100000
0xf0100000 <_start+4026531828>: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0xf0100008 <_start+4026531836>: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
```

执行mov %eax, %cr0之后，0xf0100000处的内容与0x100000处内容相同
```
(gdb) stepi
=> 0x100028:    mov    $0xf010002f,%eax
0x00100028 in ?? ()
(gdb) x /16xb 0x100000
0x100000:   0x02    0xb0    0xad    0x1b    0x00    0x00    0x00    0x00
0x100008:   0xfe    0x4f    0x52    0xe4    0x66    0xc7    0x05    0x72
(gdb) x /16xb 0xf0100000
0xf0100000 <_start+4026531828>: 0x02    0xb0    0xad    0x1b    0x00    0x00    0x00    0x00
0xf0100008 <_start+4026531836>: 0xfe    0x4f    0x52    0xe4    0x66    0xc7    0x05    0x72
```

#不开启分页的错误分析
注释掉movl    %eax, %cr0,再编译运行，在0x100025处中断后stepi单步执行：
```
(gdb) b *0x100025
Breakpoint 1 at 0x100025
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x100025:    mov    $0xf010002c,%eax

Breakpoint 1, 0x00100025 in ?? ()
(gdb) stepi
=> 0x10002a:    jmp    *%eax
0x0010002a in ?? ()
(gdb) stepi
=> 0xf010002c <relocated>:  add    %al,(%eax)
relocated () at kern/entry.S:74
74      movl    $0x0,%ebp           # nuke frame pointer
(gdb) x /10i $pc
=> 0xf010002c <relocated>:  add    %al,(%eax)
   0xf010002e <relocated+2>:    add    %al,(%eax)
   0xf0100030 <relocated+4>:    add    %al,(%eax)
   0xf0100032 <relocated+6>:    add    %al,(%eax)
   0xf0100034 <relocated+8>:    add    %al,(%eax)
   0xf0100036 <relocated+10>:   add    %al,(%eax)
   0xf0100038 <relocated+12>:   add    %al,(%eax)
   0xf010003a <relocated+14>:   add    %al,(%eax)
   0xf010003c <spin+1>: add    %al,(%eax)
   0xf010003e <test_backtrace+1>:   add    %al,(%eax)
```
可以看到执行完jmp *%eax后就跑飞了

