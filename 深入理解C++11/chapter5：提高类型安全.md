##### 1.强类型枚举
1. 枚举：在cpp11标准以前，枚举值常和整形数值对应。缺点如下：
    1. ==枚举类型的名称和枚举的成员的名称都是全局可见的，污染全局作用域，容易造成名称的冲突。==
    ```
    enum Gender {Male, Female};
    int main() {
        int i = 0, j = 1;
        cout << (i == Male) << (j == Female) << endl; // 11
        return 0;
    }
    ```
    2. 枚举类型所占用的空间大小也是一个不确定量
    ```
    enum A {
        A1 = 1,
        A2 = 0xFFFFFFF0U
    };
    enum B {
        B1 = 1,
        B2 = 0xFFFFFFFFFLL
    };
    
    int main() {
        cout << sizeof(A2) << endl; // 4
        cout << sizeof(B2) << endl; // 8
        return 0;
    }
    ```
2. 强类型枚举：cpp11引入的一种新的枚举类型，又称强类型枚举。声明强类型枚举，只需要在enum后加上关键字class或者struct。强类型枚举的优势：
    1. 强作用域，强类型枚举成员的名称不会被输出到其父作用域空间
    2. 转换限制，强类型枚举成员的值不可以与整形隐式的相互转换
    ```
    enum class Gender {Male, Female};

    int main() {
        // 使用强类型枚举成员需要指定强类型名称
        Gender gender = Gender::Male;
        // error, 强类型枚举成员不能与整形进行隐式转换
        // cout << (gender > 0) << endl;
        // ok, 将强类型枚举显式转换为整形
        cout << ((int)gender == 0) << endl; // 1
        return 0;
    }
    ```
    3. 可以指定底层类型。强类型枚举默认的底层类型为int,但是可以显式的指定底层类型，具体方法在枚举名称后面加上`:数据类型`，其中数据类型可以是除了wchar_t以外的任何整形。
    ```
    enum class A : char{A1 = 1, A2 = 2};
    enum class B : unsigned int{B1 = 1, B2 = 2};
    
    int main() {
        // 1
        cout << sizeof(A) << endl;
        // 1
        cout << sizeof(A::A1) << endl;
        // 4
        cout << sizeof(B) << endl;
        // 4
        cout << sizeof(B::B1) << endl;
        return 0;
    }
    ```
3. 不管是枚举还是强类型枚举，其成员没有公有私有之分。
##### 2.堆内存管理
1. 显式内存管理：以下问题需要注意
    1. 野指针：一些内存单元已经被释放，之前指向它的指针却还在使用。已经释放的内存有可能被运行时系统重新分配给程序使用，从而导致无法预测的错误。
    2. 重复释放：程序试图去释放已经被释放过的内存单元或者释放已经被重新分配过的内存单元，就会导致重复释放错误。
    3. 内存泄漏：不再需要使用的内存单元如果没有被释放就会导致内存泄露。如果程序不断重复进行这类操作，将会导致内存占用剧增。
2. CPP11中的智能指针
    1. cpp98中，智能指针通过类模板auto_ptr来实现。其缺点是拷贝时返回一个左值，不能调用delete[]，cpp11标准已经废弃它，而是改用unique_ptr，shared_ptr，weak_ptr等智能指针。
    2. 
    ```
    // 不能复制的unique_ptr
    unique_ptr<int> up1(new int(100));
    // error，拷贝构造函数和拷贝赋值运算符都被删除了
    // unique_ptr<int> up2(up1);
    unique_ptr<int> up3 = move(up1);
    // 空指针，0
    cout << up1.get() << endl;
    // 显式释放内存
    up3.reset();

    shared_ptr<int> sp1(new int(200));
    // sp1和sp2指向同一块堆内存
    shared_ptr<int> sp2 = sp1;
    sp1.reset();
    // 200,还存在sp2指向着堆内存，所以堆内存没有被释放
    cout << *sp2 << endl;
    ```
    3. weak_ptr：它可以指向shared_ptr指针指向的对象内存，却并不拥有该内存，使用weak_ptr的成员函数lock，则可以返回其指向堆内存的一个shared_ptr对象，如果堆内存无效则返回空指针。该类并不会影响堆内存的引用计数，可以用来验证shared_ptr指针的有效性。==一般用来解决shared_ptr智能指针中环形引用的问题。==
3. 垃圾回收的方式
    1. 基于引用计数的垃圾回收器：记录堆上对象被引用的次数，当对象被引用的次数变为0时，该对象可以当作垃圾而回收。==缺点：难处理环形引用的问题==
    2. 基于跟踪处理的垃圾回收器：
        1. 标记清除算法
        2. 标记整理
        3. 标记拷贝
4. cpp11与最小垃圾回收支持：CPP11中定义了安全派生的指针，安全派生的指针是指向由new分配的对象或者其子对象的指针。安全派生指针的操作包括如下所示。==在cpp11的规则中，最小垃圾回收支持是基于安全派生指针这个概念的==
    1. 在解引用基础上的引用，例如`&*p`
    2. 定义明确的指针操作：比如`p + 1`
    3. 定义明确的指针转换：`static_cast<void*>(p)`
    4. 指针和整形之间的reinterpret_cast
