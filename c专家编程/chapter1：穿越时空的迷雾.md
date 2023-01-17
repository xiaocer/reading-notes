1. 环境：gcc version 11.3.0
## 1.实参char** 和形参const char**不兼容
1. 在gcc version 11.3.0版本中编译如下程序，将会产生警告信息：`expected ‘const char **’ but argument is of type ‘char **’`
```
#include <stdio.h>

void foo(const char** arg)
{
    printf("%s\n", *arg);
}
int main(int argc, char** argv)
{
    // error
    foo(argv);
    return 0;
}
```
2. 参数传递过程类似于赋值。在赋值操作中，左边指针所指向的类型必须具有右边指针所指向类型的全部限定符。
```
int* ptr1 = &i;
// OK
const int* ptr2 = ptr1;

// 警告warning: initialization discards ‘const’ qualifier from pointer target type
const int* ptr3 = &i;
int* ptr4 = ptr3;
// OK
*ptr4 = 200;
```
## 2.关键字const并不能将变量变成常量
1. 在一个符号前加上const限定符只是表示这个符号不能被赋值。也就是它的值对于这个符号来说是只读的，但是他并不能防止通过程序的内部或者外部来修改这个值。
```
const int i = 100;
int* ptr = &i;
// OK
*ptr = 200;
printf("%d\n", i); // 200
```
## 3.类型转换
1. 一个bug:
```
#include <stdio.h>
#define TOTAL_ELEMENTS (sizeof(array) / sizeof(array[0]))
int array[] = {1, 3, 5};
int main() {
    int d = -1, x;
    unsigned int c = d;
    printf("%u\n", c); // 4294967295
    // 有符号数转化为无符号数进行比较
    if (d <= TOTAL_ELEMENTS - 2) {
    // OK
    // if (d <= (int)TOTAL_ELEMENTS - 2) {
        x = array[d + 1];
    }
    printf("%d\n", x); // 0
    return 0;
}
```