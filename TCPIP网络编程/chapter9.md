# 套接字的多种可选项
## 1.套接字的多种可选项
![image.png](https://upload-images.jianshu.io/upload_images/17728742-58cc5652703db755.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/17728742-2f654c2312757746.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. SOL_SOCKET层是套接字相关的通用可选项
2. IPPROTO_IP层可选项是IP协议相关事项
3. IPPROTO_TCP层可选项是TCP协议相关的事项
## 2.可选项的读取和设置
1. 可选项的读取
```
示例：读取套接字可选项的函数
#include <sys/socket.h>
int getsockopt(int sock, int level, int optname, void* optval, socklen_t *optlen);
函数返回值：
    成功时返回0，失败时返回-1
函数参数：
    sock:用于查看可选项的套接字文件描述符
    level：要查看的可选项的协议层
    optname:要查看的可选项名
    optval:保存查看结果的缓冲地址值
    optlen:第四个参数的大小
```
2. 可选项的设置
```
#include <sys/socket.h>
int setsockopt(int sock, int level, int optname, const void* optval, socklen_t optlen);
函数返回值：
    成功时返回0，失败时返回-1
函数参数：
    sock:用于更改可选项的套接字文件描述符
    level：要更改的可选项的协议层
    optname:要更改的可选项名
    optval:保存要更改的选项信息的缓冲地址值
    optlen:第四个参数的大小
```
## 3.输入缓冲大小和输出缓冲大小可选项
创建套接字将同时生成I/O缓冲。SO_RCVBUF是输入缓冲大小相关可选项，SO_SNDBUF是输出缓冲大小的相关可选项。
```
// 示例：读取、更改创建套接字时的默认的I/O缓冲区大小

#include <cstdio>
#include <sys/socket.h>
#include <netdb.h>

int main()
{
    // 创建TCP套接字
    int sockfd = socket(PF_INET, SOCK_STREAM, 0);
    int recv_buf, send_buf;
    socklen_t recv_len = sizeof(int), send_len = sizeof(int);
    // 读取TCP套接字输入输出缓冲区的大小
    getsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, (void*)&recv_buf, &recv_len);
    printf("before alter,输入缓冲区大小为:%d\n", recv_buf);

    getsockopt(sockfd, SOL_SOCKET, SO_SNDBUF, (void*)&send_buf, &send_len);
    printf("before alter,输出缓冲区大小为:%d\n", send_buf);
    
    // 设置TCP套接字输入输出缓冲区的大小
    int alter_in = 1024, alter_out = 1024; 
    setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, (void*)&alter_in, sizeof(alter_in));
    setsockopt(sockfd, SOL_SOCKET, SO_SNDBUF, (void*)&alter_out, sizeof(alter_out));
    
    getsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, (void*)&recv_buf, &recv_len);
    printf("after alter,输入缓冲区大小为:%d\n", recv_buf);

    getsockopt(sockfd, SOL_SOCKET, SO_SNDBUF, (void*)&send_buf, &send_len);
    printf("after alter,输出缓冲区大小为:%d\n", send_buf);
    return 0;
}

// before alter,输入缓冲区大小为:131072
// before alter,输出缓冲区大小为:16384
// after alter,输入缓冲区大小为:2304
// after alter,输出缓冲区大小为:4608
```
==缓冲区的大小设置需要谨慎处理，因为不会完全按照我们的要求进行。==
## 4.可选项SO_REUSEADDR及Time-wait状态
##### 1.Time-wait状态
主机A向主机B发送FIN消息，表示断开连接。套接字经过四次挥手过程后并非立即消除，而是要经过一段时间的Time-wait状态。因此先发送FIN消息的主机将会经过Time-wait状态。

==问题：服务器端先断开连接，为什么无法立即重新运行？==
```
因为先断开连接的主机还要经过一段时间的Time-wait状态，
（必须等待几分钟）相应的端口处于使用状态
```
![image.png](https://upload-images.jianshu.io/upload_images/17728742-45b587c46055481b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 2.地址再分配
1. 场景：系统发生故障从而紧急停止，这时需要尽快重启服务器端以提供服务。==但是套接字处于Time-wait状态而必须等待几分钟！！==
2. 解决方案：在套接字的选项中更改SO_REUSEADDR的状态。通过更改该参数，可以将任然处于Time-wait状态下的套接字端口号重新分配给新的套接字。
```
SO_REUSEADDR值有两个：
    默认值为0，表示无法分配正处于Time-wait状态下的套接字端口号
    值为1，表示可以

optlen = sizeof(option);
option = true;
setsockopt(serv_sock, SOL_SOCKET, SO_REUSEADDR, (void*)&option, optlen);
```
## 5.TCP_NODELAY
##### 1.Nagle算法
Nagle算法：为防止因数据包过多而发生网络过载。只有收到前一数据的ACK消息时，Nagle算法才发送下一数据。下图是使用Nagle算法和不使用的区别：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-c2d03285ec230838.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
TCP套接字默认使用Nagle算法交换数据，
因此最大限度地进行缓冲，直到收到ACK。
使用了Nagle算法的，套接字可选项TCP_NODELAY的值为0
```
##### 2.禁用Nagle算法：将套接字可选项TCP_NODELAY的值改为1
大文件数据应该禁用Nagle算法，可以提供传输速度。
```
// 将套接字可选项TCP_NODELAY的值改为1
int opt_val = 1;
setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, (void*)&opt_val, sizeof(opt_val));
```