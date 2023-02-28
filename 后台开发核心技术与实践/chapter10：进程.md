## 1.进程的创建与结束
##### 1.进程的创建
1. 进程的创建有两种方式：
    1. 由操作系统创建：在系统启动时，操作系统会创建一些进程，他们承担着管理和分配系统资源的任务，这些进程通常被称为系统进程。
    2. 由父进程创建
2. 进程的创建：使用fork函数
    1. 创建一个子进程示例：
    ```
    #include <stdio.h>
    #include <unistd.h>
    int main() {
        int pid = fork();
        if (pid == -1) {
            printf("fork()error\n");
            return -1;
        } else if (pid == 0) {
            // 子进程
            printf("父进程id:%d,子进程id:%d\n", getppid(), getpid());
        } else {
            // 父进程
            printf("父进程id:%d,子进程id:%d\n", getpid(), pid);
        }
        sleep(10);
        return 0;
    }
    ```
3. 父进程和子进程共享的资源：除了代码段共享外，其他不共享
![image.png](https://upload-images.jianshu.io/upload_images/17728742-dd8662eb3d6ede47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 2.进程的结束
1. 进程的结束：在Linux中进程退出分为正常退出和异常退出。
    1. 正常退出：
        1. 在main函数中执行return
        2. 调用exit函数
        3. 调用_exit函数
    2. 异常退出：
        1. 调用abort函数
        2. 进程收到某种信号，该信号使得程序终止
    3. 区别：exit函数是在_exit函数之上的一个封装，其会自动调用_exit，并在调用之前先刷新流数据。
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-49ba4b58fe998258.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    4. 示例：exit函数和_exit函数的区别：
    ```
    #include <stdio.h>
    #include <stdlib.h>
    // 程序输出
    // test
    // exit()
    int main() {
        printf("test\n");
        printf("exit()");
        exit(1);
    }
    
    #include <stdio.h>
    #include <unistd.h>
    // 程序输出
    // test （ 因为printf函数内部的内存缓冲区在遇到特殊字符，这里是换行符
    // 会将缓冲区的内容刷新）
    int main() {
        printf("test\n");
        printf("exit()");
        _exit(1);
    }
    ```
## 2.僵尸进程和孤儿进程
##### 1.孤儿进程
1. 孤儿进程：指一个父进程退出后，而他的一个或者多个子进程还在运行，那么那些子进程将称为孤儿进程。孤儿进程将由进程号为1的init进程收养，并由init进程对他们完成状态收集工作。
##### 2.僵尸进程
1. 僵尸进程：一个进程使用fork创建子进程，如果子进程退出，而父进程没有调用wait或者waitpid获取子进程的状态信息，那么子进程的进程描述符PCB仍然保留在系统中。这样的进程称之为僵尸进程。==当父进程退出后，僵尸进程成为孤儿进程，过继给了init进程，而init进程会周期性的调用wait来清除各个僵尸的子进程。==
2. 僵尸进程的一个简单示例
    1. 如下所示，子进程退出，而父进程尚在运行，却没有回收子进程的资源。此时子进程就是一个僵尸进程
    ```
    #include <stdio.h>
    #include <unistd.h>
    int main() {
        int pid = fork();
        if (pid > 0) {
            printf("父进程中，进程id为%d,子进程id为%d\n", getpid(), pid);
            sleep(10);
        } else if (pid == 0) {
            printf("子进程中，父进程id为:%d,进程id为:%d\n", getppid(), getpid());
            return -1;
        }
        return 0;
    }
    ```
    2. 查看僵尸进程：运行上述程序后，随后执行`ps -au | grep -w 'Z'`查看
3. 使用wait回收子进程：wait函数会阻塞
```
// 使用wait回收子进程,并获取子进程的退出状态值
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>

int main() {
    int pid = fork();
    if (pid == 0) {
        printf("子进程id：%d\n", getpid());
        sleep(3);
        exit(100);
    } else if (pid > 0) {
        int status;
        int ret = wait(&status);
        if (WIFEXITED(status)) {
            printf("回收了子进程%d的资源，子进程退出状态为:%d\n", ret,
                WEXITSTATUS(status));
        }
     
    }

    return 0;
}
```
4. 使用waitpid回收子进程的资源
```
#include <stdio.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdlib.h>

int main() {
    int pid = fork();
    if (pid == 0) {
        printf("子进程id：%d\n", getpid());
        sleep(10);
        exit(100);
    } else if (pid > 0) {
        int ret;
        do {
            ret = waitpid(pid, NULL, WNOHANG);
            if (ret == 0) {
                printf("暂无子进程退出\n");
                sleep(1);
            }
        } while (ret == 0);
        if (ret == pid) printf("子进程%d退出\n", ret);
    }
    return 0;
}
```
## 3.守护进程
1. 守护进程：脱离于终端并且在后台运行的进程。他是一个生存期较长的进程，通常独立于控制终端并且周期性的执行某种任务或者等待处理某些发生的事件。守护进程通常在系统引导装入时启动，在系统关闭时终止。
2. 创建一个简单的守护进程的步骤：
    1. 创建子进程，父进程退出：这样子进程变为孤儿进程，所有工作在子进程中完成。
    2. 在子进程中创建新会话：
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-c252a0e70e51284e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
        1. 在守护进程中调用setsid函数的原因：由于父进程在调用fork函数时，子进程全盘拷贝了父进程的会话期、进程组、控制终端等。此时将作为守护进程的子进程并不是真正意义上的独立。
    3. 改变当前工作目录为根目录：使用fork创建的子进程默认继承了父进程的当前工作目录
    4. 重新设置文件权限掩码：使用fork创建的子进程默认继承了父进程的文件权限掩码，这给子进程使用文件带来了麻烦。因此将文件权限掩码设置为0，可以大大提高守护进程的灵活性。
    5. 关闭文件描述符：使用fork创建的子进程默认继承了父进程中已经打开了的文件。而在守护进程中，文件描述符为0，1，2的三个文件不需要，所以应该关闭他们。
3. 示例：创建一个守护进程，该守护进程每隔10秒钟向文件写入数据
```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#define MAX_FILE 1024
// 将子进程变为守护进程
int main() {
    int pid = fork();
    if (pid < 0) {
        perror("fork()error\n");
        exit(-1);
    } else if (pid > 0) {
        // 父进程退出，将所有工作交给子进程
        exit(1);
    } else {
        // 创建新会话
        setsid();
        // 设置文件权限掩码
        umask(0);
        for (int i = 0; i < MAX_FILE; ++i) {
            close(i);
        }
        // 守护进程的真实工作
        int fd; 
        char buff[] = "this is daemon";
        while (1) {
            if ((fd = open("/tmp/normal.log", O_CREAT | O_WRONLY | O_APPEND, 0600)) == -1) {
                perror("open()error\n");
                return -1;
            }
            write(fd, buff, strlen(buff) + 1);
            sleep(10);
            close(fd);
        }
    }
    return 0;
}
```













