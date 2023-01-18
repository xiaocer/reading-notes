## chapter3：通用为本，专用为末
##### 1.继承构造函数
1. 在子类中可以通过使用using声明来声明继承基类的构造函数。有了继承构造函数就不用繁琐的在子类中调用父类中的构造函数完成子类对象中基类成员的初始化。
```
class A {
private:
    int x;
    double y;
public:
    A(int x, double y): x(x), y(y) {}
    A(){}
    void showInfo() const {
        cout << x << y << endl;
    }
};

class B : public A{
public:   
    // 继承构造函数
    using A::A;
    B(){}
private:
    float z = 3.0; // 子类成员变量的就地初始化
};

int main() {
    B b(200.0, 300.0);
    b.showInfo();
    return 0;
}
```
##### 2.委派构造函数
1. 在一个存在多个构造函数的类中，将一个构造函数设定为基准版本，而其他构造函数可以通过委派这个基准版本来进行初始化。==C++11中的委派构造函数是在构造函数的初始化列表位置进行构造、委派的。==
2. 委派构造：即将构造的任务派给目标构造函数完成类的构造。被委派的构造函数称为目标构造函数，委派给其他构造函数的构造函数称为委派构造函数。
3. 构造函数不能同时委派和使用初始化列表。所以委派构造函数中不能有初始化列表。
4. 示例：构造模板函数作为目标构造函数
```
#include <iostream>
#include <list>
#include <vector>
#include <deque>
using namespace std;

class Test {
private:
    list<int> l;
    // 定义模板函数作为目标构造函数
    template<class T> 
    Test(T first, T last) : l(first, last) {}
public:
    Test(const vector<int>& v) : Test(v.begin(), v.end()) {}
    Test(const deque<int>& d) : Test(d.begin(), d.end()) {}
    void showInfo()const {
        list<int>::const_iterator iter = l.begin();
        for (; iter != l.end(); iter++) {
            cout << *iter << " ";
        }
        cout << endl;
    }
};
int main() {
    vector<int> vec = {1, 3, 5};
    Test test1(vec);
    test1.showInfo();

    deque<int> d = {2, 4, 6};
    Test test2(d);
    test2.showInfo();
    return 0;
}
```
##### 3.右值引用
1. 移动构造函数：在C++11中，偷走临时变量中资源的构造函数称为移动构造函数。移动构造函数可以减少不必要的拷贝构造操作，直接引用已有的资源，提高效率。==通常在一个类中声明了移动构造函数，也会声明一个常量左值引用为参数的拷贝构造函数，以保证在移动构造函数不能使用时，也可以使用拷贝构造。==
2. 左值：在赋值表达式中出现在等号左边的就是左值
3. 右值：在赋值表达式右边出现的称为右值。
    1. 纯右值：比如非引用返回的函数返回的临时变量值、运算表达式的结果、不跟对象关联的字面量值true、'x'都是纯右值。
    2. 将亡值：比如说返回右值引用的函数的返回值、
3. 右值引用：就是对右值进行引用的类型，==也只能对右值进行引用，而不能是左值。注意：右值引用本身就是左值。== 例如：
```
// 声明一个名称为a的右值引用，func()函数返回一个右值
T&& a = func();
```
4. 左值引用：常量左值引用可以接受非常量左值、常量左值、右值对其进行初始化，俗称万能引用。非常量左值引用只能接收非常量左值对其进行初始化。
```
int a = 100;
const int b = 200;

// 非常量左值引用
int& c = a;
// 常量左值引用
const int& d = 400;
const int& e = a;
const int& f = b;
```
5. 左值引用和右值引用的判断：
```
cout << is_rvalue_reference<string&&>::value << endl; // 1   
cout << is_lvalue_reference<int&>::value << endl;     // 1
cout << is_reference<int>::value << endl;             // 0
```
6. std::move函数的使用：将左值强制转换为右值引用类型，以用于移动语义。==被转换的左值，其生命周期并没有随着左右值的转换而改变。== move函数通常用于转换一个生命周期即将结束的对象。
```
#include <iostream>
using namespace std;
class Test {
public:
    Test() : ptr(new int(100)) {}
    ~Test() {
        if (ptr) {
            delete ptr;
        }
    }
    Test(Test && test): ptr(new int(*test.ptr)) {
        test.ptr = nullptr;
    }
    int* getPtr() const {
        return ptr;
    }
private:
    int* ptr;
};
int main() {
    Test t1;
    Test t2(move(t1));
    // error，t1的ptr已经在移动构造函数中被销毁
    cout << *t1.getPtr() << endl;
    return 0;
}
```
7. 判断一个类是否具有移动语义
```
class Test {
public:
    Test() : ptr(new int(100)) {}
    ~Test() {
        if (ptr) {
            delete ptr;
        }
    }
    Test( Test& test): ptr(new int(*test.ptr)) {}
    Test(Test && test): ptr(new int(*test.ptr)) {
        test.ptr = nullptr;
    }
    int* getPtr() const {
        return ptr;
    }
private:
    int* ptr;
};
int main() {
    // Test类包含移动构造函数
    cout << is_move_constructible<Test>::value << endl; // 1
    return 0;
}
```
8. 使用移动语义完成高性能的置换函数
```
template <class T>
void swap(T& a, T& b) {
    T temp(move(a));
    a = move(b);
    b = move(temp);
}
```
9. move_if_noexcept函数：用于替代move函数，该函数在类的构造函数没有noexcept关键字修饰时返回一个左值引用从而使得变量可以使用拷贝语义，而在类的移动构造函数有noexcept关键字时，返回一个右值引用从而使变量可以使用移动语义。==在移动构造函数中抛出异常可能会导致一些指针成为悬挂指针，不安全。==
10. RVO/NRVO：RVO称为返回值优化，全称Return Value Optimization，NRVO全称Named Return Value Optimization。
    1. RVO优化的关闭：编译时使用-fno-elide-constructors选项
##### 4.完美转发
1. 完美转发（perfect forwarding）：指的是在函数模板中，完全依照模板的参数的类型，将参数传递给函数模板中调用的另外一个函数。==完美转发考虑的是转发函数的接受能力，使得目标函数既能够接收左值引用，又能接收右值引用。==
2. 术语：
```
template <typename T>
// forwarding是一个转发函数模板
void forwarding(T t) {
    // runCode称为真正执行代码的目标函数
    runCode(t);
}
```
3. ==C++11中通过引入一条所谓引用折叠的新语言规则，并结合新的模板推导规则来完成完美转发。==
    1. 在C++11中形如一下语句：
    ```
    typedef const int T;
    typedef T& TR;
    TR& v = 1
    ```
    2. C++11中的引用折叠规则：引用折叠的意思就是将复杂的未知表达式折叠为已知的简单表达式。
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-ce7dde04af86b44e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    3. 模板推导规则：当转发函数的实参是类型X的一个左值引用，那么模板参数被推导为`X&`类型，而转发函数的实参是类型X的一个右值引用的话，那么模板的参数被推导为`X&&`类型。
4. 在C++11中，用于完美转发的函数称作forward函数。它和move函数在实现上差别不大，通常就是一个static_cast，仅仅做了类型的转换。
```
// 完美转发
template <typename T>
void forwarding(T && t) {
    runcoding(forward<T>(t));
}
```
5. 完美转发示例：
```
#include <iostream>
#include <utility>
using namespace std;

void runCode(int&& m) {
    cout << "右值引用版本" << endl;
}
void runCode(int& m) {
    cout << "左值引用版本" << endl;
}
void runCode(const int&& m) {
    cout << "常量右值引用版本" << endl;
}
void runCode(const int& m) {
    cout << "常量左值引用版本" << endl;
}
template <typename T>
void perfectForward(T && t) {
    runCode(forward<T>(t));
}
int main() {
    int a;
    int b;
    const int c = 1;
    const int d = 0;
    // 左值引用版本
    // 右值引用版本
    // 常量左值引用版本
    // 常量右值引用版本
    perfectForward(a);
    perfectForward(move(b));
    perfectForward(c);
    perfectForward(move(d));
    return 0;
}

```
##### 5.显式转换操作符
1. 使用explicit关键字修饰构造函数，表示禁止从内置基础类型到自定义类型的隐式构造，只能显式构造。
```
class Test {
private:
    int x;
public:
    explicit Test(int x) : x(x) {}
    int getX() const {return x;}
};
int main() {
    // 隐式构造 将编译报错
    Test test = 100;
    cout << test.getX() << endl;
    return 0;
}
```
2. 使用explicit关键字修饰转换构造函数，表示在将自定义类型转换为内置基础类型或者其他自定义类型的时候，只能显式转换。==当然所谓显式转换并没有完全禁止从自定义类型转换为内置基础类型或者其他自定义类型==
```
template <typename T>
class Test {
public:
    Test(T* p) : _p(p) {}
    explicit operator bool() const {
        if (_p != nullptr) 
            return true;
        else
            return false;
    }
private:
    T* _p;
};
int main() {
    int a;
    Test<int> test(&a);
    // ok
    if (test)
        cout << "valid pointer!" << endl;
    // error，由于explicit关键字的作用
    // bool f = test;
    return 0;
}
```
##### 6.列表初始化
1. 在C++98中，允许使用花括号{}对数组元素进行统一的列表初始值设定，例如`int arr[] = {1, 2, 3, 4};`。在`C++11`中，允许使用花括号对STL中的容器进行初始化，这种初始化方法称为初始化列表。
```
// ok
vector<int> vec {1, 2, 3, 4};
// ok
map<int, string> m = {{1001, "张三"},{1002, "李四"}};
```
2. 自定义类型使用列表初始化功能：需要定义一个以`initialize_list<T>`模板类为参数的构造函数
```
class Test {
public:
    Test(initializer_list<pair<int, string>> l) {
        auto i = l.begin();
        for (; i != l.end(); i++) {
            vec.push_back(*i);
        }
    } 
private:
    vector<pair<int, string>> vec;
};

int main() {
    // 使用列表初始化功能初始化自定义类型
    Test test{{1001, "张三"}, {1002, "李四"}};
}
```
3. 防止类型收窄：类型收窄指的是一些可以使数据变化或者精度丢失的隐式类型转换。可能导致类型收窄的情况比如说如下情况。==cpp11中使用初始化列表进行初始化的数据，编译器是会检查其是否发生类型收窄的。==
    1. 将浮点数隐式转化为整型数
    2. 从高精度的浮点数转换为低精度的浮点数
    3. 从整型数（或者非强类型的枚举）转换为浮点型
    4. 从整形（或者非强类型的枚举）转化为较低长度的整形
    ```
    int x = 1024;
    // warning
    char y  = {x};
    // error narrowing conversion
    float* ptr = new float{1e48};
    ```
##### 7.POD类型
1. POD全称为plain old data。cpp11将POD划分为两个基本概念的合集，即平凡的和标准布局的。
2. 一个平凡的类或者结构体应该符合以下定义：
    1. 拥有平凡的默认构造函数和析构函数：即由编译器为我们生成默认的构造函数和析构函数。
    2. 拥有平凡的拷贝构造函数和移动构造函数：即由编译器为我们生成
    3. 拥有平凡的拷贝赋值运算符和移动赋值运算符：即由编译器为我们生成
    4. 不能包含虚函数以及虚基类
    5. 判断一个类是否为平凡的：使用is_trivial类模板的value属性。
    ```
    #include <iostream>
    #include <type_traits>
    using namespace std;
    class Test {
    public: 
        Test();
    };
    Test::Test() = default;
    
    struct Base {
    
    };
    int main() {
        cout << is_trivial<Test>::value << endl; // 0
        cout << is_trivial<Base>::value << endl; // 1
    
        return 0;
    }
    ```
3. 标准布局的类或者结构体应该符合以下定义：
    1. 所有非静态成员都有相同的访问权限，例如：
    ```
    // a和b访问权限不同
    struct Test {
    public:
        int a;
    private:
        int b;
    }
    ```
    2. 在类或者结构体继承时满足以下情况之一：
        1. 派生类中存在非静态成员，且只有一个仅包含静态成员的基类
        2. 基类中存在非静态成员，而派生类中没有非静态成员
    3. 类中第一个非静态成员的类型与其基类不同。如果相同，则不能称作标准布局。在C++标准中，如果基类没有成员，标准允许派生类的第一个成员与基类共享地址。因为派生类的地址总是堆叠在基类之上的，这样的地址共享表明基类并没有占据任何的实际空间，可以节省一点空间。==但是如果基类的第一个成员仍是积累类型的，编译器仍然会为基类分配1字节的空间，因为Cpp标准要求类型相同的对象必须地址不同。所以如下案例中D1和D2的布局不一样==
    ```
    struct B1 {};
    struct B2{};
    struct D1 : B1 {
        B1 b;
        int i;
    };
    struct D2 : B1 {
        B2 b;
        int i;
    };
    
    int main() {
        D1 d1;
        D2 d2;
        // 140723578952152
        cout << reinterpret_cast<long long>(&d1) << endl;
        // 140723578952153，一个字节是编译器为子类对象基类部分分配的
        cout << reinterpret_cast<long long>(&(d1.b)) << endl;
        // 140723578952156，三个字节是padding
        cout << reinterpret_cast<long long>(&(d1.i)) << endl;   
        // 140723578952160
        cout << reinterpret_cast<long long>(&d2) << endl;
        // 140723578952160，地址共享，因为d2第一个成员和基类类型不同
        cout << reinterpret_cast<long long>(&(d2.b)) << endl;
        // 140723578952164
        cout << reinterpret_cast<long long>(&(d2.i)) << endl;
        return 0;
    }
    ```
    4. 没有虚函数和虚基类
    5. 所有非静态成员都符合标准布局类型，其基类也符合标准布局。
    6. 使用模板类来帮助判断类型是否是一个标准布局的类型：`is_standard_layout的value属性`
4. POD的判断：使用类模板`is_pod`的value属性来判定一个类型是否是POD
##### 8.非受限联合体
1. 非受限联合体：在新的CPP11标准中，取消了联合体对于数据成员类型的限制。标准规定，任何非引用类型都可以成为联合体的数据成员
2. 在CPP11中，非受限联合体有一个非POD的成员，而该非POD成员类型拥有非平凡的构造函数，那么该受限联合体的默认构造函数将被编译器删除。
```
union T {
    string s; // string有非平凡的构造函数
    int n;
public:
    T() {}
};

int main() {
    // error 不能构造，因为T的构造函数已被删除
    T t;
}
```
3. 匿名的非受限联合体可以运用于类的声明中，作为类的变长成员
```
#include <iostream>
#include <cstring>
using namespace std;

struct Student {
public:
    Student(bool gender, int age) : gender(gender), age(age) {}
private:
    bool gender;
    int age;
};
class Singer {
public: 
    enum Type {STUDENT, NATIVE, FOREIGNER};
    Singer(bool gender, int age) : s(gender, age) {t = STUDENT;}
    Singer(int id) : id(id) {t = NATIVE;}
    Singer(const char* name, int size) {
        size = (size > 9) ? 9 : size;
        memcpy(_name, name, size);
        _name[size] = '\0';
        t = FOREIGNER;
    }
    ~Singer() {}
private:
    Type t;
    // 匿名的非受限联合体
    union {
        Student s;
        int id;
        char _name[10];
    };
};
int main() {
    Singer s1(true, 23);
    Singer s2(2021);
    Singer s3("Tom", 9);
    return 0;
}
```
##### 9.用户自定义字面量
1. CPP11标准中，通过定义一个后缀标识的操作符，可以将声明了该后缀标识的字面量转化为需要的类型。后缀建议以下划线开始。
```
struct Watt {unsigned int v;};
ostream& operator<< (ostream& out, Watt& watt) {
    return out << watt.v << endl;
}
Watt operator "" _w(unsigned long long v) {
    return {(unsigned int)v};
}

int main() {
    // 1024_w就是一个用户自定义字面量
    Watt capacity = 1024_w;
    cout << capacity; // 1024
    return 0;
}
```
##### 10.内联名字空间
1. CPP中引入的名字空间，目的是分隔全局共享的名字空间，防止多个程序员合作编程时变量名或者函数名的冲突。
```
// 编写者张三
namespace Zs {
    namespace Test1 {
        struct Knife {};
    }
    namespace Test2 {
        struct Carpet {};
    }
    // 对外打开内部的Test1名字空间
    using namespace Test1;
}
// 使用者李四
using namespace Zs;
int main() {
    // OK
    Knife knife;
    // error
    // Carpet carpet;
    return 0;
}
```
2. CPP11中引入内联的名字空间，通过关键字`inline namespace`就可以声明一个内联的名字空间。==内联的名字空间允许程序员在父名字空间定义或者特化子名字空间的模板。内联名字空间的缺点：名字空间的内联会破坏该名字空间本身具有的封装性==
```
// 张三编写
namespace Zs {
    inline namespace Test1 {
        template<typename T>
        struct Knife {};
    }
    inline namespace Test2 {
        struct Carpet{};
        Test1::Knife<int> knife1; // 使用另一个名字空间的类
        // OK的，名字空间的内联会破坏该名字空间本身具有的封装性
        Knife<int> knife2;
    }
}

// 李四使用
using namespace Zs;
namespace Zs {
    // 模板的偏特化
    template<> struct Knife<Carpet> {};
}
int main() {
    Knife<Carpet> knife;
    return 0;
}
```
##### 11.模板的别名
1. 在CPP中使用`typedef`关键字为类型定义别名。同样使用`using`关键字也可以定义类型的别名
```
// unsigned int的别名为uint
using uint = unsigned int;
// int的别名为UINT
typedef unsigned int UINT;
int main() {
    // is_same用于判断两个类型是否一致
    cout << is_same<uint, UINT>::value << endl; // 1
    return 0;
}
```
2. 在使用模板编程的时候，using的语法比typedef更加灵活
```
template <typename T>
using MapString = map<T, const char*>;

int main() {
    MapString<int> _map {{1001, "张三"}, {1002, "李四"}};
    for (auto i = _map.begin(); i != _map.end(); i++ ) {
        cout << i->first << " " << i->second << endl;
    }
    return 0;
}
```
##### 12.一般化的SFINEA规则
1. SFINEA：全称substitution failure is not an error，中文直译匹配失败不是错误。当对重载的模板的参数进行展开的时候，如果展开导致了一些类型不匹配，编译器并不会报错。因为存在第二个模板定义匹配成功并对其实例化。
```
struct Test {
    typedef int foo;
};
template <typename T>
void f(typename T::foo) {}
template <typename T>
void f(T) {}
int main() {
    f<Test>(10);
    // 虽然不存在int::foo这样的类型，但是并不会报错，另一个函数模板匹配成功
    f<int>(10);
}
```