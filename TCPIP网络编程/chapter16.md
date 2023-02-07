# 关于I/O流分离的其他内容

本节要点：

1. 知道如何通过FILE结构体的指针实现半关闭的效果

   ​	１.　先进行文件描述符的复制

   ​	２.　使用shutdown函数完成半关闭的效果

2. 熟悉使用ｄｕｐ、ｄｕｐ２函数进行文件描述符的复制

## 1.分离I/O流
##### 1.分离I/O流的目的
1. 为了将FILE指针按照读模式和写模式加以区分
2. 可以通过区分读写模式降低实现难度，比如一个线程进行读，一个线程进行写。
3. 通过区分I/O缓冲提高缓冲性能
##### 2.流分离带来的EOF问题
针对fdopen函数调用生成的读模式的FILE指针或者写模式的FILE指针调用fclose()函数，这样虽然可以向对方发送EOF，但是==套接字达不到半关闭的状态，套接字既不能发送数据，也不能接收数据。==
```
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <stdlib.h>

int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <PORT>", argv[0]);
        return -1;
    }
    int serv_sock = socket(PF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(atoi(argv[1]));
    bind(serv_sock, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));

    listen(serv_sock, 5);

    struct sockaddr_in cli_addr;
    socklen_t len = sizeof(cli_addr);
    int cli_sock = accept(serv_sock, (struct sockaddr*)&cli_addr, &len); 

    // 分别创建读写模式的FILE结构体指针
    FILE* readfp = fdopen(cli_sock, "r");
    FILE* writefp = fdopen(cli_sock, "w");

    fputs("hello,my name is nrvcer!\n", writefp);
    fputs("what is your name?\n", writefp);
    fflush(writefp);
    
    // 调用成功后会向对方发送EOF
    fclose(writefp);

    char buff[30];
    // 无法接收，fclose调用已经完全终止了套接字
    fgets(buff, sizeof(buff), readfp);  
    fputs(buff, stdout);

    fclose(readfp);
    return 0;
}
```
上例代码中两个FILE指针和文件描述符、套接字之间的关系如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-4c798a3e0307faff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/17728742-39e1e14baff1710a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <stdlib.h>

int main(int argc, char* argv[])
{
    if (argc != 3)
    {
        printf("usage:%s <IP> <PORT>", argv[0]);
        return -1;
    }
    int sock = socket(PF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(atoi(argv[2]));
    serv_addr.sin_addr.s_addr = inet_addr(argv[1]);
    connect(sock, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));

    FILE* readfp = fdopen(sock, "r");
    FILE* writefp = fdopen(sock, "w");

    char buff[30];
    while (1)
    {
        // 收到服务器端发送的EOF将会退出循环
        if (fgets(buff, sizeof(buff), readfp) == NULL)
            break;
        fputs(buff, stdout);
        fflush(stdout);
    }
    // 服务器端收不到这个信息，因为服务器端套接字已经完全关闭
    // 不是半关闭
    fputs("my name is client!", writefp);
    fflush(writefp);

    fclose(readfp);
    fclose(writefp);
    return 0;
}
```
## 2.文件描述符的复制和半关闭
##### 1.终止流时无法半关闭的原因
读模式FILE指针和写模式FILE指针都是基于同一个文件描述符创建的，因此针对任意一个FILE指针调用fclose()函数都会关闭文件描述符，也就是终止套接字。
##### 2.半关闭的实现
解决方案的关键：==创建FILE指针前先复制文件描述符==
![image.png](https://upload-images.jianshu.io/upload_images/17728742-13aa42ef810a59fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如上图所示，通过fclose调用读模式的FILE指针或者写模式的FILE指针并不会销毁套接字，因为只有销毁掉所有文件描述符后才能销毁掉套接字。（类似于引用计数）

如下图所示：比如针对写模式FILE指针调用fclose函数.==此时剩下的一个文件描述符任然可以同时进行IO。==
![image.png](https://upload-images.jianshu.io/upload_images/17728742-732b45b337cf8de3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
==要想进入半关闭状态，除了通过fclose调用读模式的FILE指针或者写模式的FILE指针，还需要调用shutdown函数发送EOF==
##### 3.文件描述符的复制
在一个进程内完成文件描述符的复制，与fork函数中对文件描述符的复制不同。文件描述符的复制如下图所示（文件描述符的值不能重复）：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-64889d871435f7f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. dup函数和dup2函数
```
#include <unistd.h>
int dup(int fildes);
int dup2(int fildes, int fildes2);
函数返回值：
    成功时返回复制的文件描述符，失败时返回-1
函数参数：
    fildes:需要复制的文件描述符
    fildes2:明确指定的文件描述符整数值
```
2. 文件描述符的复制示例如下
```
#include <stdio.h>
#include <unistd.h>
#include <string.h>

int main()
{
    int cfd1 = dup(1);
    int cfd2 = dup2(cfd1, 7);
    printf("fd1 = %d, fd2 = %d\n", cfd1, cfd2); // 3 7
    write(cfd1, "HELLO\n", strlen("HELLO\n"));
    write(cfd2, "WORLD\n", strlen("WORLD\n"));
    close(cfd1);
    close(cfd2);
    write(1, "HI\n", strlen("HI\n"));   // 可以输出
    close(1);
    write(1, "HI\n", strlen("HI\n"));   // 不能输出
    return 0;
}
```
##### 半关闭的示例
1. 针对1.2中的服务器端的改进：原本打算调用`fclose(writefp)`使得服务器端达到半关闭的状态（可以接收消息，但是不能发送），但是却终止了整个套接字。现改进如下：
```
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h> // for dup()

int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <PORT>", argv[0]);
        return -1;
    }
    int serv_sock = socket(PF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(atoi(argv[1]));
    bind(serv_sock, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));

    listen(serv_sock, 5);

    struct sockaddr_in cli_addr;
    socklen_t len = sizeof(cli_addr);
    int cli_sock = accept(serv_sock, (struct sockaddr*)&cli_addr, &len); 

    // 分别创建读写模式的FILE结构体指针
    FILE* readfp = fdopen(cli_sock, "r");
    FILE* writefp = fdopen(dup(cli_sock), "w");

    fputs("hello,my name is nrvcer!\n", writefp);
    fputs("what is your name?\n", writefp);
    fflush(writefp);

    // 服务器端进入半关闭状态，同时发送EOF
    shutdown(fileno(readfp), SHUT_WR); 
    fclose(writefp);

    char buff[30];

    fgets(buff, sizeof(buff), readfp);  
    fputs(buff, stdout);

    fclose(readfp);
    return 0;
}
```