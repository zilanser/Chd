---
layout: Post
title: golang学习笔记
subtitle: 每天学习一点点
author: zilanser
date: 2022-10-5
headerImage: /img/in-post/2021-12-24/header.jpg
tags:
  - test
  - tag with space
---

## golang基础

### slice

1. golang数组是值类型，赋值和函数传参操作都会复制整个数组数据，而切片是引用类型，自身为结构体类型，结构体内部包括底层的数组，len(切片长度)，cap(切片容量)<br>
2. 如果slice为nil，len = cap = 0<br>
3. 0 <= len <= cap<br>
4. 使用append操作切片，可能出现以下情况:<br>
> 情况一：append后未超过cap限制，切片容量不变<br>
> 情况二：append后超过cap限制，此时如果原切片容量小于1024，切片容量翻倍；如果大于1024，切片容量增长1/4<br>

5. 切片扩容也可能出现不同的情况：<br>
> 情况一：扩容量没有超过引用数组的长度，不会重新分配数组，扩容前后的数组是同一个，数组地址也不会变。此时修改新切片也会影响到旧切片和引用数组。<br>
> 情况二：扩容量超过引用数组的长度，重新分配数组，golang会创建一个新数组，将原数组copy到新数组，再执行append操作，此时数组地址就和原数组不同，修改新切片也不会影响到旧切片和引用数组。<br>
>
> **注意：当旧数组上存在多个切片引用时，情况一可能会产生bug，所以建议尽量避免情况一，使用情况二**

6. 切片截取：s = s[low : high : max] 切片的三个参数的切片截取的意义为 low 为截取的起始下标（含）， high 为窃取的结束下标（不含 high），max 为切片保留的原切片的最大下标（不含 max）；即新切片从老切片的 low 下标元素开始，len = high - low, cap = max - low；high 和 max 一旦超出在老切片中越界，就会发生 runtime err，slice out of range。另外如果省略第三个参数的时候，第三个参数默认和第二个参数相同，即 len = cap

**注意：Go 数组是值类型，赋值和函数传参操作都会复制整个数组数据**

### map

1. map底层存储方式为数组，在存储时key不能重复，当key重复时，value进行覆盖，我们通过key进行hash运算（可以简单理解为把key转化为一个整形数字）然后对数组的长度取余，得到key存储在数组的哪个下标位置，最后将key和value组装为一个结构体，放入数组下标处

e.g:
```go
    length = len(array) = 4
    hashkey1 = hash(xiaoming) = 4
    index1  = hashkey1% length= 0
    hashkey2 = hash(xiaoli) = 6
    index2  = hashkey2% length= 2
```

2. Go语言中判断map中键是否存在的特殊写法:
```go
value, ok := map[key]
```
3. map也支持在声明的时候填充元素
```go
userInfo := map[string]string{
    "username": "pprof.cn",
    "password": "123456",
}
```
4. Go语言中使用for range遍历map
**注意： 遍历map时的元素顺序与添加键值对的顺序无关**

5. 按照指定顺序遍历map
> 可以先将map的键取出存放在切片中，在对切片进行排序，最后根据排序后的切片遍历map
```go
 func main() {
    rand.Seed(time.Now().UnixNano()) //初始化随机数种子

    var scoreMap = make(map[string]int, 200)

    for i := 0; i < 100; i++ {
        key := fmt.Sprintf("stu%02d", i) //生成stu开头的字符串
        value := rand.Intn(100)          //生成0~99的随机整数
        scoreMap[key] = value
    }
    //取出map中的所有key存入切片keys
    var keys = make([]string, 0, 200)
    for key := range scoreMap {
        keys = append(keys, key)
    }
    //对切片进行排序
    sort.Strings(keys)
    //按照排序后的key遍历map
    for _, key := range keys {
        fmt.Println(key, scoreMap[key])
    }
}
```

6. 使用delete()函数删除键值对：
```go
delete(map, key)
```
**map容量只增不减，delete()只能删除键值对，不会删除map内存，想要得到一个容量更小的map只能创建一个新的map**

7. map实现原理
> map同样也是数组存储的，每个数组下标处存储的是一个bucket（桶）类型，每个bucket中可以存储8个kv键值对，当每个bucket存储的kv对到达8个之后，会通过overflow指针指向一个新的bucket

> **当往map中存储一个kv对时，通过k获取hash值，hash值的低八位和bucket数组长度取余，定位到在数组中的那个下标，hash值的高八位存储在bucket中的tophash中，用来快速判断key是否存在(即不是通过真值进行比较)，key和value的具体值则通过指针运算存储**

> 对于kv键值对的存放，不是k1v1，k2v2..... 而是k1k2...v1v2...，即将所有键打包在一起，然后将所有值打包在一起，这么做的原因是kv的长度不同，如果按照kv格式存放，则考虑内存对齐v会占用更多内存<br>
> e.g: map[int64]int8,如果按照kv格式存放，则考虑内存对齐v也会占用int64，而按照后者存储时，8个v刚好占用一个int64<br>

### 结构体
1. 只有当结构体实例化时，才会真正地分配内存。也就是必须实例化后才能使用结构体的字段。结构体本身也是一种类型，我们可以像声明内置类型一样使用var关键字声明结构体类型<br>
2. 在定义一些临时数据结构等场景下还可以使用匿名结构体:<br>
```go
func main() {
    var user struct{Name string; Age int}
    user.Name = "Chd"
    user.Age = 18
}
```
3. 通过new关键字对结构体进行实例化，得到的是结构体的地址。例如var p2 = new(person)，得到的p2是一个结构体指针，另外，在golang中支持对结构体指针直接使用.来访问结构体的成员<br>
4. 使用&对结构体进行取地址操作相当于对该结构体类型进行了一次new实例化操作<br>
5. 初始化结构体时可以只用值，不用键,不过使用这种方式需要注意以下几点<br>
> 1.必须初始化结构体的所有字段。<br>
> 2.初始值的填充顺序必须与字段在结构体中的声明顺序一致。<br>
> 3.该方式不能和键值初始化方式混用。<br>

6. 类型别名：<br>
> 声明：type 类型别名 = 类型    e.g: type newInt = int<br>
> 类型别名只在代码中存在，代码编译完成后不会有相应类型（type newInt = int，newInt类型只会在代码中存在，编译完成时并不会有newInt类型）<br>

7. 结构体允许其成员字段在声明时没有字段名而只有类型，这种没有名字的字段就称为匿名字段，匿名字段默认采用类型名作为字段名，结构体要求字段名称必须唯一，因此一个结构体中同种类型的匿名字段只能有一个<br>
8. 结构体中字段大写开头表示可公开访问，小写表示私有（仅在定义当前结构体的包中可访问）

## 函数
### golang函数特点：
> • 无需声明原型   
> • 支持不定变参  
> • 支持多返回值  
> • 支持命名返回参数  
> • 支持匿名函数和闭包  
> • 函数也是一种类型，一个函数可以赋值给变量  
> • 不支持 嵌套 (nested) 一个包不能有两个名字一样的函数  
> • 不支持 重载 (overload)  
> • 不支持 默认参数 (default parameter)  

### 参数
1. 当两个或多个连续的函数命名参数类型相同，则除最后一个参数需要标明类型，其他的都可以省略

2. 函数传参无论是值传递还是引用传递，都是传递变量的副本，不过值传递是值的拷贝，引用传递是地址的拷贝
    **map、slice、chan、指针、interface默认以引用的方式传递**
3. golang可变参数本质上是一个slice，一个函数中只能有一个且只能为最后一个。在参数赋值时可以不用用一个一个的赋值，直接传一个切片，只需要在形参的类型前加...即可
使用切片对象作为变参时，必须展开，举例如下
```go
func test(s string, n ...int){
    var x int
    for _, i := range n {
        x += i
    }
}

func main() {
    s := []int{1, 2, 3}
    test("sum: %d", s...)    // slice... 展开slice
}
```
### 返回值
1. 没有参数的return语句返回各个返回变量的当前值。这种用法被称作“裸”返回
```go
func add(a, b int) (c int) {
    c = a + b
    return //返回c
}
func main() {
    var a, b int = 1, 2
    c := add(a, b)
}
```

2. 多返回值可以作为另一个参数调用的传参
```go
func test() (int, int) {
    return 1, 2
}

func add(x, y int) int {
    return x + y
}

func sum(n ...int) int {
    var x int
    for _, i := range n {
        x += i
    }
    return x
}

func main() {
    add(test())
    sum(test())
}
```

### 命名返回参数
1. 命名返回参数可以看做与形参类似的局部变量，可以由return隐式返回；同时命名返回参数会被函数内的同名局部变量遮蔽，这时需要显示地返回

2. 显式 return 返回前，会先修改命名返回参数
```go
func add(x, y int) (z int) {
    defer func() {
        println(z) // 输出: 203
    }()

    z = x + y
    return z + 200 // 执行顺序: (z = z + 200) -> (call defer) -> (return)
}

func main() {
    println(add(1, 2)) // 输出: 203
}
```
### 闭包

概念：**闭包就是连接函数内部和外部的桥梁，由函数和声明了该函数的此法环境组成，这个环境包括闭包创建时能访问到的所有局部变量**   

用途：  
> 1. 读取函数内部的局部变量
> 2. 让变量的值一直存在于内从中，不被GC释放

特点：  
> 1. 闭包一定有嵌套函数
> 2. 外部函数一定有局部变量，前内部函数使用了这个局部变量
> 3. 内部函数会使用return返回外部，如果不返回内部函数，外部变量没有访问到这个内部函数，就产生不了闭包，没有外界的介入，外部函数在执行完后就会被GC掉，也无法让局部变量保存在内存中

### defer(延迟调用)
特性：
> 1. defer用于注册延迟调用
> 2. defer在return前处理，前只在return前处理，e.g:
```go
func add(x, y int) (z int) {
    defer func() {
        println(z) // 输出: 203
    }()

    z = x + y
    return z + 200 // 执行顺序: (z = z + 200) -> (call defer) -> (return)
}

func main() {
    println(add(1, 2)) // 输出: 203
}
```
3. 多个defer按先进后出的方式执行
4. defer语句中的变量在声明时就已经决定了

### 异常处理
golang没有结构化异常，使用panic抛出异常，recover捕获异常  
异常的使用场景简单描述：Go中可以抛出一个panic的异常，然后在defer中通过recover捕获这个异常，然后正常处理

#### panic
> 1. 函数如果执行了panic语句，会终止其后要执行的代码，如果函数中存在要执行的defer列表，按照defer的运行顺序执行
> 2. 直到goroutine整个退出，并报告错误

#### recover
> 1. 用来控制一个goroutine的panic行为，从而影响应用的行为  
> 2. 一般调用建议：  
>    （1）在defer函数中，通过recever来终止一个goroutine的panicking过程，从而恢复正常代码的执行  
>    （2）获取panic发出的error  
> 3. 利用recover处理panic，defer必须在panic之前定义，且recover只能在defer中才生效，否则当panic时，recover无法捕获到panic，无法防止panic扩散  
> 4. recover处理异常后，程序不会回到panic那个点，而是继续到defer之后那个点

1. 延迟调用中引发的错误，可被后续延迟调用捕获，但仅最后一个错误可被捕获
2. 捕获函数 recover 只有在延迟调用内直接调用才会终止错误，否则总是返回 nil。任何未捕获的错误都会沿调用堆栈向外传递
```go
import "fmt"

func test() {
    defer func() {
        fmt.Println(recover()) //有效
    }()
    defer recover()              //无效！
    defer fmt.Println(recover()) //无效！
    defer func() {
        func() {
            println("defer inner")
            recover() //无效！
        }()
    }()

    panic("test panic")
}

func main() {
    test()
}

```
输出：
```
    defer inner
    <nil>
    test panic
```

### 测试
#### 单元测试
**go test命令是一个按照一定约定和组织的测试代码的驱动程序**，测试文件（xx_test.go）不会被go build到最终的可执行文件中

xx_test.go文件中包含三种类型的函数：
| 类型 | 格式 | 作用
|------|------|------|
| 单元测试函数 | 函数前缀名为Test | 测试程序的逻辑是否正确
| 基准测试函数 | 函数前缀名为Benchmark | 测试程序的性能
| 示例函数 | 函数前缀名为Example | 为文档提供示例文档

go test命令会遍历所有xx_test.go文件中符合上述格式的函数，并创建临时的main包调用相应的测试函数，然后运行，报告测试结果，最后销毁临时创建的文件

golang单元测试对文件名和方法名要求很严格  
> 1. 文件名必须以xx_test.go命名  
> 2. 方法必须是Test[^a-z]开头  
> 3. 方法参数必须 t *testing.T  
> 4. 使用go test执行单元测试  

## 方法


## golang并发
### 并发介绍
#### 进程、线程和协程
1. 进程是程序的执行过程，是系统进行资源分配和调度的独立单位  
2. 线程是进程的一个执行实体，是cpu调度的，比进程更小的能独立运行的基本单位  
3. 协程本质上是用户级线程，由用户自己调度控制，拥有独立的栈空间，共享堆空间  

4. 一个进程可以创建和销毁多个线程，同一个进程下的多个线程可以并发执行  
5. 协程是运行在用户态的轻量级线程，一个线程上可以运行多个协程

goroutine主张通信共享内存，而不是共享内存进行通信

### goroutine 
1. 多个goroutine是并发执行且随机调度的
2. 主goroutine结束了，其下的子goroutine也会结束

### GMP调度模型
1. GMP调度系统是golang运行时层面的实现，是go语言自己实现的一套调度系统，区别于操作系统调度线程

> 1. G goroutine，里面存放本goroutine的信息以及于P的绑定信息 
> 2. M machine，go运行时对操作系统内核线程的虚拟，与内核线程是一一对应的关系，同时与P也是一一对应的关系，一个goroutine最终要在M上运行，一个M上可以运行多个goroutine
> 3. P processor，调度器，维护一个本地的goroutine队列，保存当前goroutine运行的上下文环境（函数指针，堆栈地址及地址边界），同时管理部分内存的分配，M在P上取goroutine运行  

GMP之间的关系：P管理着一组G挂载在M上运行。

2. GMP模型：

![golang GMP模型](/img/golang/golangGMP模型.jpg)

M代表一个工作线程，在M上有一个P和G，P是绑定到M上的，G是通过P的调度获取的，在某一时刻，一个M上只有一个G（g0除外）。在P上拥有一个本地G队列，里面是已经就绪的G，是可以被调度到线程栈上执行的协程，称为运行队列

3. 全局GMP分布：

![golang GMP分布](/img/golang/golangGMP分布.jpg)

每个进程都有一个全局的G队列，也拥有P的本地执行队列，同时也有不在运行队列中的G。如正处于channel的阻塞状态的G，还有脱离P绑定在M的(系统调用)G，还有执行结束后进入P的gFree列表中的G等等。

| GMP | 状态 | 描述 |
|------|------|------|
| G | Gidle | 刚刚被分配并且还没有被初始化，值为0，为创建goroutine后的默认值 |
| G | Grunnable | 没有执行代码，没有栈的所有权，存储在运行队列中，可能在某个P的本地队列或全局队列中(如上图) |
| G | Grunning | 正在执行代码的goroutine，拥有栈的所有权(如上图) |
| G | Gsyscall | 正在执行系统调用，拥有栈的所有权，与P脱离，但是与某个M绑定，会在调用结束后被分配到运行队列(如上图) |
| G | Gwaiting | 被阻塞的goroutine，阻塞在某个channel的发送或者接收队列(如上图) |
| G | Gdead | 当前goroutine未被使用，没有执行代码，可能有分配的栈，分布在空闲列表gFree，可能是一个刚刚初始化的goroutine，也可能是执行了goexit退出的goroutine(如上图) |
| G | Gcopystac | 栈正在被拷贝，没有执行代码，不在运行队列上，执行权在 |
| G | Gscan | GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在 |
| P | Pidle | 处理器没有运行用户代码或者调度器，被空闲队列或者改变其状态的结构持有，运行队列为空 |
| P | Prunning | 被线程 M 持有，并且正在执行用户代码或者调度器(如上图) |
| P | Psyscall | 没有执行用户代码，当前线程陷入系统调用(如上图) |
| P | Pgcstop | 被线程 M 持有，当前处理器由于垃圾回收被停止 |
| P | Pdead | 当前处理器已经不被使用 |
| M | 自旋线程 | 处于运行状态但是没有可执行goroutine的线程(如下图)，数量最多为GOMAXPROC，若是数量大于GOMAXPROC就会进入休眠 |
| M | 非自旋线程 | 处于运行状态有可执行goroutine的线程 |

4. GMP调度流程：  
> 1. 新创建的G会保存在P的本地队列中，如果这个本地队列满了，就会保存在全局队列中
> 2. M会从绑定的P上取一个可执行的G来执行，如果P的本地队列为空，则会到全局队列取，如果全局队列也为空，则会到其他MP组合中“偷取”可执行的G来执行
> 3. 当M执行某一个G时发生了阻塞(或者由于系统调用陷入内核态)，runtime会创建一个新的M（或者复用空闲的M），管理阻塞G的P会将其他的G挂载扫新的M上。当G阻塞完成或者死亡时，回收就的M
> 4. 当G阻塞完成时，这个G会尝试获取一个空闲的P，加入这个P的本地队列。如果获取不到P，就会进入到全局队列中，其所在的M会进入休眠状态，加入空闲线程中

特殊的M0和G0：  
M0：M0是程序在启动后创建的编号为0的主线程，其对应的实例在全局变量runtime.m0中，不需要在heap上分配，M0的作用是初始化操作和启动第一个G，之后就和其他的M一样  
G0：G0在每个M启动后都会创建一个，G0仅负责调度，不指向任何可执行的函数。在调度或者系统调用时会使用G0的空间栈。其中全局变量的G0是属于M0的

### channel
**channel，管道，引用类型（空值为nil），goroutine之间的通信机制，遵循先进先出的规则，以保证收发数据的顺序**

#### 关闭channel
1. 只有在通知接收方goroutine所有的数据都发送完毕的时候才需要关闭通道。通道是可以被垃圾回收机制回收的，它和关闭文件是不一样的，在结束操作之后关闭文件是必须要做的，但关闭通道不是必须的
2. 关闭后的通道有以下特点：
> 1. 向关闭的channel发送值会报panic
> 2. 对一个关闭的通道进行接收会一直获取值直到通道为空。
> 3. 对一个关闭的并且没有值的通道执行接收操作会得到对应类型的零值。
> 4. 关闭一个已经关闭的channel会报panic
 
## 常用标准库

### Context
Context类型专门用来简化对于单个请求的多个goroutine之间与请求域的数据、取消信号、截止时间等相关操作。

**Context与子孙Context之间是树结构，当取消了当前Context，会将这个节点下所有可以取消的Context取消**  

**一个接口，四个方法，六个函数**  
Context是一个接口，该接口定义了四个需要实现的方法：
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <- chan struct{}
    Err() error
    Value(key interface{}) interface{}
```
其中定义了四个方法：
```
1. Deadline()返回当前Context被取消的时间，即工作完成的截止时间

2. Done()返回一个Channel，这个Channel会在当前工作完成或者上下文被取消后关闭，多次调用Done方法返回同一个Channel

3. Err()返回当前Context结束的原因，它只会在Done返回的Channel被关闭时才会返回非空值；如果当前Context被取消就会返回Canceled错误；如果当前Context超时就会返回DeadlineExceeded错误

4. Value()从Context中返回键对应的值，对于同一个上下文来说，多次调用Value并传入相同的Key会返回相同的结果，该方法仅用于传递跨API和进程间求域的数据
```
六个方法：
1. Backgroud()，该方法返回一个不可取消，没有deadline，没有任何value，实现了Context的emptyCtx，主要用于main函数，初始化和初始代码中，作为最顶层的Context。

2. TODO()，该方法返回一个不可取消，没有deadline，没有任何value，实现了Context的emptyCtx，当不知道使用什么Context时可以使用TODO()

3. WithCancel:
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

4. WithDeadline:
```go
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
```

5. WithTimeout:
```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

6. WithValue：
```go
func WithValue(parent Context, key, val interface{}) Context
```
## golang GC(garbage collection 垃圾回收)

### 标记消除法

### 三色标记法

## 插入写屏障/删除写屏障

### 混合写屏障

## golang网络编程
### Socket
Socket是TCP/IP协议族通信的中间软件抽象层，用户只需要调用Socket相关函数，让Socket去组织复杂的TCP/IP协议族进行通信

| Socket模式 | 对应传输层协议 | 特性 |
|------|------|------|
| 流式 | TCP | 面向连接的，较慢，可靠传输 |
| 数据报式 | UDP | 无连接的，较快，不可靠传输 |

#### TCP编程
**TCP协议：传输控制协议/网间协议，是一种面向连接的，可靠的，基于字节流的传输层通信协议**

1. TCP服务端处理流程：
> 1. 监听窗口
> 2. 接收客户端连接请求
> 3. 创建goroutine处理请求

2. TCP客户端处理流程：
> 1. 发起与服务端连接的请求
> 2. 进行数据接收发送
> 3. 关闭连接

#### UDP编程
**UDP协议：用户数据报协议，无连接的传输层协议**

### Https编程
**Http，超文本传输协议。详细规定了浏览器和万维网服务器之间互相通信的规则，通过因特网传送万维网文档的数据传送协议。**  
**Https，超文本传输安全协议。Https基于Http进行进行通信，但利用了SSL/TSL加密数据包**  

#### web服务器工作流程
> 1. 客户端通过TCP/IP协议建立到服务器的连接
> 2. 客户端向服务器发起Https请求包，请求服务器里的资源 
> 3. 服务器向客户端发起Https应答包，如果请求的资源包中包括动态语言的内容，服务器会调动动态语言引擎处理，再将数据返回客户端
> 4. 客户端与服务器断开连接，客户端解释HTML文档，渲染出应答包中的资源  

### Websocket编程
> 1. WebSocket是一种在单个TCP连接上进行全双工通信的协议
> 2. WebSocket使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据
> 3. 在WebSocket API中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输

## golang常用框架

### Gin

#### 中间件

### Gorm


## golang内存分配

### 内存分配三大组件

> **mheap**<br>
> golang在程序启动时，首先会向操作系统申请一大块内存，并交由mheap结构全局管理。mheap会将这一大块内存，切分成不同规格的小内存块，称为mspan，根据规格大小不同，mspan大概有70类左右

> **mcentral**<br>
> golang在程序启动时，会初始化很多的mcentral ，每个mcentral只负责管理一种特定规格的mspan。<br>
> 相当于mcentral实现了在mheap的基础上对mspan的精细化管理

> **mcache**<br>
> mcentral在Go程序中是全局可见的，因此如果每次协程来mcentral申请内存的时候,都需要加锁,那频繁的加锁释放锁开销是非常大的，而mcache就是作为mcentral的二级代理来缓冲这种压力<br>
> 众所周知，每个线程M会绑定给一个处理器P。而在单一粒度的时间里只能做多处理运行一个goroutine，每个P都会绑定一个叫mcache的本地缓存。<br>
> 当需要进行内存分配时，当前运行的goroutine会从mcache中查找可用的mspan。从本地mcache里分配内存时不需要加锁，这种分配策略效率更高。

### mspan供应链
> mcache的mspan数量不足时，mcache会从mcentral申请更多的mspan；同样的，如果mcentral的mspan数量也不够的话，mcentral也会向它的上级mheap申请mspan；如果mheap里的mspan也无法满足程序的内存申请，mheap会从操作系统中申请<br>
>以上的供应流程，只适用于内存块小于64KB的场景，原因在于Go没法使用工作线程的本地缓存mcache和全局中心缓存mcentral上管理超过64KB的内存分配，所以对于那些超过64KB的内存申请，会直接从堆上（mheap）上分配对应的数量的内存页（每页大小是 8KB）给程序

![Image Example](/img/golang/golang内存分配.png)

### 堆内存和栈内存
根据内存管理（分配和回收）方式的不同，可以将内存分为堆内存和栈内存<br>

> 堆内存：由内存分配器和垃圾收集器负责回收<br>
> 由于多个线程或者协程都有可能同时从堆中申请内存，因此在堆中申请内存需要加锁，避免造成冲突。并且堆内存在函数结束后，需要GC（垃圾回收）的介入参与，如果有大量的 GC 操作，会导致程序性能下降<br>

> 栈内存：由编译器自动进行分配和释放<br>
> 每个栈内存都是由线程或者协程独立占有，因此从栈中分配内存不需要加锁，并且栈内存在函数结束后会自动回收，性能相对堆内存要高

### 逃逸分析
在堆内存和栈内存的分析中，可以看出：为了减少GC的压力，提高程序的性能，应当尽量减少内存在堆上分配<br>
但是判断一个变量是在堆上分配内存还是在栈上分配内存相对困难，对程序员要求较高，这时就需要引入golang逃逸分析

首先要明确几点：

1. golang的逃逸分析是在编译期完成的。
2. Golang的逃逸分析只针对指针。一个值引用变量如果没有被取址，那么它永远不可能逃逸。

在Go中，函数传参时为了避免额外的copy开销，一般选择传结构体指针。但是传值的内存是在栈上分配的，而传指针结构体可能会逃逸到堆上，造成额外的GC压力。所以在变量发生逃逸时应该传值，否则就传指针，但是判断变量是否逃逸很麻烦，而且不利于代码风格的统一，所以个人认为传值较好。

验证某个函数的变量是否发生逃逸的方法有两个:
1. go run -gcflags "-m -l" (-m打印逃逸分析信息，-l禁止内联编译)
2. go tool compile -S main.go | grep runtime.newobject（汇编代码中搜runtime.newobject指令，该指令用于生成堆对象

## References
1. Go语言中文文档(https://www.topgoer.com)
2. 一篇文章讲清Go的内存布局和分配原理(https://blog.csdn.net/kevin_tech/article/details/121391666)