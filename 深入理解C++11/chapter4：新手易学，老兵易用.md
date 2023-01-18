## chapter3：新手易学，老兵易用
##### 1.右尖括号的改进
1. 在C++98中，如果在实例化模板的时候出现了连续的两个右尖括号>>，那么他们之间需要一个空格来进行分隔，否则编译错误。
```
// g++ main.cpp -std=c++98将编译错误
#include <iostream>
#include <vector>
using namespace std;
int main() {
    vector<vector<int>> vec;
    return 0;
}
```
2. 上述代码在C++11中编译可以通过的。
##### 2.auto类型推导
1. 在早期C/C++标准中，auto关键字一般用于修饰函数中的变量，表示该变量是具有自动存储期的局部变量。在`C++11`中重新定义了auto关键字，作为一个新的类型指示符（和int，float一样）来指示编译器，auto声明变量的类型必须由编译器在编译时期推导而得。
2. auto声明的变量必须被初始化，因为编译器是从初始化表达式中推导变量的类型的。
```
// error，未初始化
auto x;
```
3. 使用auto类型推导，可以增强代码的可读性
```
vector<int> vec = {1, 3, 5};
for (auto iter = vec.begin(); iter != vec.end(); iter++) {
    
}
```
4. auto能够在一定程度上支持泛型的编程
```
template<typename T1, typename T2>
double sum(T1& t1, T2& t2) {
    // 变量result的类型声明为auto的，因为类型T1和T2要在
    // 模板实例化时才能确定
    auto result = t1 + t2;
    return result;
}
int main() {
    int a = 100;
    long b = 200L;
    auto ret = sum(a, b);
    return 0;
}
```
5. auto可以用来声明多个变量的类型，这些变量的类型必须相同。
##### 3.decltype
1. CPP在CPP98标准中就部分支持动态类型，因为CPP中的运行时类型识别RTTI。RTTI的机制是为每个类型产生一个type_info类型的数据，程序员可以在程序中使用typeid函数查询一个变量的类型，该函数返回变量相应的type_info数据，type_info的name成员函数可以返回类型的名称。==在cpp11中，增加了hash_code成员函数，返回类型唯一的哈希值。hash_code是运行时得到的信息==
```
class White {};
class Black {};
int main() {
    White a;
    Black b;
    cout << typeid(a).name() << endl; // 5White
    cout << typeid(b).name() << endl; // 5Black
    White c;
    // 1
    cout << (typeid(a).hash_code() == typeid(c).hash_code()) << endl;
    return 0;
}
```
2. decltype和auto一样，可以进行类型推导。它将一个普通的表达式为参数，返回该表达式的类型。==和auto一样，decltype类型推导也是在编译时进行的。==
```
int i;
decltype(i) j = 100;
cout << typeid(j).name() << endl; // i,g++编译器中表示int类型
float a;
double b;
decltype(a + b) c;
cout << typeid(c).name() << endl; // d
```
3. 使用decltype扩大模板泛型的能力
```

template <typename T1, typename T2>
void sum(T1& t1, T2& t2, decltype(t1 + t2)& s) {
    s = t1 + t2;
}

int main() {
    int a = 100;
    long b = 200l;
    long c;
    sum(a, b, c);
    // 300
    cout << c << endl;
    return 0;
}
```
4. 模板类result_of作用用于推导函数的返回类型
```
typedef double (*func) ();
int main() {
    result_of<func()>::type f;
    cout << typeid(f).name() << endl; // d，表示double类型
    return 0;
}
```
5. 当我们使用`decltype(e)`来获取类型时，编译器将依序判断decltype推导的四规则：
    1. 如果e是一个没有带括号的标记符表达式或者类成员访问表达式，那么decltype(e)就是e所命名的实体的类型。此外，如果e是一个被重载的函数，将会导致编译时错误。==所有除去关键字，字面量等编译器需要使用的标记以外的程序员自定义的标记都可以是标记符。而单个标记符对应的表达式就是标记符表达式。==
    2. 否则，假设e的类型是T，如果e是一个将亡值（右值），那么decltype(e)为T&&
    3. 否则，假设e的类型是T，如果e是一个左值，那么decltype(e)为T&
    4. 否则，假设e的类型是T，那么decltype(e)为T
```
int main() {
    int i;
    // b的类型被推导为是一个int的引用
    // (i)不是一个标记符表达式，却是一个左值表达式
    decltype((i)) b = i;
    cout << i << endl; // 0
    cout << b << endl; // 0
    return 0;
}
```
6. 可以使用模板类is_lvalue_reference的成员value查看decltype的效果。
##### 4.cv限制符的继承
1. 如果对象的定义中有const或者volatile限制符，使用decltype进行推导时，其成员不会继承const或者volatile限制符。
```
// 1
cout << is_const<decltype(test)>::value << endl;
// 0 使用decltype进行推导，其成员不会继承const限制符
cout << is_const<decltype(test.i)>::value << endl;
// 1
cout << is_volatile<decltype(t)>::value << endl;
// 0 使用decltype进行推导，其成员不会继承volatile限制符
cout << is_volatile<decltype(ptr->i)>::value << endl;
```
##### 5.追踪返回类型
1. C++11中引入新语法，追踪返回类型，来声明和定义函数。这个新语法对于编写函数模板时，参数的类型和返回值都在实例化时才决定情况下很有用。构成追踪返回类型函数的两个基本元素：auto占位符和`->返回类型`。示例：
```
template<typename T1, typename T2>
auto sum(const T1& t1, const T2& t2)-> decltype(t1 + t2) {
    return t1 + t2;
}
```
2. 追踪返回类型还可以使用在函数指针中，例如：
```
auto (*ptr) () -> int;
// 等价于
int (*ptr) ();
```
3. 追踪返回类型也适用于函数引用
```
auto (&ptr) () ->int;
// 等价于
int (&ptr) ();
```
##### 6.基于范围的for循环
1. 基于范围的for循环，不用程序员指定循环体其界限的范围。
```
vector<int> vec = {1, 3, 5};
for (auto elem : vec) {
    cout << elem << endl;
}
```