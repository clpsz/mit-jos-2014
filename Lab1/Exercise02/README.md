#调试设置
这个练习需要使用gdb调试器来跟踪程序运行， make之后，工程中已经生成了gdb的设置文件.gdbinit（根据.gdbinit.tmpl生成）。
打开配置文件可以发现它做了三项设置：
- 设置target连接到qemu
- 设置处理器芯片为8086
- 设置内核符号文件

需要在gdb全局配置文件中添加一个配置项来使用这个配置，在~/.gdbinit中增加:

add-auto-load-safe-path /path/to/project/2014-jos/.gdbinit

#启动调试
开两个终端窗口，一个运行make qemu-gdb，另一个运行gdb，即可进入gdb调试，gdb会自动调用.gdbinit进行配置。

进入gdb后，在gdb命令行中输入info r可以看到当前的寄存器情况，可以看到:
```
(gdb) info r
eip            0xfff0	0xfff0
eflags         0x2	[ ]
cs             0xf000	61440
```
因为现在工作于实模式，指令地址为%cs<<4+%eip，说明当前的PC为0xffff0，输入x /10i 0xffff0可以查看0xffff0处的指令，如下：
```
(gdb) x /10i 0xffff0
   0xffff0:	ljmp   $0xf000,$0xe05b
   0xffff5:	xor    %dh,0x322f
   0xffff9:	xor    (%bx),%bp
   0xffffb:	cmp    %di,(%bx,%di)
   0xffffd:	add    %bh,%ah
   0xfffff:	add    %al,(%bx,%si)
   0x100001:	add    %al,(%bx,%si)
   0x100003:	add    %al,(%bx,%si)
   0x100005:	add    %al,(%bx,%si)
   0x100007:	add    %al,(%bx,%si)
```
可以看出第一条指令是一个长跳转指令，跳转到0xfe05b处执行，继续看0xfe05b处的指令：
```
(gdb) x /10i 0xfe05b
   0xfe05b:	cmpl   $0x0,%cs:0x65a4
   0xfe062:	jne    0xfd2b9
   0xfe066:	xor    %ax,%ax
   0xfe068:	mov    %ax,%ss
   0xfe06a:	mov    $0x7000,%esp
   0xfe070:	mov    $0xf3c4d,%edx
   0xfe076:	jmp    0xfd12a
   0xfe079:	push   %ebp
   0xfe07b:	push   %edi
   0xfe07d:	push   %esi
```
可以通过以下命令将指令输出到文件中：
```
(gdb) set logging on
(gdb) set logging /path/to/file
```
下面是通过此文件获取到前100条指令：
```
   0xfe05b:	cmpl   $0x0,%cs:0x65a4
   0xfe062:	jne    0xfd2b9
   0xfe066:	xor    %ax,%ax
   0xfe068:	mov    %ax,%ss
   0xfe06a:	mov    $0x7000,%esp
   0xfe070:	mov    $0xf3c4d,%edx
   0xfe076:	jmp    0xfd12a
   0xfe079:	push   %ebp
   0xfe07b:	push   %edi
   0xfe07d:	push   %esi
   0xfe07f:	push   %ebx
   0xfe081:	sub    $0xc,%esp
   0xfe085:	mov    %eax,%esi
   0xfe088:	mov    %edx,(%esp)
   0xfe08d:	mov    %ecx,%edi
   0xfe090:	mov    $0xe000,%eax
   0xfe096:	mov    %ax,%es
   0xfe098:	mov    %es:0xf7a0,%al
   0xfe09c:	mov    %al,0xa(%esp)
   0xfe0a1:	and    $0xffffffcc,%eax
   0xfe0a5:	or     $0x30,%eax
   0xfe0a9:	mov    %al,0xb(%esp)
   0xfe0ae:	lea    0xb(%esp),%edx
   0xfe0b4:	mov    $0x1060,%eax
   0xfe0ba:	calll  0xf8431
   0xfe0c0:	test   %eax,%eax
   0xfe0c3:	jne    0xfe2ad
   0xfe0c7:	calll  0xf6ffc
   0xfe0cd:	test   %esi,%esi
   0xfe0d0:	je     0xfe0da
   0xfe0d2:	andb   $0xdf,0xb(%esp)
   0xfe0d8:	jmp    0xfe0e0
   0xfe0da:	andb   $0xef,0xb(%esp)
   0xfe0e0:	lea    0xb(%esp),%edx
   0xfe0e6:	mov    $0x1060,%eax
   0xfe0ec:	calll  0xf8431
   0xfe0f2:	mov    %eax,%ebx
   0xfe0f5:	test   %eax,%eax
   0xfe0f8:	jne    0xfe296
   0xfe0fc:	cmpl   $0x2ff,(%esp)
   0xfe105:	jne    0xfe164
   0xfe107:	mov    $0x3e8,%ecx
   0xfe10d:	mov    $0xff,%edx
   0xfe113:	mov    %esi,%eax
   0xfe116:	calll  0xf8ed0
   0xfe11c:	mov    %eax,%ebx
   0xfe11f:	test   %eax,%eax
   0xfe122:	jne    0xfe296
   0xfe126:	mov    $0xfa0,%ecx
   0xfe12c:	xor    %edx,%edx
   0xfe12f:	mov    %esi,%eax
   0xfe132:	calll  0xf8df9
   0xfe138:	test   %eax,%eax
   0xfe13b:	js     0xfe293
   0xfe13f:	mov    %al,(%edi)
   0xfe142:	mov    $0x64,%ecx
   0xfe148:	xor    %edx,%edx
   0xfe14b:	mov    %esi,%eax
   0xfe14e:	calll  0xf8df9
   0xfe154:	mov    %eax,%edx
   0xfe157:	not    %edx
   0xfe15a:	sar    $0x1f,%edx
   0xfe15e:	and    %edx,%eax
   0xfe161:	jmp    0xfe1eb
   0xfe164:	cmpl   $0x2f2,(%esp)
   0xfe16d:	jne    0xfe1fa
   0xfe171:	mov    $0xc8,%ecx
   0xfe177:	mov    $0xf2,%edx
   0xfe17d:	mov    %esi,%eax
   0xfe180:	calll  0xf8ed0
   0xfe186:	mov    %eax,%ebx
   0xfe189:	test   %eax,%eax
   0xfe18c:	jne    0xfe296
   0xfe190:	mov    $0x1f4,%ecx
   0xfe196:	xor    %edx,%edx
   0xfe199:	mov    %esi,%eax
   0xfe19c:	calll  0xf8df9
   0xfe1a2:	test   %eax,%eax
   0xfe1a5:	js     0xfe293
   0xfe1a9:	mov    %al,(%edi)
   0xfe1ac:	lea    -0xab(%eax),%edx
   0xfe1b4:	cmp    $0x1,%edx
   0xfe1b8:	jbe    0xfe1d2
   0xfe1ba:	cmp    $0x2b,%eax
   0xfe1be:	je     0xfe1d2
   0xfe1c0:	cmp    $0x5d,%eax
   0xfe1c4:	je     0xfe1d2
   0xfe1c6:	cmp    $0x60,%eax
   0xfe1ca:	je     0xfe1d2
   0xfe1cc:	cmp    $0x47,%eax
   0xfe1d0:	jne    0xfe1f2
   0xfe1d2:	mov    $0x1f4,%ecx
   0xfe1d8:	xor    %edx,%edx
   0xfe1db:	mov    %esi,%eax
   0xfe1de:	calll  0xf8df9
   0xfe1e4:	test   %eax,%eax
   0xfe1e7:	js     0xfe293
   0xfe1eb:	mov    %al,0x1(%edi)
   0xfe1ef:	jmp    0xfe296
   0xfe1f2:	movb   $0x0,0x1(%edi)
```

#BIOS介绍
根据我的理解，BIOS程序固化在ROM中，计算机上电后CPU直接从ROM中执行程序，第一条执行的地址是0xffff0。因为处于实模式，高于1M的内存会回绕，也可以说是0xfffffff0，ROM和RAM统一编址。BIOS的功能如下:

1. POST(Power On Self Test)，对关键硬件进行自检，完整的POST自检将包括CPU、640K基本内存、1M以上的扩展内存、ROM、主板、 CMOS存贮器、串并口、显示卡、软硬盘子系统及键盘测试。自检中若发现问题，系统将给出提示信息或鸣笛警告。
2. 设置硬件中断，提供中断服务例程。在操作系统启动之前，硬件中断由BIOS中设置的中断服务例程处理。典型的硬件中断包括读取键盘输入，向屏幕打印字符，读取硬盘状态等。
3. BIOS程序所在的存储区是不可写的，在PC出厂时已经设置好了。同时主板上还存在一个使用电池供电的CMOS RAM，BIOS需要提供一个程序来读写这个RAM中内容及相应的读写驱动。我们平常开机时按DEL键进入的BIOS设置界面就是读写CMOS RAM的设置程序，这个程序只是BIOS提供的所有功能的一小部分。
4. bootloader，在一切准备工作都做好之后，BIOS将控制权交给操作系统。BIOS会根据之前设置的启动顺序依次检测启动设备中有没有可用的MBR，如果有的话，则将MBR读取到内存地址0x7c00处，并跳转到该地址执行。

#TODO
因为关于BIOS代码的资料实在是有点少，此处暂时不再详细分析，以后有时间的话再继续。

#参考资料
[百度百科-BIOS](http://baike.baidu.com/view/361.htm)
