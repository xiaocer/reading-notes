## 1.对齐支持
##### 1.数据对齐
1. offsetof宏：可以查看结构体或者类中成员的偏移方式，来检验数据的对齐方式。
2. 数据对齐示例：
```
class Test {
public:
    char a;
    // int类型的数据必须放在一个能够整除4的地址上
    int b;
};

int main() {
    cout << sizeof(Test) << endl; // 8
    cout << offsetof(Test, a) << endl; // 0
    cout << offsetof(Test, b) << endl; // 4
    return 0;
}
```
3. 在CPP中，每个类型的数据除去长度等属性外，都还有一项“被隐藏”属性，那就是对齐方式。对于每个内置或者自定义类型，都存在一个特定的对齐方式。对齐方式通常是一个整数，他表示的是一个类型的对象存放的内存地址应当满足的条件。
##### 2.C++11中的alignof和alignas
1. alignof：查询自定义类型、内置类型或者变量的对齐方式，alignof获得的也是一个与平台相关的值
```
class Test {
public:
    char a;
    int b;
};

int main() {
    int a;
    long long b;
    char c[1024];
    cout << alignof(Test) << endl;  // 4
    cout << alignof(a) << endl;     // 4
    cout << alignof(b) << endl;     // 8
    cout << alignof(c) << endl;     // 1
    return 0;
}
```
2. alignas：设定对齐方式，alignas可以接受常量表达式，也可以接受类型作为参数。当使用常量表达式作为alignas的操作符的时候，其结果必须以2的自然数幂次作为对齐值。
    1. 示例1
    ```
    // 指定结构体Color以32字节对齐
    struct alignas(alignof(double) * 4) Color {
        double r;
        double g;
        double b;
        double a;
    };
    alignas(double) unsigned char c[sizeof(double)];
    alignas(float) unsigned char d[sizeof(double)];
    cout << alignof(c) << endl; // 8
    cout << alignof(d) << endl; // 4
    ```
3. STL中align函数可以动态的根据指定的对齐方式来调整数据块的位置
```
// 该函数在ptr指向的大小为space的内存中进行对齐方式的调整，
// 将ptr开始的size大小的数据调整为按照align对齐
void*align(size_t __align, size_t __size, void*& __ptr, size_t& __space)
```
4. C++11中类模板aligned_storage（在产生对象的实例时对对齐方式做出一定的保证），aligned_union
## 2.通用属性
##### 1.语言扩展
1. 编译器厂商或者组织为了满足编译器客户的需要，设计出了一系列的语言扩展来扩展语法。扩展语法中比较常见的就是"属性"，属性是对语言中的实体对象（比如函数、变量、类型等）附加一些的额外注解信息，其用来实现一些语言及非语言层面的功能或者实现优化代码等的一种手段、
    1. 不同的编译器有不同的属性语法，比如说对于g++，属性通过GNU的关键字`__attribute__`来声明的
    ```
    __attribute__((属性名))

    ```
    2. 在Windows平台上，比如说_declspec是微软用于指定存储类型的扩展属性关键字。
##### 2.C++11中的通用属性
1. 通用属性语法：`[[属性列表]]`
##### 3.预定义的通用属性
1. c++11预定义的通用属性包括[[noreturn]]和[[carries_dependency]]
2. [[noreturn]]：
    1. 用于标识不会返回的函数的。不会返回的函数在被调用完成后，后续代码不会再被执行。
    2. 用于标识那些不会将控制流返回给原调用函数的函数。比如说终止应用程序的函数，由异常抛出的函数等。==通过这个属性开发人员可以告知编译器某些函数不会将控制流返回给调用函数，这能帮助编译器产生更好的警告信息，同时编译器可以做更多的诸如死代码消除，免除为函数调用者保存一些特定寄存器等代码优化工作、==
2. [[carries_dependency]]

## 3.Unicode支持
##### 1.字符集
1. 在一个标准中能够表示的所有字符的集合称为字符集。ISO/Unicode所定义的字符集为Unicode，在Unicode中每个字符占据一个码位。基于Unicode字符集的编码方式有UTF-8、UTF-16、UTF-32。
    1. UTF-8：采用了1到4个字节的变长编码方式编码Unicode，英文通常使用1字节且与ASCII兼容的，而中文常用3字节进行表示。==UTF-8编码通常较为节约存储空间，因此使用比较广泛。==
    2. UTF-16：定长编码
    3. UTF-32：定长编码
2. 在中文语言地区，还有一些常见的字符集及其编码方式，比如说GB2312、Big5、
    1. GB2312在编码上采用了基于区位码的一种编码方式，采用2字节表示一个中文字符。
##### 2.C++11中的Unicode支持
1. 在C++98中，为了支持Unicode，定义了宽字符的内置类型wchar_t。但是它的宽度由编译器实现决定的，理论上它的长度可以是8位、16位、或者32位。程序员写出的包含wchar_t的代码通常不可移植。
2. c++11中引入一下新的内置数据类型来存储不同编码长度的Unicode数据。
    1. char16_t：用于存储UTF-16编码的Unicode数据
    2. char32_t：用于存储UTF-32编码的Unicode数据
    3. 而对于UTF-8编码的Unicode数据，还是使用8字节宽度的char类型的数组来保存、
3. 在C++11中还定义了一些常量字符串的前缀，在声明常量字符串的时候可以使用。
    1. u8：表示为UTF-8编码
    2. u：表示为UTF-16编码
    3. U：表示为UTF-32编码
4. 对于Unicode编码字符的书写，C++11中还规定了一些简明的方式，即在字符串中使用'\U'加上四个十六进制数编码的Unicode码位或者8个16进制数编码的Unicode码位来标识一个Unicode字符。
```
// 你好
\u4F60 表示你
cout << "\u4F60\u597D" << endl;
```
##### 3.关于Unicode的库支持
1. C11中可以使用一些新增的编码转换函数来完成各种Unicode编码间的转换。mb是multi-byte的缩写，表示多字节字符串。c16和c32是char16和char32的缩写，rt是convert的缩写，表示转换的意思。
    1. mbrtoc16
    2. c16rtomb
    3. mbrtoc32
    4. c32rtomb
2. locale：描述的是一些区域特征。比如说在美国地区采用英文和UTF-8编码，这样的locale可以表示为en_US.UTF-8，而在中国使用简体中文并采用GB2312文字编码的locale则可以表示为zh_CN.GB2312
3. c++11中，使用模板类codecvt完成从当前locale下多字符编码字符串到多种Unicode字符编码转换的facet。这里的多字符编码字符串实际依赖于locale所采用的编码方式。
![image.png](https://upload-images.jianshu.io/upload_images/17728742-782e5b2bc2ff199c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    1. 可以使用has_facet查询该locale在本机器上的支持情况。因为一种locale并不一定支持所有的codecvt的facet
    ```
    #include <locale>
    using namespace std;
    
    int main() {
        // 定义一个locale， 并查询该locale是否支持一些facet
        locale lc("en_US.UTF-8");
        bool can_cvt = has_facet<codecvt<wchar_t, char, mbstate_t>>(lc);
        if (!can_cvt)
            cout << "不支持char-wchar_t facet!" << endl;
        
        can_cvt = has_facet<codecvt<char, char, mbstate_t>>(lc);
        if (!can_cvt)
            cout << "不支持char-char facet!" << endl;
        
        can_cvt = has_facet<codecvt<char16_t, char, mbstate_t>>(lc);
        if (!can_cvt)
            cout << "不支持char-char16_t facet!" << endl;
        
        can_cvt = has_facet<codecvt<char32_t, char, mbstate_t>>(lc);
        if (!can_cvt)
            cout << "不支持char-char32_t facet!" << endl;
        return 0;
    }
    ```
## 4.原生字符串字面量
1. 原生字符串使得用户书写的字符串所见即所得，不再需要如'\t'、'\n'等控制字符来调整字符串中的格式。程序员只需要在字符串前加入前缀R。而声明UTF-8、UTF-16、UTF-32的原生字符串，将其前缀设置为u8R、uR、UR.
2. 简单示例：
```
// 如下所示输出一个原生字符串字面量，\n不会被解释为换行
cout << R"(hello,\nworld)" << endl; // hello,\nworld
```
3. 使用原生字符串，转义字符就不能使用了，这会对使用\u或者\U的方式书写Unicode字符带来影响
```
 // 你
cout << "\u4F60" << endl;
// \u4F60
cout << u8R"(\u4F60)" << endl;
// 7(加上末尾的空字符)
cout << sizeof(u8R"(\u4F60)") << endl;
```
4. 原生字符串字面量像C的字符串字面量一样遵从连接规则、
```
char u8String[] = u8R"(你好)" "hi";
// 你好hi
cout << u8String << endl;
// 9
cout << sizeof(u8String) << endl;
```

