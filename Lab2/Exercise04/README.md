#代码
这次换种方式，完整的代码贴出来一个一个的解释。首先解释几个概念：

1. 页目录表，页目录是内存虚拟地址映射到物理地址的起点，所有的映射都要从页目录开始，页目录常驻内存，页目录表的物理地址存在寄存器CR3中
2. 页目录项，页目录项中存放着二级页表的起始物理地址。
3. 页表，页表用于存放页表项。
4. 页表项，页表项用于将最终将虚拟地址映射到物理地址，页表项的高20位在放着物理页面的起始地址。

其实页目录和页表的结构是相同，页目录项和页表项的结构也是相同的，只是一个是一级，一个是二级的不同。

###pgdir_walk用于查找一个虚拟内存对应的页表项，如果页表不存在，则根据需要可以创建一个
```
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{

    uint32_t pdx = PDX(va); //页目录索引
    uint32_t ptx = PTX(va); //页表索引
    pde_t * pde;            //页目录项
    pte_t * pte;            //页表项
    struct PageInfo *pp;    //物理页数据结构

    pde = &pgdir[pdx];      //获取页目录项
    if(*pde & PTE_P)        //判断页目录项是否有效
    { 
        pte = (KADDR(PTE_ADDR(*pde)));               //获取页表项
    }
    else
    {
        if (!create)
        {
            return NULL;
        }
        if(!(pp = page_alloc(ALLOC_ZERO)))          //创建新的页表
        {
            return NULL;
        }

        pte = (pte_t *)page2kva(pp);                //设置页表指针
        pp->pp_ref++;                               //增加引用计数
        *pde = PADDR(pte) | PTE_P | PTE_W | PTE_U;  //设置页目录项
    }   

    return &pte[ptx];                               //返回页表项
}
```
###page_lookup用于查找一个虚拟页对应的物理页的数据结构
```
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
    pte_t * pte = pgdir_walk(pgdir, va, 0);         //查找页表项

    if (!pte)
    {
        return NULL;
    }

    if (pte_store)
    {
        *pte_store = pte;                           //通过指针的指针返回指针给调用者(自己听着都觉得绕^_^)
    }

    return pa2page(PTE_ADDR(*pte));                 //返回物理页数据结构(我真想叫你物理页描述符，就怕别人不懂)
}
```
###page_remove用于移除一个虚拟页到物理页的照射
```
void
page_remove(pde_t *pgdir, void *va)
{
    pte_t *pte;
    pte_t **pte_store = &pte;
    struct PageInfo *pp = page_lookup(pgdir, va, pte_store); //查找物理页数据结构
    if(!pp)
        return;

    page_decref(pp);                                         //减引用计数

    **pte_store = 0;                                         //取消映射
    tlb_invalidate(pgdir, va);
}
```
###page_insert用于增加一个虚拟页到物理页的映射到页表
```
int
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
    pte_t *pte = pgdir_walk(pgdir, va, 1);       //寻找虚拟地址的页表项，如果页表不存在，则创建一个页表
    if (!pte)
    {
        return -E_NO_MEM;
    }
    if (*pte & PTE_P)                            //如果页表项已经遇到一个物理内存页
    {
        // reinsert same page to same va
        if (PTE_ADDR(*pte) == page2pa(pp))       //如果当前页跟要映射的物理页是同一个页面
        {
            *pte = page2pa(pp) | (perm|PTE_P);   //更新权限即可
            return 0;
        }
        else
        {
            page_remove(pgdir, va);              //如果不是同一个页面，则删除对当前虚拟内存的照射
        }
    }
    *pte = page2pa(pp) | perm | PTE_P;           //设置页表项
    pp->pp_ref++;                                //更新物理页引用计数

    return 0;
}
```
###boot_map_region映射一片指定的虚拟页到指定的物理页
```
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
    uintptr_t pva = va;
    physaddr_t ppa = pa;
    pte_t *pte;

    for (; pva < va+size; pva+=PGSIZE, ppa+=PGSIZE) //循环做照射
    {
        pte = pgdir_walk(pgdir, (void *)pva, 1);    //查找页表项
        if (!pte)
        {
            return;
        }
        *pte = PTE_ADDR(ppa) | perm | PTE_P;        //设置页表项
    }
}
```
