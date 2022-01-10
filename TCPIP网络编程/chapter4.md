## 4.chapter-4实现基于TCP的客户端和服务器端
##### 1.TCP服务器端函数调用顺序
  ![image.png](https://upload-images.jianshu.io/upload_images/17728742-e2b608f0d69c6084.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 2.进入等待连接请求状态
1. listen函数：在一个套接字上侦听连接请求。
2. listen函数声明
```
//函数声明：
#include <sys/types.h>
#include <sys/socket.h>
int listen(int sockfd,int backlog);
//函数返回值：成功的话返回0，失败返回-1；
//形参：
    backlog为最大等待连接请求队列长度
    sockfd:表示监听套接字文件描述符
```
##### 3.受理客户端连接请求
1. accept函数：在一个套接字上接受新的连接请求。**调用这个函数之后，将会创建一个新的具有相同套接字协议和地址族的套接字，该套接字用于数据I/O,它是自动创建的，并自动与发起连接请求的客户端建立连接，成功的话将会返回一个对应于新创建的套接字的文件描述符。（该套接字将用于数据I/O）**
2. accept函数的声明：
```
#include <sys/socket.h>
#include <sys/types.h>
int accept(int sockfd,struct sockaddr *addr,socklen_t addrlen);
返回值：成功时返回创建的套接字文件描述符，失败时返回-1
形参：
    sockfd为服务器端套接字的描述符
    addr保存发起连接请求的客户端地址信息的变量地址值。
    有了客户端的地址信息可以做出进一步的行为
    addrlen为第二个结构体的长度
```
==综上：==

<font color = blue size = 6px>一个服务端中至少两个套接字，如下图所示：</font>
![image.png](https://upload-images.jianshu.io/upload_images/17728742-dc5cbb6b7c0e93ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 4.TCP客户端的默认调用函数顺序
![image.png](https://upload-images.jianshu.io/upload_images/17728742-91283e0fe45b4ddb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 5.客户端发起连接请求
1. connect函数
2. connect函数声明
```
#include <sys/socket.h>
int connect(int sock, struct sockaddr* servaddr, socklen_t addrlen);
函数返回值：成功时返回0，失败返回-1
函数参数：
    sock:客户端套接字文件描述符
    servaddr:保存目标服务器地址信息的变量地址值
    addrlen:第二个参数的长度
```
<font color = red>note:客户端的IP地址和端口号在调用Connect函数时自动分配，无需调用标记的bind函数进行分配。</font>

