

注意边界条件
# 功能测试Test...
## Table-driven test
挑选特殊输入集合，测试输出集合是否符合预期
函数的输入输出映射比较明确
输入的集合需要反映一般情况和边界条件

## Randomized testing
随机化的输入序列，如何验证输出：
- 与一个预期输出相同的函数比较;如比较不同算法的函数
- 输入集合随人随机，但输出却是可以预期的；如产生随机palindrome string 测试IsPalindorme()

## command program testing
`_test.go` 文件放在main.go同一个目录下，同样是“package main”；
测试时可以修改io.Reader or io.Writer， 便于比较输入输出；也可以修改全局变量；见下面white-box test
main.go 应该解耦为可以测试的几个函数；
测试函数在故障应该返回error,而不是os.Exit(1) or log.Fatal(...)，否则test driver 无法捕捉错误。

## balck-box vs white-box
- black-box testing
测试API，内部实现不透明，more rubust, 强调了client视角，can reveal flaws in API design，而且不用随着API实现改动而频繁修改测试代码.
- white-box testing
可以覆盖内部实现的trickier parts,可以为方便测试进行"fake",如重定位输入输出，修改global variable等。实际的业务很多时候不能实测，如刷信用卡等，需要进行模拟测试。这是white-box testing的专长。

## external-package testing
package 目录下的测试文件都必须命名为`_test.go`, 测试代码.go file的package name 有两种情况， 一是与测试包同名，测试文件可以访问所有包内global variables, 方便实现white-box testing; 二是包名加上`_test`后缀，相当于测试文件是一个独立的包，需要import 测试的包。测试文件只能访问API。called external package test. 功能有1）实现黑盒测试；2）可以导入其他包，并避免路径递归(导入的包依赖测试包)。

```bash
go list -f={{.GoFiles}} #normal .go files
go list -f={{.TestGoFiles}} # in-package test  files
go list -f={{.XTestGoFiles}} # exteranl package test files
```

如果external testing，同时需要访问包内变量, 可以增加一个in-package test file, convention name is "export_test.go", open a back door, 里面export unexported variables, 简单形式就是改个大写字母，如 fmt package `var IsSpace = isSpace`；这样external testing can used fmt.IsSpaces as fmt.isSpace.

## write effective tests, avoid brittle tests
抽象的测试framework 会失之具体，测试应该通过清晰、具体的上下文信息。
测试代码也需要维护， 如果实现代码稍微的变动，测试代码也需要平复修改，那么测试代码是脆弱的brttle。避免编写brittle 测试代码，获得stable：
- 关注稳定的核心接口
- 比较核心substring
- 编写抽取函数获取输出的essence

## coverage
By it's nature, testing is not complete.
Edsger Dijkstra: "Testing shows presence, not absence of bugs"
简单测量覆盖面是statements converage, 就是代码语句被测试执行的统计。
```bash
go test -coverprofile=c.out
go tool cover -html=c.out

go test -cover #只是简单显示覆盖率，不输出文件

```
100%的语句覆盖不太现实，而且不见得是好目标，语句被执行不代表没有bug, 某些语句需要特殊各站情况的特性，某些panic 永远不会被执行。
测试立足于实际，平衡成本和收益。trade-off between cost of writing test  and failures could have been prevented by tests.
覆盖率测量工具有助于发现 weakest spots, 但devising good test cases跟编程一样需要审慎思考

# 性能测试
Benchmark... 函数默认不运行，使用-bench=<reg>选择， -bench=. 表示全部选择
Test...函数 使用-run=<reg>选择，默认全部运行，进行benchmark 测试时，可以使用-run=NONE, 不运行Test...函数
性能测试有两种：函数在特定输入下的运行性能，和函数在输入增长下的时间和内存增长曲线（算法渐进性能）
如果测试初始化工作时间比较重，可以使用`b.ResetTimer()`重置。
 go test -bench=. -benchmem
 -benchmem 显示内存分配allocs/op


## profiling
https://blog.golang.org/profiling-go-programs
在程序运行的时候监听选择的events,记录信息，汇总成一个profile文件,事后对profile进行分析。
cpu profile原理:是定期（100 per second) sampling, 看采样时刻调用stack上的函数，得知：那个函数在运行，哪些函数在调用链上;注意stack 只能监视100 frame, 所以递归导致调用链条很长的话，可以会漏掉源头的哪些函数的计数。
mem profile原理：

### go test profiling
go test profiling 有三种，注意混用时测试间会互相干扰。
```bash
go test -cpuprofile=cpu.out #操作系统thread 切换时
go test -blockprofile=block.out #协程block in channel, sys call, lock
go test -memprofile=mem.out #heap profile, sampling internal memory allocation goroutine， avr per 512KB
```

### runtime/pprof
stane alone 程序可以选择进行profile,同样输出profile 文件。
## go tool pprof
go tool pprof对profile进行分析，profile只记录了函数地址，需要传入运行文件，go test会自动保存运行文件为“packageName”.test；runtime/pprof 的话需要传入编译好的运行程序文件。
```bash
go test -run=NONE -bench=ClientServerParallelTLS64 -cpuprofile=cpu.out net/http
go tool pprof -text -nodecount=10 http.test cpu.out
go tool pprof -web #浏览器 svg 流程图
go tool pprof  http.test cpu.out #进入交换模式
# top10
# top10 -cum
# web
# weblist
#list <function-name-pattern>
```

# Example...
有三个功能：
- godoc 将其显示在对应的同名函数条目下；go doc 不理会。
单独名为Exmple的函数与package对应。
- 如果跟随//Output 的注释，go test 会比较注释与标准输出的一致性
- golang.org 网站的godoc 带有GoPlayeground， 可以在线编辑、运行Example...functions
