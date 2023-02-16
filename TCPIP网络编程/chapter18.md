本节要点：
1. 熟悉线程和进程的优缺点
2. 熟悉线程的创建和执行
3. 熟悉线程的同步技术
# 多线程服务器端的实现
## 1.引入线程的背景
##### 1.多进程模型的缺点
1. 创建进程的过程带来一定的开销
2. 为了完成进程间数据交换，需要特殊的IPC技术。
3. 进程间的上下文切换开销大
##### 2.线程相比于进程的优点
1. 线程间交换数据无需特殊技术
2. 线程的创建和上下文切换比进程的快：上下文切换不需要切换数据区和堆。
![image.png](https://upload-images.jianshu.io/upload_images/17728742-1200430be7352fc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/17728742-625edbd4a09d2597.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.线程的创建和运行
##### 1.线程的创建
1. pthread_create()函数
```
#include <pthread.h>
int pthread_create (pthread_t *__restrict __newthread,
			   const pthread_attr_t *__restrict __attr,
			   void *(*__start_routine) (void *),
			   void *__restrict __arg)
函数返回值：
    成功时返回0，失败时返回其他值
函数参数：
    __newthread:保存新创建线程ID的变量地址值
    __attr:用于传递线程属性的参数，传递NULL时，创建默认属性的线程
    start_routine:在单独执行流中执行的函数地址值
    arg:通过第三个参数传递调用函数时包含传递参数信息的变量地址值
```
2. 该函数使用示例
```
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

void* startrun(void* arg)
{
    int cnt = *((int*)arg);
    for (int i = 0; i < cnt; i++)
    {
        sleep(1);
        printf("HelloWorld!\n");
    }
}
int main()
{
    pthread_t thread;
    int arg = 5;
    pthread_create(&thread, NULL, startrun, (void*)&arg);
    // 保证创建的线程执行完毕
    sleep(10);
    printf("main ends!");
    return 0;
}
```
==note:线程相关代码在编译时需要添加`-lpthread`选项==
##### 2.控制线程的执行流
1. pthread_join函数：调用该函数的进程或者线程将进入等待状态，直到第一个参数为id的线程终止为止。
```
#include <pthread.h>
int pthread_join(pthread_t thread, void** status);
函数返回值：
    成功时返回0，失败时返回其他值
函数参数：
    thread:该参数值id的线程终止后才会从该函数返回
    status：保存线程的main函数返回值的指针变量地址值
    （可以获取线程的返回值）
```
2. 该函数使用示例
```
#include <stdio.h>
#include <pthread.h>

void* startroutine(void* arg)
{
    int cnt = *((int *)arg);
    for (int i = 0; i < 5; i++)
    {
        printf("HelloWorld!\n");
    }
    return (void*)"success";
}
int main()
{
    pthread_t t_id;
    int arg = 5;
    pthread_create(&t_id, NULL, startroutine, (void*)&arg);
    char** ret;
    // ret获取线程的返回值，等待t_id线程执行完毕
    pthread_join(t_id, (void**)ret);
    printf("thread return value:%s\n", *ret);
    return 0;
}
```
##### 3.函数的分类
1. 临界区：访问共享资源的一句或者一段代码
2. 根据临界区是否存在问题，函数可以分为两类：
```
1.线程安全函数（线程安全函数的名称后缀通常为_r）
2.非线程安全函数
```
3. 如何使用线程安全的函数：不用程序员手动调用线程安全的函数
```
方法1：在头文件前定义_REENTRANT宏
方法2：编译时通过添加-D_REENTRANT选项来定义宏
```
## 3.线程同步
同步：访问共享内存中的数据的时候，多个线程具有先后的顺序。线程同步用于解决线程访问顺序引发的问题。后面主要介绍两种同步技术：互斥量（mutex）和信号量(semaphore).同步技术还有条件变量，读写锁。互斥量是mutual exclusion的简写。
##### 1.互斥量的创建和销毁函数
互斥量就是一把锁。
1. 互斥量的创建及销毁函数

   ​	1. 函数原型如下：
    ```
    #include <pthread.h>
    int pthread_mutex_init(pthread_mutex_t *mutex,
    const pthread_mutexattr_t* attr);
    int pthread_mutex_destroy(pthread_mutex_t *mutex);
    函数返回值：成功时返回0，失败返回其他值
    函数参数：
        mutex:创建互斥量时传递保存互斥量的变量地址值，销毁时传递需要销毁的互斥量地址值
        attr:传递即将创建的互斥量属性，没有特别需要指定的属性时传递NULL
    ```

    2. 互斥量的声明同样可以使用如下宏：

       ```c
       pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
       ```

       

##### 2.互斥量锁住或者释放临界区时使用的函数

```
#include <pthread.h>
int pthread_mutex_lock(pthread_mutex_t* mutex);
int pthread_mutex_unlock(pthread_mutex_t* mutex)
返回值：
    成功时返回0，失败时返回其他值
```
##### 3.示例

```c
#include <stdio.h>
#include <pthread.h>
// 互斥锁
pthread_mutex_t mutex;
// 共享变量
int sum = 0;

void* increase(void* arg) {
    for (int i = 0; i < 100; ++i) {
        pthread_mutex_lock(&mutex);
        sum++;
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}

void* decrease(void* arg) {
    for (int i = 0; i < 100; ++i) {
        pthread_mutex_lock(&mutex);
        sum--;
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}
// 四个线程，2个线程增加100次，两个线程减少100次
int main() {
    pthread_mutex_init(&mutex, NULL);
    pthread_t tids[4];
    for (int i = 0; i < 4; ++i) {
        if (i / 2 == 0) {
            pthread_create(&tids[i], NULL, increase, NULL);
        } else {
            pthread_create(&tids[i], NULL, decrease, NULL);
        }
    }
    for (int i = 0; i < 4; ++i) {
        pthread_join(tids[i], NULL);
    }

    printf("sum:%d\n", sum); // 0
    pthread_mutex_destroy(&mutex);
    return 0;
}
```



## 4.信号量
##### 1.信号量创建及销毁方法
```
#include <semaphore.h>
int sem_init(sem_t * sem, int pshared, unsigned int value);
int sem_destroy(sem_t * sem);
函数返回值：
    成功时返回0，失败时返回其他值
函数参数：
    sem:创建信号量时传递保存信号量的变量地址值
    销毁时传递需要销毁的信号量变量地址值
    pshared:传递0时，创建只允许一个进程内部使用的信号量，传递其他
    值时，创建可由多个进程共享的信号量。我们需要完成同一进程内的线程同步
    所以传递0.
    value：指定新创建的信号量初始值
```
##### 2.信号量中相当于互斥量lock，unlock的函数
1. 函数声明
```
#include <semaphore.h>
int sem_post(sem_t * sem);	// 相当于unlock
int sem_wait(sem_t * sem); // 相当于lock
函数返回值：
    成功时返回0，失败时返回其他值
函数参数：
    sem:传递保存信号量读取值的变量地址值，传递给sem_post时信号量
    增1，传递给sem_wait时信号量减一。
```
2. 同步临界区的形式
```
// 假设信号量的初始值为1
sem_wait(&sem); // 信号量变为0。由于信号量不能小于0，所以在信号量为0时调用这个函数，线程将阻塞。
// 临界区的开始
....
// 临界区的结束
sem_post(&sem); // 信号量变为1
```
==上述代码结构中，调用sem_wait函数进入临界区的线程在调用sem_post函数之前不允许其他线程进入临界区，信号量的值在0和1之间跳转，具有这种特性机制的信号量称为二进制信号量。==

##### 3.示例
线程A从用户输入得到其值后存入全局变量num,此时线程B将取走该值并累加，该过程共进行5次，完成后输出总和并退出程序。
```
#include <stdio.h>
#include <semaphore.h>
#include <pthread.h>

int num = 0;
sem_t one;
sem_t two;

void* read(void* arg)
{
    for (int i = 0; i < 5; i++)
    {
        
        sem_wait(&one);
        fputs("input a number:", stdout);
        scanf("%d", &num);
        sem_post(&two);
    }
}
void* accu(void* arg)
{
    int sum = 0;
    for (int i = 0; i < 5; i++)
    {
        sem_wait(&two);
        sum += num;
        fputs("accumulate end!\n", stdout);
        sem_post(&one);
    }
    printf("sum:%d\n", sum);
}
int main()
{
    sem_init(&one, 0, 1);
    sem_init(&two, 0, 0);

    // 创建用于读和累加的线程
    pthread_t t1, t2;
    pthread_create(&t1, NULL, read, NULL);
    pthread_create(&t2, NULL, accu, NULL);
    
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    // 销毁信号量
    sem_destroy(&one);
    sem_destroy(&two);

    return 0;
}
```
![image.png](https://upload-images.jianshu.io/upload_images/17728742-7051504b9709aa2a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 5.线程的销毁
1. 调用pthread_join()函数：调用该函数时，不仅会等待线程终止，还会引导线程销毁。==但是该函数的问题是线程终止前，调用该函数的线程将进入阻塞状态，这样的话调用这个函数的线程不能继续向下执行。==
2. 调用pthread_detach()函数:调用该函数后，不会引起线程终止或者进入阻塞状态，可以通过该函数引导销毁线程创建的内存空间。
```
#include <pthread.h>
int pthread_detach(pthread_t thread);
函数返回值：
    成功时返回0，失败时返回其他值
函数参数：
    thread：终止的同时需要销毁的线程id
```
## 6.多线程并发服务器端的实现
##### 1.示例：多个客户端之间可以交换信息的程序
1. 服务端如下：
```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>
#include <sys/socket.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include <unistd.h>

#define BUFF_SIZE 1024
#define MAX_CLIENT 256

// 客户端的数量
int clnt = 0; 
// 保存客户端套接字的数组
int clnt_sock_arr[MAX_CLIENT];
// 保存互斥量的变量
pthread_mutex_t mutex;

void send_msg(char* msg, ssize_t msg_len);
// 线程执行函数
void* handle_clnt(void* arg)
{
    ssize_t read_len;
    char msg[BUFF_SIZE];
    int clnt_sock = (*(int*)arg);
    while ((read_len = read(clnt_sock, msg, BUFF_SIZE)) != 0)
    {
        send_msg(msg, read_len);
    }
    // 客户端断开连接
    pthread_mutex_lock(&mutex);
    for (int i = 0; i < clnt; i++)
    {
        // clnt_addr数组中删除clnt_sock
        if (clnt_sock == clnt_sock_arr[i])
        {
            while (i < clnt - 1)
            {
                clnt_sock_arr[i] = clnt_sock_arr[i+1];
                i++;
            }
            break;
        }

    }
    clnt--;
    pthread_mutex_unlock(&mutex);
    close(clnt_sock);
    return NULL;
}

void send_msg(char* msg, ssize_t msg_len)
{
    // 向所有客户端套接字发送消息
    for (int i = 0; i < clnt; i++)
    {   
        int count = 0;
        while (count < msg_len) {
            int write_len = write(clnt_sock_arr[i], msg, msg_len);
            count += write_len;
        }
        
    }
}
int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("usage:%s <PORT>\n", argv[0]);
        return -1;
    }

    int serv_sock, clnt_sock;
    serv_sock = socket(PF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr, clnt_addr;
    socklen_t clnt_len = sizeof(clnt_addr);
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(atoi(argv[1]));

    bind(serv_sock, (const struct sockaddr*)&serv_addr, clnt_len);

    listen(serv_sock, 5);

    // 初始化互斥锁
    pthread_mutex_init(&mutex, NULL);
    pthread_t threadid;
    while (1)
    {
        clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_addr, &clnt_len);
        
        pthread_mutex_lock(&mutex);
        clnt_sock_arr[clnt] = clnt_sock;
        clnt++;
        pthread_mutex_unlock(&mutex);

        // 创建子线程和客户端通信
        pthread_create(&threadid, NULL, handle_clnt, (void*)&clnt_sock);
        printf("connected client IP:%s， 端口：%d\n", 
            inet_ntoa(clnt_addr.sin_addr),
            ntohs(clnt_addr.sin_port));
        pthread_detach(threadid);
    }

    close(serv_sock);
}
```
2. 客户端
```c
#include <stdio.h>
#include <sys/socket.h>
#include <string.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include <pthread.h>
#include <unistd.h>

#define MSGSIZE 1024
#define NAMESIZE 20
char name[NAMESIZE] = "[default]";
char msg[MSGSIZE]; 

// 发送消息的线程
void* send_msg(void* arg)
{
    int clnt_sock = (*(int*)arg);
    char name_msg[BUFSIZ + NAMESIZE];
    while (1)
    {
        fgets(msg, sizeof(name_msg), stdin);
        if (!strcmp(msg, "q\n") || !strcmp(msg, "Q\n"))
        {
            close(clnt_sock);
            exit(0);
        }
        sprintf(name_msg, "%s %s", name, msg);
        write(clnt_sock, name_msg, strlen(name_msg));
    }
    return NULL;
}
// 接收消息的线程
void* receive_msg(void* arg)
{
    int clnt_sock = (*(int*)arg);
    char name_msg[BUFSIZ + NAMESIZE];
    ssize_t read_len;
    while (1)
    {
        if ((read_len = read(clnt_sock, name_msg, sizeof(name_msg) - 1)) == 0)
            return (void*)-1;
        name_msg[read_len] = 0;
        fputs(name_msg, stdout);
    }
    return NULL;
}
int main(int argc, char* argv[])
{
    if (argc != 4)
    {
        printf("usage:%s <IP> <PORT> <USERNAME>", argv[0]);
        return -1;
    }

    int clnt_sock = socket(PF_INET, SOCK_STREAM, 0);

    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_addr.s_addr = inet_addr(argv[1]);
    serv_addr.sin_port = htons(atoi(argv[2]));
    serv_addr.sin_family = AF_INET;

    connect(clnt_sock, (const struct sockaddr*)&serv_addr, sizeof(serv_addr));
    sprintf(name, "[%s]", argv[3]);
    
    pthread_t snd_id, rev_id;
    // 读写分离
    pthread_create(&snd_id, NULL, send_msg, &clnt_sock);
    pthread_create(&rev_id, NULL, receive_msg, &clnt_sock);
    pthread_join(snd_id, NULL);
    pthread_join(rev_id, NULL);

    close(clnt_sock);   
    return 0;
}
```