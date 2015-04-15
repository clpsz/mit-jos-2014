
#环境搭建
建议使用ubuntu系统建行实验，需要安装的软件是qemu-system，直接sudo apt-get install qemu-system即可。我尝试用CentOS安装qemu，结果一堆依赖的问题，又不想自己去编译，最后就放弃了。Lab里面一直说要用MIT打过补丁的qemu，不过Lab1里面好像没有用到。

#下载代码
```
$git clone http://pdos.csail.mit.edu/6.828/2014-jos.git lab
$cd lab
```
#qemu运行
###正常运行
```
$make qemu
```
###调试运行
需要开两个终端，一个以调试模式启动qemu，启动后qemu不会立即开始运行指令，而是等待gdb连接后通过gdb控制qemu的运行
```
$make qemu-gdb
```
另一个运行gdb，并输入调试命令，进行调试
```
$make gdb
```


#评分
```
$make grade
```
会自动编译内核，运行，并查看是否满足Lab要求，修改代码后建议运行一下看是否正确
