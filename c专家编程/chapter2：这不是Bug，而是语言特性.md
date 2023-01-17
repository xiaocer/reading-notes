## 1.多做之过（语言中某些不应该存在的特性）
1. switch语句的case标签后记得使用break语句。
2. 一连串相邻的字符串常量将被自动合并成一个字符串，而不用在书写多行信息时必须在行末加上`\`
```
// 除了最后一个字符串外，其余每个字符串末尾的`\0`字符会被自动删除
printf("hello"
        ",hi\n");
```
3. 定义C函数时，在缺省条件下函数的名字是全局可见的，默认被extern关键字修饰。==这种全局可见行，很容易导致interpositioning，即用户编写和库函数同名的函数并取而代之的行为。== extern关键字和static关键字修饰函数的区别：
```
// 在任何地方均可见
function func1() {}
// 在任何地方均可见
extern function func2() {}
// 在文件之外不可见
static function func3() {}
```
## 2.误做之过（语言中存在不适当的特性）
1. C语言中的符号重载
![image.png](https://upload-images.jianshu.io/upload_images/17728742-567ebbeb17eff243.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 结合性：在表达式中如果有几个优先级相同的操作符，则结合性就起仲裁的作用，由他决定哪个操作符先执行。比如说赋值操作符具有右结合性，则表达式最右边的操作最先执行。
```
int a, b = 1, c = 2;
// a = 2
a = b = c;
```
3. 应当使用`fgets()`而不是gets函数读取输入，因为fgets对读入的字符数设置了一个限制。
##### 3.少做之过（语言应该提供但没有提供的特性）
