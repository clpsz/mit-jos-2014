#结论
1. kern/printf.c提供用户实际需要调用的接口cprintf
2. lib/printfmt.c提供了供cprintf函数调用的接口vcprintf，vcprintf的作用是对输出进行格式化，把不同类型的输出(%s %d %p等)按不同的方式显示在屏幕上
3. kern/console.c提供了供vcprintf调用的回调函数cputchar(实际上是kern/printf.c中的putch，但这个函数只是cputchar的一个包装而已)，让vcprintf可以不用处理向屏幕上输出一个字符这样的底层的细节。

#练习解答
练习是要求实现故意没有实现的%o格式化输出，kern/init.c(36行)使用了这%o输出格式：
```
 32     // Initialize the console.
 33     // Can't call cprintf until after we do this!                                      
 34     cons_init();
 35 
 36     cprintf("6828 decimal is %o octal!\n", 6828);
```
在没有实现之前，输出如下：
```
$ make qemu
qemu-system-i386 -hda obj/kern/kernel.img -serial mon:stdio -gdb tcp::26000 -D qemu.log 
6828 decimal is XXX octal!
```
在lib/printfmt.c(205行)修改如下(git diff格式):
```
--- a/Lab1/2014-jos-Lab1/lib/printfmt.c
+++ b/Lab1/2014-jos-Lab1/lib/printfmt.c
@@ -205,11 +205,13 @@ vprintfmt(void (*putch)(int, void*), void *putdat, const char *fmt, va_list ap)
 
         // (unsigned) octal
         case 'o':
-            // Replace this with your code.
-            putch('X', putdat);
-            putch('X', putdat);
-            putch('X', putdat);
-            break;
+          num = getint(&ap, lflag);
+          if ((long long) num < 0) {
+              putch('-', putdat);
+              num = -(long long) num;
+          }
+          base = 8;
+          goto number;
 
         // pointer
         case 'p':
```
其实就是把%d的代码复制一下，改个base就行了。再次运行qemu即可得到正确的结果：
```
$ make qemu
qemu-system-i386 -hda obj/kern/kernel.img -serial mon:stdio -gdb tcp::26000 -D qemu.log 
6828 decimal is 15254 octal!
```
#练习问题解答
##1. Explain the interface between printf.c and console.c. Specifically, what function does console.c export? How is this function used by printf.c?
printf.c调用console.c提供的接口cputchar，将这个函数封装在putch，并将这个封装好的函数作为参数传给vprintfmt函数，用于向屏幕上输出一个字符。
##2. 原题如下：
```
Explain the following from console.c:
1      if (crt_pos >= CRT_SIZE) {
2              int i;
3              memcpy(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
4              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
5                      crt_buf[i] = 0x0700 | ' ';
6              crt_pos -= CRT_COLS;
7      }
```
crt_post是当前光标位置，CRT_SIZE是屏幕上总共的可以输出的字符数(其值等于行数乘以每行的列数)，这段代码的意思是当屏幕输出满了以后，将屏幕上的内容都向上移一行，即将第一行移出屏幕，同时将最后一行用空格填充，最后将光标移动到屏幕最后一行的开始处。
##3. 原题如下：
```
For the following questions you might wish to consult the notes for Lecture 2. These notes cover GCC's calling convention on the x86.
Trace the execution of the following code step-by-step:

int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
In the call to cprintf(), to what does fmt point? To what does ap point?
List (in order of execution) each call to cons_putc, va_arg, and vcprintf. For cons_putc, list its argument as well. For va_arg, list what ap points to before and after the call. For vcprintf list the values of its two arguments.
```
直接在i386_init函数中加测试代码即可：
```
--- a/Lab1/2014-jos-Lab1/kern/init.c
+++ b/Lab1/2014-jos-Lab1/kern/init.c
@@ -34,6 +34,11 @@ i386_init(void)
     cons_init();
 
     cprintf("6828 decimal is %o octal!\n", 6828);
+    {
+        int x = 1, y = 3, z = 4;
+    Lab1_exercise8_3:
+        cprintf("x %d, y %x, z %d\n", x, y, z);
+    }
 
     // Test the stack backtrace function (lab 1 only)
     test_backtrace(5);
```
加Lab1_exercise8_3标号的目的是为了在kern/kernel.asm反汇编代码中容易找到添加的代码的位置。查看反汇编代码得知添加的代码在虚拟内存0xf01000de处：
```
    {    
        int x = 1, y = 3, z = 4; 
    Lab1_exercise8_3:
        cprintf("x %d, y %x, z %d\n", x, y, z);
f01000de:   c7 44 24 0c 04 00 00    movl   $0x4,0xc(%esp)
f01000e5:   00   
f01000e6:   c7 44 24 08 03 00 00    movl   $0x3,0x8(%esp)
f01000ed:   00   
f01000ee:   c7 44 24 04 01 00 00    movl   $0x1,0x4(%esp)
f01000f5:   00   
f01000f6:   c7 04 24 f2 19 10 f0    movl   $0xf01019f2,(%esp)
f01000fd:   e8 3f 08 00 00          call   f0100941 <cprintf>
    }
```
运行qemu输出结果如下：
```
x 1, y 3, z 4
```
直观的感觉肯定是fmt指向格式化字符串"x %d, y %x, z %d\n"，ap指向x压入的栈地址，实际上也确实如此，但是背后的原理并不简单。

函数调用压栈的顺序是从右向左，所以最先压入的是常量4，最后压入的是字符串常量的地址，使用gdb可以看到0xf01019f2是否指向该字符串：
```
(gdb) b *0xf01000de
Breakpoint 1 at 0xf01000de: file kern/init.c, line 40.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf01000de <i386_init+65>:   movl   $0x4,0xc(%esp)

Breakpoint 1, i386_init () at kern/init.c:40
40              cprintf("x %d, y %x, z %d\n", x, y, z);
(gdb) stepi
=> 0xf01000e6 <i386_init+73>:   movl   $0x3,0x8(%esp)
0xf01000e6      40              cprintf("x %d, y %x, z %d\n", x, y, z);
(gdb) step
=> 0xf0100947 <cprintf+6>:      lea    0xc(%ebp),%eax
cprintf (fmt=0xf01019f2 "x %d, y %x, z %d\n") at kern/printf.c:31
31              va_start(ap, fmt);
(gdb) 
=> 0xf010094a <cprintf+9>:      mov    %eax,0x4(%esp)
32              cnt = vcprintf(fmt, ap);
(gdb) 
=> 0xf0100914 <vcprintf+6>:     movl   $0x0,-0xc(%ebp)
vcprintf (fmt=0xf01019f2 "x %d, y %x, z %d\n", ap=0xf010ffe4 "\001") at kern/printf.c:19
19              int cnt = 0;
(gdb) x /s 0xf01019f2
0xf01019f2:     "x %d, y %x, z %d\n"
(gdb) x /16xb 0xf010ffe4
0xf010ffe4:     0x01    0x00    0x00    0x00    0x03    0x00    0x00    0x00
0xf010ffec:     0x04    0x00    0x00    0x00    0x00    0x00    0x00    0x00
```
在0xf01000de断点后，执行stepi跟到cprintf中后再执行step可以看到传给cprintf的两个实参是两个指针，其中fmt指向格式化字符串，ap指向栈中压入的常量1的地址，即x压入到栈中的地址。

从这个练习可以看出来，正是因为C函数调用实参的入栈顺序是从右到左的，才使得调用参数个数可变的函数成为可能(且不用显式地指出参数的个数)。但是必须有一个方式来告诉实际调用时传入的参数到底是几个，这个是在格式化字符串中指出的。如果这个格式化字符串指出的参数个数和实际传入的个数不一致，比如说传入的参数比格式化字符串指出的要少，就可能会使用到栈上错误的内存作为传入的参数，编译器必须检查出这样的错误。

va_start宏计算fmt后面的地址并赋值给ap。

va_end宏用于重置ap宏，使它指向NULL(不过看cprintf的反汇编代码中，va_end好像啥都没干，不知道是为什么)。

三个函数cons_putc, va_arg, and vcprintf的顺序及传入参数如下：
```
vcprintf (fmt=0xf01019b2 "x %d, y %x, z %d\n", ap=0xf010ffd4) //ap指向第一个可变个数参数的栈地址

    consputc (c=120)//x
    consputc (c=32) //空格
    va_arg(&ap, int)//调用前ap为0xf010ffd4，调用后ap为0xf010ffd8
    consputc (c=49) //1

    consputc (c=44) //,
    consputc (c=121)//y
    consputc (c=32) //空格
    va_arg(&ap, int)//调用前ap为0xf010ffd8，调用后ap为0xf010ffdc
    consputc (c=51) //3

    consputc (c=44) //,
    consputc (c=32) //空格
    consputc (c=122)//z
    consputc (c=32) //空格
    va_arg(&ap, int)//调用前ap为0xf010ffdc，调用后ap为0xf010ffe0
    consputc (c=52) //4
    consputc (c=10) //\n
```
为了方便找到调用va_arg(被getint和getuint调用)，取消了getint和getuint的static属性，否则这两个函数好像会被编译器优化成内联函数，找不到在哪调用的了。

##4. 原题如下：
```
Run the following code.
    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);
What is the output? Explain how this output is arrived at in the step-by-step manner of the previous exercise. Here's an ASCII table that maps bytes to characters.
The output depends on that fact that the x86 is little-endian. If the x86 were instead big-endian what would you set i to in order to yield the same output? Would you need to change 57616 to a different value?
```
这个题目还是挺有意思的，57616的16进制表示是e110，而ascii表中0x72,0x6c,0x64分别代表r,l,d三个字母。因此57616用%x打印出来就是e110，可以看作是ello，而0x00646c72在内存中的布局是0x7c 0x6c 0x64 0x00，可以看作是字符串"rld\0"，所以面这行代码恰好可以打印出He110 world。
```
He110 Worldentering test_backtrace 5
```
因为没有换行所以跟后面的打印信息连起来了，所以代码可以做如下修改来加上换行：
```
    unsigned int i = 0x000a646c;
    cprintf("H%x Wor%s", 57616, &i);
```
输出如下：
```
He110 World
entering test_backtrace 5
```
在原题的基础上，如果是大端的话，需要做如下改动：
```
    unsigned int i = 0x726c6400;
    cprintf("H%x Wor%s", 57616, &i);
```
无法验证。
##5. 原题如下：
```
In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?
    cprintf("x=%d y=%d", 3);
```
会把栈上的3后面的内存当作是整数打印出来，添加这行程序后gdb调试(我的这一条命令编译到了虚拟地址0xf0100102处)：
```
--- a/Lab1/2014-jos-Lab1/kern/init.c
+++ b/Lab1/2014-jos-Lab1/kern/init.c
@@ -38,6 +38,8 @@ i386_init(void)
         int x = 1, y = 3, z = 4;
     Lab1_exercise8_3:
         cprintf("x %d, y %x, z %d\n", x, y, z);
+    Lab1_exercise8_5:
+        cprintf("x=%d y=%d", 3);
     }
     {
         unsigned int i = 0x000a646c;
```
gdb查看如下：
```
(gdb) stepi
=> 0xf0100111 <i386_init+116>:  call   0xf0100981 <cprintf>
0xf0100111      42              cprintf("x=%d y=%d", 3);
(gdb) x /16xw $esp
0xf010ffd0:     0xf0101a44      0x00000003      0x00000003      0x00000004
0xf010ffe0:     0x00000000      0x00000000      0x00000000      0x00000000
```
因为栈上0x000000003上一个整型内存地址处的值也是0x00000003，可以预知，将打印出两个3，实际结果如下：
```
x 1, y 3, z 4
x=3 y=3He110 World
```
##6. Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change cprintf or its interface so that it would still be possible to pass it a variable number of arguments?
这个没法验证能，只能猜测一下，我觉得需要做两处改动，下面解释。
因为传参是从左向右，而可变个数参数必须在参数列表的最后，所以有名参数最先入栈，可以参数最后入栈，这样在进入被调用函数时，栈空间的参数部分如下：
```
high    +------+
 ||     | fmt  |
 ||     | arg1 |
 ||     | arg2 |
 ||     | ...  |
 \/     | argn |
low     +------+
```
最后一个有名参数的地址是已知的，此处假设最后一个有名参数是fmt，因此va_start计算ap地址时，需要使用fmt的地址减去fmt占用的内存大小，即&fmt - 4。

而va_arg在更新当前参数指针地址时，也是要使用当前指针的值减去当前取的参数的占用的内存空间大小。

以上结论纯属猜测，如有其它看法，欢迎讨论。

