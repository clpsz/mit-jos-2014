#代码
首先是show me the code:
```
--- a/Lab1/2014-jos-Lab1/kern/monitor.c
+++ b/Lab1/2014-jos-Lab1/kern/monitor.c
@@ -58,8 +58,17 @@ mon_kerninfo(int argc, char **argv, struct Trapframe *tf)
 int
 mon_backtrace(int argc, char **argv, struct Trapframe *tf)
 {
-    // Your code here.
-    return 0;
+    uint32_t ebp, *p;
+
+    ebp = read_ebp();
+    while (ebp != 0)
+    {
+        p = (uint32_t *) ebp;
+        cprintf("ebp %x eip %x args %08x %08x %08x %08x %08x\n", ebp, p[1], p[2], p[3], p[4], p[5], p[6]);
+        ebp = p[0];
+    }
+    
+    return 0;
 }
```
第一个编程题比较简单，给的提示也足够充分，照着做即可。

复制我练习10的分析，第一个test_backtrace函数部分的栈如下：
```
0xf010ffb0:     0x00000004      0x00000005      0x00000000      0x00010094
0xf010ffc0:     0x00010094      0x00010094      0xf010fff8      0xf0100144
             +--------------------------------------------------------------+
             |    next x    |     this x     |  don't know   |  don't know  |
             +--------------+----------------+---------------+--------------+
             |  don't know  |    last ebx    |  last ebp     | return addr  |
             +------ -------------------------------------------------------+
             |     arg 1    |     arg 2      |    arg 3      |    arg 4     |
             +------ -------------------------------------------------------+
             |     arg 5    |     arg 6      |    arg 7      |    arg 8     |
             +------ -------------------------------------------------------+
```
获取到第一个ebp之后循环，直到ebp为0停止循环。
#练习问题
##1.  The return instruction pointer typically points to the instruction after the call instruction (why?).
感觉这个似乎不用回答啊，函数返回不到下一条指令地址还能到哪？
##2. (Why can't the backtrace code detect how many arguments there actually are? How could this limitation be fixed?
因为判断有几个参数这种事情是编译器干的，编译器通过函数原型来判断有几个参数。函数内部是没有方法直接获取到有几个参数传过来了这种事情的。

要修复的话，可以把函数的第一个参数设置为总共的参数个数。

