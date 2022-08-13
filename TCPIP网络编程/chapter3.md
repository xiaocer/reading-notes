## 3.chapter-3地址族与数据序列
##### 0.分配给套接字的IP地址和端口号
1. ==IP地址是为收发网络数据而分配给计算机的值。== IP地址分为两类：IPv4和IPv6,IPv4标准的四字节IP地址分为网络号和主机号
![image.png](https://upload-images.jianshu.io/upload_images/17728742-b9f3d7594431227b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 用于区分套接字（应用程序）的端口号。==端口号由16位组成，可分配的端口范围为0到65535==
##### 1.表示ipv4地址的结构体,sockaddr_in,位于netinet/in.h头文件中。
```
struct sockaddr_in 
{
  sa_family_t sin_family;  //地址族（address family）
  uint16_t sin_port;  //16位udp/tcp端口号，以网络字节序保存
  struct in_addr sin_addr;  //32位ip地址，以网络字节序保存
  char sin_zero[8];  //不使用,无特殊含义，只是使结构体sockaddr_in和sockaddr结构体保持一致而插入的成员
}
struct in_addr
{
  In_addr_t s_addr;  //32位ip地址，实际类型为uint32_t
}
```

##### 2.每种协议族使用的地址族也不同，比如说：
```
AF_INET:ipv4网络协议中使用的地址族
AF_INET6:ipv6网络协议中使用的地址族
AF_LOCAL:本地通信中采用的unix协议的地址族
```
##### 3.第三个结构体sockaddr（并非只为IPv4设计）
```
struct sockaddr
{
  sa_family_t sin_family;  //地址族
  char sa_data[14];  //保存ip地址和端口信息，剩余部分填充为0
}
```
##### 4.主机字节序与网络字节序
１. CPU向内存保存数据的方式有两种如下：
```
主机字节序：
1.大端序(Big Endian)：高位字节数存放在低位地址
2.小端序(Little Endian):高位字节数存放在高位地址
```
2.在Intel系列CPU中是以小端序保存数据的，可以用如下程序解释：
```
#include <iostream>

using namespace std;

int main()
{
	int a = 65535;	//二进制位00000000 00000000 11111111 11111111
	int* p1 = &a;	
	
	char* p2 = reinterpret_cast<char*>(p1);
	//第一个字节（低地址）
	printf("%p %d\n", p2, *p2);	

	p2++;
	//第二个字节
	printf("%p %d\n",p2,*p2);
	
	p2++;
	//第三个字节
	printf("%p %d\n", p2, *p2);
	
	p2++;
	//第四个字节（高地址）
	printf("%p %d\n", p2, *p2);
	
	return 0;
}
//000000CC3ECFF934 - 1
//000000CC3ECFF935 - 1
//000000CC3ECFF936 0
//000000CC3ECFF937 0
```
**同样可以证明在Linux下的CPU也是以小端序保存数据的。**

3. 由于数据在主机的保存方式可能不同（可能使用大端序或者小端序），所以在网络中传输数据约定统一方式，这种约定称为网络字节序(Network Byte Order)： <font color = red>统一规定为大端序。因此计算机接受数据应识别该数据为大端序（即网络字节序），在网络上传输数据前先把数据转化为大端序格式。</font>
4. 字节序转换函数：(主机字节序与网络字节序的相互转换)
```
//将short类型的数据从主机字节序转化为网络字节序
uint16_t short htons();
//将long类型的数据从主机字节序转化为网络字节序
 uint32_t long htonl();
//将short类型的数据从网络字节序转化为主机字节序
uint16_t short ntohs();
//将long类型的数据从网络字节序转化为主机字节序
 uint32_t long ntohl();
```
**上面的函数中，h代表主机字节序（host）,n代表网络字节序(network)**
举例如下：
```
#include <cstdio>
#include <arpa/inet.h>  // for Endian Convertions
int main()
{
    // 主机字节序表示的端口和IP地址
    unsigned short host_port = 0x1234;
    unsigned long host_ip = 0x12345678;
    // 网络字节序表示的端口和IP地址
    unsigned short net_port;
    unsigned long net_ip;
    
    net_port = htons(host_port);
    net_ip = htonl(host_ip);

    //%x表示输出十六进制数
    //%#x表示带格式输出十六进制数，在输出前加0x
    printf("%#x,%#lx",net_port, net_ip);  //0x3412,0x78563412
    return 0;
}
```
**5.将字符串表示的ip地址转化为大端序(网络字节序)整形数值。**

1.inet_addr函数的应用：
```
//函数声明
in_addr_t inet_addr(const char* cp);
//本函数成功时返回一个大端序的32位整型值，失败返回INADDR_NONE(也就是值为-1).
// in_addr_t在内部声明为32位整数型
```
举例说明：
```
#include <stdio.h>
#include <arpa/inet.h>

int main()
{
    const char* host_ip1 = "1.2.3.4";
    const char* host_ip2 = "1.2.3.256";
    unsigned long net_ip1 = inet_addr(host_ip1);
    if(net_ip1 == INADDR_NONE)
    {
        printf("error ocurs!\n");
    }
    printf("%#x\n",net_ip1);
    net_ip1 = inet_addr(host_ip2);
    if(net_ip1 == INADDR_NONE)
    {
        printf("error ocus!\n");
    }
    printf("%d\n",net_ip1);
    return 0;
}
//0x4030201
//error ocus!
//-1
```

2.inet_aton函数的应用：
```
//该函数声明：
int inet_aton(const char* cp,struct in_addr* inp);
// inp表示将保存转换结果的in_addr结构体变量的地址值
//成功时返回1，失败时返回0
```
函数举例：
```
#include <stdio.h>
#include <arpa/inet.h>

int main()
{
    const char* host_ip = "1.2.3.255";
    struct sockaddr_in ip_addr;
    if(inet_aton(host_ip,&ip_addr.sin_addr) == 0)
    {
        printf("error ocurs!");
    }
    else
    {
        // 0xff030201
        printf("%#x\n",ip_addr.sin_addr.s_addr);
    }
    return 0;
}
```
<font color = red>如果调用inet_addr函数需要将转换后的IP地址信息代入sockaddr_in结构体中声明的in_addr结构体变量，而inet_aton函数不需要此过程。因此这个函数使用频率更高。</font>

3.inet_pton函数的应用
 **inet_pton函数：** 将字符串形式的IP地址转化为大端整形数值。**本函数支持IPV６，所以推荐在进行将字符串形式的IP地址主机序转化为整形形式的IP地址的网络字节序时使用这个函数。**
```
#include <stdio.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <string.h>
int main()
{
    struct sockaddr_in addr_in;
    memset(&addr_in,0,sizeof(addr_in));
    addr_in.sin_port = htons(0x1234);
    inet_pton(AF_INET,"127.0.0.1",&addr_in.sin_addr.s_addr);
    printf("%x\n",addr_in.sin_addr.s_addr);
    unsigned int ip = inet_addr("127.0.0.1");
    printf("%x\n",ip);
    return 0;
}
```
运行结果如下：
```
xiaocer@linux:~/练习$ ./a.out        
100007f
100007f
```
**6.将网络字节序整数型ip地址转化为字符串形式**

1.inet_ntoa函数的应用
```
//函数声明：
char* inet_ntoa(struct in_addr in);
//该函数成功时返回字符串的地址，失败时返回-1；
// 本函数注意事项：该函数未向程序员要求分配存储字符串的内存，而是在内部申请了静态内存并保存了字符串。
// 再次调用inet_ntoa函数之前返回的字符串地址值是有效的，如果需要长期保存，则应将字符串复制到其他内存空间
```
**函数举例：**
```
#include <stdio.h>
#include <arpa/inet.h>
#include <string.h>
int main()
{
    char arr[32] = {0};
    struct sockaddr_in addr1,addr2;
    addr1.sin_addr.s_addr = htonl(0x10203040);
    addr2.sin_addr.s_addr = htonl(0x11223344);
    //1076895760
    printf("%d\n",addr1.sin_addr.s_addr);
    //0x40302010
    printf("%#x\n",addr1.sin_addr.s_addr);
    const char* str = inet_ntoa(addr1.sin_addr);
    if(str)
    {
        //16.32.48.64
        printf("%s\n",str);
    }
    strcpy(arr,str);
    //第二次调用inet_ntoa将覆盖前一次调用函数中保存的字符串值
    str = inet_ntoa(addr2.sin_addr);
    if(str)
    {
        //17.34.51.68
        printf("%s\n",str);
    }
    //16.32.48.64
    printf("%s\n",arr);
    return 0;
}
```
2.inet_ntop函数的使用：
**inet_ntop函数：** 将网络字节序整数型IP地址转化为字符串形式。**推荐使用这个函数，因为这个函数支持过更多的地址族，比如说AF_INET6.**
示例：
```
#include <stdio.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
int main()
{
    struct sockaddr_in addr;
    addr.sin_addr.s_addr = htonl(0x12345678);
    // 78563412
    printf("%x\n",addr.sin_addr.s_addr);
    const char* ip_str = inet_ntoa(addr.sin_addr);
    // 18.52.86.120
    printf("%s\n",ip_str);

    char des[32];
    // 18.52.86.120
    inet_ntop(AF_INET,&addr.sin_addr.s_addr,des,32);
    printf("%s\n",des);

    return 0;
}
```
==综上：==

<font color = blue size = 6px>将字符串表示的ip地址转化为大端序(网络字节序)整形数值的方法有以下几种：</font>
1. 使用inet_addr函数
2. 使用inet_aton函数
3. 使用inet_pton函数

<font color = blue size = 6px>将网络字节序整数型ip地址转化为字符串形式：</font>
1. 使用inet_ntoa函数
2. 使用inet_ntop函数
##### 5.INADDR_ANY的应用
每次创建服务器端套接字都要输入IP地址有些繁琐，所以==可以利用INADDR_ANY初始化地址信息；如果采用这种方式分配服务器端的IP地址，== 就可以自动获取服务器端的IP地址。
比如说在创建服务器端时：
```
struct sockaddr_in serv_addr;
serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
```
##### 6.向套接字分配网络地址
**bind函数：** 为套接字（服务端或者客户端）初始化地址信息。

1.bind函数声明：
```
//函数声明：
#include <sys/socket.h>
int bind(int sockfd,const struct sockaddr* address,socklen_t address_len);
// 函数返回值：成功返回0，失败返回-1
// 形参
    sockfd:表示要分配地址信息的套接字文件描述符
    address:表示存有地址信息的结构体变量地址值
    addrlen:第二个结构体变量的长度
```
2. bind函数简单举例：
```
struct sockaddr_in server_addr;
memset(&server_addr,0,sizeof(server_addr));
addr.sin_family = AF_INET;  //ipv4协议地址族
addr.sin_port = htons(6666);
addr.sin_addr.s_addr = htonl(INADDR_ANY);
bind(server_socket,(struct sockaddr*)&server_addr,sizeof(server_addr));
```
3. ==在一些特殊的场景中，需要客户端程序以指定的端口去连接服务器。这时就可以使用bind函数绑定一个具体的端口。（客户端绑定端口号0，系统会随机分配一个端口号）==
```

```

##### 7.一个简单示例
1. 服务器端:往一个客户端发送"HelloWorld"
```
#include <cstdio>
#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>
#include <cstring>
#include <cstdlib>
#include <unistd.h>

int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <port>\n", argv[0]);
        return -1;
    }
    // 创建服务器端套接字
    int sockfd;
    sockfd = socket(PF_INET, SOCK_STREAM, 0);
    if (sockfd == -1)
    {
        printf("socket()error!");
        return -1;
    }
    
    // 分配套接字地址
    struct sockaddr_in addr_in;
    memset(&addr_in, 0, sizeof(addr_in));
    addr_in.sin_family = AF_INET;   // ipv4协议地址族
    addr_in.sin_port = htons(atoi(argv[1]));    
    addr_in.sin_addr.s_addr = htonl(INADDR_ANY);
    if (bind(sockfd, (struct sockaddr*)&addr_in, sizeof(addr_in)) == -1)
    {
        printf("bind()error!");
        return -1;
    }

    // 侦听客户端连接请求
    if (listen(sockfd, 15) == -1)
    {
        printf("listen()error!");
        return -1;
    }

    // 受理客户端连接请求
    struct sockaddr_in client_addr_in;
    socklen_t client_addr_size = sizeof(client_addr_in);
    int clnt_fd = accept(sockfd, (struct sockaddr*)&client_addr_in, &client_addr_size);
    if (clnt_fd == -1)
    {
        printf("accept()error!");
        return -1;
    }

    // O
    const char* message = "HelloWorld";
    write(clnt_fd, message, strlen(message));
    
    // 释放连接
    close(sockfd);
    close(clnt_fd);
     
    return 0;
}

```
2. 客户端:接受来自服务端发送的"HelloWorld"
```
#include <cstdio>
#include <sys/socket.h>
#include <cstdlib>
#include <arpa/inet.h>
#include <unistd.h>
#include <cstring>

int main(int argc, char* argv[])
{
    if (argc != 3)
    {
        printf("usage:%s <IP> <port>\n", argv[0]);
        return -1;
    }
    // 创建客户端套接字
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd == -1)
    {
        printf("socket()error!");
        return -1;
    }

    // 发起连接请求
    struct sockaddr_in servaddr;
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(atoi(argv[2]));
    inet_pton(AF_INET, argv[1], &servaddr.sin_addr);
    int ret = connect(sockfd, (struct sockaddr*)&servaddr, sizeof(servaddr));
    if (ret == -1)
    {
        printf("connect() error!\n");
        return -1;
    }

    // I
    char buff[100] = {0};
    int read_len = read(sockfd, buff, sizeof(buff));
    printf("%s\n",buff);

    // 释放连接
    close(sockfd);
    return 0;
}
```