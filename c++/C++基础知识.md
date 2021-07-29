## 基础类型

```c++
short
long 
int 
float
double 
char  字符类型
void 无类型一般用来表示指针
const 常量
指针就是内存地址
void* char*
数组
char c[2]
int arr[10]
printf
%c 字符
%d 数字
%p 地址
```

```c++
  int *a;
  a = (int *) malloc(sizeof(int));
  *a = 10;
  printf("a=%p,%p,%d\n", &a, a, *a);
	//a=0x7ffeebacf720,0x7fd7554059d0,10
  定义了一个int指针，给a分配一个内存空间大小为int大小
  &a 指针的地址
  a a的地址
  *a a的值
    
   int c[3] = {0, 1, 2};
   printf("arr=%p,%p,%d,%d,%d\n", &c, c, c[0], c[1], c[2]);
	//arr=0x7ffeebacf73c,0x7ffeebacf73c,0,1,2
```



## 结构体

```c++
struct st {
    int a;
    int b;
};
//main中
 struct st aas;
 aas.a = 10;
 aas.b = 20;
 printf("struct content = %d,%d\n", aas.a,aas.b);
//struct content = 10,20
```

## 枚举

```c++
enum em {
    RED = 0,
    GREEN,
    BLACK
};
//main中
 enum em colorEm;
 colorEm = BLACK;
 printf("the color is %d\n", colorEm);
	//the color is 2
```



## 算数运算

```c++
+ , - , * , / , %
  //if else 
 if (aas.a > aas.b) {

    } else {

    }

//for 循环
    for (int i = 0; i < 10; ++i) {
        printf("for for %d\n", i);
    }
//while循环
    while (1) {
        printf("while %d\n", 1);
        break;
    }
```



## 函数

```c++
int addC(int a) {
    return a + 10;
}
 printf("function %d\n", addC(1));
 //function 11
```



## 文件操作

```c++
 FILE *file; //新建文件
 char buf[1024] = {0,};
    file = fopen("1.txt", "a+"); //打开没有，没有就新建 a 表示append
    fwrite("hello world!", 1, 12, file); //向文件中写入数据
    rewind(file);//游标放在文件的开头
 		fread(buf, 1, 26, file);//读取文件
    fclose(file); //关闭文件
    printf("buff %s\n", buf);
//buff hello world!hello world!
```



## 指针

```c++
指针就是内存地址
栈空间不需要我们管理，系统自动管理 大小为4M？
堆空间
void* man = malloc(size)//堆空间分配
free(man)//释放内存

函数指针
int func(int x )//这是一个函数
int (*f)(int x);//这个是函数指针
f = func //将func函数的首地址赋给指针f 
//////示例
int addC(int a) {
    return a + 10;
}

int main() {
    int (*f)(int);
    f = addC;
    int result = f(5);
    printf("func zhi zhen %d", result);
    return 0;
}
//func zhi zhen 15
```



## 定义自己的库



```
clang -g -c add 
```



# namespace命名空间
- C++命名空间基本概念
- C++命名空间定义，使用语法，意义
C++namespace 命名空间，里面定义了标识符的可见范围
可以自己定义命名空间 给命名空间添加属性 有点像是java的包的概念 
namespace::a来使用命名空间的属性 也可以使用 usenamespace来导入

# 新增Bool类型关键字
bool类型用来表示true和false
c++中可以用1表示true，可以用0表示false

#  C/C++中的const
- const基础知识
const 常量 常量指针不能修改指向的内存空间 只能修改指向 const int* b = 1
指针常量不能修改指向可以修改地址  int* const a=1

- const和#define相同之处
都是表示常量
- const和#define的区别
#define是宏定义 #undef 是卸载这个定义
#define编译预处理阶段 简单的文本替换
const 是编译器处理 提供类型检查和作用域检查


# 引用
取地址符&就是C++中的引用，如果在C++中用到了引用，就不能在使用C的语法
int &b = a b就是a的引用
引用一定要初始化
可以通过引用去操作引用指向的变量
引用没有自己的存储空间 它只是一个别名
引用其实就是一个指针常量
不能返回局部变量 因为栈区的变量会被系统回收

如果是堆区没有释放的话就会内存溢出 

# C++对C的函数扩展
- inline内联函数
C++的内链函数必须写在一起，C++的内联函数和kotlin的内联函数一样，都是减少方法的入栈和出栈

- 默认参数
类似于kotlin的默认参数 在方法区设置int i= 1 当不传参数的时候参数就是1
- 函数占位参数
只需要申明参数类型不需要具体的形参值 

- 函数重载
 方法名相同，参数不同（个数不同或者类型不同）

- 函数指针
typedef void (*func)(int a,int b)
通过给func函数指针赋值，去调用真正的方法

# 类和对象
当在头文件中定义了一个方法时候，在cpp文件实现，需要中类名：：方法名才能去实现它

#对象的构造和析构
构造函数一般用来做初始化的操作(test())，析构函数用来做资源释放的操作（~test()）

#构造函数的分类及调用
默认会有一个构造函数
比如新建一个类 test t1;
如果是有参数的构造函数 那就是test t1(a,b)

- 拷贝构造函数调用时机
初始化一个函数之后 用=号在赋值给一个未初始化的函数 就会调用这个函数的拷贝函数 

深拷贝就是重新为区域生成一块区域 防止在析构函数释放资源的时候 出错


- 对象初始化列表
classname：：m1(xxx) 初始化 classname类中的M1类的有参构造函数
初始化对象的顺序  是按照成员变量的声明 来决定的
常变量必须用初始化列表去初始化


#对象的动态建立和释放
- new和delete基本语法
在栈上面创建的对象会自动释放，大小是固定的 ，
在堆上面创建的对象需要手动释放 大小是可以动态扩建的

C++在堆上面申请内存可以通过下面俩种方式
new/delete
new [] /delete[]
new的时候会自动调用构造函数 delete的时候会自动调用析构函数

# 友元
在一个类中申明一个友元函数，这个函数可以访问这个类中的私有的成员变量
在一个类中申明一个友元类，这个类中可以直接访问另外一个类中的私有变量

#运算符重载
使用operator符号来对运算符进行重载
首先确定函数调用比如我们要重载[] 运算符那就是operator[]()
第二步确定参数 是基础变量还是数组 operator[](int i)
第三步确定返回值 int& operator[](int i)
## 继承和多态
struct 继承默认是public  class继承默认是private

## 纯虚函数和抽象类
虚函数才能产生多肽

当一个类中有一个纯虚函数，那这个类就是一个抽象类


























