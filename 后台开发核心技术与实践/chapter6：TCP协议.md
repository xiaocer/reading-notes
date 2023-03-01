## 1.tcp协议
##### 1.网络模型
1. 七层网络模型：由国际标准组织ISO推出，又称开放系统互联模型，简称OSI
    1. 七层网络模型及其功能展示如下：
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-0e145a2423aa2fdb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 五层网络模型：大学教科书中常用
![image.png](https://upload-images.jianshu.io/upload_images/17728742-915fdff68b97cf32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 四层网络模型：使用最广泛的一个四层模型，TCP/IP分层模型
![image.png](https://upload-images.jianshu.io/upload_images/17728742-b0b6629c14834aa2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 2.TCP头部结构
1. TCP头部的格式如下：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-c560ab7d7221f18a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. 16位源端口号：告知主机该报文段是来自哪一个源端口
    2. 16位目的端口：告知主机传递给哪个上层协议或者应用程序的
    3. 32位序号：指的是某一个传输方向上的字节流的每个字节的编号。==用于解决网络包乱序的问题==
    4. 32位确认号：用作对另一方发送来的TCP报文段的响应
    5. 4位头部长度：标识该头部有多少个4字节。因为4个bit位最大能表示15，所以TCP头部最长为60个字节
    6. 6位标志位如下（用于操控TCP的状态机）：
        1. URG标志：表示紧急指针是否有效
        2. ACK标志：表示确认号是否有效。一般携带ACK标志的TCP报文段称之为确认报文段。==用于解决不丢包的问题==
        3. PSH标志：提示应用程序应该立即从TCP接收缓冲区中读走数据，为接收后续数据腾出空间
        4. RST标志：表示要求对方重新建立连接，一般携带RST标志的TCP报文段称之为复位报文段
        5. SYN标志：表示请求建立一个连接，一般携带SYN标志的TCP报文段称之为同步报文段
        6. FIN标志：表示通知对方本端要关闭连接了，一般携带FIN标志的TCP报文段称之为结束报文段
    7. 16位窗口大小：这是TCP流量控制的一个手段。这个窗口是接收通告窗口，他告诉对方本端的TCP接收缓冲区还可以容纳多少字节的数据。==这样对方就可以控制发送数据的速度，这个字段用于解决流量控制的问题==
    8. 16位校验和：由发送端填充，接收端根据接收到的TCP报文段执行CRC算法，以检验TCP报文段（包括TCP头部和数据部分）在传输过程中是否被损坏。==这也是TCP可靠传输的一个重要保障==
    9. 16位紧急指针：这是一个正的偏移量，它和序号字段的值相加表示最后一个紧急数据的下一字节的序号。TCP的紧急指针是发送端向接收端发送紧急数据的方法。
##### 3.TCP的状态流转
1. TCP的三次握手和四次挥手：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-41531072f39160db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. 建立三次握手的原因：主要是要初始化序列号的初始值，通信的双方要通知对方自己的数据通信的序号。
2. TCP的状态流转图如下：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-2729d8b954573196.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. CLOSED：表示初始状态
    2. LISTEN：表示服务器的某个socket处于监听状态，可以接受连接
    3. SYN_SENT：在服务端监听后，客户端socket执行connect连接时，客户端发送SYN报文，此时客户端就进入SYN_SENT状态，等待服务端的确认。
    4. SYN_RCVD：表示服务端收到了SYN报文，在正常情况下，这个状态是服务器端的socket在建立TCP链接时的三次握手会话过程中的一个中间状态。
    5. ESTABLISHED：表示连接已经建立
    6. FIN_WiIT_1：这个是已经建立连接后，其中一方请求终止连接，等待对方的FIN报文。当对方回应了ACK报文，则进入FIN_WAIT_2状态。
    7. FIN_WAIT_2：FIN_WAIT_2状态下的套接字，表示半连接，即有一方要求关闭连接。==即半关闭状态==
    8. TIME_WAIT：表示收到了对方的FIN报文，并发送出了ACK报文，就等2MSL后即可回到CLOSED可用状态了。MSL：Maximum Segment Lifetime，即报文的最大生存时间，一般1个MSL为2分钟。
    9. CLOSING：一端发送FIN报文后，并没有收到对方的ACK报文，反而收到了对方的FIN报文。CLOSING状态表示双方都正在关闭套接字连接。
    10. CLOSE_WAIT：当对方关闭一个套接字后发送FIN报文给自己时，系统就回应一个ACK报文给对方，此时就进入CLOSE_WAIT状态。
    11. LAST_ACK：它是被动关闭一方在发送FIN报文后，最后等待对方的ACK报文
    12. CLOSED：当收到ACK报文后，就进入到CLOSED可用状态了
##### 4.TCP的超时重传
1. 超时重传：TCP每发送一个报文段，就会对这个报文段设置一次计时器。只要计时器设置的重传时间到了，但没有收到确认，就要重传这一报文段。
2. 影响超时重传机制协议效率的一个关键参数是RTO（Retransmission Timeout）超时重传时间。
3. TCP协议使用自适应算法以适应互联网分组传输时延的变化，这种算法的基本要点是TCP监视每个连接的性能，由每一个TCP的连接情况推算出合适的RTO值。自适应重传算法的关键是对当前RTT的准确估计，以便调整RTO
4. RTT：全称Round Trip Time，即连接往返时间。指的是发送端从发送TCP包开始到接受他的立即响应所耗费的时间。
##### 5.TCP滑动窗口
1. 在TCP头部中和TCP滑动窗口相关的字段是16位窗口大小，她表示的是窗口的字节容量。
2. 另外TCP的选项字段中还包含一个TCP窗口扩大因子。
## 2.TCP网络编程API
1. TCP的交互流程
![image.png](https://upload-images.jianshu.io/upload_images/17728742-cf0473037c17b926.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 常用的网络编程API：
    1. socket：创建套接字，调用成功返回一个套接字描述符，套接字描述符存在每个进程的进程空间的套接字描述符表中。==这个表中的一个字段存放新创建的套接字的描述符，另一个字段存放套接字数据结构的地址，套接字数据结构都存放在操作系统的内核缓冲里。==
    2. bind：给套接字绑定IP和端口号
    3. listen：创建一个侦听队列
    4. accept：从监听队列取出一个连接
    5. read/write、recv/send、readv/writev、recvmsg/sendmsg、recvfrom/sendto
    6. close：使得相应的套接字描述符的引用计数减一，当引用计数为0的时候，才会触发TCP客户端向服务器发送终止连接请求。
3. TCP协议选项
    1. SO_REUSEADDR
    2. SO_RECVBUF/SO_SNDBUF
    3. SO_KEEPALIVE
    4. SO_LINGER
    5. TCP_CORK
    6. TCP_NODELAY
    7. TCP_KEEPCNT/TCP_KEEPIDLE/TCP_KEEPINTVL
    8. SO_SNDTIMEO/SO_RCVTIMEO
    9. TCP_DEFER_ACCEPT
    10. SO_RCVBUF和SO_SNDBUF
4. 选项SO_REUSEADDR：
    1. 该选项的功能：重用已经处于TIME_WAIT状态的套接字占用的端口。
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-1f44cb084cbeb1aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
5. 选项TCP_NODELAY/TCP_CHORK
    1. 拥塞控制领域的Nagle算法：TCP中的Nagle算法默认是启用的。如果发送端欲多次发送包含少量字符的数据包，则发送端会先将第一个小包发送出去，而将后面到达的少量字符数据都缓存起来而不立即发送，直到接收到接收端对前一个数据包报文段的ACK确认为止。
    2. TCP_NODELAY和TCP_CORK基本控制了包的Nagle化。
        1. TCP_NODELAY不使用Nagle算法，不会将小包进行拼接成大包再进行发送，而是直接将小包发送出去。
        2. TCP_CORK：
        ![image.png](https://upload-images.jianshu.io/upload_images/17728742-92ca423f565de643.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
6. SO_LINGER：延缓套接字的close操作，默认close操作立即返回，将发送缓冲区的数据发送给对端。
    1. SO_LINGER可以改变close的行为，控制SO_LINGER通过下面这个结构体
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-6eceec7db8807808.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
7. 选项TCP_DEFER_ACCEPT：当接收到第一个数据后才会创建连接。这个字段可以用来防御空连接攻击（只是建立连接但是不发送任何数据）。
8. 选项SO_RCVBUF和SO_SNDBUF：每个TCP套接字都有一个发送缓冲区和接收缓冲区。这两个套接字选项可以改变默认缓冲区的大小
9. 选项SO_SNDTIMEO和SO_RCVTIMEO：分别设置socket的发送和接收超时时间。如果在超时时间内还没有收到数据，就会关闭连接。
10. SO_KEEPALIVE：用于保持连接，检测对方主机是否崩溃，避免服务器永久阻塞于TCP连接的输入。
	1. 设置该选项可能导致的情况：
	![image.png](https://upload-images.jianshu.io/upload_images/17728742-6fe4fe975047ec2b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	![image.png](https://upload-images.jianshu.io/upload_images/17728742-4f2200b3d99ca895.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	2. 有关SO_KEEPALIVE的三个参数的解释如下
	![image.png](https://upload-images.jianshu.io/upload_images/17728742-e870ea0d1ddc41ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.网络字节序和主机字节序
1. 主机字节序：指的是整数在内存中保存的顺序。常见的有两种：
	1. 小端序：将低位有效字节存放在起始地址处
	2. 大端序：将高位有效字节存放在起始地址处
2. 网络字节序：所有网络协议都是采用大端序的方式传输数据的。所以大端序又称之为网络字节序。
## 4.封包和解包
1. 粘包：TCP是一个基于字节流的协议，所谓流，就是没有界限的一串数据。在发送端连续发送两段数据，接收端接收时无法将接收到的数据进行区分拆成两段。==UDP协议就不存在粘包问题==
	1. 粘包的问题示例：在数据传输时假设连续两次调用send发送两段数据data1和data2，在接收端可能有如下情况：
		1. 先接收到data1，然后接收data2(正常的，正是我们需要的)
		2. 先接收到data1的部分数据，然后接收到data1余下的部分以及data2的全部
		3. 先接收到data1的全部数据和data2的部分数据，然后接收到了data2的余下的数据
		4. 一次性接收到了data1和data2的全部数据
	2. 粘包可能发生在发送端，也可能发生在接收端。粘包的问题原因分析如下：
		1. 对于粘包问题34，有可能因为Nagle算法造成的发送端的粘包。当要提交一段数据给TCP发送时，TCP并不立即发送此段数据，而是等待一小段时间，看看在等待期间是否还有要发送的数据，如果有则会一次性将多段数据发送出去
		2. 对于粘包问题2，可能由于接收端接收数据不及时造成的接收端粘包。
2. 粘包问题的解决：对数据包进行封包和解包
	1. 封包：给一段数据加上固定长度的包头，例如4个字节表示包体的长度。包头中有个字段记录了包体的长度，这样一段数据就分为包头和包体。
3. 封包解包示例：客户端向服务端发送一段数据
	1. 服务端：
	```
	// 服务端接受一个字符串
    #include <arpa/inet.h>
    #include <unistd.h>
    #include <stdio.h>
    #include <stdlib.h>
    #include <errno.h>

    int readN(int fd, char* buffer, int count) {
        int read_len = 0;
        while (read_len < count) {
            int ret;
            do {
                ret = read(fd, buffer, count - read_len);
            } while (ret < 0 && errno == EINTR); // read读取中被信号中断则继续
            if (ret == -1) {
                return ret;
            } else if (ret == 0) {
                return read_len;
            } 
            read_len += ret;
            buffer += ret;
        }
        return read_len;
    }
    int main(int argc, char* argv[]) {
        if (argc != 3) {
            printf("usage:%s <IP> <PORT>\n", argv[0]);
            return -1;
        }
        int lfd = socket(PF_INET, SOCK_STREAM, 0);

        struct sockaddr_in serv_addr;
        serv_addr.sin_family = AF_INET;
        serv_addr.sin_port = htons(atoi(argv[2]));
        inet_pton(AF_INET, argv[1], (void*)&serv_addr.sin_addr);
        bind(lfd, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));

        listen(lfd, 128);

        char buffer[128];
        int client_fd = accept(lfd, NULL, NULL);

        // 接收4个字节的包头
        int read_len = readN(client_fd, buffer, 4);
        if (read_len != 4) {
            return -1;
        }

        int packetHead = ntohl(*(int*)buffer);
        printf("包体的大小为:%d\n", packetHead);

        read_len = readN(client_fd, buffer, packetHead);
        if (read_len != packetHead) {
            return -1;
        }
        buffer[read_len] = 0;
        printf("客户端发送的数据为:%s\n", buffer);

        close(client_fd);
        close(lfd);

        return 0;
    }
	```
	2. 客户端：
	```
	#include <stdio.h>
    #include <arpa/inet.h>
    #include <unistd.h>
    #include <stdlib.h>
    #include <string.h>
    #include <errno.h>

    int sendN(int fd, char* buffer, int count) {
        int send_len = 0;
        while (send_len < count) {
            int write_len;
            do {
                write_len = write(fd, buffer, count - send_len);
            } while (write_len < 0 && errno == EINTR);

            if (write_len == -1) {
                return -1;
            } else if (write_len == 0) {
                return send_len;
            }
            send_len += write_len;
            buffer += write_len;
        }
        return send_len;
    }
    int main(int argc, char* argv[]) {
        if (argc != 3) {
            printf("usage:%s <IP> <PORT>\n", argv[0]);
            return -1;
        }
        int fd = socket(PF_INET, SOCK_STREAM, 0);

        struct sockaddr_in serv_addr;
        serv_addr.sin_family = AF_INET;
        serv_addr.sin_port = htons(atoi(argv[2]));
        inet_pton(AF_INET, argv[1], (void*)&serv_addr.sin_addr);

        connect(fd, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));

        char data[] = "0123456789";
        int data_len = strlen(data);
        char* buffer = (char*)malloc(128);
        // 封包：四个字节的包头
        *(int*)buffer = htonl(data_len);
        memcpy(buffer + sizeof(data_len), data, data_len);

        int ret = sendN(fd, buffer, data_len + 4);
        if (ret <= 0) {
            return -1;
        } 
        printf("客户端发送的数据长度：%d,发送的数据为:%s\n", data_len + 4, data);

        free(buffer);

        close(fd);
        return 0;
    }
	```