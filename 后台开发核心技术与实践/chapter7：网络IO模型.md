## 1.四种网络IO模型
1. 阻塞IO模型：
    1. 在Linux中，默认情况下所有的套接字都是阻塞的。
    2. 大部分的socket接口都是阻塞的，比如说accept、connect、send、recvfrom
    3. 阻塞IO模型示例如下：
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-2d2c869281b8b3e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 非阻塞IO模型：
    1. 在Linux下，可以通过设置socket使得IO变为非阻塞状态。
    2. 在非阻塞IO中，用户进程需要不断主动的询问内核数据是否已经准备好。非阻塞的接口在被调用之后立即返回。
    3. 非阻塞IO模型示例如下：
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-db035493bb4c16b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. IO多路复用模型：又称为事件驱动IO
    1. IO多路复用模型的一个实例：
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-92a6f1656488f589.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
4. 异步IO模型：
    1. 异步IO模型流程图
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-95cd02f5a976ae68.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
==总结：==
1. 阻塞IO、非阻塞IO和多路IO复用都属于同步IO。同步IO在进行IO操作时会阻塞进程。==非阻塞IO在执行接收数据的系统调用时，如果内核的数据没有准备好，这时候不会阻塞进程；但是当内核中的数据准备好时，接收数据的系统调用会将数据从内核拷贝到用户内存中，这时候进程被阻塞。==
2. 各种IO模型的比较：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-2e9f5f530406f094.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.select
1. 示例1：使用select循环读取键盘输入
```
// 使用select循环读取键盘输入
#include <sys/select.h>
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <assert.h>

int main() {
    // 非阻塞的读取终端上的信息
    int fd = open("/dev/tty", O_RDONLY | O_NONBLOCK);
    assert(fd > 0);
    printf("%d\n", fd);

    fd_set readfds;
    char c;

    while (1) {
        struct timeval timeout;
        timeout.tv_sec = 2;
        timeout.tv_usec = 0;
        FD_ZERO(&readfds);
        FD_SET(fd, &readfds);
        int ret = select(fd + 1, &readfds, NULL, NULL, &timeout);
        assert(ret >= 0);
        if (ret == 0) {
            printf("time out\n");
            continue;
        } 
        if (FD_ISSET(fd, &readfds)) {
            int read_len = read(fd, &c, 1);
            if ('\n' == c) continue;
            if ('q' == c) break;
            printf("input character is:%c\n", c);
        }
    }

    close(fd);

    return 0;
}
```
## 3.poll
## 4.epoll
1. epoll的操作过程需要三个接口，分别是：
    1. epoll_create
    2. epoll_ctl
    3. epoll_wait
2. 示例：实现回声服务器端
    1. 服务器端：
    ```
    #include <stdio.h>
    #include <sys/epoll.h>
    #include <sys/socket.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <string.h>
    #include <assert.h>
    #include <arpa/inet.h>
    #define EPOLL_EVENTS 1024
    
    int main(int argc, char* argv[]) {
        if (argc != 3) {
            printf("usage:%s <IP> <PORT>\n", argv[0]);
            return -1;
        }
        int lfd = socket(PF_INET, SOCK_STREAM, 0);
        assert(lfd != -1);
    
        struct sockaddr_in serv_addr;
        serv_addr.sin_family = AF_INET;
        serv_addr.sin_port = htons(atoi(argv[2]));
        inet_pton(AF_INET, argv[1], &serv_addr.sin_addr);
    
        int ret = bind(lfd, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));
        assert(ret != -1);
    
        listen(lfd, 128);
    
        // 创建epoll实例
        int epfd = epoll_create(1);
    
        struct epoll_event event;
        event.events = EPOLLIN;
        event.data.fd = lfd;
        // 监视用于监听客户端连接请求的套接字的读事件
        ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &event);
        assert(ret != -1);
    
        struct epoll_event events[EPOLL_EVENTS];
        char buff[1024];
        while (1) {
            ret = epoll_wait(epfd, events, EPOLL_EVENTS, -1);
            assert(ret != -1);
    
            int fd;
            for (int i = 0; i < ret; ++i) {
                fd = events[i].data.fd;    
                // 根据描述符的类型和事件的类型进行处理
                if (fd == lfd && events[i].events & EPOLLIN) {
                    struct sockaddr_in client_addr;
                    socklen_t client_addr_len = sizeof(client_addr);
                    int client_fd = accept(lfd, (struct sockaddr*)&client_addr, &client_addr_len);
                    if (client_fd == -1) {
                        perror("accept()error\n");
                    } else {
                        event.events = EPOLLIN;
                        event.data.fd = client_fd;
                        epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &event);
                        char ip[64];
                        printf("客户端ip:%s,端口：%d成功建立连接\n", inet_ntop(AF_INET, &client_addr.sin_addr, ip, sizeof(ip)),
                            ntohs(client_addr.sin_port));
                        
                    }
                } else if (events[i].events & EPOLLIN) {
                    // 用于通信的套接字有数据可读
                    int read_len = read(fd, buff, sizeof(buff) - 1);
                    if (read_len == 0) {
                        close(fd);
                        printf("客户端断开连接\n");
                        epoll_ctl(epfd, EPOLL_CTL_DEL,fd, NULL);
                    } else if (read_len == -1) {
                        close(fd);
                        epoll_ctl(epfd, EPOLL_CTL_DEL,fd, NULL);
                    } else {
                        buff[read_len] = 0;
                        printf("接收到客户端的消息：%s\n", buff);
                        // 监视通信文件描述符的写缓冲区是否可写，服务端进行回声
                        event.data.fd = fd;
                        event.events = EPOLLOUT;
                        epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &event);
                    }
                } else if (events[i].events & EPOLLOUT) {
                    // 用于通信的套接字有数据可写
                    int write_len = write(fd, buff, strlen(buff));
                    if (write_len == -1) {
                        close(fd);
                        epoll_ctl(epfd, EPOLL_CTL_DEL,fd, NULL);
                    } else {
                        event.data.fd = fd;
                        event.events = EPOLLIN;
                        epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &event);
                    }
                    memset(buff, 0, sizeof(buff));
                }
            }
        }
    
        close(lfd);
        close(epfd);
    }
    ```
## 5.select/poll/epoll的区别
1. 三者都属于IO多路复用的机制。
2. poll相比于select来说，更加高级。原因如下：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-fcfaae94d49d8048.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. select的优点如下：
    1. select的可移植性好，在某些UNIX系统上不支持poll
    2. select对于超时值提供了更好的精度，而poll精度较差。
3. epoll的优点如下：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-ada11b9ae49a5a07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
