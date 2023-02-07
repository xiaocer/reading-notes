# I/O复用

本节要点：

1. 熟悉select并且使用select实现一个简易的IO复用服务器端
2. 了解poll
3. 知道IO复用技术之一的select的优缺点

## 1.基于IO复用的服务器端
IO复用技术：可以在不创建多个进程的同时向多个客户端提供服务。
1. 多进程服务器端模型
![image.png](https://upload-images.jianshu.io/upload_images/17728742-ca31a50b80e6b0d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	1. 多进程服务器的缺点是：
		1. 创建进程代价大
		2. 进程间通信相对复杂
2. I/O复用服务器端模型
![image.png](https://upload-images.jianshu.io/upload_images/17728742-de81893ee8e44ace.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.select函数
##### 1.功能
使用select函数可以将多个文件描述符集中到一起统一监视。
##### 2.函数原型
```c
#include <sys/select.h>
#include <sys/time.h>
int select(int maxfd, fd_set* readset,
fd_set* writeset, fd_set* exceptset,
const struct timeval* timeout);
函数返回值：
    成功时返回大于0的值（就绪文件描述符的总数），失败时返回-1，超时返回时返回0
函数参数：
    maxfd:表示监视对象文件描述符的数量
    readset:将所有关注是否存在待读取数据的文件描述
    符注册到fd_set变量，并传递其地址值
    writeset:将所有关注是否可传输无阻塞数据的文件描述
    符注册到fd_set变量，并传递其地址值
    exceptset:将所有关注是否发生异常的文件描述
    符注册到fd_set变量，并传递其地址值
    timeout:调用select函数后，为防止陷入无限阻塞的状态，传递超时信息.
    过了指定时间，就可以从该函数中返回
```
##### 3.该函数调用步骤
![image.png](https://upload-images.jianshu.io/upload_images/17728742-1f578c7b426a1ce6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. 设置文件描述符：将要监视的文件描述符集中到一起，集中时按照监视项进行区分，分为接收、异常、传输这三类。==fd_set数组是一个存储了0和1的长整型数组，一共1024个位。对fd_set数组变量的操作以位为单位进行，如果某个位设置为1，则表示该文件描述符是监视对象。（最左端的位表示文件描述符0）== 在fd_set变量中注册或者更改值的操作由下列宏表示

![image.png](https://upload-images.jianshu.io/upload_images/17728742-098e96ce94fcfaba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
==其中FD_ISSET函数用于验证select函数的调用结果，select函数会在监视的文件描述符发生变化时返回==
2. 设置监视范围：将最大的文件描述符值加1作为第一个参数。（文件描述符值从0开始）
![image.png](https://upload-images.jianshu.io/upload_images/17728742-c43836b18d74c6c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 设置超时时间:将秒数填入结构体的第一个成员，微秒数填入第二个成员。
```
struct timeval
{
    long tv_sec; // seconds
    long tv_usec;// microseconds
}
```
4. 调用select函数后查看结果
5. select函数调用示例：读取控制台输入的数据，标准输入对应文件描述符0
```
#include <cstdio>
#include <sys/select.h>
#include <unistd.h> // for read()
int main()
{
    fd_set reads, temps;
    // reads数组每一位置为0
    FD_ZERO(&reads);
    // 监视标准输入的变化
    FD_SET(0, &reads);  // 0是控制台标准输入，将文件描述符0对应的位设置为1
    struct timeval timeout;
    int result;
    char buff[30];
    while (1)
    {
        temps = reads;
        // 设置超时时间
        timeout.tv_sec = 5;
        timeout.tv_usec = 0;
        result = select(1, &temps, nullptr, nullptr, &timeout);
        if (result == -1)
        {
            break;
        }
        else if (result == 0)
        {
            puts("time out!");
        }
        else
        {
            // 验证select函数的调用结果
            if (FD_ISSET(0, &temps))
            {
                ssize_t read_len = read(0, buff, sizeof(buff) - 1);
                buff[read_len] = '\0';
                printf("message from console:%s", buff);
            }
        }
    }

    return 0;
}
```
![image.png](https://upload-images.jianshu.io/upload_images/17728742-54193dcac6b146b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 4.select函数的优缺点
1. 优点：跨平台
2. 缺点
```
1. 每次调用select，都需要将fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大。调用成功返回后，还需要将fd集合从内核态拷贝到用户态。
2. 每次调用select时都需要在内核遍历传递进来的所有fd,这个开销在fd很多时会很大。
3. select支持的文件描述符数量太小了，默认是1024。如果需要更改，需要重新编译内核。
```
##### 5.实现I/O复用回声服务器端
```c
#include <stdio.h>
#include <sys/socket.h>
#include <stdlib.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/select.h>
#include <sys/time.h>
#include <unistd.h>

int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <PORT>", argv[0]);
        return -1;
    }
    int serv_sock, clnt_sock;
    serv_sock = socket(PF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr, clnt_addr;
    socklen_t clnt_addr_len = sizeof(clnt_addr);
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(atoi(argv[1]));
    bind(serv_sock, (const sockaddr*)&serv_addr, sizeof(serv_addr));

    listen(serv_sock, 5);

    fd_set reads, temps;
    FD_ZERO(&reads);
    FD_SET(serv_sock, &reads);
    struct timeval time; 
    int fd_max = serv_sock, fd_num;

    char buff[30] = {0};
    ssize_t len;
    char dest_ip[INET_ADDRSTRLEN];
    while (1)
    {
        temps = reads;
        time.tv_sec = 5;
        time.tv_usec = 0;
        if ((fd_num = select(fd_max + 1, &temps, NULL, NULL, &time)) == -1)
            break;
        else if (fd_num == 0)
        {
            printf("超时返回\n");
            continue;
        }
        for (int i = 0; i < fd_max + 1; i++)
        {
            if (FD_ISSET(i, &temps))
            {
                // 连接请求
                if (i == serv_sock)
                {
                    clnt_sock = accept(serv_sock, (sockaddr*)&clnt_addr, &clnt_addr_len);
                    FD_SET(clnt_sock, &reads);
                    if (fd_max < clnt_sock)
                        fd_max = clnt_sock;
                    printf("connected client:%d, IP:%s, port:%d\n",
                        clnt_sock,
                        inet_ntop(AF_INET, (const void*)&clnt_addr.sin_addr.s_addr, dest_ip, sizeof(dest_ip)),
                        ntohs(clnt_addr.sin_port));                   
                }
                // 通信
                else
                {
                    len = read(i, buff, sizeof(buff));
                    // 客户端断开连接
                    if (len == 0)
                    {
                        FD_CLR(i, &reads);
                        close(i);
                        printf("closed client:%d\n", i);
                    }
                    else if (len > 0) {
                        printf("message from client:%s\n", buff);
                        write(i, buff, len);    // 回声
                        memset(buff, 0, sizeof(buff));
                    }
                }
            }   
        }
    }
    close(serv_sock);
}
```
## 3.poll函数
##### 1.功能
和select一样。区别比如说select函数中一个描述符对应一个比特位，而poll函数中一个文件描述符对应一个结构体。
##### 2.函数原型
```
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
函数返回值：IO发生变化的文件描述符的个数
函数参数：
    fds:监视的文件描述符集合
    nfds:表示监视对象文件描述符的数量
    timeout:
        -1：永久阻塞
        0：调用完成立即返回
        >0:等待的时长
```
##### 3.pollfd结构体
```
struct pollfd {
   int   fd;         /* file descriptor */
   // 等待的事件
   short events;     /* requested events */
   // 实际发生的事件
   short revents;    /* returned events */
};
```
![image.png](https://upload-images.jianshu.io/upload_images/17728742-6aa06be822a6ec06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
