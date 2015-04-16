#原题如下：
```
Assuming that the following JOS kernel code is correct, what type should variable x have, uintptr_t or physaddr_t?
	mystery_t x;
	char* value = return_a_pointer();
	*value = 10;
	x = (mystery_t) value;
```

#解答
是uintprt_t
第三句对*value进行了赋值，所以value肯定是一个虚拟地址，因为直接解引用物理地址是没有意义的。

那么x肯定也是虚拟地址，要不然将虚拟地址赋值给一个物理地址也没有意义的。
