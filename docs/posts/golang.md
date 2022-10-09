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
## golang并发

### 进程、线程和协程

### goroutin

### GMP调度模型


## golang GC(garbage collection 垃圾回收)

### 标记消除法

### 三色标记法

## 插入写屏障/删除写屏障

### 混合写屏障


## golang网络编程

### https

### websocket


## golang常用框架

### Gin

#### 中间件

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