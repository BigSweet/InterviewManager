jni是java调用C/C++的手段
ndk是通过C/C++实现特定功能的手段，类似于sdk是一个开发工具包

C/C++编译过程
1，预处理 gcc -E main.c -o main.i
预处理阶段主要处理include和define等。它把#include包含进来的.h 文件插入到#include所在的位置，把源程序中使用到的用#define定义的宏用实际的字符串代替

2，编译  gcc -S main.i -o main.s
编译阶段，编译器检查代码的规范性、语法错误等，检查无误后，编译器把代码翻译成汇编语言

3，汇编  gcc -c main.s -o main.o
汇编阶段把 .s文件翻译成二进制机器指令文件.o,这个阶段接收.c, .i, .s的文件都没有问题

4，链接  gcc main.o -o main.exe
链接阶段，链接的是其余的函数库，比如我们自己编写的c/c++文件中用到了三方的函数库，在连接阶段就需要连接三方函数库，如果连接不到就会报错

cmakelist文档
https://www.zybuluo.com/khan-lau/note/254724

makefile文档
https://quanzhuo.github.io/2016/06/06/Makefile/

指针是一个特殊的变量，它指向另外一个变量的内存地址，同时它具有自己的内存地址。
在32位的系统上面，所有的指针变量的字节大小都是4，如果是64位系统，那指针变量的大小都是8

定义一个指针会在栈中为他分配地址
malloc方法是在堆中分配地址
calloc也是分配地址但是会初始化都为0
realloc 重新分配地址
free 释放地址，释放内存

指针的值去进行交换不会有效果，需要通过指针去操作才能修改真正的值
数组*(arr+3)=arr[3]
*取值 &取地址


