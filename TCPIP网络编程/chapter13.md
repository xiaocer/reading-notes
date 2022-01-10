# 多种I/O函数
## 1.send函数和recv函数
##### 1.send函数
```
#include <sys/socket.h>
ssize_t send(int sockfd, const void* buff, 
size_t nbytes, int flags);
函数返回值：
    成功时返回发送的字节数，失败时返回-1
函数参数：
    sockfd:表示与数据传输对象的连接的套接字文件描述符
    buff:保存待传输数据的缓冲地址值
    nbytes:待传输的字节数
    flags：传输数据时指定的可选项信息
```
##### 2.recv函数
```
#include <sys/socket.h>
ssize_t recv(int sockfd, void* buff, 
size_t nbytes, int flags);
函数返回值：
    成功时返回接收的字节数，收到EOF返回0，失败时返回-1
函数参数：
    sockfd:表示与数据接收对象的连接的套接字文件描述符
    buff:保存接收数据的缓冲地址值
    nbytes:可接收的最大字节数
    flags：接收数据时指定的可选项信息
```
##### 3.send函数和recv函数的可选项
该可选项可利用位或运算同时传递多个信息。
![image.png](https://upload-images.jianshu.io/upload_images/17728742-5685a43f19d9fe54.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. MSG_OOB:
```
//1.发送紧急信息
//紧急消息的传送只需要在调用send函数时指定MSG_OOB选项
send(sock, "123", strlen("123"), MSG_OOB);

//2.紧急消息的接收:recv函数的第四个参数指定为MSG_OOB
```
2. MSG_PEEK:设置MSG＿PEEK选项并调用recv函数时，即使读取了输入缓冲的数据也不会删除。==这个选项通常和MSG＿DONTWAIT合作，用于调用以非阻塞方式验证待读数据存在与否的函数==
```
// 客户端
#include <cstdio>
#include <sys/socket.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <cstring>
#include <cstdlib>

int main(int argc, char* argv[])
{
    if (argc != 3)
    {
        printf("usage:%s <IP> <PORT>", argv[0]);
        return -1;
    }
    int serv_sock = socket(PF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(atoi(argv[2]));
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    connect(serv_sock, (const sockaddr*)&serv_addr, sizeof(serv_addr));
    
    send(serv_sock, "HelloWorld", strlen("HelloWorld"), 0);
    
    close(serv_sock);
    return 0;
}

// 服务端
#include <cstdio>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <cstdlib>
#include <cstring>
#include <unistd.h>

int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <port>", argv[0]);
        return -1;
    }
    int serv_sock = socket(PF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr, cli_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(atoi(argv[1]));
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    bind(serv_sock, (const sockaddr*)&serv_addr, sizeof(serv_addr));
    
    listen(serv_sock, 5);

    socklen_t cli_len = sizeof(cli_addr);
    int cli_sock = accept(serv_sock, (struct sockaddr*)&cli_addr, &cli_len);

    char buff[30];
    ssize_t str_len;
    while (1)
    {
        // MSG_DONTWAIT可选项表示调用recv函数不阻塞，即使输入缓冲中不存在数据
        // MSG_PEEK可选项表示验证输入缓冲中是否存在数据
        // 本次读取的数据不会从输入缓冲中删除
        str_len = recv(cli_sock, buff, sizeof(buff) - 1, MSG_PEEK | MSG_DONTWAIT);
        if (str_len > 0)
            break;
    }
    buff[str_len] = '\0';
    printf("Input buffer %ld bytes:%s\n", str_len, buff);
    str_len = recv(cli_sock, buff, sizeof(buff) - 1, 0);
    buff[str_len] = '\0';
    printf("read again:%s\n", buff);
        
    close(serv_sock);
    close(cli_sock);
    return 0;
}
```
运行结果如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-f93faecf55df5a02.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.readv函数和writev函数
readv和writev函数的功能：对数据进行整合传输以及发送。==通过writev函数可以将分散保存在多个缓冲中的数据一并发送，通过readv函数可以由多个缓冲分别接收数据。（将输入缓冲中的数据读入不同位置）== 因此适当使用这两个函数可以减少I/O函数的调用次数。
##### 1.writev函数
1. 函数声明
```
#include <sys/uio.h>
ssize_t writev(int filedes, const struct iovec* iov, int iovcnt );
函数参数：
    filedes:表示数据传输对象的套接字文件描述符。
    该函数并不只限于套接字，可以向其传递文件或者标准输出描述符
    iov:iovec结构体数组的地址值
    iovcnt:向第二个参数传递的数组长度
函数返回值：
    成功时返回发送的字节数，失败时返回-1
```
iovec结构体声明如下：
```
struct iovc
{
    void* iov_base; //缓冲地址
    size_t iov_len; //缓冲大小
}
```
2. 该函数的使用方法：下图表示向控制台（对应的文件描述符为1）输出数据
![image.png](https://upload-images.jianshu.io/upload_images/17728742-60815989128b462d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
#include <cstdio>
#include <sys/uio.h>

int main()
{
    struct iovec vec[2];
    char buff1[] = "ABCDEFG";
    char buff2[] = "1234567";
    vec[0].iov_base = buff1;
    vec[0].iov_len = 4;
    vec[1].iov_base = buff2;
    vec[1].iov_len = 4;

    // 往控制台输出数据
    ssize_t len = writev(1, vec, 2);
    printf("write bytes:%ld\n", len);
    return 0;
}
// 输出如下：
// ABCD1234write bytes:8
```
##### 2.readv函数
1. 函数声明
```
#include <sys/uio.h>
ssize_t readv(int filedes, const struct iovec* iov, int iovcnt);
函数返回值：
    成功时返回接收的字节数，失败时返回-1
函数参数：
    filedes:传递接收数据的文件描述符
    iov:包含数据保存位置和大小信息的iovec结构体数组的地址值
    iovcnt:第二个参数中数组的长度
```
2. readv函数的使用示例
```
#include <stdio.h>
#include <sys/uio.h>

int main()
{
    char buff1[30] = {0};
    char buff2[30] = {0};
    struct iovec vec[2];
    vec[0].iov_base = buff1;
    vec[0].iov_len = 4;
    vec[1].iov_base = buff2;
    vec[1].iov_len = 4;

    ssize_t read_len = readv(0, vec, 2);
    printf("read bytes:%ld\n", read_len);
    printf("first message:%s\n", buff1);
    printf("first message:%s\n", buff2);
    return 0;
}
// helloworld
// read bytes:8
// first message:hell
// first message:owor
```
##### 3.writev函数和readv函数的使用场景
![image.png](https://upload-images.jianshu.io/upload_images/17728742-8003597981cf224a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
