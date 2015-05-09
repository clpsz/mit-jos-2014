#总结
这个题目实在是太简单的，题目已经提示的足够清楚了，就是在trap_dispatch中判断一下当前的异常是不是缺页错误，如果是的话，就把这个异常传递给page_fault_handler来处理。

两行代码搞定。

#测试
使用 make run-faultwrite-nox命令可以让内核执行执行user/faultwrite.c中的代码(将经一个env初始化为faultwrite的代码)。
