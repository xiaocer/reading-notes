## chapter2：保证稳定性和兼容性
##### 1.保持与C99兼容
C99是1999年指定的C标准。
1. 预定义标识符__func__：用于返回所在函数的名字
```
const char* func() {
    return __func__; 
}
struct Test {
public:
    Test() : name(__func__) {}
    const char* name;
};
int main() {
    cout << func() << endl;// func
    Test test;
    cout << test.name << endl; // Test
    return 0;
}
```
2. _Pragma操作符：在C/`C++`标准中，`#pragma`是一条预处理的指令，该指令会指示编译器该指令所在的头文件应该只被编译一次。这个指令的功能和`#ifndef XXX_H #define XXX_H #endif`来定义头文件达到的效果一样。在C++11中，标准定义了与预处理指令#pragma功能相同的操作符_Pragma
```
# 在头文件中定义
_Pragma("once")
```
3. 变长参数的宏定义以及__VA_ARGS__：一般用在轻量级调试中
```
#define LOG(...) {\
    fprintf(stderr, "%s:line %d:\t", __FILE__, __LINE__);\
    fprintf(stderr, __VA_ARGS__);\
    fprintf(stderr, "\n");\
}
int main() {
    int x = 3;
    // main.cpp:line 11:       x = 3
    LOG("x = %d", x);
    return 0;
}
```
4. long long整形
```
long long int x = -1LL;
unsigned long long int y = 1ull;
// 8 8
cout << sizeof(x) << " " << sizeof(y) << endl;
```
5. 扩展的整形：C++11中定义了5种标准的有符号整形。
    1. signed char
    2. short int
    3. int
    4. long int
    5. long long int
6. 宏__cplusplus：
    1. 可以用来检验代码是否支持使用C++11编译器进行编译
    ```
    #if __cplusplus < 201103L
        #error "should use C++11 implementation"
    #endif
    ```
    2. 在C和`C++`混合编写的代码中，可以在头文件中使用如下声明，这样这种类型的头文件可以被#include到C文件中进行编译，也可以被#include到`C++`文件中进行编译。因为extern "C"可以抑制`C++`对函数名称，变量名等符号进行名称重整，因此编译出的C目标文件和`C++`目标文件中的变量名称、函数名称等符号都相同。
    ```
    #ifdef __cplusplus
    extern "C" {
    #endif
    int func(int a, int b);    
    #ifdef __cplusplus
    }
    #endif
    ```
7. 断言：将一个返回值总是需要为真的判别式放在语句中，用于排除在设计的逻辑上不应该产生的情况。
    1. 运行时断言：使用assert宏，在运行的时候进行逻辑的判断。然后再发布程序时定义NDEBUG宏来禁用assert宏。例如编译时指定宏`g++ main.cpp -o main -D NDEBUG`
    ```
    char* allocArray(int n) {
        assert(n > 0); // 断言 n > 0
        return new char[n];
    }
    int main() {
        char* ptr = allocArray(0);
    }
    ```
    2. 静态断言：==在编译时期进行断言==，在C++11中引入了static_assert，它接收两个参数，一个是断言表达式，该表达式的结果必须是在编译时期就可以计算出结果，即常量表达式。另一个则是警告信息。
    ```
    static_assert(sizeof(int) == 4, "int大小不为4");
    ```
8. noexcept关键字：
    1. 作为修饰符：修饰函数表示函数不会抛出异常。语法格式为：`noexcept(常量表达式)`，当常量表达式的结果为true则表示函数不会抛出异常，反之则可能抛出异常。它和通过`throw()`动态异常声明不同的是，在`C++11`中如果noexcept修饰的函数抛出了异常，编译器可以选择直接调用std::terminate函数来终止程序的运行，从而阻止异常的传播和扩散。`C++11`标准中类的析构函数默认是`noexcept(true)`的
    ```
    void test() noexcept {
        throw 1; // 程序将调用terminate终止程序运行
    }
    // 和上述等同
    //void test() noexcept(true) {
    //    throw 1; // 程序将调用terminate终止程序运行
    //}
    int main() {
        try {
            test();
        }
        // 不会执行
        catch(...) {
            cout << "捕获异常进行处理" << endl;
        }
        return 0;
    }
    ```
    2. 作为操作符：通常用于模板
    ```
    // func函数是否是一个noexcept的函数，将由T()表达式是否会抛出异常决定
    template <class T>
    void func() noexcept(noexcept(T())){}
    ```
9. 快速初始化成员变量
    1. 在C++11中，标准允许使用等号或者花括号进行就地的非静态成员变量初始化。如果同时存在就地初始化和在构造函数中使用的成员初始化列表进行初始化，前者先执行。
    ```
    class Test {
    private:
        int x = 100;
    public:
        Test(int x) : x(x) {}
        int getX()const {return x;}
    };
    int main() {
        Test test(200);
        cout << test.getX() << endl; // 200
        return 0;
    }
    ```
    2. 对于非常量的静态成员变量，`C++11`和`C++98`保持了一致，程序员需要到头文件以外或者类外去定义它。
    3. C++98中，对于常量的静态成员，可以在类中就地声明。
10. 非静态成员的sizeof：在`C++98`中，如下写法是错误的，不支持。但是在C++11中sizeof作用的表达式包括了类成员表达式。
```
class Test {
public:
    int x = 100;
};

int main() {
    // error
    cout << sizeof(Test::x) << endl;
    return 0;
}
```
11. friend关键字
    1. 在C++11中声明一个类为另一个类的友元时不再需要使用class关键字。
    ```
    class Test2;
    class Test1 {
        friend  Test2;
        // friend class Test2;
    };
    class Test2 {
    
    };
    ```
    2. 在类模板中声明友元，使用int类型进行实例化类模板，就不会有友元。
    ```
    template <typename T> 
    class Test2 {
        friend T;
    };
    class Test1 {
    
    };
    ```
12. final/override关键字
    1. final关键字：使用final关键字修饰虚函数，==只能修饰虚函数==，表示在派生类中不可以覆盖它所修饰的虚函数。
    2. override关键字：一般用于修饰派生类中的虚函数表示重写其基类中的同名函数
13. 模板函数的默认模板参数：在C++11中支持函数模板的默认模板参数，并且对于函数模板来说，默认模板参数的位置比较随意，不像在类模板中，必须从右往左定义默认类模板参数。==默认模板参数通常是和默认函数参数一起使用==
```
template <typename T1 = int>
void func(T1 t = 200) {
    cout << t << endl;
}
int main() {
    func(100);
    func();
    func<const char*>("test");
    return 0;
}
```
14. 外部模板
    1. 外部模板的声明，声明外部模板表示在编译当前文件后，不再进行模板的实例化。==而在链接时使用其他目标文件中显式实例化的模板。这个和使用extern关键字声明外部变量的道理一致。==
    ```
    extern template void func<int>(int);
    ```
    2. 模板的显示实例化：
    ```
    template void func<int>(int);
    ```
15. 局部和匿名类型作为模板实参，模板包括模板类和模板函数
```
template <typename T> class Test{};
template <typename T> void func(T t) {};

struct A{}a;
struct {}b; // b是匿名类型变量
typedef struct {}C; // c是匿名类型

int main() {
    struct B{} d; // 局部类型变量

    Test<A> t1;
    Test<C> t2;
    Test<B> t3;

    func(a);
    func(b);
    func(d);
    return 0;
}
```