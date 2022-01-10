# 优于select的epoll
==epoll方式只在Linux下支持，相比而言，select大部分操作系统都支持。==
## 1.实现epoll时必要的函数和结构体
##### 1.克服select函数缺点的epoll函数具有如下优点：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-e84fced3eb650320.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 2.epoll_create函数
1. 函数功能：创建保存epoll文件描述符的空间，这个函数创建的资源和套接字一样，由操作系统管理。
2. 函数声明
```
#include <sys/epoll.h>
int epoll_create(int size);
函数返回值：
    成功时返回epoll文件描述符（树的根节点），失败时返回-1
函数参数：
    epoll实例的大小，内核监听的数目（该参数值只是向操作系统提的建议）
```
##### 3.epoll_ctl函数
1. 函数功能：向空间注册或者注销文件描述符等（操纵epoll例程）
2. 函数声明
```
#include <sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);
函数返回值：
    成功时返回0，失败时返回-1
函数参数：
    epfd:用于注册监视对象的epoll例程的文件描述符
    op:用于指定监视对象的添加、删除、更改等操作
    fd:监视对象文件描述符
    event：监视对象的事件类型
```
3. 该函数调用示例
```
// 含义为：epoll例程A中注册文件描述符B，主要目的是监视参数C中的事件
epoll_ctl(A, EPOLL_CTL_ADD, B, C);
```
4. 可以向op形参传递的常量
![image.png](https://upload-images.jianshu.io/upload_images/17728742-e835bd6a5fce4b4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 4.epoll_event结构体
1. epoll_event结构体将发生事件的文件描述符单独集中在一起
```
struct epoll_event
{
    __uint32_t events;
    epoll_data_t data;
}
typedef union epoll_data
{
    void* ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
}epoll_data_t
```
2. events中可以保存的常量及其所指的事件类型
![image.png](https://upload-images.jianshu.io/upload_images/17728742-c894711ce5daf16a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 5.epoll_wait函数
1. 函数功能：与select函数类似，在一段超时时间内等待一组文件描述符上的事件。
2. 函数声明
```
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
函数返回值：
    成功时返回发生事件的文件描述符的数量
    失败时返回-1
函数参数：
    epfd:表示事件发生监视范围的epoll例程的文件描述符
    events:保存发生事件（epoll_wait检测到的就绪事件）的文件描述符集合的结构体地址值
    maxevents:第二个参数中可以保存的最大事件数（指定最多监听多少个事件）
    timeout:以1/1000秒为单位的等待时间，传递-1时表示一直等待直到发生事件
    0表示立即返回
```
==第二个参数所指缓冲需要动态分配。==
##### 6.基于epoll的回声服务器端
```
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>
#include <unistd.h>
#define EPOLL_SIZE 50

// 保存地址信息
typedef struct SockInfo
{
    int fd;
    struct sockaddr_in addr;
}SockInfo;

int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <PORT>", argv[0]);
        return -1;
    }
    int serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    int cli_sock;

    struct sockaddr_in serv_addr, cli_addr;
    socklen_t len = sizeof(struct sockaddr_in);
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(atoi(argv[1]));
    bind(serv_sock, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));

    listen(serv_sock, 5);

    // 创建保存epoll文件描述符的空间,一个epoll实例
    // epfd为一个树的根节点
    int epfd = epoll_create(EPOLL_SIZE);   
    struct epoll_event* epoll_events;
    epoll_events = (struct epoll_event*)malloc(sizeof(struct epoll_event) * EPOLL_SIZE);

    struct epoll_event event;
    event.events = EPOLLIN;
    SockInfo info = {serv_sock, serv_addr};
    event.data.ptr = (void*)&info;
    // 注册监听套接字的文件描述符
    epoll_ctl(epfd, EPOLL_CTL_ADD, serv_sock, &event);
    int event_cnt;

    ssize_t read_len;
    char buff[30];
    while (1)
    {
        // 等待文件描述符发生变化
        event_cnt = epoll_wait(epfd, epoll_events, EPOLL_SIZE, -1);
        if (event_cnt == -1)
        {
            puts("epoll_wait()error");
            break;
        }
        for (int i = 0; i < event_cnt; i++)
        {
            SockInfo* infop = (SockInfo*)epoll_events[i].data.ptr;
            // 客户端连接到来
            if (infop->fd == serv_sock)
            {
                cli_sock = accept(serv_sock, (struct sockaddr*)&cli_addr, &len);
                SockInfo info = {cli_sock, cli_addr};
                event.data.ptr = (void*)&info;
                event.events = EPOLLIN;
                // 注册新创建的文件描述符（用于与客户端通信的套接字文件描述符）
                epoll_ctl(epfd, EPOLL_CTL_ADD, cli_sock, &event);
                printf("connected client:%d ip:%s port:%d\n",
                    cli_sock, 
                    inet_ntoa(info.addr.sin_addr),
                    ntohs(info.addr.sin_port));
            }
            // 通信
            else
            {
                // 判断是否为读事件发生,不是读事件发生就退出本次循环
                if (!epoll_events[i].events & EPOLLIN)
                {
                    continue;
                }
                read_len = read(infop->fd, buff, sizeof(buff));
                // 客户端断开连接
                if (read_len == 0)
                {
                    epoll_ctl(epfd, EPOLL_CTL_DEL, infop->fd, NULL);
                    printf("closed client:%d ip:%s port:%d\n", 
                    infop->fd, 
                    inet_ntoa(infop->addr.sin_addr),
                    ntohs(infop->addr.sin_port));
                    close(infop->fd);
                }
                else
                    write(infop->fd, buff, read_len);
            }
        }
    }

    close(epfd);
    close(serv_sock);
    free(epoll_events);

    return 0;
}
```
服务端运行如下：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-565a4f850042972d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.epoll的三种工作模式
##### 1.水平触发模式（epoll的默认工作模式，Level Trigger）
1. 根据读来解释。只要服务端fd对应的缓冲区有数据，epoll_wait就会返回。返回的次数和客户端发送数据的次数没有关系。==又称条件触发，条件触发模式中，只要输入缓冲中有数据epoll_wait就会一直通知该事件。== epoll_wait调用次数越多，系统的开销越大。
2. 示例
```
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <unistd.h>
#define EPOLL_SIZE 1024
int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <port>\n", argv[1]);
        return -1;
    }

    int serv_sock, clnt_sock;
    // 创建TCP服务端套接字
    serv_sock = socket(PF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr, clnt_addr;
    socklen_t clnt_addr_size = sizeof(clnt_addr);
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(atoi(argv[1]));
    
    bind(serv_sock, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));

    listen(serv_sock, 5);

    int epfd = epoll_create(EPOLL_SIZE);
    struct epoll_event event;
    event.events = EPOLLIN;
    event.data.fd = serv_sock;
    epoll_ctl(epfd, EPOLL_CTL_ADD, serv_sock, &event);

    int event_cnt;
    struct epoll_event events[EPOLL_SIZE];
    char buff[5];
    ssize_t read_len;

    while (1)
    {
        if ((event_cnt = epoll_wait(epfd, events, EPOLL_SIZE, -1)) == -1)
        {
            puts("epoll_wait error!");
            break;
        }
        puts("epoll_wait()called!");
        for (int i = 0; i < event_cnt; i++)
        {
            // 新的客户端连接到来
            if (events[i].data.fd == serv_sock)
            {
                clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_addr, &clnt_addr_size);
                event.events = EPOLLIN;
                event.data.fd = clnt_sock;
                epoll_ctl(epfd, EPOLL_CTL_ADD, clnt_sock, &event);
                printf("new connection:%d\n", clnt_sock);
            }
            // 通信
            else
            {
                if (!events[i].events & EPOLLIN)
                {
                    continue;
                }
                read_len = recv(events[i].data.fd, buff, sizeof(buff), 0);
                // 客户端断开连接
                if (read_len == 0)
                {
                    printf("connection closed:%d\n", events[i].data.fd);
                    epoll_ctl(epfd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                    close(events[i].data.fd);
                }
                else
                    // 回声
                    send(events[i].data.fd, buff, read_len, 0);
            }
        }
    }

    close(epfd);
    close(serv_sock);

    return 0;
}
```
![image.png](https://upload-images.jianshu.io/upload_images/17728742-d75624f603bad144.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/17728742-8f6d509e662b0032.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 2.边沿触发模式（Edge Trigger，边沿触发阻塞模式）
1. 客户端发送一次数据，服务端的epoll_wait返回一次。不在乎服务端套接字的输入缓冲区中的数据是否读完。==即边缘触发中输入缓冲收到数据时仅仅注册一次该事件。即使输入缓冲中还留有数据，也不会进行注册。==
2. 数据读不完怎么办？循环读，但是也有问题，数据读完之后recv存在阻塞的问题。
```
// 循环读
while (recv())
{
    ...
}
```
3. 示例
```
将上述示例代码中从event.events = EPOLLIN;
改为event.events = EPOLLIN | EPOLLET;
```
![image.png](https://upload-images.jianshu.io/upload_images/17728742-c1289e65a7380dcf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/17728742-e7fdf086928473e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 3.边沿非阻塞模式
1. 三种工作模式中效率最高。因为ET模式在很大程度上降低了同一个epoll事件被触发的次数，因此效率比LT模式高。
2. 将套接字改为非阻塞的方法：使用fcntl函数
```
#include <fcntl.h>
int fcntl(int filedes, int cmd,...);
函数返回值：
    成功时返回cmd参数相关值，失败时返回-1
函数参数：
    filedes:属性更改目标的文件描述符
    cmd:表示函数调用目的
        向其传递F_GETFL:表示获取第一个参数所指的文件描述符属性
        向其传递F_SETFL:表示更改文件描述符属性

// 示例如下：
// 获取之前设置的属性信息
int flag = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flag | O_NONBLOCK);
// 通过上述这两条语句后，调用read/write函数时，无论是否存在数据，都会形成非阻塞套接字
```
3. 通过errno变量验证错误原因
```
#include <error.h>
// 每种函数发生错误时，保存到errno变量的值都不同
// 比如说read函数发现输入缓冲中没有数据可读时返回-1，
// 同时在errno中保存EAGAIN
```
==通过errno变量确认错误原因：边缘触发模式中，接收数据时仅仅注册一次该事件。所以一旦服务端中发生输入事件，就应该读取输入缓冲中的全部数据，因此需要验证输入缓冲是否为空。==
4. 边沿触发非阻塞模式的回声服务器端如下：
```
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#define EPOLL_SIZE 50

// 保存地址信息
typedef struct SockInfo
{
    int fd;
    struct sockaddr_in addr;
}SockInfo;

int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <PORT>", argv[0]);
        return -1;
    }
    int serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    int cli_sock;

    struct sockaddr_in serv_addr, cli_addr;
    socklen_t len = sizeof(struct sockaddr_in);
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(atoi(argv[1]));
    bind(serv_sock, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));

    listen(serv_sock, 5);

    // 创建保存epoll文件描述符的空间,一个epoll实例
    // epfd为一个树的根节点
    int epfd = epoll_create(EPOLL_SIZE);   
    struct epoll_event* epoll_events;
    epoll_events = (struct epoll_event*)malloc(sizeof(struct epoll_event) * EPOLL_SIZE);

    struct epoll_event event;
    event.events = EPOLLIN;
    SockInfo info = {serv_sock, serv_addr};
    event.data.ptr = (void*)&info;

    // 将监听套接字设置为非阻塞的
    int flags = fcntl(serv_sock, F_GETFD);
    fcntl(serv_sock, F_SETFD, flags | O_NONBLOCK);

    // 注册监听套接字的文件描述符
    epoll_ctl(epfd, EPOLL_CTL_ADD, serv_sock, &event);
    int event_cnt;

    ssize_t read_len;
    char buff[30];
    while (1)
    {
        // 等待文件描述符发生变化
        event_cnt = epoll_wait(epfd, epoll_events, EPOLL_SIZE, -1);
        if (event_cnt == -1)
        {
            puts("epoll_wait()error");
            break;
        }
        puts("return epoll_wait()!");
        for (int i = 0; i < event_cnt; i++)
        {
            SockInfo* infop = (SockInfo*)epoll_events[i].data.ptr;
            // 客户端连接到来
            if (infop->fd == serv_sock)
            {
                cli_sock = accept(serv_sock, (struct sockaddr*)&cli_addr, &len);
                
                // 将通信套接字设置为非阻塞的
                int flags = fcntl(cli_sock, F_GETFD);
                fcntl(cli_sock, F_SETFD, flags | O_NONBLOCK);

                SockInfo info = {cli_sock, cli_addr};
                event.data.ptr = (void*)&info;
                event.events = EPOLLIN | EPOLLET;
                // 注册新创建的文件描述符（用于与客户端通信的套接字文件描述符）
                epoll_ctl(epfd, EPOLL_CTL_ADD, cli_sock, &event);
                printf("connected client:%d ip:%s port:%d\n",
                    cli_sock, 
                    inet_ntoa(info.addr.sin_addr),
                    ntohs(info.addr.sin_port));
            }
            // 通信
            else
            {
                // 判断是否为读事件发生,不是读事件发生就退出本次循环
                if (!epoll_events[i].events & EPOLLIN)
                {
                    continue;
                }
                // 循环读
                while (1)
                {
                    read_len = read(infop->fd, buff, sizeof(buff));
                    // 客户端断开连接
                    if (read_len == 0)
                    {
                        epoll_ctl(epfd, EPOLL_CTL_DEL, infop->fd, NULL);
                        printf("closed client:%d ip:%s port:%d\n", 
                        infop->fd, 
                        inet_ntoa(infop->addr.sin_addr),
                        ntohs(infop->addr.sin_port));
                        close(infop->fd);
                        break;
                    }
                    // 输入缓冲中没有数据可读
                    else if (read_len == -1)
                    {
                        if (errno == EAGAIN)
                        {
                            break;
                        }
                    }
                    else
                        write(infop->fd, buff, read_len);
                    
                }
                
            }
        }
    }

    close(epfd);
    close(serv_sock);
    free(epoll_events);

    return 0;
}
```
## 3.文件描述符突破1024限制
##### 1.select
通过数组实现。突破不了，需要编译内核
##### 2.poll和epoll
可以突破1024限制。poll内部链表，epoll红黑树。

查看受计算机硬件限制的文件描述符上限
```
cat /proc/sys/fs/file-max
// 9223372036854775807
```
通过配置文件/etc/security/limits.conf修改上限值









