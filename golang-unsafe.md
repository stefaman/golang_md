
unsafe 包的源文件只有类型声明，功能是编译器层面实现的。
## 内存布局
unsafe.Sizeof(v), unsafe.Alignof(v), unsafe.Offsetof(s.m) //of是小写的
是编译器时 constant expression,含义是C语言的
unsafe的操作获得是constant, 因为是编译器运算的，参数是表达式，不是值，表达式不会被计算。这点类似C。
Go语言规范没有指定结构的内部存储顺序，目前的编译器都是按照声明顺序存储，所有在某些内存关键场合，调整结构field顺序，可以减少hole.这也如同C。
疑问：一个类型的对其与其是否是结构的field有关系吗？？？
Type.Size() int //unsafe.Sizeof() uintptr
Type.Align() int //unsafe.Alignof() uintptr
Type.FieldAlign() int //测试不出与Align的区别， unsafe.Alignof(s.f)
StructField.Offset // unsafe.Offsetof() uintptr

## unsafe.Pointer and uintptr

Pointer 可以存储任何指针，但是丢失的类型；可以指针类型相互转换。比如对同一个内存区域进行不同的解读，实现C union or conversion;
**注意** 类型+地址=value; 比如&x == &x[0], 同一个地址可以解读为数组，或者元素，这也类型C。
下面代码可以实现 C union的类型功能。
```go
func floatBits(f float64) int64 {
	return *(*int64)(unsafe.Pointer(&f))
}
```

###Pointer 与 uitptr 可以相互转换。
uintptr直接就是整数，不再是内存的引用;可以1）实现指针算术运行，打印等；2）reflect 函数返回uintptr，强迫调用方必须import "unsafe"，才能读取修改内存。
**注意** 坑
实现的GC有可能会移动内存以减少fragment，自然，会自动变更所有指针的值，应对应新的内存地址。但是如果，将unsafe.Pointer 存入为uintptr，就变成了一个值，过后在变位unsafe.Pointer时内存地址可能已经变动了。另外一方面，如果将唯一的引用指针存为uintptr,就成了没有引用的变量了，会导致该内存被GC回收。
对于返回uintptr value的函数，要立马convert to unsafe.Pointer, the result should be immediately converted to an unsafe.Pointer to ensure that it continues to point to the same variable
```go
//!+wrong
//tmp只是一个number,x没有被引用了或者内存位置（地址）因为GC发生了变动，pb有可能指向一个没有意义错误的旧地址
// NOTE: subtly incorrect!
tmp := uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)//只允许整数加法
pb := (*int16)(unsafe.Pointer(tmp))
*pb = 43
//!-wrong
//correct, 写在一个表达式
pb := (*int16)(unsafe.Pointer(uintptr(unsafe.Pointer(&x) + unsafe.Offsetof(x.b))))

pT := uintptr(unsafe.Pointer(new(T))) //err, 先创建的变量会被回收
```

### unsafe and relfect
uintptr直接使用Value.SetUint, Value.Uint
```go
package reflect
Value.Kind() == reflect.UnsafePointer
Value.Kind() == relfect.Uintptr
Value.SetPointer(p uintptr) //It panics if v's Kind is not UnsafePointer.

func (Value) Pointer() uintptr//It panics if v's Kind is not Chan, Func, Map, Ptr, Slice, or UnsafePointer.

func (Value) UnsafeAddr() uintptr//addressable Value
//== Value.Addr().Pointer()

func (Value) InterfaceData() [2]uintptr // (type, value)
//It panics if v's Kind is not Interface.

//按照typ 解释p指向的内存，赋予其类型; Value = pointer + Type
reflect.NewAt(typ Type, p unsafe.Pointer) Value

//注意事项见 unsafe package
reflect.StringHeader
reflect.SliceHeader
```

Value.InterfaceData 解读
比较：Value.Interface(), 返回的(interface{})(dynamic value); 而InterfaceData()[1]是 pointer to dynamic value 少了一次值copy;
返回Value.Interface(), 丢失了接口类型的信息，从InterfaceData()[0]可以获知是什么接口类型；
```go
type W string
func (w W) Write([]byte) (int, error) { return 1, nil }

var i = W("string...")
var w io.Writer = i
wv := reflect.ValueOf(&w).Elem() //interface kind Value
cValPtr := unsafe.Pointer(wv.InterfaceData()[1])
cTypPtr := unsafe.Pointer(wv.InterfaceData()[0])
cTypValue := reflect.NewAt(reflect.TypeOf(reflect.TypeOf(0)), cTypPtr).Elem()//Type Value
cTyp := cTypValue.Interface().(reflect.Type)  //Type
cVal := reflect.NewAt(wv.Elem().Type(), cValPtr).Elem()
p := *(*W)(cValPtr)
t.Log(p, cVal.Interface().(W), cTyp)//string... string... io.Writer
```
