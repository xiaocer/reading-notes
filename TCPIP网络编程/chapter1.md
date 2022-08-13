## 1.chapter-1.理解网络编程和套接字
**网络编程：**          网络编程其实就是编写程序使得两台联网的计算机相互交换数据。
在进行编写之前，我们应当知道这些基础：
1. Linux平台下运行c源程序
我们使用c语言编译器gcc编译c源程序。比如说现在有一个源文件hello_server.c，则对该文件进行编译的命令为:
```
//-o是用来指定可执行文件名的可选参数，server为可执行文件名
[xiaocer@localhost~]$gcc hello_server.c -o server
```
运行可执行文件的命令为：
```
[xiaocer@localhost~]$./server
```
2. 基于Linux的文件操作
```
1. 打开文件
#include <fcntl.h>
int open(const char* path, int flag); 
返回值：成功时返回文件描述符，失败时返回-1
形参：path表示文件名的字符串地址，flag表示文件打开模式信息，文件打开模式见下图

2. 关闭文件
#include <unistd.h>
int close(int fd);
返回值：成功时返回0，失败时返回-1
形参：fd表示需要关闭的文件或者套接字的文件描述符

3. 将数据写入文件
#include <unistd.h>
ssize_t write(int fd, const void* buf, size_t nbytes);
返回值：成功时返回写入的字节数，失败返回-1
形参：
    fd表示显示数据传输对象的文件描述符
    buf表示保存要传输数据的缓冲地址值
    nbytes:要传输数据的字节数
4. 读取文件中的数据
#include <unistd.h>
ssize_t read(int fd, void* buf, size_t nbytes);
返回值：成功时返回接收的字节数（遇到文件末尾则返回0），失败时返回-1
形参：
    fd表示数据接收对象的文件描述符
    buf表示要保存接收数据的缓冲地址
    nbytes表示要接收数据的最大字节数
```
![image.png](https://upload-images.jianshu.io/upload_images/17728742-bbbdd8a8776e9d5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 使用基本文件操作的一个示例：向一个文件中写入数据以及读该文件
```
#include <sys/socket.h>
#include <fcntl.h>
#include <string>
#include <unistd.h>

int main()
{
    int wr_fd;
    std::string src_str = "HelloWorld";
    char data[100] = {0};
    
    wr_fd = open("dat.txt", O_CREAT | O_RDWR | O_APPEND);
    printf("%d\n", wr_fd);  // 3
    write(wr_fd, src_str.data(), src_str.size());
    read(wr_fd, data, sizeof(data));

    close(wr_fd);
    return 0;
}

```
**文件描述符：** 每当创建文件或者套接字，操作系统将返回整数。**相当于window中的术语“句柄”。** 分配给标准输入输出及标准错误的文件描述符如下。==程序员创建的文件或者套接字分配的文件描述符一般从3开始==：
1. 0表示标准输入
2. 1表示标准输出
3. 2表示标准错误
