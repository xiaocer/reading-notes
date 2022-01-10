## 基于UDP的服务器端和客户端
##### 1.TCP和UDP的区别
流控制是区分UDP和TCP的最重要的标志。UDP不会进行流控制，与对方套接字连接和断开连接都属于流控制的一部分。
##### 2.UDP的高效使用
通过网络实时传输视频或者音频考虑使用UDP
##### 3.实现基于UDP的服务器端和客户端
1. UDP中的服务器端和客户端没有连接，因此不必调用TCP连接过程中的listen函数和accept函数。==UDP中只有创建套接字和数据交换两个过程==
2. UDP中的服务器端和客户端都只要一个套接字。==只需要一个UDP套接字就能和多台主机通信==
3. 基于UDP的数据I/O函数
```
1. sendto函数（用于传输数据）
函数声明：
#include <sys/socket.h>
ssize_t sendto(int sock, void* buff, size_t nbytes, int flags, struct sockaddr *to, socklen_t addrlen);
函数参数：
    sock:用于传输数据的UDP套接字文件描述符
    buff:保存待传输数据的缓冲地址值
    nbytes:待传输的数据长度，以字节为单位
    flags：可选项参数，如果没有则传递0
    to:存有目标地址信息的sockaddr结构体变量的地址值
    addrlen：to这个结构体变量长度
函数返回值：成功时返回传输的字节数，失败返回-1

2. recvfrom函数（用于接收数据）
#include <sys/socket.h>
ssize_t recvfrom(int sock, void* buff, size_t nbytes, int flags, struct sockaddr *from, socklen_t
* addrlen);
函数参数：
    sock:用于接受数据的UDP套接字文件描述符
    buff:保存接收数据的缓冲地址值
    nbytes:可接收的最大字节数，不能超过参数buff所指的缓冲区大小
    flags：可选项参数，如果没有则传递0
    from:存有发送端地址信息的sockaddr结构体变量的地址值
    addrlen：from这个结构体变量长度的地址值
函数返回值：成功时返回接收的字节数，失败返回-1
```
<font color = blue size = 6px>每次调用sendto函数传输数据都要添加目标地址信息。又由于UDP的数据发送端不固定，所以recvfrom函数定义为可接收发送端地址信息的形式</font>
4. 基于UDP的回声服务器端和客户端
##### 4.UDP客户端套接字的地址分配
调用sendto函数时自动分配客户端套接字的IP地址和端口号

