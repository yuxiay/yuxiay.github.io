---
title: "go并发编程"
description: 
date: 2024-07-13T21:32:56+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:   
     - go
categories:
     - go
---

## go并发编程

### 协程

```
package main

import (
	"fmt"
	"time"
)

// go并发编程
// 协程，内存占用小（2k），切换快
// go语言没有线程，go语言诞生之后只有协程可用 -goroutine  非常方便

// 主协程
// 主协程执行完，子协程会结束
func main() {
	// 主死随从
	// 匿名函数启动goroutine

	// 1.闭包
	// 2.for循环问题 for循环的时候，每个变量会重用
	// 每次for循环时，i变量会被重用，当我进行到第二轮for循环时候，这个i就变了
	for i := 0; i < 100; i++ {
		go func(i int) {
			fmt.Println(i)
		}(i)
	}

	fmt.Println("main")
	time.Sleep(1 * time.Second)
}

```

### 	go gmp调度原理

Go语言的GMP调度模型是其并发能力的核心，通过高效管理Goroutine（G）、操作系统线程（M）和逻辑处理器（P）的协作，实现了轻量级线程的高效调度。以下是对其原理的详细解析：

------

#### **GMP组件角色**

1. **Goroutine（G）**
   轻量级用户态线程，由Go运行时管理，栈初始仅几KB，动态扩展。通过`go`关键字创建，开销远小于OS线程。
2. **Machine（M）**
   对应操作系统线程，由内核调度。M必须绑定一个P才能执行G，否则休眠。阻塞的系统调用会触发M与P解绑，避免资源浪费。
3. **Processor（P）**
   逻辑处理器，管理G的上下文环境（如本地运行队列）。数量默认等于CPU核心数（由`GOMAXPROCS`设置），决定并行执行的G数量。

------

#### **调度流程**

1. **G的创建与分配**
   新G优先放入当前P的本地队列（避免锁竞争）；若本地队列满（容量256），则放入全局队列。
2. **M执行G**
   M需绑定P后，从P的本地队列获取G执行。若本地队列空，按以下顺序获取任务：
   - 从全局队列取（需加锁，每次取一批）。
   - 从其他P的本地队列**窃取一半任务**（Work-Stealing机制）。
3. **阻塞处理**
   - **系统调用阻塞**：M与P解绑，P被其他空闲M接管继续执行。原M执行完系统调用后，尝试获取P，若失败则将G放入全局队列，自身休眠。
   - **Channel/锁阻塞**：G进入等待队列，M释放P执行其他G。G被唤醒后重新放入P的队列。
4. **抢占式调度**
   - **协作式抢占**：在函数调用时插入检查点（如栈扩容），触发调度。
   - **信号式抢占**（Go 1.14+）：`sysmon`监控线程检测运行超过10ms的G，发送信号强制中断，实现抢占。

------

#### **关键机制**

1. **Work-Stealing**
   空闲P从其他P的本地队列窃取任务，平衡负载，避免部分P闲置。
2. **Hand Off机制**
   M阻塞时释放P，由其他M接管，确保CPU资源不被浪费。
3. **全局队列与本地队列**
   - 本地队列无锁操作，提升性能。
   - 全局队列作为备用，解决任务分配不均问题。

### 	互斥锁

```go
package main

import (
	"fmt"
	"sync"
)

// 锁，资源竞争
var total int
var wg sync.WaitGroup

// 锁能复制吗 本质是结构体是可以复制的，但是复制后就失去了锁的效果
var lock sync.Mutex

func add() {
	defer wg.Done()
	for i := 0; i < 1000000; i++ {
		lock.Lock()
		total += i // 竞争
		lock.Unlock()
	}
}

func sub() {
	defer wg.Done()
	for i := 0; i < 1000000; i++ {
		lock.Lock()
		total -= i
		lock.Unlock()
	}
}

func main() {
	wg.Add(2)
	go add()
	go sub()
	wg.Wait()
	fmt.Println(total)
}

// 资源竞争，当加载a的值时，如果俩个都同时加载值，那么a的值将变得不可确定
// 这时我们可以用锁来进行串行运算
```



如果只是想用简单的加减运算逻辑可以使用atomic包，进行原子操作

```go
atomic.AddInt64(&total, 1)
```



### 读写锁

  

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// 锁本质上是将并行的代码串行化了，使用lock肯定会影响性能
// 即使是设计锁，那么也应该尽量保证并行
// 我们有俩组协程，其中一组负责写数据，另一组负责读数据
// web系统中绝大多数场景是读多写少
// 虽然有多个goroutine，但是仔细分析我们发现，读协程是可以并发的，读和写应该串行，读和读之间也不应该并行
// 读写锁
func main() {
	var rwlock sync.RWMutex

	// 使主函数是在协程结束时结束
	var wg sync.WaitGroup

	wg.Add(11)

	// 写的goroutine
	go func() {
		defer wg.Done()
		time.Sleep(1 * time.Second)
		rwlock.Lock() // 加写锁，写锁会防止别的写锁获取和读锁获取
		defer rwlock.Unlock()
		fmt.Println("get write")
		time.Sleep(1 * time.Second)
	}()

	// 读的goroutine
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			for {
				rwlock.RLock() // 加读锁，读锁不会阻止别人的读
				time.Sleep(1 * time.Microsecond)
				fmt.Println("get read")
				rwlock.RUnlock()
			}
		}()
	}
	wg.Wait()
}

```



### goroutine之间的通信

```go
package main

import "fmt"

// goroutine之间的通信方式

/*
不要通过共享内存来通信，而要通过通信来实现内存共享
php, python, java, 多线程编程的时候，俩个goroutine之间的通信最常用是一个全局
也会提供消息队列的机制		消费者与生产者之间的关系
channel 再加上语法糖让使用channel更加简单
*/

/*
go 中channel的应用场景：
	1. 消息传递， 消息过滤
	2. 信号广播
	3. 事件订阅和广播
	4. 任务分发
	5. 结果汇总
	6. 并发控制
	7. 同步异步
*/

func main() {
	var msg1 chan string
	var msg2 chan string

	// channel的初始化值 如果为0的话，你放进去会阻塞

	// 有缓冲channel	适用于消费者与生产者之间的通信
	msg1 = make(chan string, 2)

	// 无缓冲channel	适用于通知，B要第一时间知道A是否已经完成
	msg2 = make(chan string, 0)

	// go有一种happen-before机制，可以保障无缓冲channel的写高于读进行操作
	go func(msg chan string) {
		data := <-msg
		fmt.Println(data)
	}(msg2)

	// 放值到channel中
	msg2 <- "hello"
	msg1 <- "world"

	// 取值
	data := <-msg1
	fmt.Println(data)
}

// 出现死锁
// 1. waitgroup 如果缺少了done调用
// 2. 无缓冲的channel 也容易出现

```

### for range 对channel进行遍历

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	var msg chan int

	msg = make(chan int, 2)

	go func(msg chan int) {
		for data := range msg {
			fmt.Println(data)
		}
		fmt.Println("all done")
	}(msg)

	// 放值到channel中

	msg <- 1
	msg <- 2

	close(msg) // 与其他语言有很大区别，可以关闭channel

	msg <- 3   // 已经关闭的channel不能再放值了
	d := <-msg // 已经关闭的channel可以再取值
	fmt.Println(d)

	time.Sleep(time.Second * 10)
}

```



### 单向channel的应用场景

```go
package main

import (
	"fmt"
	"time"
)

// 单向channel
// 默认情况下，channel是双向的
// 但是，我们经常一个channel作为参数进行传递，希望对方是单向使用

func producer(out chan<- int) {
	for i := 0; i < 10; i++ {
		out <- i * i
	}
	close(out)
}

func consumer(in <-chan int) {
	for v := range in {
		fmt.Println(v)
	}
}

func main() {
	//var ch1 chan int       // 双向channel
	//var ch2 chan<- float64 // 单向channel，只能写入float64数据
	//var ch3 <- chan bool	// 单向channel，只能读取

	// 可以将双向channel改为俩个单向channel
	// 不能将单向channel转为双向
	//c := make(chan int, 3)
	//var send chan<- int = c	// send-only
	//var read <-chan int = c	// recv-only
	//send <- 1
	//<-read

	ch := make(chan int)
	// 可以内部自动转换为单向
	go producer(ch)
	go consumer(ch)
	time.Sleep(time.Second * 5)
}

```



### 小练习题

```go
package main

import (
	"fmt"
	"time"
)

/*
使用俩个goroutine交替打印序列，一个goroutine打印数字，另外一个goroutine打印字母
*/

var number, letter = make(chan bool), make(chan bool)

func printNumber() {
	i := 0
	for {
		<-number
		fmt.Printf("%d%d", i, i+1)
		i = i + 2
		letter <- true
	}
}

func printLetter() {
	i := 0
	str := "ABCDEFGHIJKLMNOPQRSTUVWSYZ"
	for {
		<-letter
		if i >= len(str) {
			return
		}
		fmt.Printf(str[i : i+2])
		i = i + 2
		number <- true
	}
}

func main() {

	go printNumber()
	go printLetter()
	number <- true
	time.Sleep(time.Second * 15)
}

```



### select 监控goroutine运行

```go
package main

import (
	"fmt"
	"time"
)

// 监控goroutine的运行
// select 类似于switch case语句，但是select的功能和我们操作linux里面提供的io的select，poll，epoll
// select主要作用于多个channel

// 现在有个需求，我们现在有俩个goroutine都在执行，但是呢，我在主goroutine中，当某一个执行完成以后，这个时候我会立马知道这个

var done = make(chan struct{}) // channel是多线程安全的

func g1(ch chan struct{}) {
	time.Sleep(time.Second)
	ch <- struct{}{}
}

func g2(ch chan struct{}) {
	time.Sleep(time.Second)
	ch <- struct{}{}
}

func main() {
	g1Ch := make(chan struct{})
	g2Ch := make(chan struct{})
	go g1(g1Ch)
	go g2(g2Ch)

	// 我要监控多个channel，任何一个channel返回都知道
	// 1. 某一个分支就绪了就执行该分支
	// 2. 如果两个都就绪了，随机的, 目的是防止饥饿
	timer := time.NewTimer(time.Second * 5)
	for {
		select {
		case <-g1Ch:
			fmt.Println("g1 done")
		case <-g2Ch:
			fmt.Println("g2 done")
		case <-timer.C:
			fmt.Println("timeout")
			return
		}
	}

}

```



### 通过context解决goroutine信息传递问题

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// 通过context解决goroutine信息传递
// 渐进式的方式
// 需求：有一个goroutine监控cpu的信息
// 需求: 我们可以主动退出监控程序

var wg sync.WaitGroup
var stop = make(chan struct{})

func cpuInfo(ctx context.Context) {
	// 这里能拿到一个请求的id
	fmt.Printf("tracid: %s\r\n", ctx.Value("traceid"))

	defer wg.Done()
	for {
		select {
		case <-ctx.Done():
			fmt.Println("退出cpu监控")
			return
		default:
			time.Sleep(2 * time.Second)
			fmt.Println("cpu信息")
		}
	}
}

func main() {
	wg.Add(1)

	// context包提供了三种函数，WithCancel, WithTimeout, WithValue
	// 如果你的goroutine，函数中，如果希望被控制，超时，传值，但是我不希望影响我原来的接口信息时候，函数参数中第一个参数就尽量的要加上一个ctx

	// 1. WithCancel 主动cancel
	//ctx1, cancel1 := context.WithCancel(context.Background())
	// 子context调用父context的cancel也有效
	//ctx2, _ := context.WithCancel(context.Background())
	//go cpuInfo(ctx2)
	//cancel1()

	// 2. timeout 主动超时
	ctx, _ := context.WithTimeout(context.Background(), 6*time.Second)

	// 3. WithDeadline 在时间点cancel

	// 4. WithValue
	valueCtx := context.WithValue(ctx, "traceid", "123")

	go cpuInfo(valueCtx)

	wg.Wait()
	fmt.Println("监控完成")
}

```

