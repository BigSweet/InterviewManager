##kotlin和java相比的优势
可空特性 
扩展方法
activity和recyclerview的adapter不需要findview（后面官方不推荐了）

对于集合类，列表类的操作符语法糖

方法可以当做参数进行传递（回调更加方便）

参数可以设置默认值

数据model自带get和set方法



##kotlin实现静态变量
将变量放在companion object中 const val
如果在java中调用这个变量需要类名.Companion.变量名，可以通过添加@jvmfiled来直接通过类名去点变量名


##kotlin操作符let with run
let操作符，定义一个作用域，在这个作用域里面，it表示自己本身，通常用来进行判空操作
with操作符 定义一个域，这个域中直接就是自己本身，可以访问自己的属性。通过用来做数据赋值  作用域中需要自己判空
run操作符 let和with的结合体，可以操作符前面判空。作用域中表示自己本身。访问自身的方法和属性，返回值为最后一行代码
apply 和run操作符类似，返回值不同，返回的是自己本身

## kotlin内联函数

### inline

调用一个方法是一个压栈和出栈的过程,这个过程是会消耗资源的。获取传递高阶函数作为参数的时候也可以使用内联函数
标记为内联函数就会在编译期间直接执行方法体，不需要压栈和出栈，节省资源，

在高阶函数作为参数传递的时候，在执行的时候，高阶函数会被创建成一个对象，如果放在循环体中，会创建多个对象

比如

```
fun hello(action:()->Unit){
 print("hello!")
 action
}
fun main(){  
	hello{
		print("bye!")
	}
}
===========
fun main(){
val post = object :Function0<Unit>{
		override fun invoke(){
				return print("hello!")
		}
	}
}
hello(post)
```

如果我们标记了hello为内涵函数

那么main方法为

```
fun main(){  
 print("hello!")
 print("bye!")
}
```

### noinline用法

noinline用来局部的，指向性地关闭函数的内联优化，效果如下

![image-20210721102106704](/Users/yanzhe/android/知识整理/kotlinimage/image-20210721102106704.png)

### crossinline

如果我们在main中写了一个return

让内联函数里面的函数类型参数可以被间接的调用，但是标记了crossinline之后，在lambda表达式里面就不能使用return

其实return的是main方法

![image-20210721102538590](/Users/yanzhe/android/知识整理/kotlinimage/image-20210721102538590.png)



![image-20210721103023695](/Users/yanzhe/android/知识整理/kotlinimage/image-20210721103023695.png)
