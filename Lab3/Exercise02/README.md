#总结
这个练习需要写的代码就比较多了，不过按照题目的提示，完成各个函数也就可以正常工作了。

#调用关系
题目中给出了函数间的调用关系，有几个函数省略了，下面是我自己补充的：
```
start
  i386_init
    cons_init
    env_init
    trap_init
    env_create
      evn_alloc
        env_setup_vm
      load_icode
        region_alloc
  env_run
    env_popup_tf
```
画的不太直观，就是同一个缩进级别是在同一个函数里面被调用了。

#函数解释
###env_init
初始化envs中的所有env，并添加到空闲链表。
###env_setup_vm
设置一个env的虚拟地址空间，即页目录表和必要的内核数据结构。
###region_alloc
照射物理内存到虚拟地址空间。
###load_icode
将一个可执行文件（因为没有文件系统，所以其实就是一个内存地址）载入到一个env中。
###env_create
创建一个env，其实就是从env空闲链表中分配一个env，并且将可执行文件载入到这个env的地址空间中。这个函数仅仅在内核初始化的时候用到，后面不是通过这种方式来创建env。
###env_run
调试一个新的env，设置当前正在运行的env的状态为runnable（但是在哪保存之前的env的呢？）。






