## 2.chapter-2套接字类型与协议设置

本节重点：
1. 了解套接字使用的协议族，重点知道ipv4互联网协议族和ipv6互联网协议族
2. 知道套接字使用的套接字类型（数据传输方式）：面向连接的套接字和面向消息的套接字
3. 知道计算机之间通信中使用的协议信息，例如AF_INET协议族下，面向连接的套接字种的TCP协议

##### 1.Linux下创建套接字所用的函数
1. socket函数：
```
socket函数声明：
#include <sys/socket.h>
#include <sys/types.h>
int socket(int domain,int type,int protocol);
函数返回值：使用该函数创建套接字成功时返回文件描述符，失败时返回-1；
函数形参：
    1. domain:套接字使用的协议族信息(protocol family)。protocol family协议族常见有如下：
        PF_INET:ipv4互联网协议族
        PF_INET6:ipv6互联网协议族
        PF_LOCAL:本地通信的Unix协议族
        PF_PACKET:底层套接字的协议族
        PF_IPX:IPX Novell协议族
    2. type:套接字数据传输类型信息，两种具有代表性的数据传输方式。第二参数的目的是：同一协议族中也存在多种数据传输方式：
        //套接字类型1：
        面向连接的套接字：SOCK_STREAM
        //套接字类型2：
        面向消息的套接字：SOCK_DGRAM
    
    3. protocol:计算机之间通信中使用的协议信息，指定具体的协议信息。
    第三参数的目的是：同一协议族中可能存在多个数据传输方式相同的协议。即同一协议族，同一种数据传输方式，多种协议也是有可能存在的。   
        比如说：
        //创建ipv4协议族中面向连接的套接字（TCP套接字）
        int tcp_socket = socket(PF_INET,SOCK_STREAM,IPPROTO_TCP);
        //创建ipv4协议族中面向消息的套接字
        int udp_socket = socket(PF_INET,SOCK_DGRAM,IPPROTO_UDP);
        //当然，上面两个函数的第三参数可以省率，因为符合前两个参数的条件的协议只有一个。
        
```
##### 2.两种套接字类型的区别
套接字类型指的是套接字的数据传输方式，通过socket函数的第二参数传递。
1. 套接字类型1：面向连接的套接字，是一个可靠的，按序传递的，基于字节的面向连接的数据传输方式的套接字。
2. 套接字类型2：面向消息的套接字，不可靠的，不按序传递的，以数据的高效传输为目的的套接字。
##### 3.面向连接的套接字：传输的数据不存在数据边界
1. 服务端程序如下：
```
#include <stdio.h>
#include <sys/socket.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <stdlib.h>

int main() {
    int lfd = socket(PF_INET, SOCK_STREAM, 0);
    if (lfd == -1) {
        perror("socket() error\n");
        exit(0);
    }
    
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(9999);
    inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr.s_addr);
    if (bind(lfd, (const struct sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        perror("bind() error\n");
        close(lfd);
        exit(0);
    }

    if (listen(lfd, 128) == -1) {
        perror("listen() error\n");
        close(lfd);
        exit(0);
    }

    int clifd = accept(lfd, NULL, NULL);
    if (clifd == -1) {
        perror("accept() error\n");
        close(lfd);
        exit(0);
    }
    char buff[] = "hello,world";
    if (write(clifd, buff, sizeof(buff)) == -1) {
        perror("write() error\n");
        close(lfd);
        close(clifd);
        exit(0);
    }

    close(clifd);
    close(lfd);
    return 0;
}
```
2. 客户端程序如下：
```
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd == -1) {
        perror("socket() error\n");
        exit(0);
    }
    
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(9999);
    inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr.s_addr);
    if (connect(fd, (const struct sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        perror("connect() error\n");
        close(fd);
        exit(0);
    }

    char buff[1024] = {0};
    int count = 0, read_len, callTimes = 0;
    while (read_len = read(fd, &buff[count++], 1)) {
        if (read_len == -1) {
            perror("read() error\n");
            close(fd);
            exit(0);
        }
        callTimes += read_len;
    }
    printf("message from server:%s\n", buff);
    printf("read call times:%d\n", callTimes);

    close(fd);
    return 0;
}
```
3. 观察客户端运行结果：服务器端发送了12个字节的数据，客户端却需要调用12次read函数进行读取。
```
root@hecs-349784:~/practice# ./client
message from server:hello,world
read call times:12
```