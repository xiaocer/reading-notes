##### 1.多线程基本理论
1. 线程：一个进程内存在多个控制权。包含多个线程的进程内部，每个线程对应一个栈。
2. 多线程的创建：`pthread_create`
    1. 创建线程时向线程传递参数 
    ```
    #include <pthread.h>
    #include <stdio.h>
    
    void* worker(void* arg) {
        int param = *(int*)arg;
        printf("param:%d\n", param);
        pthread_exit((void*)1);
    }
    int main() {
        pthread_t tid;
        int param = 1024;
        pthread_create(&tid, NULL, worker, &param);
        void* ret;
        pthread_join(tid, &ret);
        printf("子线程返回的值：%ld\n", (long)ret);
    
        return 0;
    }
    ```
3. 线程的结束：
    1. 函数结束，则调用它的线程就结束
    2. 一般子线程通过pthread_exit函数调用，并且传递一个返回值来结束子线程的运行；主线程可以通过pthread_join获得该返回值。
    3. 示例：线程的创建与结束
    ```
    #include <pthread.h>
    #include <stdio.h>
    
    void* worker(void* arg) {
        printf("子线程id:%ld\n", pthread_self());
        static int number = 1024;
        return (void*)&number;
    }
    int main() {
        pthread_t tid;
        int ret = pthread_create(&tid, NULL, worker, NULL);
    
        void* status;
        pthread_join(tid, &status);
        printf("子线程返回的值为：%d\n", *(int*)(status));
    
        return 0;
    }
    ```
4. 获取线程id：
    1. 在线程的执行函数中调用pthread_self获取线程的id
    2. 执行pthread_create函数调用生成的id
    3. 示例：
    ```
    #include <pthread.h>
    #include <stdio.h>
    
    void* worker(void* arg) {
        printf("子线程id：%ld\n", pthread_self());
        return NULL;
    }
    int main() {
        pthread_t tid;
        pthread_create(&tid, NULL, worker, NULL);
        pthread_join(tid, NULL);
        printf("子线程id:%ld\n", tid);
        return 0;
    }
    ```
5. 线程的属性：线程属性对象的类型为pthread_attr_t
    1. 线程属性对应的结构体如下：
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-38b6e207cc0c6907.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    2. 获取和设置pthread_attr_t结构体里面的各个属性
    ```
    // 分离状态
    pthread_attr_getdetachstate
    pthread_attr_setdetachstate
    // 设置和获取线程的栈地址
    pthread_attr_getstackaddr
    pthread_attr_setstackaddr
    // 获取和设置线程的栈大小
    pthread_attr_getstacksize
    pthread_attr_setstacksize
    // 同时获取和设置栈地址和栈大小
    pthread_attr_getstack
    pthread_attr_setstack
    
    // 栈保护区大小：在线程栈顶留出一段空间，防止栈溢出。
    // 当栈指针进入这段保护区时，系统会发出错误提示，通常发送信号给线程
    
    // 获取/设置栈保护区的大小
    pthread_attr_getguardsize
    pthread_attr_setguardsize
    
    // 获取和设置线程的优先级，新线程的优先级为0
    pthread_attr_getschedparam
    pthread_attr_setschedparam
    
    // 
    ```
    3. 调度策略和继承父进程优先级
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-795d8cc2f55a388c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    4. 争用范围
      ![image.png](https://upload-images.jianshu.io/upload_images/17728742-57b180e58fd10a83.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    5. 线程并行级别
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-802d33476989b830.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 2.多线程同步
1. 多线程同步：对于多线程程序来说，同步是指在一定的时间内只允许某一个线程访问某个共享资源。而在此时间内，不允许其他的线程访问该资源。线程同步的方法主要有互斥锁、条件变量、信号量、读写锁。
2. 互斥锁：一种特殊的变量，它有锁上和打开两种状态。
    1. 互斥锁的相关函数如下：
    ```
    // 互斥锁的初始化
    pthread_mutex_init
    // 互斥锁的上锁
    pthread_mutex_lock
    // 互斥锁的解锁
    pthread_mutex_unlock
    // 互斥锁的销毁
    pthread_mutex_destroy
    // 尝试上锁，尝试失败会返回EBUSY
    pthread_mutex_trylock
    ```
    2. 示例：使用pthread_mutex_trylock
    ```
    // 模拟4个线程售卖100张票
    #include <pthread.h>
    #include <stdio.h>
    #include <unistd.h>
    #include <errno.h> // for EBUSY
    pthread_mutex_t mutex;
    int tickets = 100;
    
    void* sell_ticket(void* arg) {
        while (tickets > 0)
        {
            int ret = pthread_mutex_trylock(&mutex);
            if (ret == EBUSY) {
                printf("互斥锁已经被另一线程占用\n");
            } else if (ret == 0) {
                
                printf("子线程id：%ld,售卖出第%d张票\n", pthread_self(),
                100 - tickets + 1);
                tickets--;
                pthread_mutex_unlock(&mutex);
                sleep(1);
            }
            
        }
        
    }
    int main() {
        pthread_mutex_init(&mutex, NULL);
        pthread_t tids[4];
        for (int i = 0; i < 4; ++i) {
            pthread_create(&tids[i], NULL, sell_ticket, NULL);
        }
        sleep(20);
        for (int i = 0; i < 4; ++i) {
            pthread_join(tids[i], NULL);
        }
    
        pthread_mutex_destroy(&mutex);
        return 0;
    }
    ```
3. 条件变量：当线程在等待满足某些条件时使得线程进入睡眠状态，一旦条件满足，就唤醒因等待满足特定条件而睡眠的线程。
    1. 条件变量的相关函数：
    ```
    // 初始化条件变量 
    pthread_cond_init
    // 销毁条件变量
    pthread_cond_destroy
    // 条件等待
    pthread_cond_wait
    // 计时等待
    pthread_cond_timedwait
    // 激活一个等待某条件的线程
    pthread_cond_signal
    //  激活所有等待线程
    pthread_cond_broadcast
    
    ```
    2. 示例：旅客坐车,两辆车，1名旅客
    ```
    #include <pthread.h>
    #include <stdio.h>
    #include <unistd.h>
    pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
    int count = 0; // 记录旅客的数量
    
    void* car_arrive(void* arg) {
        printf("车辆%s来了\n", (char*)arg);
        while (1) {
            pthread_mutex_lock(&mutex);
            if (count > 0) {
                pthread_cond_signal(&cond);
                pthread_mutex_unlock(&mutex);
                break;
            }
            pthread_mutex_unlock(&mutex);
        }    
    }
    void* people_arrive(void* arg) {
        pthread_mutex_lock(&mutex);
        count++;
        pthread_cond_wait(&cond, &mutex);
        pthread_mutex_unlock(&mutex);
        printf("Mery上车了\n");
    }
    int main() {
        pthread_t tids[3];
        pthread_create(&tids[0], NULL, car_arrive, (void*)"Jack");
        printf("time wating...Jack arrive\n");
        sleep(1);
        pthread_create(&tids[1], NULL, people_arrive, NULL);
        printf("time wating...Mery arrive\n");
        sleep(1);
        pthread_create(&tids[0], NULL, car_arrive, (void*)"Lucy");
        printf("time wating...Lucy arrive\n");
        sleep(1);
    
        for (int i = 0; i < 3; ++i) {
            pthread_join(tids[i], NULL);
        }
    
        return 0;
    }
    ```
3. 读写锁：适用于读比写情况更多的场景。写比读优先级更高，防止读模式锁长期占用，而写模式的线程处于长期饥饿的状态。
    1. 读写锁相关的函数：
    ```
    // 初始化读写锁
    pthread_rwlock_init
    // 销毁读写锁
    pthread_rwlock_destroy
    // 获取读出锁(阻塞式)
    pthread_rwlock_rdlock
    // 获取写入锁(阻塞式)
    pthread_rwlock_wrlock
    // 获取读出锁(非阻塞式)
    pthread_rwlock_tryrdlock
    // 获取写入锁(非阻塞式)
    pthread_rwlock_trywrlock
    // 释放读出或者写入锁
    pthread_rwlock_unlock
    ```
4. 信号量：信号量允许多个线程同时进入临界区，而互斥锁只允许一个线程进入临界区。
    1. 信号量相关的函数如下：
    ```
    #include <semaphore.h>
    // 创建信号量
    int sem_init(sem_t *sem, int pshared, unsigned int value);
    函数参数：
        value：信号量的初始值
        pshared：
            0：表示信号量在同一个进程下共享
            非0：表示信号量在进程之间共享
    // 将信号量的值-1
    sem_wait
    // 将信号量的值+1
    sem_post
    // 清理信号量
    sem_destroy
    ```
    2. 示例：用信号量模拟两个窗口向顾客服务
    ```
    #include <pthread.h>
    #include <semaphore.h>
    #include <stdio.h>
    #include <unistd.h>
    
    sem_t sem;
    void* receive_service(void* arg) {
        if (!sem_wait(&sem)) {
            usleep(100);
            printf("顾客%ld接受服务\n", pthread_self());
            sem_post(&sem);
        }
    }
    int main() {
        // 同时只有两个窗口向顾客服务
        sem_init(&sem, 0, 2);
        pthread_t tids[10];
        for (int i = 0; i < 10; ++i) {
            pthread_create(&tids[i], NULL, receive_service, NULL);
            printf("顾客%ld到来\n", tids[i]);
            usleep(10);
        }
        
        for (int i = 0; i < 10; ++i) {
            pthread_join(tids[i], NULL);
        } 
    
        sem_destroy(&sem);
        return 0;
    }
    ```
5. 多线程可重入：定义宏_REENTRANT告诉编译器需要可重入功能，这个宏定义必须出现在程序中的任何#include语句之前。这个宏的功能：
    1. 会对部分函数重新定义他们的可安全重入版本，在函数名之后添加_r字符串