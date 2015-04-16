#代码
下面的代码映射物理页面数据结构数组到物理页
```
   boot_map_region(kern_pgdir, 
                    UPAGES, 
                    ROUNDUP((sizeof(struct PageInfo)*npages), PGSIZE),
                    PADDR(pages), 
                    (PTE_U | PTE_P));
```
下面的代码映射内核栈到物理页
```
    boot_map_region(kern_pgdir, 
                    (KSTACKTOP-KSTKSIZE), 
                    KSTKSIZE,
                    PADDR(bootstack), 
                    (PTE_W | PTE_P));
```
下面的代码映射内核到物理内存
```
    boot_map_region(kern_pgdir, 
                    KERNBASE, 
                    ROUNDUP((0xFFFFFFFF-KERNBASE), PGSIZE),
                    0, 
                    (PTE_W | PTE_P));
```
