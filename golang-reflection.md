# Reflection
reflection 可以在runtime 窥视或改变任何变量的类型和值，对fmt, json等“泛型”都依赖reflect package.
http://localhost:6060/blog/laws-of-reflection
https://jimmyfrasche.github.io/go-reflection-codex/

## reflect.Type
reflect.Type 是接口类型，表示各种类型
func Typeof(i interface{}) Type //获得参数的具体类型信息
Type 类型是 comparable ==
fmt `%T`内部使用了reflect.Typeof

注意：
传入接口类型变量，只会获得具体类型;如果需要可以传入pointer to interface,可以获得接口类型。
参数的名字丢失了。
包内部不导出、隐藏的数据结构也会暴露出来
- byte, rune类型会显示为uint8, int32,
- 其余的named type能被显示，不会显示underlying type
- 传入nil, 或者任何接口类型的zero value, 返回nil.
**z注意** 这个nil不包括refrence type的zero value,(输入map[string]bool(nil), 参数i 不是nil, 对应的接口类型变量的dyanmic type is “map[string]bool", dynamic value 才是 nil；但是输入io.Reader(nil), 就是nil,因为输入参数根本没有dynamic type and value， io.Reader就相当于没有出现)

```go
fmt.Printf("%s\n", reflect.TypeOf(reflect.TypeOf(1)))
fmt.Printf("%s\n", reflect.TypeOf(reflect.ValueOf(nil)))
fmt.Printf("%s\n", reflect.TypeOf(time.Second))
fmt.Printf("%s\n", reflect.TypeOf(byte(1)))
fmt.Printf("%s\n", reflect.TypeOf(rune(1)))
//Output
// *reflect.rtype
// reflect.Value
// time.Duration
// uint8
// int32
```
### Type.Kind()
Type类型的方法很多时针对特定类型的，调用会painic。调用可能panic的方法前，用Type.Kind()先获取类型的种类。
??? Type.Kind() 什么时候返回reflect.Invalid

以下方法时通用的：
Type.String() 返回类型说明string;
Type.Name() 返回包内的名字；
Type.PkgPath() return import PkgPath
Type.Size() analogous unsafe.Sizeof
Type.Align()
Type.FieldAlign()

### Type.Elem() Type
获得元素类型
不能获得interface 的dynamic type
It panics if the type's Kind is not Array, Chan, Map, Ptr, or Slice.
传入&nil_interface, 获得interface type.
```go
writerType := reflect.TypeOf((*io.Writer)(nil)).Elem()

fileType := reflect.TypeOf((*os.File)(nil))
fmt.Println(fileType.Implements(writerType))//true
```

### create a new Type
reflect.MapOf(K, T Type)
reflect.SliceOf(T Type)
reflect.ArrayOf(count int, T Type)
reflect.PtrOf(T Type)
reflect.ChanOf(reflect.BothDir, T Type)
reflect.ChanOf(reflect.SendDir, T Type)
reflect.ChanOf(reflect.RecvDir, T Type)
reflect.StrucOf(fields []reflect.StructField) //StructField{Name, Type, Tag}
reflect.FuncOf(in, out []Type, variadic bool), if varaiadic, in[last] must be slice  Type

### conversion
类型转换不成功是painic；可以先检查在转换
Type.ConvertableTo(T Type), Type.Implements(T Type),  Type.AssignableTo(T Type), Type.Comparable(),
```go
if S := v.Type(); S.ConvertableTo(T) {
	t := v.Convert(T)
}
```
### type assertion
转换为具体类型,有两种方式
```go
if conrete := SomeReflected.Elem(); concrete.IsValid() {
	//avoid panic: if SomeReflected is nil interface
	concrete.Type() == T
	...
}

if SomeReflected.CanInterface() {//避免nil interface painic or unexported field painic
	if v, ok := SomeReflected.Interface().(conrete_type); ok {
		...
	}
}
```
转化为接口类型, 使用Value.Convert;
Type.Implements(T Type), T is interface Type 判断是否现实了接口
```go
var i io.Writer = os.Stdout
var c io.Closer
id := reflect.ValueOf(&i).Elem()  //id is io.Writer, dynamic type is *os.File
concrete := id.Elem()        // *os.File
assertType := reflect.TypeOf(&c).Elem()   // io.Closer type
cd := concrete.Convert(assertType)  //io.Closer,  dynamic type is *os.File
t.Log(cd.Elem() == concrete)    //true
```
## reflect.Value reflect.ValueOf
reflect.Value 是Struct类型，表示各种value. 注意，接口定义的某些方法只能用于对应的具体类型，否则是panic。先用Value.Kind()检测，防误。
`func Valueof(i interface{}) Value`

Value 接口的String()返回动态类型说明字符串,如"<int Value>", 或者字符串类型本身，
fmt `%v` verb don't call Value's String() method，but display conrete value. fmt `%s` verb 需要明确传入 v.String(), 这是比较特殊的。

### zero Value
Value类型也是有默认zero, called zero Value;
The zero Value represents no value. Its IsValid method returns false, its Kind method returns Invalid, its String method returns "<invalid Value>", and all other methods panic.
zero Value 没有存储类型和值，很多方法会painic; 安全的方法只有Kind, IsValid, String
zeroValue.Kind() == reflect.Invalid
Value.IsValid() 用于判断

返回zero Value的方法：
nil_ptr.Elem()
ValueOf(nil)
ValueOf(zeroValue)
ValueOf{}
reflect.New(reflect.TypeOf(reflect.Value{}))
reflect.Zero(reflect.TypeOf(reflect.Value{}))

reflect.Zero(T Type) 返回的不是zero Value, 是T类型的zero value;当然，如果T is reflect.TypeOf(reflect.Value{})除外；

### Value.Type()
Value.Type() //TypeOf(存储的static type value)
if Value is zero Value, painic

### Value.Elem()
Value 只能是reflect.Inerface or relfect.Ptr

## kind vs type
虽然type 有无数种，但是Value.Kind()， or Type.Kind()是有限的：
- basic types Bool, String, and all the numbers;
- the aggregate types Array and Struct;
- the reference types Chan, Func, Ptr, Slice, and Map;
- Interface types;
虽然传入reflect.ValueOf 的接口类型参数不在显示为接口类型了，但是aggregate type的内部成员如果是接口类型，可以检测是相应的接口类型。
vp.Elem().Kind() == reflect.Interface, vp is pointer to interface,
- Invalid, meaning no value at all. zeroValue.Kind() is Invalid.

Value.Type().Kind() == Value.Kind(); 例外是zero Value.

## reflect.Value to interface{}
Value.Interface(),会返回interace{}(<concrte value>)
是ValueOf()的逆运算,
可以将v.Inerface()传入fmt函数，相当于传入对应的具体类型。
但是
- zeroValue.Interface() is painic
- It panics if the Value was obtained by accessing unexported struct fields
Value.CanInterface() 用于判断。
- 接口类型的interface() 返回的(interface{})(dynamic value); 丢失了接口类型的信息，从Value.InterfaceData()[0]可以获知是什么接口类型；
```go
var i io.Writer = os.Stdout
id := reflect.ValueOf(&i).Elem()
t.Log(reflect.DeepEqual(id.Interface(), id.Interface())) //true
```

## create a Value
除了ValueOf, 还有
- New(T Type) return pointer to zero value T;
New(nil) panic
reflect.New(reflect.TypeOf(reflect.Value{})) is zeroValue

- Zero(T Type)
reflect.Zero(T Type) 返回的不是zero Value, 是T类型的zero value;当然
reflect.Zero(reflect.TypeOf(reflect.Value{})) is zeroValue

v := New(T).Elem() // v is settable
v := Zero(T) // v is not settable

对于引用类型， zero value is nil, 需要单独的initialize function
- MakeSlice, MakeMap, MakeChan, MakeFunc

- NewAt ???

## changed value
reflect not only can interpret value, also can changea a value. 但是函数都是传值的，reflect.ValueOf(x) 返回的都是value, not variable, not addressable; but
reflect.Valueof(&x).Elem() is addressable, refer to x;
传入指针的套路依然可用

### addressable
与具体类型的&有些区别[addressability](https://golang.org/ref/spec#Address_operators)
A value is addressable if it is an element of a slice, an element of an addressable array, a field of an addressable struct, or the result of dereferencing a pointer. If CanAddr returns false, calling Addr will panic.
map.MapIndex() return value is not addressable.
### Value.Adrr()
相应&v

### settability
reflect.Value 类型的具体含义和可调用的方法视其具体“存储的内容值”而定。某些Value是可以设置值的settability，谓词函数Value.CanSet() return true;
A Value can be changed only if it is addressable and was not obtained by the use of unexported struct fields.包括
- 指针类型Value.Elem()
- 隐含的指针类型, slice.Index(i)；
- filed of settable struct
- index of settable array

### reflect.Value.Elem()
- Value is pointer, 相当于*p, p.Elem() is settable value
The Elem method returns the variable pointed to by a pointer, again as a reflect.Value. This operation would be safe even if the pointer value is nil, in which case the result would have kind Invalid.
逆运算是Value.Addr()

- Value is interface
i.Elem() is the dyanmic value, is not settable value

### reflect.Indirect(v Value) Value
v 是指针时返回Value.Elem()，其余情况直接返回Value本身；不会painic.
某些情况下可以节约代码

注意区分： 接口，具体类型，pointer to interface，pointer to concrete type
```go
var w io.Writer
w = os.Stdout
v := reflect.ValueOf(w) // v.Type() is *os.File
v := reflect.ValueOf(&w) //v.Type() is *io.Writer
v := reflect.ValueOf(&w).Elem() //v.Type() is io.Writer
v := reflect.ValueOf(&w).Elem().Elem() //v.Type() is *os.File
```
### arithmetic type
Type.Bits()
注意类型的匹配，否则各种panic
```go
x := 2
xp := reflect.ValueOf(&x).Elem()
fmt.Println(xp.CanAddr(), xp.CanSet())
xd := xp.Addr().Interface().(*int)
*xd = 3
fmt.Println(x)

xp.Set(reflect.ValueOf(4))
fmt.Println(x)

xp.SetInt(5)
fmt.Println(x)
```

Set??类型的输入参数是最大类型，动适应目标value类型；那就有溢出的可能（合法的），Value.Overflow??? 可以检测是否赋值会溢出。
```go
xp := reflect.ValueOf(new(int)).Elem()
xp.Set(reflect.ValueOf(int64(4)))//panic: value of type int64 is not assignable to type int//输入参数类型，必须和目标value匹配
xp.SetInt(x int64)//输入参数的类型固定的最大化int64，自动适应目标value类型

if a.CanSet() && !a.OverflowInt(b) {
	a.SetInt(b)
}
```
## slice array string
虽然切片值不能addressable, 但是a[i] 是implicit pointer.所以修改其内容，不需要传入指针，但是数组时copy 传入函数的，所以修改数组的内容依然需要传入指针。

### Type method
reflect.SliceOf(T Type)
reflect.ArrayOf(count int, T Type)
Type.Elem() //元素的类型，数组和slice均适用
array Type: Type.Len() //只有数组类型有长度的概念，不要跟Value.Len()搞混淆

### Value method

slice and array both have Value.Cap(), reflect.Copy(dst, src)

模拟 s[i]
slice, array, string both have Value.Index(i int)
模拟len(s)
lice, array, string both have Value.Len()
模拟 s[i:j]
slice, array, string both have Value.Slice(i,j int)
模拟 s[i:j:k]
slice, array both have Value.Slice3(i,j, k int)
模拟 s = s[:len(s):i]
slice has Value.SetCap(i int); Value.Len() <= i <= Value.Cap();
模拟 s = s[:i]
slice has Value.SetLen(i int); i <= Value.Cap();
模拟 append
reflect.Append(s Value, x ...Value) Value
reflect.AppendSlice(s Value, t Value) Value
构建
reflect.SliceOf(T Type) (S Type)
reflect.MakeSlice(S Type, len, cap int)
数组直接使用reflect.Zero

reflect.ValueOf(e).Index(i) is variable, although e is not addressable value;前提是[i]是分配了内存的，否则panic;而传入&e却可以修改e,更加灵活。
```go
e := []int{1, 2, 3}
ev := reflect.ValueOf(&e).Elem().Index(2)
//ev := reflect.ValueOf(e).Index(2) //is ok
t.Log(ev.CanAddr(), ev.CanSet()) //true true
ev.SetInt(55)
t.Log(e) //[1 2 55]
ep := reflect.ValueOf(&e).Elem()
ep.Set(reflect.ValueOf([]int{2, 3, 4}))
ep.Set(reflect.Append(ep, reflect.ValueOf(5)))
ep.Set(reflect.AppendSlice(ep, reflect.ValueOf([]int{7, 8})))
t.Log(e) // [2 3 4 5 7 8]
```
ValuOf(接口类型) 会丢失接口类型的信息，只有具体类型；如果传入nil接口类型，那么只有得到invalid; 解决方法时传入pointer to interface; 这样可以获得传入接口类型的类型信息，也可以传入nil 接口类型。
可以赋值为实现了该接口的具体类型。
```go
type W int
func (w W) Write(p []byte) (int, error) {
	return int(w), nil
}

var w io.Writer
fmt.Println(reflect.TypeOf(w), reflect.ValueOf(w).Kind() == reflect.Invalid) //<nil> true
w = os.Stdin
fmt.Println(reflect.TypeOf(w), reflect.ValueOf(w).Kind() == reflect.Ptr) //*os.File true
t.Log(reflect.TypeOf(&w), reflect.ValueOf(&w).Elem().Type(), reflect.TypeOf(&w).Elem()) //*io.Writer io.Writer io.Writer
rw := reflect.ValueOf(&w).Elem()
var ws W = 33
rw.Set(reflect.ValueOf(ws))
fmt.Println(reflect.TypeOf(w), w) //test.W 33
```
空接口interface{}比较特别,可以赋值给他任何具体类型，但不能直接调用Set???()设置具体类型,但可以Set Valueof(任何具体类型)
```go
var y interface{}
fmt.Println(reflect.TypeOf(y), reflect.ValueOf(y).Kind() == reflect.Invalid) //<nil> true
ry := reflect.ValueOf(&y).Elem()
// ry.SetInt(2)                     // panic: SetInt called on interface Value
ry.Set(reflect.ValueOf(3))        // OK, y = int(3)
fmt.Println(reflect.TypeOf(y), y) //int 3
// ry.SetString("hello")            // panic: SetString called on interface Value
ry.Set(reflect.ValueOf("hello"))  // OK, y = "hello"
fmt.Println(reflect.TypeOf(y), y) //string hello
```

## map
Type.Key() //返回key's Type
reflect.MapOf(K, V Type) (S Type)

```go
//读取
// key is Value type
for _, key := range m.MapKeys() {nil slice for nil map
	v := m.MapIndex(key) // zero Value means not found
	...
}

//decode
m.Set(reflect.MakeMap(m.Type()))
m.SetMapIndex(k, v Value)// zero value means delete; painic if m is nil
```
## struct
有Type 和 Value 两个视角，不要搞混
### StructField
这是获得成员的类型信息
type StructField struct {
    Name string
		//只有unexported filed 非空，形如"gopl.io/ch12/test"
    PkgPath string
    Type      Type      // field type
    Tag       StructTag // field tag string
    Offset    uintptr   // offset within struct, in bytes
    Index     []int     // index sequence for Type.FieldByIndex
    Anonymous bool      // is an embedded field
}
###struct Type
Type.NumField()
Type.Field(i) reflect.StructField;//painc if i not in [0, NumField())
用NumField, Field, 只能变量最外成员，StructField.Anonymous 判断是否内嵌成员。
外层内嵌匿名结构成员的FieldStruct.Name 省略了包的名字

Type.FieldByName("name") (reflect.StructField, bool)
FieldByName模拟了s.name; 可以访问内嵌匿名结构成员；命名冲突时，返回false;

因为反射可以窥视包外结构的Unexported field；这会导致一个在struct 没有的名字冲突。
内嵌的包内结构成员的未导出field可能与内嵌包外结构的未导出field成员冲突；或者两个内嵌包外结构的未导出field发生名字冲突。这不带符合隐藏原则；FieldByName may return one of the fields named x or may report that there are none. https://github.com/golang/go/issues/4876

Type.FieldByIndex([]int) StructField
用于镶嵌结构的循序调用Filed(i)；
比如说S.FiledByIndex([]int{0,1,2}), 访问S.Field(0).Type.Field(1).Type.Field(2)

Type.FieldByNameFunc(match Func(string)bool) (StructField, bool)
模拟了struct的成员寻找方式，会一层层访问的内嵌成员；名字冲突时返回false;

### struct Value
Value.NumField()
Value.Field(i) Value
只能访问第一层成员，

Vaue.FieldByName("name") Value
模拟了s.name; 可以访问内嵌匿名结构成员(包括未导出的成员)；跟上述Type.FieldByName一样的寻找规则，失败时返回zero Value;

Value.FieldByNameFunc(match Func(string)bool) Value
同Type.FieldByNameFunc, 失败时返回 zero Value

### 可以访问，不能修改unexported field
这里讨论的是所有小写字母开头的field name, 包括包内的struct type
unexported value 可用被reflect interpret, can addressable, 但是can not Set, can not Interface;
这也是JSON等只处理exported field的原因.
传入pointer to struct, 可以修改其expotred field, 或者修改指针指向另外一个同类型的struct variable
```go
//unexported value can not be changed
stdout := reflect.ValueOf(os.Stdout).Elem() // *os.Stdout, an os.File var
t.Log(stdout, stdout.CanAddr(), stdout.CanSet(), stdout.CanInterface())
stdout.Set(reflect.ValueOf(*os.Stderr))
// stdout.Set(reflect.ValueOf(*os.Stdin)) //设置为*os.Stdin test无法输出???
fd := stdout.FieldByName("fd")
t.Log(fd.CanAddr(), fd.CanSet(), fd.CanInterface()) //true false false
t.Log(fd.Kind() == reflect.Uintptr, fd.Uint())      //windows:ture, XXX
// t.Log(fd.Kind() == reflect.Int, fd.Int()) //linux:true, 1
```

### struct tag
标签是关联到field的信息，注意标准写法value要加"",
`key1: "value1" key2: "value2"`
StructField.Tag, StructTag, underlying type is string
StructTag.Get //找不到返回""
StructTag.Lookup(key string)(name string, ok bool) //用于识别""

## function
### function Type
FuncOf(in, out []Type, variadic bool);
//if variadic, T.In(T.NumIn()-1) MUst be slice Type
Type.IsVariadic() bool
Type.NumIn() int
Type.In(i int) Type
Type.NumOut() int
Type.Out(i int) Type

### Make a function
reflect.MakeFunc(typ Type, fn Func(in []Value)(out []Value)) Value
构建的函数可以：
- Value.Call()
- Value.Interface() 回到现实世界
- f.Set(fValue) 改变现实世界
这种方法类型和算法分开，比较头疼；可以用来编写泛型函数

### invoke a function
Value.CallSlice 只能用于variadic function, 没个鸟用；因为
Value.Call 适用于全部函数，其signature
Call([]Value) []Value; 使用起来好费力啊。。。。

### reflect.Swapper



## method
在接口Value.Method() 只能访问接口的方法，Value.Elem().Method()访问具体类型的方法（可能比接口方法多）
T `*T` Methdo()的互相访问？？？

有Type and Value 两个视角，不要搞混
### Type.Method
获得是方法的类型信息，存储在结构Method中
### type Method
type Method struct {
    Name    string
    PkgPath string

    Type  Type  // method type, is function Type
    Func  Value // func with receiver as first argument, is function Value
    Index int   // index for Type.Method
}

Type.NumMethod()
Type.Method(i) Method
Type.MethodByName("name") (Method, bool)
Type 其实表示了很多种类型，具体类型和接口类型调用Type.Method返回的Method.Type 和Method.Func含义是不同的
### 具体类型的Method
Method.Type 和Method.Func 表示的是concrete type's method expression,
具体类型是函数的第一个参数，Func.Call调用需要传入具体类型参数
Func is a reflect.Value, is a method expression, should pass reciever as first parameter
Type.NumMethod, 模拟s.F(), 包括了结构内嵌的匿名成员方法。
如果方法名字冲突，该方法不会包括进去。
Type.NumMethodByName 模拟s.F(), 包括了结构内嵌的匿名成员方法,名字冲突的方法返回false.
同struct一样，该类型并没有实现了某种接口，Type.Implements(T) return false.
**注意**： 搜索匿名成员方法只包括绑定在该类型上的方法，不同于selector s.F()编译器自动的完成*、&转换。只会搜索到类型匹配的方法。原因是搜索的方法Call()调用必须参数类型匹配，这里头不会有什么编译器自动类型转换。
但是需要注意的，代码上`(T).Func`都是（隐含）同时实现`(*T).Func`;反过来不成立。可以通过Value.Addr()获得&s Value, 来搜索方法。
```go
type st struct {
	io.Writer //(1)
	os.File //(2)
}
s := st{}
PrintMethods(&s) //可以调用*os.File，io.Writer的方法；不要求st内是否是指针成员
//1,2都有Write()，Write()方法丢失
PrintMethods(s) //只能调用os.File（没有方法），io.Writer的方法;要求st内是*os.File
//Write()依然冲突，没有可以遍历的方法

s.Close();(&s).Close();//无论st{}包含的os.File or *os.FIle, 都是OK

//遍历具体类型绑定的方法
func PrintMethods(i interface{}) { //i is not nil
	v := reflect.ValueOf(i)
	typ := v.Type()
	fmt.Printf("value type is %s\n", typ)
	for i := 0; i < typ.NumMethod(); i++ {
		fmt.Printf("func (%s) %s%s\n", typ, typ.Method(i).Name, strings.TrimPrefix(v.Method(i).Type().String(), "func"))
	}
}
```
### 接口类型的Method
获取接口类型的方法（传入pointer to interface， 如`（*io.Writer）nil`）
Method.Type 函数是接口类型的method signature
Method.Func is zero Value
```go
//遍历接口类型方法
func PrintInterface(i interface{}) { //i is *interface
	typ := reflect.TypeOf(i).Elem()
	fmt.Printf("interface type %s\n", typ.String())
	for i := 0; i < typ.NumMethod(); i++ {
		fmt.Printf("func %s%s\n", typ.Method(i).Name, strings.TrimPrefix(typ.Method(i).Type.String(), "func"))
	}
}
```
### Value.Method
Value.Method(i) or Value.MethodByName 返回的是Value, 表示了method function Value, 绑定了reciver；
Value.Call()不需要传入reciever。
Value.Method is panic if Value is nil interface;
方法的搜索规则同上述Type.Method

```go
//f relfect.Value, is a method value, a closure that has a reciver
f := reflect.ValueOf(time.Hour).MethodByName("Hours")
// t.Log(f.Type().String()) // func() float64
retF := f.Call(nil)[0].Interface().(float64)
t.Log(retF) //1
```

## chanel

构建
- reflect.ChanOf(relfect.ChanDir, T) (S type)
- reflect.MakeChan(S, 0) (v Value)
- reflect.MakeChan(S, buffer_size) (v Value)

性质
- Value.Cap() int
- value.Len() int
- Type.Dir() ChanDir

通讯
- Value.Close()
- Value.Recv() (v value, ok bool)
- Value.Send(x value)
- Value.TryRecv() (v value, ok bool)
no-block return, 如果通道阻塞，返回 zero Value, false; 如果通道关闭，返回 zero element value, false;
- Value.TrySend(x value) bool
no-block

没有range, 只能自己动手
```go
for {
	v, ok := iterable.Recv()
	if !ok {
		break
	}
	//body
}
```
###select
cumbersome, but flexible;可以在runtime 创建大量的通道slice
实现下面的code
```go
select {
case m1 := <-c1:
	...
case c2 <- m2:
	...
case m3, ok <- c3:
	...
default:
	...
}
```

```go
cases := []reflect.SelectCase{
	{Dir: reflect.SelectRecv,   Chan: c1},
	{Dir: reflect.SelectSend,   Chan: c2, Send: m2},
	{Dir: reflect.SelectRecv,   Chan: c3},
	{Dir: reflect.SelectDefault},
}
switch selected, recv, recvOK := reflect.Select(cases); selected {
case 0:
	//c1 was selected, recv is the value received
	//we get recvOK whether we need it or not
case 1:
	//c2 was selected, therefore m2 was sent
	//recv and recvOK are meaningless in this case
case 2:
	//c3 was selected, recv is the value received if recvOK == true
	//recvOK is false if c3 is closed
case 3:
	//the default case
	//recv and recvOK are meaningless in this case
}
```
## unsafe
见unsafe 笔记

## encode and decode
### 循环引用
fmt.Print 只显示指针的地址，可以处理循环指针和结构，但是循环的slice or map 一样可以让其stuck....
json.Marshal(所有内部引用自身的值)//fatal error: stack overflow

## caution
reflect 虽然提供泛型和灵活性，但是避免过度使用。有几点：
- 错误会导致panic,这样代码需要做很多type swicth; 而且不一定捕捉了所有panic
- 没有static type check, code is hard to understand
- 性能差，one or two orders of magnitude slow er than code specialized for a particular type.
