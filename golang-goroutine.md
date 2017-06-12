# 实践

## pipelines
https://blog.golang.org/pipelines
上游在1）发送完全部数据后，2）下游通知cancle后，要关闭通道，该stage 才terminate
下游如果需榨干通道（range 通道直至其closed），或者提前cancle 通知上游停止发送。
关闭广播通道是现行广播通知cancle的最佳实践

## context
https://blog.golang.org/context
context package 提供了更加便利的控制pipeline各stage 终止的方案，包括了cancle, deadline, timeout 三种终止方法


# Goroutines and Channels
todo:
协程的调度、内存

concurrent programming vs sequential programming
intuitions acquired from sequential programming may at times lead us astray.
communicating sequential processes or CSP, vs shared memory multithreading.
The differences between goroutines and threads are essentially quantitative, not qualitative.
program 由多个goroutine 构成，func main是main goroutine, 但main goroutine return, or gouroutine call exit funcs, entire program exit. 但是如果是长寿面程序，注意goroutine 需要显示退出，否则就是goroutine leak（比如stuck in writing a non-receiver channel), 导致资源的浪费.

通道有几个特性可以利用：
- 传递events,启动同步作用
- 传递values
- 阻塞、唤醒
- 构建负责模型

## unbuffered channel
阻塞读、写 goroutine, 也reawaken 相应的写 、读 goroutine
goroutine是抢占式多线程，unbuffered channel 起到了synchronize的作用，可以实现yield-resume
Communic ation over an unbuffered channel causes the sending and receiving goroutines to synchronize. Because of this, unbuffered channels are sometimes called synchronous channels.
When a value sending to a unbuffered channel the receipt of value **hanppens before** the reawakening of sending goroutine.
x **hanppens berfore** y: 不存在竞争和冲突
x **concurrent with** y: 不能确定x, y执行的先后，对共享变量的存储存在竞争和冲突

c := make(chan int) // 通道传送的值的是有类型, c's type is chan int
c <- v //写入通道
v <- c //read
<- c// read and discard, 用于同步
通道作为同步工具时，通道可以是make(chan struct{}),传递的信号值可以选择struct{}{}。
go func{}() //可以使用closure

通道是reference value, 可以比较(指向同一通道时相等)， zero value is nil
读一个关闭的通道只是返回zero value, 写一个关闭的通道是panic, 关闭一个已经closed的通道或者nil通道也是panic
value , ok := <-channel // ok 可以省略，失败channel 是否closed
for v := range channel {} //迭代直至channel closed

除非为了通知接收方，不需要显示关闭channel, garbage collector will do that auto

通道是全双工bidirectional，两端各有send and receive, 用于pipe lines时，某些通道只使用了read or write direction, 可以显示声明unidirectional channel, 强制使用某个方向
chan<- int // send-only channel
<-chan int // receive-only channel
only can close send-only channel

Conversions （implicitly）from bidirectional to unidirectional channel types are pemitted in any assignment. There is no going back, however: once you have value of a unidirectional channel such as chan<- int, there is no way to obtain from it a value of type chan int thar refers to the same channel data structure.

## buffer channel
bufc := make(chan int 3) // buffer channel has a queue, 在队列有空间时不会阻塞写的goroutine, 在队列非空时不会阻塞读的goroutine.
cap() 通道容量
len() 当前数量，注意返回值已经不可靠了
没有阻塞，也就没有了同步。利弊需要权衡。
1. 多个写， 一个读：
- 只读一次，可以起到一个最快选择器，如果使用unbuffered channel, 将导致goroutine leak, 慢速的写goroutine 都会阻塞住。
2. pipe lines
buffer channel 用于pipe lines时可以起到缓冲 assembly line 上各生产/消费者的速率波动，注意是缓冲的是速率的波动，而不是向存储器山那样的缓冲。比如，如果消费者与生产者速率差别很大，使用buffer是没有用的，遵循Amdahl's law, 需要增加卡脖子环节的goroutine. 如果大家速率波动变差可以忽略，那没必要使用buffer, 直接使用unbuffer channel。

## concurrent programming patterns
注意goroutine leak,
阻塞在写没人读的通道上，使用buffer channel;
阻塞在读没人写的通道上，使用定时闹钟；
range永远不关闭的通道,导致死循环，使用closer goroutine.

### embarrassingly parallel

各个activity直接没有顺序上的执行要求，独立的，不互相依赖，这是最简单的并发，而且增加并发协程数量可以线性的提高性能（在满足系统资源的限制前提下）。
注意新手错误：
- for循环中go func 使用闭包时直接捕获循环变量index，这样只会捕获到循环变量的终止值，需要通过wrapper function的参数传入循环变量index。
- main 函数循环派发任务后退出，这样整个程序都退出了，执行协程可能来不及执行。即使是普通函数将其需要完成的任务分割成几个goroutine，派发任务后直接退出，这样也是bug,因为函数的caller假定函数返回后已经完全了他所赋予的责任，不会假定他偷懒委托给他人了，所有主函数需要等待执行子任务的协程运行完毕。这样goroutine和子协程需要同步，子协程退出前需要通知主协程。因为知道子协程的数量，所以使用一个unbuffer channel C，子协程退出前C <-,主协程重复N次 <-C，即可。可以考虑使用N size buffer channel，这样子协程不用扎堆阻塞在等待回收上。

下面主协程指的是派发任务并回收执行协程的状态，不一定是main
- 派发任务的协程回收执行协程们的返回状态量，需要注意主协程不能过早返回，导致子协程僵尸在等待回收（写入通道阻塞）。
 - 如果主协程需要提前返回，而且执行协程数量可知，那么可以使用buffer channel,执行协程不会阻塞在写入上;虽然主协程已经返回，但执行协程仍在运行，仍然消耗资源。如果主协程也是在一个循环中，注意主协程过早返回时导致循环加快，需要注意进程资源的消耗，如打开的文件数量是否超标。
 - 如果子协程数量不可知，如从通道传入；或者子协程也可能提前返回，不一定需要写入通道，这时候需要记录派生协程的数量变化，使用`var wgsync.WaitGoup`。

 并发数量的选择
 - CPU消耗型 最佳并发数量是CPU核的数量，再多也不能压榨CPU计算能力，反而因为协程调度等消耗降低性能。
 - IO等待型  并发数量越多越好，但满足资源限制的前提很重要，否则过度并发exessive coconcurrency不但是性能下降，而且会出现错误。比如：打开文件数量（包括普通文件，socket）,网络的带宽，磁盘的读写速度，缓存容量等。
 - 如果执行任务的不是独立的，那多协程就只会恶化性能。比如math/rand的外层函数是协程安全的，因为内部使用了锁sync.Mutex,所有如果任务是产生10000个随机数，因为是CPU密集运算型，i5 系统分割4个协程每个产生2500个随机数，1）使用rand外层函数，因为锁，4个协程不能独立运行，堵车，性能还不如单协程；2）每个协程使用独立的源rand.New(),可以提升约3倍性能。
性能影响研究

 并发数量的控制
 - 简单的控制派发协程的数量
 - 控制资源的使用数量，超过数量时阻塞。对于IO类型应用，不去现在协程数量，而是限制资源访问数量，是更高效的做法。阻塞的地方离资源越近越好。
 使用buffer channel model a concurrence primitive called counting semaphore。
 It's good practice to keep the semaphore operations as close as possible to the IO operation they regulate.

 ```go
//tokens is a counting semaphore for limiting the concurrency in f
//创建N个token, vacant slot is token, avoid filled tokens atfer create
tokens := make(chan struct{}, 20)
func f(){
	// get a token if permit, or stuck in waiting
	tokens <- struct{}{}
	//release a token, 使用defer 或者靠近资源释放处
	defer func(){<- tokens}()
	//do someting
	//<- tokens
	//资源释放后函数还有重运算

}
 ```

### 并发顺序的控制
比如需要控制网页crawl的depth,
e8.6 研究

### select
1, multiplex, 多路通道复用， 可以实现abort

```go
//控制允许最长阻塞时间
select {
	case <- time.After(duration):
		//...
	case <- 正常操作返回信号:
		//...
}

//中断
select {
	//特指某个时刻内多长时间
	case <-time.After(10 * time.Second):
		// Do nothing.
	//如果select 在循环内部，可以使用周期信号
	case <- time.Tick(10 * time.Second):
		//Do nothing
	case <-abort:
		fmt.Println("Launch aborted!")
		return
}
```
2. 非阻塞的polling  a channel

```go
//非阻塞的读
read := make(chan int)
go func(){
	//read
	read <- retValue
}()
//重复可以 polling a channel
select {
	case ret := <- read:
		//...
	defualt:
	//nothing to do
}
```

3. timer channel
time 包有函数可以返回时钟channel，
1）周期性信号，
注意time.Tick()调用会产生一个leak goroutine, 可选time.NewTicker
time.Tick()与time.Sleep的交换???
```go
tick := time.Tick(1 * time.Second)
<- tick //tick 通道按照设定周期性返回time.Time value

timer := time.NewTicker(1 * time.Second)
<- time.C
timer.Stop()
```
2）闹钟信号

```go
<- time.After(duration)
```

4. nil channel
读 或写nil 通道均是阻塞，结合select
a cas e in a selec t st atement whos e ch annel is nil is never selec ted. This lets us use nil to enable or dis able cas es that corresp ond to features like handling timeouts or cancellat ion, responding to other input events,
or emitt ing out put.
```go
var tick <-chan time.Time
if *verbose {
	tick = time.Tick(500 * time.Millisecond)
}

select {
	case <-norml:
		//...
	<- tick:
		// if nil, just bypassed
}

```
5. closed channel
channel不能实现一个值送给多人使用，先读先得，读写必须匹配，否则就是阻塞僵尸。
读取closed channel,立即返回(zero value, false),这样的特性除了用于通道EOF，也可以实现broadcast,允许多人读取同一个通道，一人某时刻关闭通道，实现广播效果。
写closed channel is panic

6. cancellation
协程只有当程序整天terminate or 自身主动退出才会结束。这样为了让执行任务的协程提前退出，外部只能通过通道传入event。很多情况下，需要同时退出N个协程，虽然可以通知方写一个通道N次，而被取消的N个协程分别读该通道。但是，这样的程序结构容易出错，导致读或写的一方阻塞在通道上。而且，很多时候，N不是确定的数值。
利用广播通道，实现1对多，用于gouroutins cancellation.
注意：
- 退出时已经允许goroutine的处理，如果不是程序暴力强退，需要注意阻塞在通道上的协程。可以使用select
- 为了查看退场时是否goroutine清理干净，可以在main中defer panic();但是panic只会显示所在goroutine的stack，测试没有显示其他goroutine；如何显示程序全部goroutine？？e8.9_v2.go
- 获取函数可以传入done通道，和其余参数，返回两个通道，值通道，error通道，这样故障可以提前输入通道，调用方可以根据故障类型选择关闭done来去终止函数。

```go
done := make(chan struct{})
go func(){
	//detect cancelled events
	if cancelling {
		close(done)
	}
}()

//polling function
func cancelled(){
	select{
		case <-done：
		 return true
	 default:
		 return false
	}
}

select {
	case <-c:
		//...
	case <-done:
		//return		
}
```
## shared memory
通道的排队顺序？？

如果两个事件不能确定谁happens before,则称两个事件concurrent；注意不是parallelism,事件不一定需要同时发生。
concurrency-safe
deadblock, liveblock, resource starvation
race condition: data race, undefined behavior
"There is no such thing as a benign data race"

### avoid data race
- only read
- variables are confined to a single goroutine, other goroutine shared with channels.
"Do not communicate bye sharing memory, but shared memory by communicating"
monitor goroutine: A goroutine that bro kers access to a confine d var iable using channel requests is cal le d a moni tor goroutine for that var iable.
pipeline 中变量的生命周期跨越不同的stage, 如果每个stage处理后送到下一个阶段，不再touch该变量，这种方式也避免了data race, 叫 serial confinement。metaphor:蛋糕流水线的蛋糕
有时需要同monitor goroutine 传入数据，传出处理后数据，可以传入数值+通道的composition value,monitor gouroutins 通过传入的通道传出数据。这里体现了channel is first class.
- mutua exclusion
mutex brokers access to variable, called monitor of the variable;两者的声明最好紧密挨着；而且不要暴露，encapsulated;A locked Mutex is not associated with a particular goroutine. It is allowed for one goroutine to lock a Mutex and then arrange for another goroutine to unlock it.
binary semaphore, 可以实现mutua exclusion, 注意与sync.Mutex有差别
```go
mu := make(chan struct{}, 1)
mu <- struct{}{} //lock
<- mu //unlock

var mu sync.mutex
mu.Lock()
mu.Unlock()
```
critical section 要包围住shared variable, 不能有缝隙；
避免重复上锁,deadlock; 避免Lock/Unlcok配对错乱，It is a run-time error if m is not locked on entry to Unlock.
使用defer mu.Unlock()，实现任何分支，包括painic 均解锁。defer 的性能损坏很轻微
注重程序代码的清晰，而不是 premature optimization

- multi readers, one Writer
适用于读操作数量多，写操作数量少的场合
可以多个reader,期间写被闭锁；只有一个writer,期间所有读写均被闭锁。
读操作mu.RLock()可以共存，期间写操作被锁住,只有当所有reader mu.URLock(), 才可以mu.Lock();
mu.Lock()是互斥的，期间所有mu.RLock() and mu.Lock()均被block；那么写操作结束时唤醒的是？？？
```go
var mu sync.RWMutex

mu.RLock()
return balance
mu.RUnlock()

mu.Lock()
balance += amount
mu.Unlock()
```

- memory synchronization
**特别注意** 在goroutine内部观察的内存变化与顺序执行的指令是一致；但是其他goroutine不保证观察的相同的内存变化。channel synchronization or mutua exclusion 不仅仅是指令的同步，也是内存的同步；多核CPU的核内cache会被刷新到main memory，使得变量的更新被所有同步的协程观察到。
**同一内存位置的所有的读和写操作都必须同步**

## lazy initialization
初始化数据的消耗如果比较高，而且不一定会被使用，可以推迟到时候的时候在初始化。这里面存在一个多协程下保证只初始化一次的问题的。使用读写锁的复杂实现。
```go
var mu sync.RWMutex // guards icons
var icons map[string]image.Image//等待被初始化的变量
// Concurrency-safe.
func Icon(name string) image.Image {
	mu.RLock()
	if icons != nil {
		icon := icons[name]
		mu.RUnlock()
		return icon
	}
	mu.RUnlock()
	// acquire an exclusive lock
	mu.Lock()
	if icons == nil { // NOTE: must recheck for nil
		loadIcons()
	}
	icon := icons[name]
	mu.Unlock()
	return icon
}
```
使用sync.Once实现函数的只执行一次
```go
var loadIconsOnce sync.Once
var icons map[string]image.Image
// Concurrency-safe.
func Icon(name string) image.Image {
	loadIconsOnce.Do(loadIcons)
	return icons[name]
}
```

## 协程模型
- growable stacks
动态增长的stack, 初始值2K？，
i5，8G系统测试：百万协程的pipeline， “taks 2.7076782s from pass across 1000000 stages”
- scheduling cost is cheaper
协程调度消耗很低， 协程的阻塞和唤醒很轻松；
i5，8G系统测试：两个协程通过unbuffer channel ping-pong 通讯："can ping-pong 1366232 pass per second"
runtime.Gosched()可以让出协程时间片。
调度机制？？？？深入研究
- GOMAXPROCS
协程是用户空间的m:n线程模型， n is GOMAXPROCS, 默认是CPU数runtime.NumCPU(),可以通过enviroment variable or runtime.GOMAXPROCS 修改
Goroutines that are sleeping or blocked in a communication do not need a thread at all. Goroutines that are blocked in IO, or other system calls, or are calling no-Go funcions, do need an OS thread, but GOMAXPROCS need not account for them.
这编程的角度，这个不应该暴露出来控制。协程调度系统应该最大化利用多核CPU性能。早期GO有缺陷，会集中一个CPU去运行。v1.5 默认NumCpu
这个数量的最优选择是个问题。线程体现是parallelism, 不是concurrence;很多程序内在不是并行的，提高线程数，反而因为线程资源消耗及操作系统调度，**跨线程的两个协程间的channel通讯** 而导致性能下降。
- goroutine has no identity
传统线程ID作为key 实现thread-local storage，goroutine 没有identity, 函数的运行是通过参数明确的，在每个协程中的行为是一致的。

## channel 实现
跨线程的两个协程间的channel通讯
