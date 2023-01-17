## 1.Intel80*86内存模型以及它的工作原理
1. 段的概念：在Intel`80*86`内存模型中，段是内存模型设计的结果，在`80*86`的内存模型中，各个处理器的地址空间并不一致，但是他们都被分割成以64KB为单位的区域。每个这样的区域称为段。
## 2.虚拟内存
1. 虚拟内存的基本思路：使用廉价但是缓慢的磁盘来扩充快速却昂贵的内存。在任意给定时刻，程序实际需要使用的虚拟内存区段的内容被载入物理内存当中，当物理内存中的数据有一段时间未被使用，他们就可能转移到硬盘中，节省下来的物理内存用于载入需要使用的其他数据。
2. 虚拟内存通过页的形式组织。页就是操作系统在磁盘和内存之间移来移去或者进行保护的单位，一般为几K字节
3. 与进程有关的所有内存都将被系统所使用，如果该进程可能不会马上运行，OS可能暂时取回所有分配给他的物理内存资源，将该进程的所有相关信息都备份到磁盘上。在磁盘中有一个特殊的交换区，用于保存从内存中换出的进程。在一台机器中交换区的内存一般是物理内存的几倍。只有用户进程才会被换进换出，内存常驻于内存中。
## 3.Cache存储器
1. 所有的现代处理器都使用了Cache存储器，当数据从内存读入时，整行的数据被装入Cache。如果程序具有良好的地址引用局部性，那么CPU以后对邻近数据的引用就可以从快速的Cache读取，而不用从缓慢的内存中读取。
2. Cache的相关术语
![image.png](https://upload-images.jianshu.io/upload_images/17728742-a179f3c2b34ec75a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 每个进程内部的内存布局
![image.png](https://upload-images.jianshu.io/upload_images/17728742-4257b7648f0f2e18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 4.堆
1. 堆经常会出现两种类型的问题：
    1. 内存损坏：释放或者改写正在使用的内存。
    2. 内存泄漏：未释放不再使用的内存。
## 5.总线错误
1. 当硬件告诉OS一个有问题的内存引用时，就会出现这两种错误。操作系统通过向出错的进程发送一个信号与之交流。信号就是一种事件通知或者软件中断。在缺省情况下进程收到总线错误或者段错误信号后将进行信息转储并终止。
##### 1.总线错误
1. 总线错误：几乎都是由于未对齐的读或者写引起的。对齐的意思就是数据项只能存储在地址是数据项大小的整数倍的内存位置上。
##### 2.段错误
1. 在Sun的硬件中，段错误是由于内存管理单元MMU（负责支持虚拟内存的硬件）的异常所致，而该异常则通常是由于解除引用一个未经初始化或者非法值的指针引起的。
```
int* ptr;
// Segmentation fault
// printf("%d\n", *ptr);

int* p = NULL;
// Segmentation fault
*p = 100;
```
2. 通常导致段错误的几个直接原因：
    1. 解除引用一个包含非法值的指针
    2. 解除引用一个空指针
    3. 在未得到正确的权限时进行访问。例如试图往一个只读的文本段存储值就会引起段错误。
    4. 用完了堆栈或者堆空间
3. 捕捉段错误信号的信号处理程序
```
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
void handler(int signal) {
    if (signal == SIGBUS) printf("get a bus error signal\n");
    else if (signal == SIGSEGV) printf("get a segmentation violation signal\n");
    else if (signal == SIGILL) printf("get a illegal instruction signal\n");
    exit(1);
}
int main() {
    int* p = NULL;
    //注册SIGSEGV信号的处理程序, Invalid access to storage
    signal(SIGSEGV, handler);
    signal(SIGBUS, handler);
    signal(SIGILL, handler);
    
    *p = 100;

    return 0;
}
```
4. 使用`setjmp`和`longjmp`函数从信号中恢复程序的运行。示例：运行下列程序，当按下ctrl + c（作为SIGINT信号）时程序将重新启动，而不是立即退出。
```
#include <setjmp.h>
#include <signal.h>
#include <stdio.h>

jmp_buf buf;
void handler(int signal) {
    if (signal == SIGINT) printf("get a SIGINT signal\n");
    longjmp(buf, 1);
}
int main() {
    signal(SIGINT, handler);
    // setjmp函数调用将返回0
    if (setjmp(buf)) {
        printf("back in main functon\n");
        return 0;
    } else {
        printf("first time through\n");
    }
    loop:
        goto loop;
    return 0;
}

// 运行结果
./a.out
first time through
^Cget a SIGINT signal
back in main functon
```