## 1.strace
#### 1.系统调用
1. 系统调用：所有操作系统在其内核都有一些内建的函数，这些函数可以用来完成一些系统级别的功能，一般称Linux系统上的这些函数为系统调用。
2. 系统调用代表了用户空间到内核空间的一种转换。例如在用户空间上调用open函数，在内核空间则会调用sys_open函数。
##### 2.strace跟踪系统调用
1. 示例程序test.cpp如下：
```
#include <iostream>

int main() {
    int a;
    std::cin >> a;
    std::cout << a << std::endl;
    return 0;
}
```
2. 编译运行
```
g++ test.cpp 
strace ./a.out
```
## top命令
1. top命令是Linux下常用的性能分析工具，能够实时动态显示系统中各个进程的资源占用情况，类似于Windows中的任务管理器。
2. top命令结果的第一行：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-03621a80636a7738.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. 系统当前时间
    2. 系统运行时间
    3. 当前用户登录数
    4. 系统负载：这里有三个数值，分别表示系统最近1分钟、5分钟、15分钟的平均负载
2. top命令结果的第二行：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-59f82e528559fa08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. total：进程总数
    2. running：正在运行的进程总数
    3. sleeping：睡眠的进程数
    4. stopped：停止的进程数
    5. zombie：僵尸进程数目
3. top命令的第三行：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-2c4e45488ca48523.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. us：用户空间占用CPU百分比
    2. sy：内核空间占用CPU百分比
    3. ni：用户进程空间内改变过优先级的进程占用CPU百分比
    4. id：空闲CPU百分比
    5. wa：等待输入输出的CPU百分比
    6. hi：CPU处理硬件中断的时间
    7. si：CPU处理软件中断的时间
    8. st：用于有虚拟CPU的情况
4. top命令结果的第四行：显示物理内存的数据
![image.png](https://upload-images.jianshu.io/upload_images/17728742-7e958360afee710e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. total：物理内存总量
    2. used：已经使用的物理内存总量
    3. free：空闲内存总量
    4. buffers：用作内核缓存的内存量
5. top命令结果的第五行：显示交换区的数据
![image.png](https://upload-images.jianshu.io/upload_images/17728742-3b23f9ce03a80996.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. total：交换区总量
    2. used：已经使用的交换区总量
    3. free：空闲的交换区总量
    4. cached：缓冲的交换区总量
6. top命令结果的第六行：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-cc05f988a9ee08ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. pid：进程号
    2. usr：运行进程的用户
    3. pr：优先级（priority）
    4. ni：任务nice值，它反应一个进程优先级状态的值，其取值范围是-20到19，一共40个级别。这个值越小，表示进程优先级越高。==nice值虽然不是优先级，但是却可以影响到进程的优先级。==
    5. virt：虚拟内存使用量，一般等于交换区+RES的大小
    6. res：物理内存总量
    7. shr：共享内存总量
    8. %cpu：CPU占用比
    9. %mem：物理内存占用比
    10. time+：累积CPU占用时间
    11. command：命令行（命令名称）
## ps命令
1. ps命令列出当前在运行的进程的快照，就是执行ps命令的那个时刻的快照。
2. Linux上进程的多种状态：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-3571c2ddbf92fa5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. ps命令的结果的第一行相关信息如下：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-6ec013ce7ba09be8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. USER：运行进程的用户
    2. PID：进程id
    3. %CPU：进程使用掉的CPU资源百分比
    4. %MEM：该进程所占用的物理内存百分比
    5. vsz：该进程使用掉的虚拟内存量，KB为单位
    6. RSS：该进程占用的固定的内存量。Resident Set Size
    7. TTY：登录者的终端机的位置。如果与终端机无关则显示？;tty1~tty6是本机上面的登录者程序；如果为pts/x等，则表示为网络连接进主机的程序。
    8. STAT：该进程目前的状态
    9. START：该进程被触发启动的时间
    10. TIME：该进程实际占用CPU运作的时间
    11. COMMAND：运行该程序的命令名称
4. ps命令常用的命令选项：
 ![image.png](https://upload-images.jianshu.io/upload_images/17728742-6566d0eb8d39486e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. ps -aux | grep 进程名：显式指定进程信息
    2. ps -u 用户名：显示指定用户信息
## Valgrind
1. Valgrind：是一套Linux下的开放源代码的仿真调试工具的集合。它由内核以及基于内核的其他调试工具组成。
![image.png](https://upload-images.jianshu.io/upload_images/17728742-21b398f6a9f6e38f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 程序的内存空间布局
![image.png](https://upload-images.jianshu.io/upload_images/17728742-4a14f1a210bc338c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. 代码段：这部分区域的大小在程序运行前就已经确定，通常属于只读的
    2. 初始化的数据段：存放程序已经初始化的全局变量的一块内存区域。例如位于函数之外的全局变量，函数中初始化的静态变量
    3. 未初始化的数据段：通常指用来存放程序中未初始化的全局变量的一块内存区域。BSS全称Block Started By Symbol
    4. 堆：堆被用来存放进程运行中被动态分配的内存段，堆的大小可以动态的扩张和缩小。==堆是向高地址扩展的数据结构==
    5. 栈：又称堆栈，存放程序的局部变量。==栈是向低地址扩展的数据结构==
        1. 使用ulimit -a命令可以查看栈的大小限制
3. Memcheck的内存检测原理：关键在于其建立了两个全局表，一个Valid-Value表，另一个Valid-Address表。
![image.png](https://upload-images.jianshu.io/upload_images/17728742-9acaad60dfa9336d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
4. Valgrind的安装：
    1. 下载安装包：`wget https://sourceware.org/pub/valgrind/valgrind-3.20.0.tar.bz2`
    2. 解压：tar xvf valgrind-3.20.0.tar.bz2
    3. 进入解压后的目录执行`./configure --prefix=/usr/bin`
    4. 执行make进行编译
    5. make install进行安装
    6. 配置环境变量：
        1. vim ~/.bashrc
        2. 向bashrc文件写入：export PATH=$PATH:/usr/bin/bin/
        3. 使得bashrc文件的改变重新生效：source ~/.bashrc
    7. 可能报错Fatal error at startup: a function redirection：sudo apt-get install libc6-dbg
5. Valgrind的基本使用
    1. 内存读写越界（访问非法内存）：Valgrind默认使用的工具是Memcheck
    ```
    #include <iostream>

    int main() {
        int a;
        std::cin >> a;
        std::cout << a << std::endl;
        return 0;
    }
    // 编译,-g生成调试信息
    g++ test.cpp -g -o test
    valgrind ./test
    ```
    2. 使用未初始化的内存
    ```
    int a[5];
    int b = a[4];
    ```
    3. 内存覆盖
    4. 动态内存（堆）管理错误：
        1. 申请和释放的方式不一致：例如使用malloc申请内存，却用delete释放
        2. 申请和释放的量不匹配，可能造成内存泄漏或者释放已经被释放过的内存
        3. 释放内存后仍然读写
    