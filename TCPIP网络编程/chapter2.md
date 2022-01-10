## 2.chapter-2套接字类型与协议设置
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
    
    3. protocol:计算机之间通信中使用的协议信息
    第三参数的目的是：同一协议族中存在多个数据传输方式相同的协议   
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