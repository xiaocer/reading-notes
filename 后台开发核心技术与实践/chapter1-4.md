## CPP编程基础知识
1. 使用在项目下使用include包含其他文件时，`include <>和include ""`的区别：
    1. include<>常用来包含系统提供的头文件，编译器会在保存系统标准头文件的位置查找
    2. include""常用来包含程序员自己编写的头文件，编译器会先在项目目录下查找指定名称的头文件，然后从保存系统标准头文件的位置查找
2. 数组指针和指针数组的区别：
    1. 数组指针：指向一维数组的指针，亦称行指针。本质还是一个指针变量。
    ```
    int a[2][2] = {1, 2, 3, 4};
    int (*p)[2] = a;
    cout << *p[0] << endl; // 1
    ++p; // 指向a[1]
    cout << *p[0] << endl; // 3
    ```
    2. 指针数组：数组中每一个元素都是一个指针变量。本质是一堆指针变量。
3. cpp中引用的本质：本质就是一个指针常量，因为引用再初始化后就不能再作为其他变量的引用。常引用的本质既是一个常量指针，也是一个指针常量。
4. 共用体：cpp中使用关键字union定义共用体。在一个共用体里可以定义多种不同的数据类型，但是同一时间只能存储其中一个成员变量的值，这些数据共享一段内存。使用示例：测试系统使用的是大端序还是小端序
```
union TEST
{
    short a;
    char b[sizeof(a)];
};
int main() 
{
    TEST test;
    test.a = 0x0102;
    if (test.b[0] == 0x01)
    {
        cout << "系统使用的是大端序" << endl;
    }
    else if (test.b[0] == 0x02)
    {
        cout << "系统使用的是小端序" << endl;
    }
    else
    {
        cout << "未知" << endl;
    }
    return 0;
}
```
5. 结构体、共用体在内存单元占用字节数的计算：
    1. 共用体：以最长的为准，并且需要遵守内存对齐方式（padding）
6. C++中提供的预处理包括宏定义、文件包含、条件编译、布局控制
    1. 宏定义比如说：`#define PI 3.14`
    2. 文件包含：`#include <iostream>`
    3. 条件编译：
    ```
    #ifdef 标识符
        程序段1
    #else
        程序段2
    #endif
    
    #if 表达式
        程序段1
    #else
        程序段2
    #endif
    ```
7. extern "C"块的应用：作用就是告诉CPP编译器块中的代码按照c标准进行编译。编译后的函数命名是以C的方式。
```
#ifdef __cplusplus
    extern "C" {
#endif
    ...
#ifdef __cplusplus
    }
#endif
```
8. 一个类中如果定义了全是默认参数的构造函数后，就不能再定义重载构造函数。
9. 静态数据成员不属于某一个对象，在为对象分配的空间中不包括静态数据成员所占的空间。
10. 对象的存储空间的计算：
    1. C++中每个空类型的实例占1字节空间
    2. 成员函数（包括析构函数和构造函数）不占用对象的存储空间
    3. 静态的数据成员也不占用对象的存储空间
    4. 类中含有虚函数，则对象会拥有一个指向虚函数表的指针占用8字节。
11. 基类成员在派生类中的访问属性由不同的继承方式决定：
    1. 公有继承：基类的公有成员和保护成员在派生类中保持原有访问属性，其私有成员仍为基类私有，子类中是不能访问的。
    2. 私有继承：基类的公有成员和保护成员在派生类中成了私有成员，其私有成员仍为基类私有
    3. 保护继承：基类的公有成员和保护成员在派生类中成了保护成员（保护成员在外界不能被访问），其私有成员仍为基类私有。
12. 在派生类中是不能访问基类的私有成员的，私有成员只能被本类的成员函数访问。
13. 当一个成员函数被声明为虚函数后，其派生类中的同名函数都自动成为虚函数
14. 纯虚函数：在基类中声明的虚函数，而且在基类中没有定义。含有纯虚函数的类称为抽象类，他不能生成对象。例如：
```
virtual void func() = 0;
```
15. 基类包含虚函数，则其析构函数也应该加上virtual关键字（编译器动态绑定），==否则会造成内存泄漏，子类对象的基类部分可以被析构，但是自身的不行。==
16. strlen()函数计算字符串的长度不会将"\0"计算进去
```
char* ptr = "Hello\0World";
cout << strlen(ptr) << endl; // 5
string str = "Hello\0World";
cout << str.length() << endl;  // 5
```
17. C++字符串和C字符串的相互转换：
    1. 将C字符串转为C++字符串，直接使用string类的构造函数`string(const char* str)`
    2. 将C++字符换转为C字符串：使用data()、c_str()、copy()方法
    ```
    char* cstr = new char[20];
    string str = "Hello,World";
    strncpy(cstr, str.c_str(), str.size());
    ```
18. string和int的转换：
    1. int转string的方法：使用`int snprintf(char *restrict buffer, size_t bufsz,  const char*restrict format,...);`方法，功能是将可变个参数按照format格式转化为字符串，将其复制到buffer中。
    2. string转整数的方法：使用strtol等
19. 在vector容器被构造之后应当立即使用reserve方法将容器的容量设置为足够大，减少vector容器重新分配内存的次数。
20. map中插入数据的方式：
    1. 使用insert函数插入pair类型的数据
    ```
    map<int, string> m;
    m.insert(pair<int, string>(1, "张三"));
    m.insert(pair<int, string>(2, "李四"));
    m.insert(pair<int, string>(3, "王五"));
    ```
    2. 使用insert函数插入value_type类型的数据
    ```
    map<int, string> m;
    m.insert(map<int, string>::value_type (1, "张三"));
    m.insert(map<int, string>::value_type (2, "李四"));
    m.insert(map<int, string>::value_type (3, "王五"));
    ```
    3. 使用数组方式插入数据
    ```
    map<int, string> m;
    m[1011] = "张三";
    m[1012] = "李四";
    m[1013] = "王五";
    ```
    4. 使用insert和数组两种方式插入数据的区别：当map中已经存在对应的关键字时，再执行insert操作时是不会插入数据的；而使用数组进行插入则会覆盖已存在关键字的值，可以插入。==如何判断insert方法是否插入数据成功是有必要的，示例如下==
    ```
    # 通过pair的第二个属性判断
    map<int, string> m;
    m.insert(map<int, string>::value_type (3, "王五"));
    pair<map<int, string>::iterator, bool> insert_pair;
    insert_pair = m.insert(map<int, string>::value_type (3, "王"));
    if (insert_pair.second == true) {
        cout << "插入成功！" << endl;
    } 
    else {
        cout << "插入失败！" << endl;
    }
    ```
21. map中的排序默认按照key从小到大排序。
    1. 按照key从大到小排序
    ```
    // greater是预定义好的比较器
    // 本质就是一个重载了函数调用运算符的结构体
    map<int, string, greater<int>> m;
    m[1011] = "张三";
    m[1013] = "王五";
    m[1012] = "李四";
    ```
    2. 按照键的长度进行从小到大排序，需要自定义比较器
    ```
    struct UserDefinedComparator {
        bool operator()(const string& s1, const string& s2)const {
            return s1.length() < s2.length();
        }
    };
    int main() {
        map<string, string, UserDefinedComparator> m;
        m["666666"] = "666666";
        m["55555"] = "55555";
        m["333"] = "333";
        
        map<string, string, UserDefinedComparator>::iterator iter = m.begin();
        for (; iter != m.end(); iter++) {
            cout << iter->first << " " << iter->second << endl;
        }
    }
    ```
    3. map按照value进行排序：将map中的元素按照pair形式插入到vector中，然后再对vector使用sort算法进行排序。
    ```
    bool compareByValue(const pair<string, int>& left, const pair<string, int>& right) {
        // 按照元素值从小到大排序
        return left.second < right.second;
    }
    struct CmpByValue {
        bool operator()(const pair<string, int>& left, const pair<string, int>& right) const {
            return left.second < right.second;
        }
    };
    int main() {
        map<string, int> name_score_map;
        name_score_map["zs"] = 78;
        name_score_map.insert(make_pair("ls", 67));
        name_score_map.insert(pair<string, int>("ww", 89));
        // 将map中的元素转存到vector中
        vector<pair<string, int>>vec(name_score_map.begin(), name_score_map.end());
        // 对map中的元素进行排序
        // sort(vec.begin(), vec.end(), compareByValue);
        sort(vec.begin(), vec.end(), CmpByValue());
        for (pair p : vec) {
            cout << p.first << " " << p.second << endl; 
        }
        return 0;
    }
    ```
## 编译
##### 1.编译与链接
1. 预处理：将源代码文件xxx.cpp和相关的头文件使用预处理器CPP预处理成xxx.i文件，预处理主要处理所有预编译指令，比如说`#include`和`#define`
```
# -E表示只执行到预编译，直接输出预编译结果
g++ -E  main.cpp -o main.i
```
2. 编译：对预处理完成的文件进行一系列的词法分析，语法分析，语义分析，以及优化后产生相应的汇编代码文件。
```
# -S表示只执行源代码到汇编代码的生成
g++ -S main.i -o main.s
# -c表示只执行到编译，输出目标文件xxx.o
g++ -c main.cpp
```
3. 链接：主要将各个模块之间相互引用的部分都处理好，使得各个模块之间能够正确的衔接。各个模块比如说xxx.o文件和静态库libc.a
    1. 静态链接：对函数库的链接放在编译时期完成的
    2. 动态链接：对库函数的链接放在程序运行时期
##### 2.g++与gcc的区别
1. gcc和g++两者都能编译C和C++代码。但是后缀为c的，gcc将它当作是C程序，而g++将它当作是C++程序；后缀为CPP的，两者都会认为是C++程序。
##### 3.makefile的编写
1. makefile的作用是在工程化开发中对工程项目定义编译规则。使用make命令，整个工程就会自动编译。make命令是一个命令工具，是一个解释makefile中指令的命令工具。
##### 4.目标文件
1. ELF（全称Executable Linkable File）是一种用于二进制文件、可执行文件、目标代码、共享库和核心转储的标准文件格式。ELF标准的目的是为软件开发人员提供一组二进制接口定义，这些接口延伸到多种操作环境中，从而减少重新编码、编译程序的需要。
2. ELF的文件类型:
    1. 可重定位的目标文件：这是由汇编器汇编生成的.o文件
    2. 可执行的目标文件
    3. 可被共享的目标文件：即动态库文件
3. 查看文件属于哪一种ELF文件：使用file命令
![image](9E1E80889D4F48AAB0A8DC4495AD1E27)
4. 读取ELF文件头的内容：使用`readelf -h xxx.o`命令
![image](2FB162A961F94AFC8021B11AD5FA6038)
5. 查看所有section表的内容：使用`readelf -S xxx.o`命令
![image](127FBE527FE949D6A4C1EB4496491E29)
6. 查看ELF文件中指定section text段的内容：`objdump -d -j .text add.o`
7. 查看ELF文件中指定section data段的内容：`objdump -d -j .data add.o`
8. 
