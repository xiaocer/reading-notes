## 1.段
1. 目标文件和可执行文件可以有几种不同的格式。在绝大多数实现中都采用了一种称作ELF的格式。（ELF全称Executable and Linking Format，即可执行文件和链接格式）有些系统中，可执行文件也采用COFF格式（COFF全称Common　Object－File　Format）。
2. 所有这些不同格式都有一个共同的概念，那就是段。在Unix中，段表示一个二进制文件相关的内容块。一个段一般包含几个section。section是ELF文件中的最小组织单位。
3. 查看可执行文件中三个段的信息：使用size命令
```
# bss段通常是指用来存放程序中未经初始化的全局变量的一块内存区域
# bss全称block started by symbol（由符号开始的块）
size a.out
text    data     bss     dec     hex filename
1568     620       4    2192     890 a.out
```
4. 编译器和链接器在哪些段写入东西
![image.png](https://upload-images.jianshu.io/upload_images/17728742-e5a2cb1d0353ff04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.操作系统在a.out文件中做的事
1. 可执行文件中的段在内存中如何布局
![image.png](https://upload-images.jianshu.io/upload_images/17728742-e479c039c2fe5bb2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 考虑共享库的虚拟地址空间布局
![image.png](https://upload-images.jianshu.io/upload_images/17728742-bbf6179d4a048560.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.C语言运行时系统在a.out里做的事
C语言如何组织正在运行的程序的数据结构的细节。运行时数据结构有以下几种：堆栈、活动记录、数据、堆等
##### 1.堆栈段
1. 堆栈段实现了一种后进先出的结构。
2. 堆栈段的作用：
    1. 为函数内部声明的局部变量（又称自动变量）提供存储空间
    2. 进行函数调用时，堆栈存储与此相关的一些维护性信息，比如说函数调用地址，这些信息称为栈帧（stack frame）。或者说过程活动记录。
    3. 堆栈可以用作暂时存储区
##### 2.过程活动记录
1. 当每个函数被调用时，都会产生一个过程活动记录。它是一种数据结构，用于支持过程调用（函数调用），记录调用结束后返回调用点所需要的全部信息。
2. 过程活动记录的规范描述
![image.png](https://upload-images.jianshu.io/upload_images/17728742-3282aedd7df59f0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 4.auto和static关键字
##### 1.static关键字的应用场景
1. 一个函数，不能返回一个指向该函数中的局部自动变量的指针，因为函数调用结束时，堆栈空间将被回收。即使通过函数调用接收函数内部指针的值，该指针也会成为悬挂指针。例如如下程序将在运行时产生段错误
```
#include <stdio.h>
char* func() {
    char ptr[] = "test";
    return ptr;
}
int main() {
    char* ptr = func();
    printf("%s\n", ptr);
    return 0;    
}
```
2. 使用static关键字：想返回一个指向在函数内部定义的变量的指针时，将变量声明为static。这样就保证该变量被保存在数据段中而不是堆栈中。该变量的生命周期和程序一样长。
##### 2.auto关键字
1. auto关键字用于修饰函数内部的变量，但是函数内部的数据默认就是auto修斯的。
## 5.setjmp和longjmp函数
1. 两者通过操纵过程活动记录来实现。
##### 1.setjmp
1. 函数原型：`setjmp(jmp_buf j)`表示使用变量j记录现在的位置，例如保存一份程序的计数器和当前的栈顶指针。
2. 函数返回0
##### 2.longjmp函数
1. 函数原型：`longjmp(jmp_buf j, int i)`表示回到j所记录的位置，即恢复setjmp函数记录的值。返回i是使得代码知道他是从这个函数返回的
2. 和goto语句的区别：
    1. goto语句不能跳出C语言中当前的函数
    2. longjmp函数则甚至都可以跳到其他文件中的函数中
3. 两者的简单示例
```
#include <setjmp.h>
jmp_buf buf;

void func() {
    printf("in func()\n");
    longjmp(buf, 1);
    // 以下代码不会被执行
    printf("程序已经跳转！");
}
int main() {
    // 保存了PC和栈顶指针的值，返回0
    if (setjmp(buf)) {
        printf("back in main()\n");
    } else {
        printf("first time through\n");
        func();
    }
    return 0;
}
// first time through
// in func()
// back in main()
```
4. 应用场景：用于错误恢复，只要还没有从函数中返回，一旦发现一个不可恢复的错误，可以将控制转移到主输入循环，从那里重新开始。