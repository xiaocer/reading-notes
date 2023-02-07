# 多进程服务器端

本节要点：

1. 熟悉僵尸进程的产生原因以及回收僵尸进程的多种方法
2. 了解孤儿进程
3. 熟悉信号处理机制
4. 知道如何分割IO，在客户端程序中，一个进程用于读，一个进程用于写。

## 1.并发服务器端实现模型和方法
1. 多进程服务器：通过创建多个进程提供服务
2. 多路复用服务器：通过捆绑并统一管理I/O对象提供服务
3. 多线程服务器：通过生成与客户端等量的线程提供服务
## 2.进程
##### 1.进程ID
所有进程都会从OS分配到ID。如下图所示，使用ps -au命令查看进程的id。PID即为进程ID。
![image.png](https://upload-images.jianshu.io/upload_images/17728742-7966504f9e0406b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 2.fork函数创建进程
1. fork函数：fork函数复制正在运行的，调用fork函数的进程。执行fork后，两个进程都将会执行fork函数返回后的语句。
```
#include <unistd.h>

pid_t fork(void);
函数返回值：
    成功时返回进程id，失败时返回-1
```
2. 如何区分父进程和子进程？
```
对于父进程（调用fork函数的主体）：fork函数返回子进程ID
对于子进程：fork函数返回0
```
3. fork函数使用示例
```
#include <stdio.h>
#include <unistd.h>

int global = 100;
int main()
{
    int autovar = 200;
    pid_t id = fork();
    // 子进程
    if (id == 0)
    {
        global += 100;
        autovar += 100;
    }
    // 父进程
    else
    {
        global -= 100;
        autovar -= 100;
    }
    if (id == 0)
    {
        printf("pid:%d,global:%d,autovar:%d\n", getpid(), global, autovar);
    }
    else
    {
        printf("pid:%d,global:%d,autovar:%d\n", getpid(), global, autovar);
    }
    return 0;
}
// pid:78501,global:0,autovar:100
// pid:78502,global:200,autovar:300
```
## 3.僵尸(zombie)进程
进程完成工作后（执行完main中的程序后），应当被销毁但是有时这些进程占用系统的重要资源，变成僵尸进程。==即僵尸进程：子进程死了，父进程没有回收子进程的资源==
##### 1.创建僵尸进程示例

​	1. 示例代码如下：

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main()
{
    pid_t child_id = fork();
    //子进程
    if(child_id == 0)
    {
        printf("I am child,child id:%d parent id:%d\n",getpid(),getppid());
        sleep(2);
        printf("child process died!\n");
    }
    //父进程
    else if(child_id > 0)
    {
        while(1)
        {
            printf("I am parent,parent id:%d\n",getpid());
            sleep(2);
        }
    }
    return 0;
}  
```
2. 验证：

   ```
   gcc server.c -o server
   ./server
   # 可以看到对应的子进程的状态
   ps -au | grep server
   ```

   

##### 2.销毁僵尸进程

1. 利用wait函数：该函数将会阻塞，直到有子进程终止。有多少个子进程，就需要调用多少次wait函数。
```
#include <sys/wait.h>
pid_t wait(int* statloc);
函数返回值：
    成功时返回终止的子进程id
    失败返回-1
函数参数：
    statloc:调用此函数时如果已有子进程终止，
    那么子进程终止时传递的返回值（比如exit函数的返回值，return语句
    的返回值）将会保存到该参数中。
使用宏分离来获取子进程的返回值：
    WIFEXITED：子进程正常终止时返回真
    WEXITSTATUS:返回子进程的返回值
    
// 示例
pid_t child_id = fork();
// 子进程
if (child_id == 0)
{
    exit(100);
}
else
{
    int status;
    pid_t id = wait(&status);
    if (WIFEXITED(status))
    {
        // child process id:348059,ret value:100
        printf("child process id:%d,ret value:%d\n", id, WEXITSTATUS(status));
    }
    sleep(30);
}
```
2. 使用waitpid函数：该函数不会阻塞，推荐使用。
```
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int* statloc, int options);
函数返回值：
    成功时返回终止的子进程id
    失败返回-1
函数参数：
    pid:等待终止的目标子进程的id,如果传递-1，
    则与wait函数相同，可以等待任意子进程终止
    statloc:与上诉wait函数相同
    options：传递常量WNOHANG，表示即使没有终
    止的子进程也不会阻塞，而是返回０退出函数
    
示例：
#include <cstdio>
#include <sys/wait.h>   //for waitpid()
#include <unistd.h>     //for fork()
#include <cstdlib>

int main()
{
    pid_t child_id = fork();
    // child process
    if (child_id == 0)
    {
        sleep(15);
        exit(100);
    }
    else
    {
        int status;
        // waitpid函数：没有子进程终止，也不会阻塞，而是返回0
        while (!waitpid(child_id, &status, WNOHANG))
        {
            sleep(3);
            puts("sleep 3 seconds!");
        }
        if (WIFEXITED(status))
        {
            printf("child process ret value:%d\n", WEXITSTATUS(status));
        }
    }
    return 0;
}
```
![image.png](https://upload-images.jianshu.io/upload_images/17728742-bebf390a0f07eca9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 使用信号处理机制：在子进程终止时，OS会向父进程发送SIGCHLD信号。因此在父进程中注册SIGCHILD信号的处理函数，在该处理函数中进行子进程资源的回收。
## 4.信号处理（signal handling）
##### 1、信号与signal函数
1. 信号：在特定事件发生时由操作系统向进程发送的消息。执行(由操作系统执行)与消息相关的自定义操作的过程称为信号处理。
2. 信号注册函数
```
#include <signal.h>
// 为了再产生信号时调用，该函数返回注册的函数指针
// 该函数表示：发生第一个参数代表的情况时，调用第二个参数所指的函数
void (*signal(int signo, void (*func)(int)))(int);
    函数名signal
    参数：signo, void(*func)(int)
        第一个参数表示特殊情况信息（信号信息）
        第二个参数表示特殊情况下将要调用的函数的地址值
    返回类型：参数为int,返回void型函数指针
    
可以在signal函数中注册的部分特殊情况和对应的常数如下：
1. SIGALRM：已经到达通过调用alarm函数注册的时间
2. SIGINT：输入CTRL + C
３.SIGCHLD：子进程终止时
```
3. alarm函数
```
#include <unistd.h>
unsigned int alarm(unsigned int seconds);
函数返回值：
    返回0或者以秒为单位的距离SIGALRM信号发生所剩时间
函数参数：
    传递正整数参数，表示相应时间后将产生SIGALRM信号
    0：表示之前对SIGALRM信号的预约取消
```
4. 示例
```
#include <cstdio>
#include <signal.h>
#include <unistd.h>

// 定义信号处理函数
void timeout(int sig)
{
    if (sig == SIGALRM)
    {
        puts("timeout!");
    }
    // 每隔两秒产生SIGALRM信号
    alarm(2);
}
void keycontrol(int sig)
{
    if (sig == SIGINT)
    {
        puts("CTRL + C pressed!");
    }
}
int main()
{
    // 注册SIGINT和SIGALRM这两个信号的信号处理函数
    signal(SIGALRM, timeout);
    signal(SIGINT, keycontrol);
    // 两秒后产生SIGALRM信号
    alarm(2);
    for (int i = 0; i < 3; i++)
    {
        puts("wait......");
        sleep(100);
    }
    return 0;
}
```
对于上述程序的注意事项：发生信号时，将会唤醒由于调用sleep函数而进入阻塞状态的进程，此时不管进程有没有睡够都将继续执行。
##### 2.利用sigaction函数进行信号处理
1. sigaction函数
```
#include <signal.h>
// Get and/or set the action for signal SIG.
int sigaction (int __sig, const struct sigaction *__restrict __act,
		      struct sigaction *__restrict __oact)
函数返回值：
    成功时返回0，失败时返回-1
函数参数：
    __sig:传递信号信息
    __act:对应于第一个参数的信号处理函数
    __oact:通过此参数获取之前注册的信号处理函数指针，若不需要则传递0
```
2. sigaction结构体如下：
```
struct sigaction
{
    //保存信号处理函数的指针值
    void (*sa_handler)(int);    
    //Additional set of signals to be blocked.
    sigset_t sa_mask;   
    // Special flags
    int sa_flags;
     /* Restore handler.  */
    void (*sa_restorer) (void);
    ...
}
```
3. 示例：使用sigaction函数对4.4程序改进如下
```
#include <cstdio>
#include <signal.h>
#include <unistd.h>

// 定义信号处理函数
void timeout(int sig)
{
    if (sig == SIGALRM)
    {
        puts("timeout!");
    }
    // 每隔两秒产生SIGALRM信号
    alarm(2);
}
int main()
{
    struct sigaction sig;
    sig.sa_handler = timeout;
    sig.sa_flags = 0;
    sigemptyset(&sig.sa_mask);
    sigaction(SIGALRM, &sig, nullptr);
    // 两秒后产生SIGALRM信号
    alarm(2);
    for (int i = 0; i < 3; i++)
    {
        puts("wait......");
        sleep(100);
    }
    
    return 0;
}
```
4. 示例：利用信号处理技术消灭僵尸进程
```
#include <cstdio>
#include <sys/wait.h>
#include <signal.h>
#include <unistd.h>
#include <cstdlib>

// 注册信号SIGCHLD的信号处理函数
void destroy_zom(int sig)
{
    int status;
    pid_t id = waitpid(-1, &status, WNOHANG);
    if (WIFEXITED(status))
    {
        printf("child process:%d,ret value:%d\n", id, WEXITSTATUS(status));
    }
}
int main()
{
    // 注册信号
    struct sigaction sig;
    sig.sa_flags = 0;
    sigemptyset(&sig.sa_mask);
    sig.sa_handler = destroy_zom;
    sigaction(SIGCHLD, &sig, nullptr);

    pid_t child_id = fork();
    if (child_id == 0)
    {
        sleep(3);
        printf("child process:%d,ready to dead!", child_id);
        exit(100);
    }
    else
    {
        for (int i = 0; i < 3; i++)
        {
            puts("wait...");
            sleep(10);
        }
    }
    return 0;
}
```
## 5.基于多任务的并发服务器端
##### 1.基于进程的并发服务器端
1. 示例：实现并发的回声服务器端
```
#include <cstdio>
#include <sys/socket.h> // for socket()
#include <unistd.h>
#include <arpa/inet.h>  // for struct sockaddr_in...
#include <cstdlib>
#include <cstring>
#include <signal.h>
#include <wait.h>   // for waitpid()

// 回收子进程资源
void distroy(int sig)
{
    int status;
    pid_t id = waitpid(-1, &status, WNOHANG);
    printf("remove child process:%d\n", id);
}
int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <port>", argv[0]);
        return -1;
    }
    // 创建服务器端套接字
    int sever_sock = socket(PF_INET, SOCK_STREAM, 0);
    
    // 绑定地址信息
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(atoi(argv[1]));
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(sever_sock, (const sockaddr*)&serv_addr, sizeof(serv_addr));

    // 侦听客户端连接
    listen(sever_sock, 5);

    struct sockaddr_in cli_addr;
    socklen_t cli_len = sizeof(cli_addr);
    char buff[1024];

    // 注册信号以及信号处理函数
    struct sigaction sig;
    sig.sa_flags = 0;
    sigemptyset(&sig.sa_mask);
    sig.sa_handler = distroy;
    sigaction(SIGCHLD, &sig, nullptr);

    while (1)
    {
        int cli_sock = accept(sever_sock, (struct sockaddr*)&cli_addr, &cli_len);
        if (cli_sock == -1)
            continue;
        else
            puts("new client coming...");
        // 创建子进程
        pid_t child_id = fork();
        // 创建子进程出错
        if (child_id == -1)  
        {
            close(cli_sock);
            continue;
        } 
        // 子进程运行区域
        else if (child_id == 0)
        {
            // fork()执行后，子进程也会将服务器端套接字和与客户端连接
            // 的套接字复制
            close(sever_sock);
            ssize_t read_len;
            while ((read_len = read(cli_sock, buff, sizeof(buff))) != 0)
            {
                write(cli_sock, buff, read_len);
            }
            close(cli_sock);
            puts("client disconnect...");
            return 0;
        }
        // 父进程
        else
        {
            close(cli_sock);
        }
    }
    close(sever_sock);
    return 0;
}
```
2. 客户端如下：
```
#include <cstdio>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <cstring>
#include <cstdlib>
#include <unistd.h>

int main(int argc, char* argv[])
{
    if (argc != 3)
    {
        printf("usage:%s <IP> <PORT>", argv[0]);
        return -1;
    }
    // 创建客户端套接字
    int cli_sock = socket(PF_INET, SOCK_STREAM, 0);

    // 初始化服务器端地址信息
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(atoi(argv[2]));
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    int ret = connect(cli_sock, (const sockaddr*)&serv_addr, sizeof(serv_addr));    
    if (ret == 0)
        puts("connected...");
    
    char message[1024];
    while (1)
    {
        fputs("Input message(q\\Q to quit):", stdout);
        fgets(message, sizeof(message), stdin);
        if (!strcmp(message, "q\n") || !strcmp(message, "Q\n"))
            break;
        write(cli_sock, message, strlen(message));
        ssize_t read_len = read(cli_sock, message, sizeof(message) - 1);
        message[read_len] = '\0';
        printf("message from server:%s", message);
    }
    close(cli_sock);
    return 0;
}
```
##### 2.分割I/O程序
1. 分割数据的发送和接收过程，主要通过多进程实现：一个父进程用于从套接字的输入缓冲区读数据，另一个子进程将数据写入套接字的输出缓冲区。分割I/O程序有利于提高同一时间内传输的数据量。理解：默认的分割模型如下图所示：
  ![image.png](https://upload-images.jianshu.io/upload_images/17728742-84092d0061a31e3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 不同进程负责输入和输出，分割IO的回升客户端如下

   ```c
   // 分割I/O的示例
   #include <cstdio>
   #include <sys/socket.h>
   #include <arpa/inet.h>
   #include <cstring>
   #include <cstdlib>
   #include <unistd.h>
   
   #define BUFFSIZE 1024
   void read_routine(int sockfd, char* buff) {
       while (1)
       {
           ssize_t read_len = read(sockfd, buff, BUFFSIZE);
           if (read_len == 0) {
               break;
           }
           buff[read_len] = '\0';
           printf("message from server:%s", buff);
       }
   }
   
   void write_routine(int sockfd, char* buff) {
       while (1) {
           // fputs("Input message(q\\Q to quit):", stdout);
           fgets(buff, BUFFSIZE, stdin);
           if (!strcmp(buff, "q\n") || !strcmp(buff, "Q\n")) {
               shutdown(sockfd, SHUT_WR);
               break;
           }
           
           write(sockfd, buff, strlen(buff));
       } 
   }
   int main(int argc, char* argv[])
   {
       if (argc != 3)
       {
           printf("usage:%s <IP> <PORT>", argv[0]);
           return -1;
       }
       // 创建客户端套接字
       int cli_sock = socket(PF_INET, SOCK_STREAM, 0);
   
       // 初始化服务器端地址信息
       struct sockaddr_in serv_addr;
       memset(&serv_addr, 0, sizeof(serv_addr));
       serv_addr.sin_family = AF_INET;
       serv_addr.sin_port = htons(atoi(argv[2]));
       serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
   
       int ret = connect(cli_sock, (const sockaddr*)&serv_addr, sizeof(serv_addr));    
       if (ret == 0)
           puts("connected...");
       
       char message[BUFFSIZE];
   
       pid_t pid = fork();
       if (pid > 0) {
           // 父进程读
           read_routine(cli_sock, message);
       } else if (pid == 0) {
           // 子进程写
           write_routine(cli_sock, message);
       }
       
       close(cli_sock);
       return 0;
   }
   ```

   