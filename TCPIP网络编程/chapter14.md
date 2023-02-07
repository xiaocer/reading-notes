# 多播与广播
## 1.多播
##### 1.多播的概念
多播方式的数据传输基于UDP完成的，所以多播数据包的格式和UDP数据包相同。采用多播方式时，可以同时向多个主机传输数据。多播需要借助路由器完成，路由器将复制该数据包并传递到同一个网络中加入同一个多播组的多个主机
##### 2.多播的数据传输方式
1. 多播的数据传输特点
```
1. 多播服务器端针对特定多播组，只发送一次数据。
2. 即使发送一次数据，该组内的所有客户端都会接收数据
3. 多播组数可在IP地址范围内任意增加
4. 加入特定组即可接收发往该多播组的数据
```
多播组是D类IP地址：224.0.0.0~239.255.255.255
##### 3.生存时间（TTL：time to live）的设置
TTL是决定数据包传递距离的主要因素，TTL用整数表示，每经过一个路由器就减一，TTL变为0时，该数据包就无法再被传递，只能销毁。
1. TTL的设置方法：通过套接字的可选项完成。==与设置TTL相关的协议层为IPPROTO_IP,选项名为IP_MULTICAST_TTL.==
```
// 示例：
int time_live = 64;
inte sendsock = socket(PF_INET, SOCK_DGRAM, 0);
setsockopt(sendsock, IPPROTO_IP, IP_MULTICAST_TTL,
(void*)&time_live, sizeof(time_live));

```
##### 4.加入多播组的设置
加入多播组的设置也是通过设置套接字选项来完成。==加入多播组相关的协议层为IPPROTO_IP,选项名为IP_ADD_MEMBERSHIP==
```
// 示例
struct ip_mreq join_addr;
int recv_sock = socket(PF_INET, SOCK_DGRAM, 0);
join_addr.imr_multiaddr.s_addr = "多播地址信息";
join_addr.imr_interface.s_addr = "加入多播组的主机地址信息";
setsockopt(recv_sock, IPPROTO_IP, IP_ADD_MEMBERSHIP，
(void*)&join_addr, sizeof(join_addr));
```
ip_mreq结构体如下：
```
struct ip_mreq
{
    struct in_addr imr_multiaddr;   // 加入的组的IP地址
    struct in_addr imr_interface;   // 加入该组的套接字所属
    //主机的IP地址
}
```
##### 5.实现多播Sender和Receiver
1. 多播数据的发送主体Sender示例如下,Sender的功能为：向AAA组广播文件中保存的新闻信息
```
#include <sys/socket.h>
#include <stdio.h>
#include <arpa/inet.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

#define TTL 64;
int main(int argc, char* argv[])
{
    if (argc != 3)
    {
        printf("usage:%s <group id> <port>", argv[0]);
        return -1;
    }
    int send_sock = socket(PF_INET, SOCK_DGRAM, 0);
    
    // 设置TTL
    int time_live = TTL;
    setsockopt(send_sock, IPPROTO_IP, IP_MULTICAST_TTL, (const void*)&time_live, sizeof(time_live));
    
    struct sockaddr_in mul_addr;
    memset(&mul_addr, 0, sizeof(mul_addr));
    mul_addr.sin_family = AF_INET;
    mul_addr.sin_port = htons(atoi(argv[2]));
    mul_addr.sin_addr.s_addr = inet_addr(argv[1]);

    // 打开文件
    FILE* fp = fopen("news.txt", "r");
    if (fp == NULL)
        return -1;
    // 未读到文件尾部
    char buff[30];
    while (!feof(fp))
    {
        fgets(buff, sizeof(buff), fp);
        sendto(send_sock, buff, strlen(buff), 0, (const struct sockaddr*)&mul_addr, sizeof(mul_addr));
        sleep(2);
    }
    fclose(fp);
    close(send_sock);
    return 0;
}
```
2. Reciever：接受传递到AAA组的新闻信息
```
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char* argv[])
{
    if (argc != 3)
    {
        printf("usage:%s <group ip> <port>", argv[0]);
        return -1;
    }
    int sock = socket(PF_INET, SOCK_DGRAM, 0);
    
    // 给套接字sock绑定本地地址
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(atoi(argv[2]));
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(sock, (const struct sockaddr*)&addr, sizeof(addr));

    // 加入多播组
    struct ip_mreq join_addr;
    // 多播组IP地址
    join_addr.imr_multiaddr.s_addr = inet_addr(argv[1]);
    // 待加入主机的IP地址
    join_addr.imr_interface.s_addr = htonl(INADDR_ANY);
    setsockopt(sock, IPPROTO_IP, IP_ADD_MEMBERSHIP, (const void*)&join_addr, sizeof(join_addr));
    
    char buff[30];
    while(1)
    {
        ssize_t len = recvfrom(sock, buff, sizeof(buff) - 1, 0, NULL, NULL);
        buff[len] = 0;
        if (len < 0)
            break;
        fputs(buff, stdout);
    }
    close(sock);
    return 0;
}
```
## 2.广播
广播：一次性向同一网络中的多个主机发送数据。==与多播的区别：传输数据的范围有区别，多播即使在跨越不同网络的情况下，只要加入多播组就可以接收数据。==
##### 1.广播的实现方法
1. 广播的分类：广播是向同一网络中的所有主机传输数据的方法。广播也是基于UDP完成，根据传输数据时使用的IP地址的形式，广播分为两种：本地广播（Local Broadcast）和直接广播（Directed Broadcast）。
2. 本地广播和直接广播的区别：
```
1. 直接广播的IP地址中除了网络地址外，其余主机IP地址全部设置为1
（可以采用直接广播的方式向特定区域内的所有主机传输数据）
2. 本地广播的IP地址固定为255.255.255.255
    例如：某个网络中的主机向255.255.255.255传输数据时，
    数据将传输到该网络中的所有主机
```
3. 默认生成的套接字会阻止广播，可以通过更改套接字可选项来更改这个默认设置。
```
int send_sock = socket(PF_INET, SOCK_SGRAM, 0);
int bcast = 1;
setsockopt(send_sock, SOL_SOCKET, SO_BROADCAST, (void*)&bcast, sizeof(bcast));
```
##### 2.实现广播的Receiver和sender
对多播中的Receiver和Sender改进如下
1. Sender
```
#include <sys/socket.h>
#include <stdio.h>
#include <arpa/inet.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char* argv[])
{
    if (argc != 3)
    {
        printf("usage:%s <broadca ip> <port>", argv[0]);
        return -1;
    }
    int send_sock = socket(PF_INET, SOCK_DGRAM, 0);
    
    // 设置UDP套接字支持广播
    int bcast = 1;
    setsockopt(send_sock, SOL_SOCKET, SO_BROADCAST, (const void*)&bcast, sizeof(bcast));

    struct sockaddr_in mul_addr;
    memset(&mul_addr, 0, sizeof(mul_addr));
    mul_addr.sin_family = AF_INET;
    mul_addr.sin_port = htons(atoi(argv[2]));
    mul_addr.sin_addr.s_addr = inet_addr(argv[1]);

    // 打开文件
    FILE* fp = fopen("news.txt", "r");
    if (fp == NULL)
        return -1;
    // 未读到文件尾部
    char buff[30];
    while (!feof(fp))
    {
        fgets(buff, sizeof(buff), fp);
        sendto(send_sock, buff, strlen(buff), 0, (const struct sockaddr*)&mul_addr, sizeof(mul_addr));
        sleep(2);
    }
    fclose(fp);
    close(send_sock);
    return 0;
}

gcc news_receive.c -o rr
./nn 255.255.255.255 8888
```

2. Receiver:
```c
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <port>", argv[0]);
        return -1;
    }
    int sock = socket(PF_INET, SOCK_DGRAM, 0);
    
    // 给套接字sock绑定本地地址
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(atoi(argv[1]));
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(sock, (const struct sockaddr*)&addr, sizeof(addr));

    
    char buff[30];
    while(1)
    {
        ssize_t len = recvfrom(sock, buff, sizeof(buff) - 1, 0, NULL, NULL);
        buff[len] = 0;
        if (len < 0)
            break;
        fputs(buff, stdout);
    }
    close(sock);
    return 0;
}
```