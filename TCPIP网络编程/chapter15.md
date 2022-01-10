# 套接字和标准I/O
## 1.标准I/O函数
##### 1.标准IO函数的两个优点
1. 具有良好的移植性
2. 可以利用缓冲提高性能（I/O缓冲和套接字缓冲之间的关系）
![image.png](https://upload-images.jianshu.io/upload_images/17728742-a42dd6f7f72b2c00.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 2.标准I/O函数与系统函数之间的性能对比
1. 使用系统函数复制一个较大的文件
```
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
```
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
1. fdopen函数：将创建套接字时的文件描述符转换为标准IO函数中使用的FILE结构体指针
```
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
```
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