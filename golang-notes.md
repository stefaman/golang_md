
## function
在有返回值的函数中，不允许将“最终的” return语句包含在if...else...结构中，
否则会编译失败：
闭包绑定的是变量，而不是值或者闭包创建是变量的snapshot，但是可以通过函数传入参数的方式实现变量snapshot，这在循环内引用循环变量的闭包的idom。

## switch
要习惯多用switch.
case 条件表达式不用是整型常量了，可以多个并列，强制fallthrough,抛弃break
switch {} 与多个if...else...的逻辑作用等同，但是更紧凑。
case a,b,c: 的形式比if a || b || c 好看
switch 简单语句; {} //可以声明局部变量

## defer
函数的参数计算和传送是在defer 语句执行时
defer function call afer return valus evaluated and before return, so it can get or modify return values.
Go’s panic mechanism runs the defer red functions before it unwind s the stack, so in defer function can get stack information, like runtime.Stack().
defer函数总会是执行，执行过程中出现panic也是会中断该defer的执行，后续defer函数继续执行

```go

for i := 0; i < 5; i++ {
	defer fun(i int){
		fmt.Println(i)
	}(i)
}
//4 3 2 1 0
```

## panic and recover
panic时先执行defer函数， 如果defer中出现panic, 打印的是此最新的panic
打印的源文件相关信息是编译时确定的“快照”，运行时跟源文件已经没有关系了.
recover()可以获得panic(value), 此机制可以实现，比如：函数清理工作， 从镶嵌很深的代码处panic()抛出用户自定义类型的故障信息， recover()捕获并处理。这样可以避免err层层返回的冒泡，而是从故障出直接panic（err）中断执行。

## 错误处理
可以定义特定数据结构err_st表示错误，在其指针类型上实现error interface, 这样调用方可以检测错误类型`if e, ok :=err.(*err_st); ok && e != nil {}`;而且使用指针，每一个返回的错误都是唯一的不相等的。

## 多重赋值
=右边时 range, a channel or map operation, or a type assertion 的时候才可以省略后面的值，注意type assertion在失败是，如果左边参数只有一个，会panic
多重返回值函数出现在=右边时，左边的变量数量必须必须明确匹配，不用的值也要使用匿名变量`_`配对。
fmt.Println(f()) //可以接受多重返回值，但是多个输入参数时不能接受多重返回值的函数
