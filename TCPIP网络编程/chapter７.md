## 优雅的断开套接字连接

本节重点：

1. 知道如何在基于TCP协议的服务端中实现半关闭

##### 1.基于TCP的半关闭
1. Linux下的close函数意味着完全断开连接，完全断开就不能传输数据，也不能接收数据。==为了解决这个问题，只关闭一部分数据交换中使用的流的方法产生，断开一部分连接指的是可以传输数据但是无法接收或者可以接收数据但是无法传输。即只关闭流的一半（流分为输入流和输出流）==
2. 套接字生成的流如下：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-b24e2d76f116bddf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
半关闭即断开IO流1或者IO流２
##### 2.shutdown函数（关闭全双工连接的部分流）
1. 函数声明
```
int shutdown(int sock, int howto);
函数参数：
    sock:需要断开的套接字文件描述符
    howto:传递断开方式信息
函数返回值：
    成功时返回0，失败时返回-1
```
2. 参数2howto的可能值如下：
```
1. SHUT_RD:断开输入流
2. SHUT_WR:断开输出流
3. SHUT_RDWR:同时断开输入输出流
```
##### 3.示例
实现一个收发文件的服务器端和客户端：服务器端将约定的文件传给客户端，客户端收到后发送字符串"Thank you"给服务器端。==这里用到半关闭技术，只关闭服务器端的输出流，这样就可以既保留输入流，可以接收客户端的数据，又可以发送EOF（断开输出流时向对方主机发送EOF）==

<font color = red>note:EOF的值为0</font>
1. 发文件的服务器端如下：
```
#include <stdio.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char* argv[])
{
    FILE* fp;
    // 以读的方式打开一个文件
    fp = fopen("des.txt", "rb");
    char buff[30];
    if (argc != 2)
    {
        printf("usage:%s <port>",argv[0]);
        exit(-1);
    }

    // 创建TCP服务端套接字
    int sockfd = socket(PF_INET, SOCK_STREAM, 0);

    // 分配套接字地址
    struct sockaddr_in  addr_in;
    memset(&addr_in, 0, sizeof(addr_in));
    addr_in.sin_family = AF_INET;   // ipv4协议地址族
    addr_in.sin_port = htons(atoi(argv[1]));    
    addr_in.sin_addr.s_addr = htonl(INADDR_ANY);

    bind(sockfd, (const struct sockaddr*)&addr_in, sizeof(addr_in));

    listen(sockfd, 5);

    struct sockaddr_in client_in;
    socklen_t client_in_len = sizeof(client_in);
    int cli_fd = accept(sockfd, ( struct sockaddr*)&client_in, &client_in_len);
    size_t size = sizeof(buff);
    // 循环读取文件，写出数据到客户端
    while (1)
    {
        size_t read_len = fread(buff, 1, size, fp);
        // 表示读取完毕，则直接退出循环
        if (read_len < size)
        {
            write(cli_fd, buff, read_len);
            break;
        }
        write(cli_fd, buff, size);
    }
    // 关闭输出流（半关闭）
    shutdown(cli_fd, SHUT_WR);
    // 还可以通过输入流接收数据
    read(cli_fd, buff, size);
    printf("Message from client:%s\n", buff);

    fclose(fp);
    close(sockfd);
    close(cli_fd);

    return 0;
}
```
2. 收文件的客户端如下：
```
#include <stdio.h>
#include <sys/socket.h>
#include <string.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include <unistd.h> //for read

int main(int argc, const char* argv[])
{
    if (argc != 3)
    {
        printf("usage: %s <IP> <PORT>", argv[0]);
        exit(-1);
    }
    // 创建客户端套接字
    int sockfd = socket(PF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(atoi(argv[2]));
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    connect(sockfd, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));
    
    // 创建文件，保存服务器端发来的文件数据
    FILE* fp = fopen("destination.txt", "wb");
    char buff[30];
    ssize_t read_len;
    while ((read_len = read(sockfd, buff, sizeof(buff))) != 0)
    {
        fwrite(buff, 1, read_len, fp);
    }
    printf("Recieved file data");
    write(sockfd, "Thank you", 10);

    fclose(fp);
    close(sockfd);
    return 0;
}
```