# 进程间通信
## 1.进程间通信的基本概念
1. 进程间通信（inter process communication）：不同进程之间交换数据，简称IPC。为了实现这一点，OS应当提供两个进程可以同时访问的内存空间，比如说管道。
## 2.通过管道实现进程间通信
管道并非属于进程的资源，他和套接字一样属于操作系统。
##### 1.通过管道进行进程间单向通信
1. 创建管道的函数
```
#include <unistd.h>
int pipe(int fileds[2]);
函数返回值：
    成功时返回0，失败返回-1
函数参数：
    fileds[0]表示从管道接收数据时使用的文件描述符，即管道出口
    fileds[1]表示管道传输数据时使用的文件描述符，即管道入口
```
2. 示例：
```
#include <cstdio>
#include <unistd.h> // for pipe()
#include <wait.h>

int main()
{
    char buff[30] = "Hello,World!世界";
    // 创建管道
    int pipedes[2];
    pipe(pipedes);
    // 创建子进程
    pid_t child_id = fork();
    // 子进程
    if (child_id == 0)
    {
        // 往管道写入数据
        write(pipedes[1], buff, sizeof(buff));

    }
    // 父进程
    else if (child_id > 0)
    {
        // 读取管道的数据
        read(pipedes[0], buff, sizeof(buff));
        puts(buff);
        waitpid(-1, nullptr, WNOHANG);
    }
    return 0;
}
```
==上述示例中，父进程用于从管道出口读取数据，子进程向管道写入数据。==
##### 2.通过管道进行进程间双向通信
1.示例一：通过一个管道进行双向通信，双向通信模型如下：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-540c590bba735e71.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
==这种方式的问题：一个进程向管道写入数据，无法控制是哪一个进程从管道读取数据。==

2.示例二：创建两个管道进行双向通信,双向通向模型如下：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-ee3e6f6e0317e3be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
#include <cstdio>
#include <unistd.h>

int main()
{
    int pipedes1[2], pipedes2[2];
    // 创建两个管道
    pipe(pipedes1);
    pipe(pipedes2);

    char mess1[30] = "Hello,who are you?";
    char mess2[30] = "hi, I am NrvCer!";
    char buff[30];
    // 创建子进程
    pid_t child_id = fork();
    if (child_id == 0)
    {
        write(pipedes1[1], mess1, sizeof(mess1));
        read(pipedes2[0], buff, sizeof(buff));
        printf("child process recieve:%s\n", buff);
    }
    else if (child_id > 0)
    {
        read(pipedes1[0], buff, sizeof(buff));
        printf("parent process recieve:%s\n", buff);
        write(pipedes2[1], mess2, sizeof(mess2));
    }
    return 0;
}
```
==上述示例中，子进程通过管道1向父进程传输数据，父进程通过管道2向子进程传输数据。==
##### 3.运用进程间通信示例
对chapter10中的回声服务器端进行改进：将回声客户端传输的字符串按序保存到文件中。
