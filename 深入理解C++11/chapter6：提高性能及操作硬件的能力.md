
## 1.常量表达式
1. 运行时常量性：指的是运行期间数据的不可更改性，这是由const关键字保证的。例如：
```
const int i = 100;
```
2. 编译时期常量：cpp11以前使用宏替换可以获得编译时期的常量，==cpp11中，使用constexpr关键字声明常量表达式。这样编译器就可以在编译期对常量表达式进行值计算。==
    1. 常量表达式函数：在函数返回类型前加入关键字constexpr就可以使其成为常量表达式函数。例如常量构造函数，它也可以用于非常量表达式中的类型构造，因此程序员不用为类再重写一个非常量表达式版本。
    ```
    class Test {
    private:
        int x;
    public:
        // 常量构造函数函数体必须为空
        // 初始化列表只能由常量表达式来赋值 
        constexpr Test(int x) : x(x) {}
        int getX()const {return x;}
    };
    
    int main() {
        constexpr Test test(100);
        cout << test.getX() << endl;
        // ok
        Test t(200);
        return 0;
    }
    ```
    2. 常量表达式值：通常情况下常量表达式必须被一个常量表达式赋值。例如上例中的test变量。
3. 常量表达式的应用：
    1. 用于模板函数：当声明为常量表达式的模板函数后，而某个该模板函数的实例化结果不能满足常量表达式的需求的话，constexpr会自动忽略。该实例化后的函数将成为一个普通函数。
    ```
    struct Test {
        Test() {i = 5;}
        int i;
    };
    template <typename T>
    constexpr T func( T t) {
        return t;
    }
    int main() {
        Test  test;
        // ok
        Test t1 = func(test);
        // ok
        constexpr int a = func(100);
        // error，结构体Test不是一个定义了常量构造函数的类型
        // constexpr Test t2 = func(test);
        return 0;
    }
    ```
    2. 用于switch-case的case表达式
    3. 用于初始化数组的声明中
    4. 符合CPP11标准的编译器对常量表达式函数应该至少支持512层的递归
    ```
    
    constexpr int fibonacci(int n) {
        return (n == 1) ? 1 : ((n == 2) ? 1 : fibonacci(n - 1) + fibonacci(n - 2));
    }
    
    int main() {
        int fib[] = {fibonacci(3), fibonacci(4)};
        for (int i : fib) {
            cout << i << endl; // 2 3
        }
        return 0;
    }
    ```
## 2.变长模板
1. 变长函数：参数的数量和类型是不受限制的
```
// count表示参数的数量
double sum(int count, ...) {
    va_list ap;
    double sum = 0;
    va_start(ap, count);            // 获得变长列表的句柄ap
    for (int i = 0; i < count; i++) {
        sum += va_arg(ap, double);  // 每次获得一个参数
    }
    va_end(ap);
    return sum;
}
int main() {
    printf("%f\n", sum(3, 1.2f, 3.4, 5.6));
    return 0;
}
```
2. 变长模板：cpp11中提出
    1. 模板参数包：编译器将多个模板参数打包成单个的模板参数包。模板参数包使得模板能够接受变长的参数
        1. 简单示例
        ```
        // 如下tuple类的声明中，_Elements就是一个模板参数包
        template<typename... _Elements>
        class tuple;
        
        // 模板参数包可以是非类型的，即非类型模板参数
        template<int... A>
        class Test{};
        int main() {
            Test<1, 2, 3> test;
            tuple<int, float, double> t1 {1, 2.2f, 3.4};
            tuple<int, string> t2 {2023, "nrvcer"};
            return 0;
        }
        ```
        2. 模板参数包的解包：通过包扩展的表达式来完成。例如下例中的`A...`就是一个包扩展，将模板函数包解包的。
            1. 解包示例：
            ```
            template<typename T1, typename T2>
            class B{};
            
            template <typename... A>
            class Test: private B<A...> {};
            int main() {
                Test<int, double> test;
                // error, 类模板B只支持两个参数
                Test<int, double, char> test1;
                return 0;
            }
            ```
            2. 更过有趣的解包示例
            ```
            1. 声明Arg为参数包 使用Arg&&... 这样的包扩展表达式
            等价于Arg1&&, ... Argn&&
            2. template <typename... A>
            class Test: private B<A>... {};
            表示Test类多继承自多个B类。
            ```
        3. 可以使用模板作为变长模板的参数包
        ```
        template <typename A, typename B> 
        struct Test {};
        // 包含两个模板参数包，两个变长模板
        template<template<typename...>class T, typename... Targs
        ,template<typename...>class U, typename... Uargs>
        struct Test<T<Targs...>, U<Uargs...>> {};
        
        int main() {
            Test<int, float> t1;
            Test<tuple<int, char>, tuple<float>> t2;
            return 0;
        }
        ```
    2. 函数参数包：变长的函数参数也可以声明成函数包。cpp11中要求函数参数包必须唯一，而且是函数的最后一个参数。例如：
    ```
    template<typename... T>
    void func(T... args);
    ```
3. 参数包可以展开的位置：标准定义了7种参数包可以展开的位置
    1. 表达式
    2. 初始化列表
    3. 基类描述列表
    4. 类成员初始化列表
    5. 模板参数列表
    6. 通用属性列表
    7. lambda函数的捕捉列表
4. sizeof...操作符：用于计算参数包中的个数
```
template<class... A>
void func(A... arg) {
    int size = sizeof...(A); // 计算参数包中的个数
    cout << "可变参数的格个数：" << size << endl;
}

// 特化2个参数的版本呢
void fun(int a, int b) {
    cout << a << "," << b << endl;
}

int main() {
    fun(100, 200);
    func(1, 2, 3, 4);
    return 0;
}
```

## 3.原子类型与原子操作
在CPP11之前，程序中使用线程主要使用POSIX线程和OpenMP编译器指令两种编程模型来完成程序的线程化。在CPP11中引入了多线程的支持，使得CPP在进行线程编程时，不必依赖第三方库和标准。
##### 1.原子操作与CPP11中的原子类型
1. 内置类型的原子类型定义，定义在头文件cstdatomic中
![image.png](https://upload-images.jianshu.io/upload_images/17728742-d1dbce97ccaff558.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 除了使用上诉的内置类型的原子类型定义原子类型外，还可以使用atomic类模板。
```
atomic<T> t;
```
3. automic类型及其相关的操作如下：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-d9e06932dbba0268.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. 布尔类型的atomic类型atomic_flag：他是无锁的，线程对其访问不需要加锁。通过它的成员test_and_set以及clear可以实现自旋锁。
    ```
    #include <iostream>
    #include <atomic>
    #include <unistd.h>
    #include <thread>
    
    using namespace std;
    
    atomic_flag lock = ATOMIC_FLAG_INIT;
    void f(int n) {
        while (lock.test_and_set(memory_order_acquire))
            // 自旋， 直到t2线程将lock的值设置为false
            cout << "waiting from ather thread:" << n << endl;
        cout << "Thread:" << n << "start working" << endl;
    }
    
    void g(int n) {
        cout << "Thread:" << n << "is going to start" << endl;
        // 将lock的值设置为false
        lock.clear();
        cout << "Thread:" << n << "start working" << endl;
    }
    int main() {
        lock.test_and_set();
        thread t1(f, 1);
        thread t2(g, 2);
        t1.join();
        usleep(100);
        t2.join();
        return 0;
    }
    ```
##### 2.内存模型以及顺序一致性
1. 在默认情况下，在CPP11中的原子类型的变量在线程中总是保持着顺序执行的特性，这样的特性称为顺序一致性。（非原子类型则没有必要，因为不需要在线程间进行同步）==顺序一致性属于CPP11中多种内存模型的一种。==
    1. 简单示例
    ```
    // 线程t1中，因为原子类型变量的顺序一致性，a的赋值语句一定发生在b的赋值语句之前
    #include <thread>
    #include <atomic>
    #include <iostream>
    using namespace std;
    
    atomic<int> a = 0;
    atomic<int> b = 0;
    
    int func1(int) {
        int t = 1;
        a = t;
        b = 2;
    }
    int func2(int) {
        while (b != 2)
            ;
        cout << a << endl; // 1
    }
    int main() {
        thread t1(func1, 0);
        thread t2(func2, 0);
        t1.join();
        t2.join();
        return 0;
    }
    ```
    2. 对于CPP11中的内存模型，要保证代码的顺序一致性，就必须同时做到以下几点：
        1. 编译器保证原子操作的指令间顺序不变，即保证产生的读写原子类型的变量的机器指令与代码编写者看到的是一致的
        2. 处理器对原子操作的汇编指令的执行顺序不变。这对于X86这样的强顺序的体系结构而言，没有任何的问题。而对于PowerPC这样的弱顺序的体系结构而言，要求编译器在每次原子操作后加入内存栅栏（内存屏障）。
    3. 强顺序和弱顺序的概念：强顺序即处理器对原子操作的汇编指令的执行顺序不变，弱顺序即处理器对原子操作的汇编指令的执行顺序可能打乱（指令重排序），而为了处理器保持较高的流水线吞吐率和运行时性能。
2. 为原子操作指定所谓的内存顺序memory_order。在CPP11中，标准一共定义了7种memory_order的枚举值，如下所示：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-a93cefafad08082b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. memory_order_seq_cst是默认值
    2. memory_order_relaxed表示该原子操作是松散的，可以被任意重排序的
    ```
    #include <thread>
    #include <atomic>
    #include <iostream>
    using namespace std;
    
    atomic<int> a = 0;
    atomic<int> b = 0;
    
    int func1(int) {
        int t = 1;
        // 使用松散的内存模型
        a.store(t, memory_order_relaxed);
        b.store(2, memory_order_relaxed);
    }
    int func2(int) {
        // 即使打印出0，2也合理
        cout << a << "," << b << endl; // 1, 2或者0，0
    }
    int main() {
        thread t1(func1, 0);
        thread t2(func2, 0);
        t1.join();
        t2.join();
        cout << a << "," << b << endl; // 必然为1， 2
        return 0;
    }
    ```
##### 3.线程局部存储TLS
1. 线程局部存储TLS全称是thread local storage。线程局部存储实际上是由单线程程序中的全局变量、静态变量被应用到多线程程序中被线程共享而来。全局变量和静态变量是被多个线程共享的。
2. C++11中，在变量的声明中加上修饰符thread_local，即可将变量声明为TLS变量。每个线程将拥有全局变量和静态变量的拷贝，一个线程对全局变量的读写不会影响另外一个线程中的全局变量的数据。
##### 4.快速退出：quick_exit和at_quick_exit
1. terminate函数：在没有捕获抛出的异常将会导致terminate函数的调用。而terminate函数在默认情况下是去调用abort函数的。用户可以通过set_terminate函数来改变默认的行为。==默认情况下，abort函数和terminate函数不会调用任何的析构函数。==
2. exit函数：exit函数会正常调用自动变量的析构函数，并且还会调用atexit注册的函数
```
#include <cstdlib>
#include <iostream>
using namespace std;

void openDevice() {cout << "device is opened" << endl;}
void resetDeviceStat() {cout << "device stat is reset" << endl;}
void closeDevice() {cout << "device is closed" << endl;}
int main() {
    // 注册函数，当exit被调用时
    atexit(closeDevice);
    atexit(resetDeviceStat);
    openDevice();
    exit(0);
    return 0;
}
// device is opened
// device stat is reset
// device is closed
```
3. C++11标准引入了quick_exit函数，该函数并不执行析构函数而只是使得程序终止。它和exit函数同属于正常退出。而使用at_quick_exit注册的函数也可以在quick_exit的时候被调用。
```
struct A {~A() {cout << "destruct A" << endl;}};

void closeDevice() {cout << "device is closed" << endl;}

int main() {
    A a;
    at_quick_exit(closeDevice);
    quick_exit(0);
}
// 输出结果， 析构函数不会被调用
// device is closed
```