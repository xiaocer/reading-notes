1. Linux下常用的进程间通信的方法：
    1. 管道：用于同一台机器上的进程间通信
    2. 消息队列：用于同一台机器上的进程间通信
    3. 共享内存：用于同一台机器上的进程间通信
    4. 信号量：用于同一台机器上的进程间通信
    5. 套接字：不同机器之间的网络通信
## 1.管道
##### 1.无名管道
1. 因为管道传递数据的单向性，管道又被称之为半双工管道。
2. 管道的特点：
    1. 数据只能由一个进程流向另一个进程，其中一个是读管道，另一个是写管道。如果要进行双工通信，需要建立两个管道。
    2. 管道只能用于父子进程或者兄弟进程之间通信。
    3. 管道没有名字
    4. 管道的缓冲区大小受限制
3. 无名管道的创建：使用pipe函数
4. 示例：使用无名管道实现父子进程之间的通信，一个进程从管道读端读数据，另一个进程向管道写端写数据。
```
#include <stdio.h>
#include <unistd.h>
#include <assert.h>
#include <string.h>
#include <stdlib.h>

int main() {
    int fd[2];
    int ret = pipe(fd);
    assert(ret != -1);

    int pid = fork();
    if (pid < 0) {
        perror("fork()error\n");
        exit(-1);
    } else if (pid == 0) {
        // 子进程
        close(fd[0]);
        char buff[] = "hello, this is child process sending message to you!";
        write(fd[1], buff, strlen(buff));
        close(fd[1]);
    } else {
        // 父进程
        close(fd[1]);
        char buff[1024];
        int read_len = read(fd[0], buff, sizeof(buff));
        buff[read_len] = 0;
        printf("接收到子进程的消息：%s\n", buff);
        close(fd[0]);
    }
    return 0;
}
```
##### 2.有名管道
1. 有名管道又称之为FIFO。它不同于无名管道的地方在于有名管道提供一个路径名与之关联。
2. 有名管道的特点：
    1. 没有血缘关系的进程也可以通信。（只要能够访问该路径的进程就可以和通过创建FIFO所在的进程通信）
    2. 该管道通过路径名来指出，并且在文件系统中是可见的。
3. 有名管道和无名管道的区别：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-a144a8abe722a34c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
4. 有名管道的创建：通过mkfifo函数
5. 示例：两个进程使用有名管道进行通信
```
#include <stdio.h>
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#define FIFO_FILE "/tmp/fifo"

// 一个用于从有名管道非阻塞式读取数据的进程
int main() {
    // 管道已经文件存在
    if (access(FIFO_FILE, F_OK) == 0) {
        // 删除管道文件
        execlp("rm", "-f", FIFO_FILE, NULL);
    }
    // 创建有名管道
    if (mkfifo(FIFO_FILE, 0777) < 0) {
        perror("mkfifo()error\n");
    }    
    int fd = open(FIFO_FILE, O_NONBLOCK | O_RDONLY);
    char buff[1024];
    int read_len;
    while (1) {
        memset(buff, 0, sizeof(buff));
        if ((read_len = read(fd, buff, sizeof(buff) - 1)) == 0) {
            printf("no data\n");
            sleep(3);
        } else if (read_len > 0) {
            buff[read_len] = 0;
            printf("接收到数据：%s\n", buff);
        }
    }
    close(fd);
    return 0;
}

// 一个用于向有名管道写数据的进程
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#define FIFO_FILE "/tmp/fifo"

int main(int argc, char* argv[]) {
    if (argc < 2) {
        printf("no write data\n");
        return -1;        
    }
    int fd = open(FIFO_FILE, O_WRONLY | O_NONBLOCK);
    write(fd, argv[1], sizeof(argv[1]));
    close(fd);    
    return 0;
}
```
## 2.消息队列
1. 在系统内核中用来保存消息的队列，他在系统内核中是以消息链表的形式存在。消息链表中节点的结构使用msg声明。
2. 消息队列相关的函数：
    1. 创建新的消息队列或者获取新的消息队列
    ```
    int msgget(key_t key, int msgflg);
    函数参数：
        key：一个端口号
        msgflag：
            IPC_CREAT：如果没有该队列，则创建一个并返回新标识符，如果已经存在则返回原标识符
            IPC_EXCL：如果没有该队列则返回-1，如果已经存在，则返回0
    ```
    2. 向消息队列写消息：msgsnd
    ```
    int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
    函数参数：
        mqid：消息队列的标识码
        msgp：指向消息缓冲区的指针。这是一个结构体，一般是以下形式
            struct msgbuf {
               long mtype;       /* message type, must be > 0 */
               char mtext[1];    /* message data */
           };
        msgsz：消息的大小，即mtxet的大小
        msgflag：指明程序在队列里没有数据的情况下应该采取的行动
            0：表示在队列满或者空时采取阻塞等待的处理模式
    ```
    3. 从消息队列读消息：msgrcv
    ```
    ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp,
                      int msgflg);
    函数参数：
        msgtyp：指明从消息队列里读取的消息形态
            0：所有消息都会被读取
    ```
    4. 设置消息队列属性：msgctl
    ```
    int msgctl(int msqid, int cmd, struct msqid_ds *buf);
    函数参数：
        cmd：指明要对消息队列执行的操作
            IPC_STAT：获取消息队列对应的msqid_ds数据结构，并将其保存在buf中
            IPC_SET：设置消息队列的属性，要设置的属性存储在buf中
            IPC_RMID：从内核中删除msqid标识的消息队列
    ```
## 3.共享内存
1. 不同进程之间共享的内存通常安排在同一段物理内存中。一个进程向共享内存写入数据，将会影响到其他访问共享内存的进程。因此需要有一种同步机制保护共享资源。
2. 共享内存相关的函数如下：
    1. 创建共享内存：
    ```
    #include <sys/shm.h>

    int shmget(key_t key, size_t size, int shmflg);
    函数参数：
        key：为共享内存段命名
        size：确定需要共享得内存容量
        shmflg：权限标志
            IPC_CREATE：在key标识得共享内存不存在得情况下创建它
            IPC_EXCL：如果共享内存已经存在则函数调用失败
    函数返回值：
        成功则返回一个共享内存标识符，用于后续得共享内存操作
        失败则返回-1
    ```
    2. 将共享内存连接到进程得自身得地址空间中
    ```
    void *shmat(int shmid, const void *shmaddr, int shmflg);
    函数参数：
        shmid：为shmget函数返回的共享存储标识符
        addr和flag参数决定了以什么方式来确定连接的地址
    返回值：该进程连接到共享内存段得实际地址
    ```
    3. 将共享内存从当前进程中分离
    ```
    int shmdt(const void *shmaddr);
    函数参数：
        shmaddr：shmat函数返回得地址指针
    返回值：成功返回0，失败返回-1
    ```
    4. 对共享内存执行操作
    ```
    int shmctl(int shmid, int cmd, struct shmid_ds *buf);
    ```
3. 示例：两个互不相关得进程使用共享内存来进行通信
4. 共享内存得优缺点：
    1. 优点：使用共享内存进行进程间得通信非常方便，数据的共享使得进程间得数据不用传送，而是直接访问内存。
    2. 缺点：共享内存没有提供同步的机制
## 4.SYSTEM V信号量
1. 信号量得相关函数：
    1. 创建和打开信号量：semget
    2. 改变信号量得值：semop
    3. 控制信号量得信息：semctl
## 5.ipcs命令
1. ipcs命令用于报告系统得消息队列、信号量、共享内存等。
2. ipcs -a：用于列出本用户所有相关的ipcs参数
3. ipcs -q：用于列出进程中的消息队列
4. ipcs -s：用于列出所有的信号量
5. ipcs -m：用于列出所有的共享内存信息
6. ipcs -l：用于列出系统的限额
7. ipcs -t：列出最后的访问时间
![image.png](https://upload-images.jianshu.io/upload_images/17728742-1926d297836e67ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
8. ipcs -u：列出当前的使用情况
![image.png](https://upload-images.jianshu.io/upload_images/17728742-bd81431b26715123.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
