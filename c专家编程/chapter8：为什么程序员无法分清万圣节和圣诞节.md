## 强制类型转换
1. 已知一个void指针，并且事实上该指针指向一个函数。则使用类型转换调用该函数示例如下：
```
extern int printf(const char*, ...);
void* func = (void*) printf;

int main() {
    (*(int (*)(const char*, ...))func)("Hello, World!");
    return 0;
}
```