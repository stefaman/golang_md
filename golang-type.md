
## nil and empty and zerovalue
引用类型,接口类型的zerovalue是T(nil)
var v T; // v is zerovalue
复合类型empty value T{}虽然当前没有存储内容，但不是nil;
单独一个`nil`是没有类型的，可以作为函数参数，或者 == nil, 如果需要操作，需要转换为具体类型的nil

## 接口类型
注意区分nil， 和 dyanminc value is nil; nil 接口值调用方法时painic，因为没有reciever;而后者可以，结果是具体类型而定，当然可能是panic or abuse.
如果类型T bound T.method, 那T（nil) 也是该方法的合法reciever。 当然，nil是否有意义的值，如通常代表空集合，要看具体函数。最好在函数documentation comment体现出来
需要注意函数参数为nil的情况。
```go
var w io.Writer // w is nil
// w.Write([]byte("a")) //panic: runtime error: invalid memory address or nil pointer dereference
var f *os.File                    // f is a nil pointer
w = f                             // w is not nil, by dynamic value is nil
//same as f.Write(...), not panice
fmt.Println(w.Write([]byte("a"))) //return:0   invalid argument
```

## map
`var v, ok T = a[x]`, 是`var v, ok= a[x]`冗余的写法，ok 依然是bool, called untyped bool
map 类型的nil可以读[],得到zero value; 但是delete, or m[k] = v操作nil 会panic;
map 类型empty value m{} ; 可以delete（虽然没鸟用）, 和扩展赋值 m[k] = v
**注意** 当v是slice 类型时，m[k][i] = x 这样的操作是bug, 需要 `if m[k] != nil {}`

## slice and array
nil 和 []T{}，他们都不可以[], 得到panic; 可以s = append(nil, x)；
x=s[0:0] 虽然也是empty(len(x) == 0),但是cap(x)=cap(s)
```go
$: go doc reflect.SliceHeader
type SliceHeader struct {
        Data uintptr
        Len  int
        Cap  int
}
```
**注意** 虽然名义上slice是引用类型，不可用==，但内部实现是一个"slice head", struct{len,cap,ptr};是值copy的，创建slice后就不能更改；
```go
type Values map[string][]string
var v Values // v is nil
Values(nil)["any_key"] is a nil slice
Values(nil)["any_key"] = nil //error
Value{}["any_key"] is a nil slice
Value{}["any_key"] = []string{"", ""} //ok
```

数组是值类型，slice是引用类型
make([]int, 0, 10) //len is 0, 预先分配空间，提高append()效率
make([]int 10) //len is 10, value is zero
切片是数组的一个view,像是JS的array buffer;切片基于数组或者旧切片构造。
切片类型的len and cap：外部看见和可用的元素数量是len(s), 而cap(s)关注点有两个：\
1）append时候超过了cap内部会copy内存，某些已经容量的情境下可以先分配给slice足够的cap；
2）基于切片创建新的切片时，两种基于同一个数组，cap相等，新切片的len只受制于cap,可以比旧切片的len大，如oldSlice[:cap(oldSlice)]
合理地设置存储能力的值，可以大幅降低数组切片内部重新分配内存和搬送内存块的频率，从而大大提高程序性能。
基于数组或切片构造切片时，可以指定cap大小
a[low:high:max] 0<=low<=high<=max<=cap(a) //新切片的cap = max - low,

### append
唯一合法的操作 s= append(s, x)
nothing happen in: append(slice, nil...) or append(slice,) or append(slice)

## string
不可用修改 unmodifiable，内部是byte slice；
fmt 按照UTF-8格式处理，
[i]， len 都是byte为单位，
range 操作返回值， index是utf-8编码单元的第一个byte index, 返回值是以rune为单位，错误编码返回'\ufffd'，index只跳过一个byte;（通过index的增长区分错误编码和真的是'\ufffd'）
[]byte []rune string 三种类型可以相互转换
string 是{len, ptr}表示的byte slice, reflect.StringHeader是runtime
```go
$: go doc reflect.StringHeadr
type StringHeader struct {
	Data uintptr
	Len  int
}
```
str[low:high], 0<=low<=high<=len(str);返回子字符串；slice 操作生成子字符串是两个word的消耗，因为字符串不可修改，所以可以共享内存。

## constant
literal constant 的类型编译器会灵活选择，自由度比较高。特别算术类型基本上都是通用的，比如一个named type 的underlying type is arithmetic type, then literal arithmetic constant can assign to it.
Go语言的字面常量更接近我们自然语言中的常量概念，它是无类型的。只要这个常量 **在相应类型的值域范围内**，就可以作为该类型的常量，否则编译报错。
比如常量-12，它可以赋值给int、 uint、 int32、int64、 float32、 float64、 complex64、 complex128等类型的变量。
命名常量可以不用指明类型，让编译器在需要时候自动转换类型。
指定类型的常量只能表示给类型值域内的值，而且赋值给其他类型是需要明确conversion。

## named type
named type 跟原有类型不一样的类型,但是他们的underly type都是一样的。

因为跟old type 具备一样的underlying type, 原有的operation依然可以保留，比如：算术运算，T{}实现的购置方式，
命名有几点用途：
- 强调是不同的类型，如 rune, byte
- 消除old type的旧方法，可以添加新方法

不能对*T， interface type 命名一个新类型。

## type indentity
表征一个类型
named type是单独的类型，underlying type是不一样的类型，
两个named type必须idtentifier，underlying type一样才是indentical

结构类型的identical比较，看顺序，tags
函数类型不看参数名字，只看参数类型和数量

## Assignability
算术类型的named type可以直接使用literal 赋值。
rune 和 int32 可以互相`=`, 但是最好还是明确进行T()
named type和 underlying type 之间可以相互赋值
A value x is assignable to a variable of type T ("x is assignable to T") in any of these cases:
>>
x's type is identical to T.
x's type V and T have identical underlying types and at least one of V or T is not a named type.
T is an interface type and x implements T.
x is a bidirectional channel value, T is a channel type, x's type V and T have identical element types, and at least one of V or T is not a named type.
x is the predeclared identifier nil and T is a pointer, function, slice, map, channel, or interface type.
x is an untyped constant representable by a value of type T.

## conversion
byte(整数) 可以实现取整数低8位
浮点数的转换规范没有指明round的具体规则；但是constant float number 规定了Round to the nearest representable constant if unable to represent a floating-point or complex constant due to limits on precision.
算术类型转换overflow会成功，结果implement defined，貌似是二进制暴力转换；
字符串和slice的转换需要考虑运行时性能消耗
A constant value x can be converted to type T in any of these cases:
>>
x is representable by a value of type T.
x is a floating-point constant, T is a floating-point type, and x is representable by a value of type T after rounding using IEEE 754 round-to-even rules, but with an IEEE -0.0 further rounded to an unsigned 0.0. The constant T(x) is the rounded value.
x is an integer constant and T is a string type. The same rule as for non-constant x applies in this case.

A non-constant value x can be converted to type T in any of these cases:
整数可以直接转换为字符串，string(0x8521) == "蔡"
>>
x is assignable to T.
ignoring struct tags (see below), x's type and T have identical underlying types.
ignoring struct tags (see below), x's type and T are unnamed pointer types and their pointer base types have identical underlying types.
x's type and T are both integer or floating point types.
x's type and T are both complex types.
x is an integer or a slice of bytes or runes and T is a string type.
x is a string and T is a slice of bytes or runes.

### 指针
结构类型值的成员取值`s.a`，数组的index取值`a[1]`, range, [:],前面的值可以是指针类型，会自动完成转换。但是slice类型不行,不能对指针类型值取下标。
函数返回类型指针的一层好处是：返回值都不相等，相等类型也不相等了
**循环引用可以通过编译**
fmt.Print 只显示指针的地址，可以处理循环指针和结构，但是循环的slice or map 一样可以让其stuck....
```go
type P *P
var p P
p = &p

// a map that contains itself
type M map[string]M
m := make(M)
m[""] = m

// a slice that contains itself
type S []S
s := make(S, 1)
s[0] = s

// a linked list that eats its own tail
type Cycle struct {
	Value int
	Tail  *Cycle
}
var c Cycle
c = Cycle{42, &c}
```
## composition literal
复合类型字面量 T{...}，T is named type, or struct{...}, [n]T, []T, map[kT]vT,
{...}, 如果最后不是紧靠着‘}’,必须追加tailing common尾部逗号“,”;
T{}: T是数组和结构时表示zerovalue，同 var x T一样；但是slice,map 表示的是empty value, 不是zeroe value nil。mapT{} 等于 make(mapT), 可以x[]=v;
T{} 可以 "&", "." or "[]"操作
```go
[...]int{1,2,3}, or [3]{1,2,3} //
[...]rune{2: 'a','b', 'c', 8: 'k'}
type S struct{
	a int
	b bool
}
S{a:3, b: true} //忽略成员时zerovalue
S{2, true} //顺序一致，全部列出，不可忽略

type M map[S]bool
M{
	S{2, false}: false,
	S{a:3, b:true}: true,
}

//avoid ambiguity, 否则第一个 { 会当作block的开始
if x == (T{a,b,c}[i]) { … }
if (x == T{a,b,c}[i]) { … }
```
### 简写表示
slice, array, map 等复合字面量，如果成员也是复合字面量的类型，那么可以忽略成员重复的类型标示；而且指针复合类型不用明确&操作。注意结构类型没有简略字面量表示法。
```go
[...]Point{{1.5, -3.5}, {0, 0}}     // same as [...]Point{Point{1.5, -3.5}, Point{0, 0}}
[][]int{{1, 2, 3}, {4, 5}}          // same as [][]int{[]int{1, 2, 3}, []int{4, 5}}
[][]Point{{{0, 1}, {1, 2}}}         // same as [][]Point{[]Point{Point{0, 1}, Point{1, 2}}}
map[string]Point{"orig": {0, 0}}    // same as map[string]Point{"orig": Point{0, 0}}
map[Point]string{{0, 0}: "orig"}    // same as map[Point]string{Point{0, 0}: "orig"}

type PPoint *Point
[2]*Point{{1.5, -3.5}, {}}          // same as [2]*Point{&Point{1.5, -3.5}, &Point{}}
[2]PPoint{{1.5, -3.5}, {}}          // same as [2]PPoint{PPoint(&Point{1.5, -3.5}), PPoint(&Point{})}
```

## method
T 必须是nemed type, and underlying type can not be pointer or interface

type T Oldtype
T将失去所有绑定在Oldtype的方法，只有结构的export field 仍是可见的. 如果原来类型是组合类型，可以“继承”其成员，方法不能“继承”；如果旧类型内嵌了一个匿名接口类型（或绑定了方法的）成员，因为“继承”了成员，自然新类型N.m可以调用O.m方法。
```go
type Com struct {
	io.Writer
}
type C Com
c := C{Writer: os.Stdout}
c.Write([]byte("蔡"))
```

变量或者值v 有类型T， p 有类型*T
func (t T) Func() {}
`func (p *T) Pfunc() {}`
类型T上定义了method Func, v is a receiver of `T.Func`
或者类型*T 上定义了method Pfunc, p is receiver  of `(*T).Pfunc`

一个方法signature可以选择绑定在T or `*t`上， 绑定在指针类型可以1）避免vaue copy消耗;2）修改reciver；
通常，即使不修改reciever, 大多数情况下都是绑定在指针类型上。但是如果reciver 本身是slice, map等引用类型，就绑定在T上，省去指针操作的麻烦。

在选择方法时， 编译器会自动在T 和 `*T` 间转换。但是这里有微妙的区别。
1. 只有T变量才能&，才能获得`(*T).Pfunc`，rhv是不行的；而*T值就可以获得`T.Func`
T类型 **变量**（只有变量可以取地址&） 可以调用*T绑定的方法，`*T`类型 **值** (指针值就可以解引用*)
2. 类型T实现`T.Func`，那么*T也同时自动实现`（*T）.Func`。但反过来，不成立。实现了`(*T).Pfunc`，那么T不会自动实现。这样理解：编译器可以自动生成函数`（*T）Func{*T...}`;但是反过来不行，`(*T).Pfunc`(可能)会修改参数，所以编译器无法自动生成一个`(T).Pfunc{&T...}`，这样只能修改参数，而不是外部传入的那个变量，是没有效果。

### method value
在函数式编程时可以传递闭包
v.Func or p.Pfunc 是一个绑定了reciever 的函数，一个closure
signature is the same as method

### method expression
T.Func or `(*T).Pfunc` 是一个函数，
signature is `fucn(T, ...)` or `func(*T, ...)`, 是原方法函数的扩展，第一个参数是receiver

## struct
成员tag 标签只能在reflect中获得
composition or embeded 不是JS's prototype, 相当于一种语法便利
但是要注意这不是其他语言的class 继承 或者原型的意义，outer.x 只是 outer.Iner.x的简写
anonymous field 的写法相当于内层Iner结构的field or method可以使用outer.f这样的方式引用，但就是outer.Inner.f的语法糖.

小写的成员只能在包内访问，这是GO的“私有成员”
包外成员访问链条上每个环节必须都是exported field;也就是field name都是大写字母开头的。

### method and struct
outer.f()只是outer.Iner.f()的简写，成员方法的receiver 仍然是原来的类型，即是 outer.Iner。被组合的方法不可能获得外部结构的成员。
**注意**  包内内嵌匿名不导出结构的大写方法，是作为可导出方法，可以被导出的包外
```go
package pack1
type Test struct {
	us
}
func (us) Ufunc() {
	fmt.Println("Ufunc called")
}

package pack2
import "pack1"
type S struct {
	pack.T
}
s := S{}
s.Ufunc() //Ufunc called
s.Test.Ufunc() //Ufunc called
s.Test.us.Ufunc() //compile error
```

### 指针成员 指针方法
type S struct {T}; var s S
内嵌成员绑定的`T.Func`，`(*T).Pfunc`都可以调用，可以s.F, s.P, (&s).F, （&s).P
内嵌成员可以是指针类型;
`type SP struct {*T}`; var sp SP{&T{}}
内嵌成员绑定的`T.Func`，`(*T).Pfunc`都可以调用，可以sp.F, sp.P, (&sp).F, （&sp).P
需要注意指针的初始化，避免nil指针引用错误。
需要注意只有只有变量才可以调用绑定在指针类型上的方法。
内嵌类型不能是pointer to interface

## 名字冲突
内层同一rank结构的成员或者方法的有同名冲突时，不能使用outer.x的简写方法，否则会编译报错。但是仍然可以使用传统链式成员引用方法。
**注意**， 不论匿名成员方法函数是`T.Func`，`(*T).Pfunc`，成员是否指针类型，都算入同一个名字空间。
**注意**，如果两个内嵌成员，一个是包外，一个是包内，那么他们都共同的小写成员，这样不冲突。或者都是包外，也可以有共同的小写成员名字。
```go
type T struct {
	r int
}
type S struct {
	time.Timer //has a Timer.r
	T
}
s := S{r:1}
```
## 匿名结构调用方法
匿名结构类型通过内嵌具备了方法的成员，也就可以使用方法了。这语法糖妙用。
Methods can be declared only on named types (like Point) and pointers to them (`*Point`), but thanks to embedding, it’s possible and sometimes useful for unnamed struct types to have methods too.
```go
var cache = struct {
sync.Mutex
mapping map[string]string
} {
mapping: make(map[string]string),
}

func Lookup(key string) string {
cache.Lock()
v := cache.mapping[key]
cache.Unlock()
return v
}
```

## encapsulation
hiding information
API compatibility
opaque or encapsulation

## interface
https://research.swtch.com/interfaces
generalization or abstraction
generality
satisfied implicitly

interface as contract
This freedom to substitute one type for another that satisfies the same interface is called substitutability, and is a hallmark of object-oriented programming

一个接口类型是方法的结合， 一个接口类型value是{接口类型，dynamic value, dynamic type}三个部分组成的。
一个type 只有持有interface 所有的method,那就说此类型satisfy 此接口，可以赋值
一个类型可以满足很多接口，所有类型都是满足空接口interface{}
一个类型值赋值给满足的某个接口类型，那么其余特性都被隐藏, 不可见了。也就说一个接口类型 conceals origin type's other information, except the methods that interface possess
```go
var w io.Writer
w = os.Stdout // os.Stdout 同时满足Writer and Closer 两种接口
os.Stdout.Close() // ok
w.Close() // error
```

接口类型作为函数参数或者返回值，实现了generalization, interface{}尤甚
empty inteface, 任何值均可以赋值给 type `interfance{}`, 可用于函数paramet 类型声明为`interface{}`

concrete type 的方法是绑定在 T or `*T`上的， 所以只有绑定的类型满足接口，不会有T 和 `*T`的自动转换， 如 `.Func()`那样；一个类型T在T和*T上分别实现了接口F的部分方法，那么*T类型满足接口，T类型不满足接口；如container/heap的实现

可以使用下面的代码进行编译时assert
```go
// *bytes.Buffer must satisfy io.Writer
var _ io.Writer = (*bytes.Buffer)(nil)
```
运行时检查某个类型是否实现某接口类型，除了使用reflect外，可以
```go
var _, ok = (interface{})((*bytes.Buffer)(nil)).(io.Writer)
```

因为golang的类型声明时不用明确声明满足的接口，可以新创建某种接口类型，使得现有的具备某种共性方法concrete type都满足接口类型，而不用修改原有的类型API。这种便利性在操作现有代码库类型时特别方便。

interface value 包含了两个部分，concrete type and it's value，称为dynamic type and dynamic value。
在运行时通过赋值满足该接口的具体类型，implicit conversion, 也可以explicitly conversion,
接口类型初始化或者=nil时，dynamic type is nil and dynamic value is nil.
当interface value't dynamic value is nil of a conrete type, then it's not nil.
```go
var b *bufio.Buffer // b == nil
var w io.Writer = b //w dynamic type is *bufio.Buffer, dynamic value is nil
w != nil // true
w.Write() // panic 此例子中，(*bufio.Buffer).Write() receiver, nil is invalid
```
所以编译器需要动态的分配函数调用地址，dynamic dispatch
fmt "%T" 可以显示接口类型的dynamic type.

接口类型的可以== or ！=， 如果都是nil or 动态类型相同，执行该类型的比较。注意：如果两个接口类型的动态类型相同，但是不能比较（slice, map, function), 会panic。这点确实蛋疼，因为动态类型是运行动态的。

接口类型的区别在于method set, 特殊情况，为了避免ambiguity， 可以增加一个注释性的方法
```go
type Fooer interface {
    Foo()
    ImplementsFooer()
}
//这样避免歧义
type Bar struct{}
func (b Bar) ImplementsFooer() {}
func (b Bar) Foo() {}
```

### 函数 adapter
因为只有named type才能实现方法，也就才能满足接口类型，所以如果某个函数的signature 跟接口唯一的方法一致，或者为了修饰函数使其满足接口要求；
可以通过构建一个adapter 类型 T， T(f)类型转换后满足接口类型
```go
net/http
package http
type HandlerFunc func(w ResponseWriter, r *Request)
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
f(w, r)
//这里还可以做很多事情
}
//HandlerFUnc 实现了接口Handler {ServeHTTP(w ResponseWriter, r *Request)}
http.Handle("/", HandlerFUnc(f))
```
web server: 函数式编程+接口类型+channel+concurrent，可以深入研究

### type assertion
value, ok := v.(T) // v is a value of interface type I,
如果没有ok,单个返回值时而fail时，直接panic;
失败时返回nil, false
if v is nil ,always fail
if T is conrete type, test if v's dynamic type is T, if so return v's conrete value;the effect is extract v's dynamic value.可以应用于：动态类型的识别，如不同类型error的识别
if T is interface type, test if v's dynamic type C satisfies T, if so return T(v's dynamic value), same as T(v.(C)). the effect is type conversion, usually interface type T has difference methods with I.可以应用于：动态类型额外方法的获取,如io.Writer 变量是否含有WriteString() method. interface{}动态类型的识别。有时候需要临时构造一种接口类型T用于识别动态类型是否包含某方法。

接口有两个不同的侧重点：
1. 接口是抽象、封装、信息隐藏、contract,如io.Writer,http.Handler,
强调的是接口类型的方法，具体类型的共性和一般化，不关心具体类型的细节和实现

2. 接口作为各种具体类型的并集， discriminated unions, 最典型是空接口interface{}，
强调了具体类型的差异，需要去识别各种不同的动态（具体）类型，针对性的进行操作，典型如函数接口类型参数的动态类型判断

## type switch
识别接口类型的动态类型，case 可以混用具体类型和接口类型;
optional simple statement;
not fallthrough;
```go
func sqlQuote(x interface{}) string {
	switch x := x.(type) {//<可选的>赋值给x,省去case执行语句每一步都要去 type assertion
		case nil:
			return "NULL"
		case int, uint:
			return fmt.Sprintf("%d", x) // x has type interface{} here.
		case bool:
			if x {
					return "TRUE"
			}
			return "FALSE"
		case string:
			return sqlQuoteString(x) // (not shown)
		default:
			panic(fmt.Sprintf("unexpected type %T: %v", x, x))
	}
}
```
## 接口类型总结
变量或者值都是有类型的，分为{interface type, concrete type}
但一个变量是接口类型时，只能对他调用接口类型规定的method，接口变量的zero value is nil, nil接口变量依然可以调用对应的method,但是大多数接口类型这么做会导致panic; nil接口变量没有对应的dynamic type and value.

一个任意concrete type value 可以赋值给任意接口类型变量，只要满足接口要求；一个接口类型的value可以赋值给其他接口类型的变量，只要其满足该接口类型(也就是说，后者是前者的子集)；该接口变量的dynamic type and dynamic value is that concrete type value.
interface{} type has no method，可以容纳任何具体类型, 和任何接口类型。可以利用空接口类型编写函数泛型参数，在函数内部重新获得其动态类型信息。
使用泛型参数时，对接口类型变量的窥视可以使用type assertion or type switch。但是这处理方法要求编程时先知道所有相关的类型信息，如果是未知的用户定义了类型或者复合类型，就没有办法了。refelect派上用场了。
类型断言在must场合，除了类型转换，还启动类型检查作用，失败则panic.在试探的场合，使用v，ok= 方式，避免panic
```go
var w io.writer = os.Stdout //隐藏，只能对w 调用w.Write()方法，os.Stdout其余特性被隐藏
var file  = w.(*os.File) // 抽取，out的类型是具体类型 *os.File,
var closer = w.(io.closer) //因为w dynamic value satisfy io.closer interface,可以转化任何具体类型满足的接口类型；closer 现在是io.closer interface，其余特性被隐藏。
switch w.(type) {
//只要是w 动态类型能满足的接口类型，或者动态类型本身， case 就符合要求。
case io.writer:
case os.File:
case io.closer:
}
```
