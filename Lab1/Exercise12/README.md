#代码
代码也可以看我在Lab1/2014-jos-Lab1目录中的提交
```
$git show 430ca52
--- a/Lab1/2014-jos-Lab1/kern/kdebug.c
+++ b/Lab1/2014-jos-Lab1/kern/kdebug.c
@@ -180,6 +180,13 @@ debuginfo_eip(uintptr_t addr, struct Eipdebuginfo *info)
     //  which one.
     // Your code here.
 
+    stab_binsearch(stabs, &lline, &rline, N_SLINE, addr);
+    if (lline <= rline) {
+        info->eip_line = stabs[lline].n_desc;
+    }
+    else {
+        cprintf("line not find\n");
+    }
 
     // Search backwards from the line number for the relevant filename
     // stab.
diff --git a/Lab1/2014-jos-Lab1/kern/monitor.c b/Lab1/2014-jos-Lab1/kern/monitor.c
index 383fa22..399506f 100644
--- a/Lab1/2014-jos-Lab1/kern/monitor.c
+++ b/Lab1/2014-jos-Lab1/kern/monitor.c
@@ -24,6 +24,7 @@ struct Command {
 static struct Command commands[] = {
     { "help", "Display this list of commands", mon_help },
     { "kerninfo", "Display information about the kernel", mon_kerninfo },
+  { "backtrace", "Display backtrace info", mon_backtrace },
 };
 
--- a/Lab1/2014-jos-Lab1/kern/monitor.c
+++ b/Lab1/2014-jos-Lab1/kern/monitor.c
@@ -24,6 +24,7 @@ struct Command {
 static struct Command commands[] = {
     { "help", "Display this list of commands", mon_help },
     { "kerninfo", "Display information about the kernel", mon_kerninfo },
+  { "backtrace", "Display backtrace info", mon_backtrace },
 };
 #define NCOMMANDS (sizeof(commands)/sizeof(commands[0]))
 
@@ -58,13 +59,21 @@ mon_kerninfo(int argc, char **argv, struct Trapframe *tf)
 int
 mon_backtrace(int argc, char **argv, struct Trapframe *tf)
 {
-    uint32_t ebp, *p;
+    uint32_t ebp, eip, *p;
+    struct Eipdebuginfo info;
 
     ebp = read_ebp();
     while (ebp != 0)
     {
         p = (uint32_t *) ebp;
-        cprintf("ebp %x eip %x args %08x %08x %08x %08x %08x\n", ebp, p[1], p[2], p[3], p[4], p[5], p[6]);
+        eip = p[1];
+        cprintf("ebp %x eip %x args %08x %08x %08x %08x %08x\n", ebp, eip, p[2], p[3], p[4], p[5], p[6]);
+        if (debuginfo_eip(eip, &info) == 0)
+        {
+            int fn_offset = eip - info.eip_fn_addr;
+
+            cprintf("%s:%d: %.*s+%d\n", info.eip_file, info.eip_line, info.eip_fn_namelen, info.eip_fn_name, fn_offset);
+        }
         ebp = p[0];
     }
```
#说明
获取行号这个代码有点复杂，还没来得及仔细看，照着上面的例子抄了一下，发现也能工作，等看明白了再添加说明。其它的应该比较简单，练习的提示也比较清楚。

printf("%.*s", length, string)这个用法还是很牛B的啊。

添加命令这个找下已有的命令在哪，照着写即可：
```
$ grep -rn 'help\|kerninfo' ./Lab1/2014-jos-Lab1
./kern/monitor.c:25:    { "help", "Display this list of commands", mon_help },
./kern/monitor.c:26:    { "kerninfo", "Display information about the kernel", mon_kerninfo },
```
