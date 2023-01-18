## 1.指针空值nullptr
##### 1.指针空值：从0到NULL，再到nullprt
1. 将指针初始化为指向一个空的位置，比如说0。==可以这么做因为大多数计算机系统不允许用户程序写地址为0的内存空间。==
2. 使用NULL，NULL是一个宏定义
```
#ifndef __cplusplus
#define NULL ((void *)0)
#else   /* C++ */
#define NULL 0
#endif  /* C++ */
```
3. 使用CPP11标准中的关键字nullptr，nullptr是一个指针空值类型nullptr_t的常量。nullptr是有类型的，且仅可以被隐式转化为指针类型，不能转换为整形或者布尔型
```
typedef decltype(nullptr) nullptr_t;
```
4. 示例：
```
void func(char* ptr) {
    cout << "invoke f(char*)" << endl;
}
void func(int) {
    cout << "invoke f(int)" << endl;
}
int main() {
    // 调用func(char* ptr)版本
    func(nullptr);
    // 编译报错，不确定调用哪一个版本
    func(NULL);
}
```
##### 2.nullptr和nullptr_t
1. nullptr_t类型的数据可以转换为任意一个指针类型
```
int* ptr = nullptr;
cout << ptr << endl; // 0
```
2. nullptr与nullptr_t类型的变量可以做比较，两者是相等的
```
nullptr_t ptr;
cout << (ptr == nullptr) << endl;   // 1
cout << (0 == nullptr) << endl;     // 1
```
3. 将nullptr_t应用于模板的时候，模板会将它作为一个普通的类型来进行推导。
```
template<typename T>
void func(T* t) {}
int main() {
    // 编译报错，因为nullptr的类型为nullptr_t,而不是指针
    func(nullptr);
    // OK,推导T=float
    func((float*) nullptr);
    return 0;
}
```
4. nullptr到任何类型的指针的转换是隐式的，而无类型指针`void*`必须经过类型转换后才能使用。所以说nullptr方便多了。
5. nullptr是一个右值常量，不能使用`&`符号获得nullptr的地址，而nullptr_t类型的变量的地址可以获得。
```
template<typename T>
void func(T* t) {}
int main() {
    nullptr_t ptr;
    // 0x7fff47d341a0
    cout << &ptr << endl;
    nullptr_t&& rvalue = nullptr;
    // 0x7fff47d341a8
    cout << &rvalue << endl;
    return 0;
}
```
## 2.默认函数的控制
##### 1.类与默认函数
1. 默认函数：
    1. 编译器会在自定义类中默认生成一些程序员没有定义的成员函数。这些成员函数包括：
        1. 构造函数
        2. 拷贝构造函数
        3. 拷贝赋值函数
        4. 移动构造函数
        5. 移动拷贝函数
        6. 析构函数
    2. 为自定义类型提供全局默认操作符函数：
        1. operator,
        2. operator&
        3. operator&&
        4. operator*
        5. operator->
        6. operator->*
2. CPP11中重用了default关键字，程序员可以在默认函数定义或者声明时加上`=default`从而显式的指示编译器生成该函数的默认版本。
3. 在CPP11中，在函数的定义或者声明加上`=delete`指示编译器不生成函数的缺省版本。CPP11标准称`=default`修饰的函数为显式缺省函数，而称`=delete`修饰的函数为删除函数。
    1. 示例：使用显式删除来删除自定义类型的operator new操作符，避免在堆上分配该类型的对象
    ```
    class Test {
    public:
        void* operator new(size_t) = delete;
    };
    
    int main() {
        // 编译失败，无法引用被删除的函数
        Test* ptr = new Test();
        return 0;
    }
    ```
    2. 显式删除析构函数来限制自定义类型在栈上或者静态的构造
    ```
    class Test {
    public:
        ~Test() = delete;
    };
    
    int main() {
        // 编译失败，无法引用已经删除的函数
        Test test;
    }
    ```
## 3.lambda函数
##### 1.lambda函数的基本使用
1. lambda函数的语法定义：[capture] (parameters) mutable ->return-type{statement}
    1. capture：捕捉列表，捕捉上下文中（这里指的是父作用域）的变量以供lambda函数使用
    2. parameters：参数列表。如果不需要参数传递则可以连同括号省略。
    3. mutable：默认情况下，lambda函数总是一个const函数，mutable可以取消其常量性
    4. ->return-type：返回类型
    5. {statement}：函数体，函数体中可以使用参数和捕获的外部变量
```
// 最简单的lambda函数
[]{};
int a = 100;
int b = 200;
// 省略了参数列表和返回类型
cout << ([=] {return a + b;})() << endl;
auto func = [=, &b] (int c) ->int {return b += a + c;};
cout << func(300) << endl;
```
2. 捕捉列表由多个捕捉项组成，并以逗号分隔。捕捉列表有如下几种形式：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-54c42515df30c2b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 仿函数的概念：重定义了成员函数`operator()`的一种自定义类型对象。仿函数可以拥有初始状态，即在自定义类型内部定义私有成员变量。==仿函数是编译器实现lambda的一种方式。lambda函数的函数体部分，被转化为仿函数之后会成为一个类的常量成员函数。在现阶段，通常编译器都会把lambda函数转化为一个仿函数对象。lambda函数书写简单，可以使用lambda函数代替仿函数来书写代码。==
4. lambda函数在`C++11`标准中默认是内联的，代码在行内展开。
5. 在效果上，lambda函数等同于一个局部函数。（即在函数作用域中定义的函数）
6. lambda的类型被定义为闭包的类，而每个lambda表达式则会产生一个闭包类型的临时对象（右值）。因此严格的讲，lambda函数并非函数指针。
7. lambda函数的常量性及mutable关键字，默认情况下，lambda等价的是有常量operator()的仿函数。而mutable修饰符可以消除其常量性、
```
int x = 100;
// 编译错误，在常量的lambda中修改常量
// auto func = [=] () {x = 200;};

// 非const的lambda可以修改常量数据，但是不影响外部的数据
auto func = [=] () mutable {x = 200;};

// const的lambda，改动引用本身。
// auto func = [&] () {x = 200;};
```
##### 2.STL中使用lambda
1. STL算法for_each简单示例
```
const int bound = 100;
vector<int> input = {20, 121, 45, 300};
vector<int> output;
inline void filter(int i) {
    if (i > bound)
        output.push_back(i);
}
int main() {
    for_each(input.begin(), input.end(), filter);
    for_each(input.begin(), input.end(), [=] (int i) {
        if (i > bound)
            output.push_back(i);
    });
    for (int i : output)
        cout << i << " ";
    return 0;
}
```
2. 函数指针方式与lambda方式传参的比较：
    1. 函数定义在别的地方，阅读不方便
    2. 出于效率考虑，使用函数指针很可能导致编译器不对其进行内敛优化，再循环次数较多的时候，内敛的lambda和没有能够内敛的函数指针可能存在着巨大的性能差别。
    3. 函数指针的应用范围相对狭小，不能传参。
3. 仿函数和lambda的取舍：
    1. 在现行CPP11中，捕捉列表仅能捕捉父作用域的自动变量，而对于超出这个范围的变量是不能被捕捉的
    ```
    int d = 0;
    int test() {
        // 编译报错：无法在 lambda 中捕获带有静态存储持续时间的变量
        auto func = [d]() {};
    }
    ```
    2. 仿函数可以被定义以后在不同的作用域范围内取得初始值（即构造仿函数的外部传递的初始值）。
    ```
    int d = 1024;
    class Test {
    private:
        int _data;
    public:
        Test(int d): _data(d) {}
        void operator() ()const {
            cout << _data << endl;
        }
    };
    int main() {
        // 定义仿函数， d即初始值
        Test test(d);
        test();
        return 0;
    }
    ```