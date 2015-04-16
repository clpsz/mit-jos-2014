#代码
这个练习大概写了一百行代码，之前没有仔细看，直接看kern/pmap.c，看到所有的// Fill this function in都写，以为这个文件写完才算完成练习1，结果居然写了一天还没完完（水平也确实比较渣），make qemu总是有assert过不了。再仔细一看，原来只要写boot_alloc(), mem_init(), page_init(), page_alloc(), page_free()即可，瞬间轻松了。


mem_init()中需要注释掉未完成的部分，代码可参见Lab2/2014-jos-Lab2，不再贴出来了。

#函数介绍
1. static void * boot_alloc(uint32_t n)用于在没有内存分配函数之间的内存分配，只在page_alloc不可用之前使用。作用是从可用物理内存上返回足够n字节的空间，返回该空间的起始地址(虚拟地址)。并保证每次分配的内存都对齐到PGSIZE页面边界。
2. void mem_init(void)用于进行整个内存初始化。
3. void page_init(void)用于PageInfo数组的初始化。每个物理页面对应一个PageInfo数据结构，用于指出该物理页面被多少个虚拟页面照射。如果没有映射到虚拟页面，则表示物理页面空闲。所有的空间物理页面的数据结构构成一个链表，如果是空闲物理页面，它的PageInfo数据结构还要负责标示出下一个空闲物理页面数据结构的位置。
4. struct PageInfo page_alloc(int alloc_flags)用于分配一个物理页面，返回分配的物理页面的PageInfo数据结构。
5. void page_free(struct PageInfo *pp)负责释放一个物理页面，将该页面放回到可用物理内存。

#注意点
- malloc是肯定不能用的，因为内存自己都不知道怎么分配内存呢
- 在page_init()执行之前，只能使用boot_alloc分配内存，所以要用boot_alloc来分配页目录(pgdir)和物理内存数据结构数组：
```
kern_pgdir = (pde_t *) boot_alloc(PGSIZE);
...
pages = (struct PageInfo*)boot_alloc(sizeof(struct PageInfo)*npages);
```
- page_init()中需要知道最低可用物理内存位置，因为无法直接获取到nextfree的值(局部变量)，我一开始的做法是用pages的值加上pages占用的大小，然后再计算对应的物理内存。后来才发现直接boot_alloc(0)就可以返回nextfree，然后一个PADDR就算出了物理内存地址。
