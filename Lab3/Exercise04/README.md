#结论
在这个练习中定义了一系列中断入口函数，用好JOS提供的宏可以轻松完成这些工作。

1. 使用宏TRAPHANDLER和TRAPHANDLER_NOEC定义一系列中断入口地址。
1. 因为要切换到内核代码，所以需要将ES和DS设置成内核的ES和DS。
2. 使用pushal命令来压入中断调用帧(trapframe)，然后将这栈中的这个帧的地址传给函数trap。
3. 在trap_init中需要设置IDT。

#分析
下面这两个结构体定义对于理解中断很有用：
```
struct PushRegs {
	/* registers as pushed by pusha */
	uint32_t reg_edi;
	uint32_t reg_esi;
	uint32_t reg_ebp;
	uint32_t reg_oesp;		/* Useless */
	uint32_t reg_ebx;
	uint32_t reg_edx;
	uint32_t reg_ecx;
	uint32_t reg_eax;
} __attribute__((packed));

struct Trapframe {
	struct PushRegs tf_regs;
	uint16_t tf_es;
	uint16_t tf_padding1;
	uint16_t tf_ds;
	uint16_t tf_padding2;
	uint32_t tf_trapno;
	/* below here defined by x86 hardware */
	uint32_t tf_err;
	uintptr_t tf_eip;
	uint16_t tf_cs;
	uint16_t tf_padding3;
	uint32_t tf_eflags;
	/* below here only when crossing rings, such as from user to kernel */
	uintptr_t tf_esp;
	uint16_t tf_ss;
	uint16_t tf_padding4;
} __attribute__((packed));
```
从TrapFrame的注释中可以得知，在进入中断入口时，内核栈中已经由CPU压入了ESS(用户态进入内核态才有), ESP(用户态进入内核态才有), EFLAGS, CS, EIP, ERR_NO。

进入中断入口之后，如果该中断没有错误码，则压入一个0以保持TrapFrame的致，接着是压入中断号方便后面trap_dispatch函数处理不同的中断。因为可能要更改代码段和数据段描述符，所以将es和ds也压入堆栈。最后是调用pushal压入当前的寄存器值。

读代码的时候需要注意的是堆栈的增长方向是从大地址到小地址，而结构体的field从上到下是从小地址到大地址。因此最先压入堆栈的是结构体中的最后一个字段。且压栈和出栈都是以4个字节为一个块进入操作的，对于不足4字节的数据，会被填充。

中断入口的代码如下：
```
#define TRAPHANDLER_NOEC(name, num)					\
	.globl name;							\
	.type name, @function;						\
	.align 2;							\
	name:								\
	pushl $0;							\
	pushl $(num);							\
	jmp _alltraps

...

.globl _alltraps
_alltraps:
pushl %ds                                                                                  
pushl %es
pushal

movw $GD_KD, %ax
movw %ax, %ds
movw %ax, %es

pushl %esp  /* trap(%esp) */
call trap
```
