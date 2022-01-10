## 域名及网络地址
##### 1.域名系统
1. 域名系统简称DNS（domain name system），DNS是对IP地址和域名进行相互转换的系统，其核心是DNS服务器。
2. 域名：将容易记，易表述的域名分配并取代IP地址
##### 2.域名转换为IP地址
1. 利用域名获取IP地址
```
// 通过传递字符串格式的域名获取IP地址
#include <netdb.h>
struct hostent* gethostbyname(const char* hostname);
函数返回值：
    成功时返回hostent结构体地址，失败时返回NULL
```
2. hostent结构体如下
![image.png](https://upload-images.jianshu.io/upload_images/17728742-3be7393840f2be8b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
成员解释如下
```
1. h_name:存有官方域名
2. h_aliases:存有除官方域名外的其他域名
（可以通过多个域名访问同一主页，同一IP也可以绑定多个域名）
3. h_addrtype:获取h_addr_list的IP地址的地址族信息。
（如果是IPV4，则此变量存有AF_INET）
4. h_length:保存IP地址长度（其值为4或者16）
5. h_addr_list:保存域名对应的IP地址
```
3. 示例
```
#include <cstdio>
#include <netdb.h>
#include <cstdlib>
#include <arpa/inet.h>

int main(int argc, const char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <addr>", argv[0]);
        exit(-1);
    }
    struct hostent* host;
    host = gethostbyname(argv[1]);
    if (host)
    {
        printf("official name:%s\n", host->h_name);
        for (int i = 0; host->h_aliases[i]; i++)
        {
            printf("Aliases:%s\n", host->h_aliases[i]);
        }
        printf("addr type:%s\n", host->h_addrtype == AF_INET ? "AF_INET" : "AF_INET6");
        for (int i = 0; host->h_addr_list[i]; i++)
        {
            printf("address:%s\n", inet_ntoa(*(struct in_addr*)host->h_addr_list[i]));
        }
    }
    return 0;
}
```
运行结果如下：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-6988f4597a0b6817.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 3.根据IP地址获取域名
1. gethostbyaddr函数
```
#include <netdb.h>
struct hostent* gethostbyaddr(const char* addr, socklen_t len, int family);
函数返回值：
    成功时返回hostent结构体变量地址值，失败则返回NULL
函数参数：
    addr:含有IP地址信息的in_addr结构体指针
    len:向第一个参数传递的地址信息的字节数，IPV4为4
    family:传递地址族信息，AF_INET或者AF_INET6
```
2. 示例
```
#include <cstdio>
#include <netdb.h>
#include <cstdlib>
#include <arpa/inet.h>
#include <cstring>

int main(int argc, const char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <addr>", argv[0]);
        exit(-1);
    }
    struct hostent* host;
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_addr.s_addr = inet_addr(argv[1]);
    host = gethostbyaddr(&addr.sin_addr, 4, AF_INET);
    if (host)
    {
        printf("official name:%s\n", host->h_name);
        for (int i = 0; host->h_aliases[i]; i++)
        {
            printf("Aliases:%s\n", host->h_aliases[i]);
        }
        printf("addr type:%s\n", host->h_addrtype == AF_INET ? "AF_INET" : "AF_INET6");
        for (int i = 0; host->h_addr_list[i]; i++)
        {
            printf("address:%s\n", inet_ntoa(*(struct in_addr*)host->h_addr_list[i]));
        }
    }
    return 0;
}
```