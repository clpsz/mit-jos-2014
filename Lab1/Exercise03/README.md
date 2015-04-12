#总体分析
boot sector由工程目录下的boot/子目录中的boot.S和main.c两个文件编译生成，生成的文件在obj/boot/目录下。通过分析boot/Makefrag可以知道，boot sector生成的步骤如下：
```
boot.S->boot.o
main.c->main.o
boot.o main.o -> boot.out
boot.out -> boot
```
最后一步将boot.out中多余的信息去掉，只留下.text段，因为boot.o和main.o都没有.data和.bss段，所以只留下.text段不会影响程序的运行。使用file命令可以验证boot.out和boot两个文件的格式分别为ELF和x86 boot sector文件格式。

#boot.S
boot.S的第一条语句(cli)会被BIOS加载到内存地址0x7c00处，BIOS执行完必要的准备工作后就跳到0x7c00执行boot sector。

boot.S开始运行时处在16位模式下，最后会切换到32位保护模式，它的代码非常简洁，也很干脆，流程如下：

1. 使能A20地址线
2. 执行lgdt加载gdt
3. 进入保护模式
4. 设置各个段选择子及栈寄存器esp
5. 调用main.c中的bootmain函数

#main.c
main.c的入口为bootmain函数，bootmain函数也非常的简洁，流程如下：

1. 读取kernel的ELF文件头部
2. 依次加载每一个程序段
3. 跳转到kernel程序入口去执行

#练习问题
##1. At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?
回答：从设置CR0开始进入32模式，具体的代码是（汇编的32行）：
```
movl    %eax, %cr0
ljmp    $PROT_MODE_CSEG, $protcseg
```
它后面的那一句比较有意思，跳转到下一句执行，看似没有什么作用，其实是为了清空掉之前已前预取的指令（或者说是设置正确的CS和EIP），只有这一条指令可以同时设置CS和EIP，保证后面的指令在分段机制下正确的执行。

##2. What is the last instruction of the boot loader executed, and what is the first instruction of the kernel it just loaded?
最后一条C语句是：
```
     // call the entry point from the ELF header                                                                                                                                            
      // note: does not return!
      ((void (*)(void)) (ELFHDR->e_entry))();
```
从boot.asm中可以找到它对应的指令是：
```
7d61:   ff 15 18 00 01 00       call   *0x10018
```
7d61是指令的内存地址，可以通过gdb调试来验证，在7d61处设置一个断点，注意此时已经进入32位保护模式，程序的地址为0x00007d61(当前CS段选择子的基址为0)：
```
 (gdb) b *0x00007d61
 (gdb) c
 (gdb) x /10i $pc
=> 0x7d61:  call   *0x10018
   0x7d67:  mov    $0x8a00,%edx
   0x7d6c:  mov    $0xffff8a00,%eax
   0x7d71:  out    %ax,(%dx)
   0x7d73:  mov    $0xffff8e00,%eax
   0x7d78:  out    %ax,(%dx)
   0x7d7a:  jmp    0x7d7a
   0x7d7c:  add    %al,(%eax)
   0x7d7e:  add    %al,(%eax)
   0x7d80:  add    %al,(%eax)
 (gdb) x /10i *0x10018
   0x10000c:    movw   $0x1234,0x472
```
最后一句可以得出，内核的第一条指令是:
```
movw   $0x1234,0x472
```
可以看出boot.asm没有欺骗我们，进入kernel前的第一条执行确实是call *0x10018。
##3. Where is the first instruction of the kernel?
因为当前CS指定的段基址为0，此时还没有开启分布，从之前的调试过程可以轻松得出，kernel的第一条执行地址在物理内存的0x00010018处。
 
##4. How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?
回答：kernel是一个ELF格式的文件，可以从ELF头部得到程序有多少个程序头部，bootmain函数循环把所有的程序头部读到内存中。每个程序头部的信息该头部的大小，readseg函数循环从磁盘读取扇区，一直到读完为止，不用计算要读多少个扇区。

读取整个内核代码：
```
// load each program segment (ignores ph flags)
ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
eph = ph + ELFHDR->e_phnum;
for (; ph < eph; ph++)
    // p_pa is the load address of this segment (as well
    // as the physical address)
    readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
```
读取程序段代码：
```
end_pa = pa + count;
...
while (pa < end_pa) {
...
}
```
