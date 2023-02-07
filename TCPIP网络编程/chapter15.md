# 套接字和标准I/O
## 1.标准I/O函数
##### 1.标准IO函数的优点和缺点：
1. 优点：
    1. 具有良好的移植性
    2. 可以利用缓冲提高性能（I/O缓冲和套接字缓冲之间的关系）：比如说使用fputs函数传输字符串Hello时，首先将数据传递到标准IO函数的输出缓冲，然后数据将移动到套接字输出缓冲，最后将字符串发送到对方主机。
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-a42dd6f7f72b2c00.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 缺点：
	1. 不容易进行双向通信，因为创建套接字默认返回的是文件描述符，而不是FILE结构体指针
	2. 有时可能频繁调用fflush函数
	3. 需要以FILE结构体指针的形式返回文件描述符（其中包含成员表示文件描述符）
##### 2.标准I/O函数与系统函数之间的性能对比：在传输的数据很多时，使用标准IO函数可以提高性能
1. 使用系统函数复制一个较大的文件
```c
#include <fcntl.h>
#include <unistd.h>
#include <time.h>
#include <stdio.h>

int main()
{
    int fd1 = open("sour.txt", O_RDONLY);
    int fd2 = open("copy.txt", O_CREAT | O_TRUNC | O_WRONLY);
    ssize_t read_len;
    char buff[3];
    long begin_time = time((time_t*)NULL);
    while ((read_len = read(fd1, buff, sizeof(buff))) > 0)
    {
        write(fd2, buff, sizeof(buff));
    }
    long end_time = time((time_t*)NULL);
    printf("copy spend time:%ld\n", end_time - begin_time);
    close(fd1);
    close(fd2);
    return 0;
}
// 运行结果为
// copy spend time:1253
```
2. 标准IO函数复制一个较大的文件（IO缓冲可以提高性能）
```c
#include <stdio.h>
#include <time.h>

int main()
{
    FILE* fp1 = fopen("sour.txt", "r");
    FILE* fp2 = fopen("copy1.txt", "w");
    
    char buff[3];
    long begin_time = time((time_t*)NULL);
    while (fgets(buff, sizeof(buff), fp1) != NULL)
    {
        fputs(buff, fp2);
    }
    long end_time = time((time_t*)NULL);
    printf("copy spend time:%ld\n", end_time - begin_time);
    fclose(fp1);
    fclose(fp2);
    return 0;
}
//运行结果为
//copy spend time:34
```
##### 3.使用标准IO函数
1. fdopen函数：将创建套接字时的文件描述符转换为标准IO函数中使用的FILE结构体指针。转换为FILE结构体指针的目的是可以通过该指针调用标准IO函数。
```c
#include <stdio.h>
FILE* fdopen(int fildes, const char* mode);
函数参数：
    fildes:需要转换的文件描述符
    mode:将要创建的FILE结构体指针的模式信息
函数返回值：
    成功时返回转换的FILE结构体指针，失败时返回NULL
    
#include <fcntl.h>
#include <stdio.h>

int main()
{
    int fd = open("ppp.txt", O_WRONLY | O_CREAT | O_TRUNC);
    // 返回写模式的指针
    FILE* fp = fdopen(fd, "w");
    if (fp == NULL)
        return -1;
    fputs("HelloWorld", fp);
    fclose(fp);
    return 0;
}
```
2. fileno函数：将FILE结构体指针转化为文件描述符
```c
#include <stdio.h>
int fileno(FILE* stream);
函数返回值：
    成功时返回转换成功后的文件描述符，失败返回-1
    
#include <stdio.h>
#include <unistd.h>

int main()
{
    FILE* fp = fopen("news.txt", "r");
    printf("%p\n", fp);
    int fd = fileno(fp);
    printf("%d\n", fd); // 3
    close(fd);
    return 0;
}
```

##### 4.基于套接字的标准IO函数的使用

1. 基于标准IO函数的数据交换形式的回声服务器端如下：

   ```c
   #include <stdio.h>
   #include <arpa/inet.h>
   #include <stdlib.h>
   #include <string.h>
   #include <unistd.h>
   
   int main(int argc, char* argv[]) {
       if (argc != 2) {
           printf("usage:%s <port>\n", argv[0]);
           return -1;
       }
       int lfd = socket(PF_INET, SOCK_STREAM, 0);
       
       struct sockaddr_in serv_addr;
       serv_addr.sin_family = AF_INET;
       serv_addr.sin_port = htons(atoi(argv[1]));
       serv_addr.sin_addr.s_addr = INADDR_ANY;
       bind(lfd, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));
   
       listen(lfd, 128);
   
       int client_fd = accept(lfd, NULL, NULL);
       
       FILE* readp;
       FILE* writep;
   
       readp = fdopen(client_fd, "r");
       writep = fdopen(client_fd, "w");
   
       char message[128] = {0};
       while (!feof(readp)) {
           fgets(message, sizeof(message), readp);
           printf("message from client:%s\n", message);
           fputs(message, writep);
           // 保证数据从IO缓冲的输出缓冲输出到套接字的输出缓冲
           fflush(writep);
       }
   
       fclose(readp);
       fclose(writep);
       close(lfd);
   
       return 0;
   }
   ```

   

2. 基于标准IO函数的数据交换形式的回声客户端如下：

```c
#include <stdio.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char* argv[]) {
    if (argc != 3) {
        printf("usage:%s <ip> <port>\n", argv[0]);
        return -1;
    }

    int fd = socket(AF_INET, SOCK_STREAM, 0);
    
    struct sockaddr_in serv_addr;
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(atoi(argv[2]));
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    connect(fd, (const sockaddr*)&serv_addr, sizeof(serv_addr));

    FILE* readp = fdopen(fd, "r");
    FILE* writep = fdopen(fd, "w");

    char message[128] = {0};
    while (1) {
        fputs("Input message(Q to quit):", stdout);
        fgets(message, sizeof(message), stdin);
        if (!strcmp(message, "q\n") || !strcmp(message, "Q\n")) {
            break;
        }
        fputs(message, writep);
        fflush(writep);
        fgets(message, sizeof(message), readp);
        printf("Message from server:%s\n", message);
    }

    fclose(readp);
    fclose(writep);

    return 0;
}
```

