## 基于UDP的服务器端和客户端

本节重点：

1. 熟悉TCP协议和UDP协议的区别

##### 1.TCP和UDP的区别
流控制是区分UDP和TCP的最重要的标志。UDP不会进行流控制，他提供的是不可靠的数据传输服务。与对方套接字建立连接和断开连接、超时重传机制等都属于流控制的一部分。
##### 2.UDP的高效使用
虽然可靠性方面比不上TCP，但是UDP在结构上比TCP更简洁。UDP不会发送类似ACK的应答消息，也不会像SEQ那样给数据包分配序号。==因此UDP的性能有时比TCP高出很多==。 通过网络实时传输视频或者音频考虑使用UDP
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
<font color = blue size = 6px>每次调用sendto函数传输数据都要添加目标地址信息，因为UDP套接字不会像TCP一样保持与对方套接字的连接状态。又由于UDP的数据发送端不固定，所以recvfrom函数定义为可接收发送端地址信息的形式</font>

##### 4.UDP客户端套接字的地址分配
如果不显式使用bind函数给客户端套接字绑定IP和端口信息，则调用sendto函数传输数据前自动分配客户端套接字的IP地址和端口号

##### 5.基于UDP的回声服务器端和客户端

1. 服务端：

   ```
   // 基于UDP实现的回声服务器端
   
   #include <stdio.h>
   #include <arpa/inet.h>
   #include <stdlib.h>
   #include <string.h>
   #include <unistd.h> // for close()
   
   int main(int argc, char* argv[]) {
       if (argc != 2) {
           printf("usage:%s <port>\n", argv[0]);
           return -1;
       }
   
       int fd = socket(PF_INET, SOCK_DGRAM, 0);
       if (fd == -1) {
           perror("socket()error\n");
           return -1;
       }
   
       struct sockaddr_in serv_addr;
       serv_addr.sin_family = AF_INET;
       serv_addr.sin_port = htons(atoi(argv[1]));
       serv_addr.sin_addr.s_addr = INADDR_ANY;
       if (bind(fd, (const struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
           perror("bind()error\n");
           close(fd);
           return -1;
       }
   
       char buff[128];
       struct sockaddr_in client_addr;
       socklen_t len = sizeof(client_addr);
       while (1) {
           memset(buff, 0, sizeof(buff));
           int number = recvfrom(fd, buff, sizeof(buff), 0, (struct sockaddr*)&client_addr, &len);
           if (number == -1) {
               perror("recvfrom()error\n");
               break;
           } else if (number > 0) {
               printf("收到客户端ip:%s，端口:%d的数据\n", 
                   inet_ntoa(client_addr.sin_addr),
                   ntohs(client_addr.sin_port));
               sendto(fd, buff, number, 0, (const struct sockaddr*)&client_addr, len);
           }
       }
   
       close(fd);
       return 0;
   }
   ```

   2. 客户端：

      ```
      // 基于UDP实现的回声客户端
      
      #include <stdio.h>
      #include <arpa/inet.h>
      #include <stdlib.h>
      #include <unistd.h>
      #include <string.h>
      
      int main(int argc, char* argv[]) {
          if (argc != 3) {
              printf("usage:%s <ip> <port>\n", argv[0]);
              return -1;
          }
      
          int fd = socket(PF_INET, SOCK_DGRAM, 0);
          if (fd == -1) {
              perror("socket()error\n");
              return -1;
          }
      
          struct sockaddr_in serv_addr;
          socklen_t serv_addr_len = sizeof(serv_addr);
          serv_addr.sin_family = AF_INET;
          inet_pton(AF_INET, argv[1], &serv_addr.sin_addr.s_addr);
          serv_addr.sin_port = htons(atoi(argv[2]));
      
          char message[128];
          while (1) {
              bzero(message, 0);
              fputs("insert message(q to quit):", stdout);
              fgets(message, sizeof(message), stdin);
              if (!strcmp(message, "q\n") || !strcmp(message, "Q\n")) {
                  break;
              }
              sendto(fd, message, strlen(message), 0, (const struct sockaddr*)&serv_addr, serv_addr_len);
              int read_len = recvfrom(fd, message, sizeof(message), 0, (struct sockaddr*)&serv_addr, &serv_addr_len);
              message[read_len] = 0;
              printf("message from server:%s\n", message);
          }
      
          close(fd);
          return 0;
      }
      ```

      ##### 6.UDP的数据传输特性

      1. UDP数据传输中是存在数据边界的，这和UDP协议有关系，每个UDP数据报报文格式中都有一个长度字段记录UDP的数据报长度。而TCP传输的数据不存在数据边界。必须保证在UDP通信过程中使得IO函数的调用次数保持一致。

      2. 示例：

          1. 服务端：

             ```
             #include <stdio.h>
             #include <arpa/inet.h>
             #include <unistd.h>
             #include <string.h>
             
             int main() {
                 int fd = socket(PF_INET, SOCK_DGRAM, 0);
             
                 struct sockaddr_in serv_addr;
                 socklen_t len = sizeof(serv_addr);
                 serv_addr.sin_addr.s_addr = INADDR_ANY;
                 serv_addr.sin_family = AF_INET;
                 serv_addr.sin_port = htons(9996);
                 bind(fd, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));
             
                 char buff[128];
             
                 // 客户端调用三次发送数据，服务端将会调用三次接收
                 for (int i = 0; i < 3; ++i) {
                 	// 保证客户端调用三次sendto发送的数据成功接收
                     sleep(5);
                     bzero(buff, sizeof(buff));
                     recvfrom(fd, buff, sizeof(buff), 0, (struct sockaddr*)&serv_addr, &len);
                     printf("message %d:%s\n", i + 1, buff);
                 }
                 close(fd);
                 return 0;
             }
             ```

             

          2. 客户端：

             ```
             #include <stdio.h>
             #include <arpa/inet.h>
             #include <unistd.h>
             
             int main() {
                 int fd = socket(PF_INET, SOCK_DGRAM, 0);
                 
                 char message1[] = "Hi";
                 char message2[] = "where you are from?";
                 char message3[] = "nice to meet you.";
             
                 struct sockaddr_in serv_addr;
                 serv_addr.sin_addr.s_addr = INADDR_ANY;
                 serv_addr.sin_family = AF_INET;
                 serv_addr.sin_port = htons(9996);
             
                 sendto(fd, message1, sizeof(message1), 0, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));
                 sendto(fd, message2, sizeof(message2), 0, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));
                 sendto(fd, message3, sizeof(message3), 0, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));
                 
                 close(fd);
                 return 0;
             }
             ```

             ##### 7.已连接UDP套接字和未连接UDP套接字

             1. 通过sendto函数传输数据的过程大致分为以下三个阶段：
                 	1. 向UDP套接字注册目标IP和端口号
                 	2. 传输数据
                 	3. 删除UDP套接字中注册的目标地址信息。

             2. 每次调用sendto函数需要重复上述过程。每次都变更地址，因此可以重复利用同一UDP套接字向不同目标传输数据。这种未注册目标地址信息的套接字称之为未连接套接字（没有和指定的IP端口绑定）。==考虑一种情况，向一个指定IP和端口的主机准备了三个数据，调用sendto函数进行传输。此时由于是未连接的UDP套接字，sendto函数需要重复执行上述的三个步骤。== 因此将未连接的UDP套接字变成已连接的UDP套接字可以提高效率。

             3. 已连接UDP套接字的创建：针对UDP套接字调用connect函数。由于指定了收发对象，因此除了使用recvfrom、sendto函数，还可以使用write、read函数进行通信。

                ```c
                // 回声客户端的改进
                #include <stdio.h>
                #include <arpa/inet.h>
                #include <unistd.h>
                #include <string.h>
                
                int main() {
                    int fd = socket(PF_INET, SOCK_DGRAM, 0);
                
                    struct sockaddr_in serv_addr;
                    serv_addr.sin_family = AF_INET;
                    serv_addr.sin_port = htons(9999);
                    inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr.s_addr);
                    
                    // 只是向UDP套接字注册目标IP和端口信息
                    connect(fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
                
                    char buff[1024];
                    while (1) {
                        memset(buff, 0, sizeof(buff));
                        fputs("insert message(q to quit):", stdout);
                        fgets(buff, sizeof(buff), stdin);
                        if (!strcmp(buff, "q\n") || !strcmp(buff, "Q\n")) {
                            break;
                        }
                        write(fd, buff, strlen(buff));
                        int read_len = read(fd, buff, sizeof(buff));
                        buff[read_len] = 0;
                        printf("客户端接收到数据:%s\n", buff);
                    }
                
                    close(fd);
                    return 0;
                }
                ```

                