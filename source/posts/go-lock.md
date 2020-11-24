---
title: Go语言中的锁
date: 2020-01-15 14:57:01
tags:
    - go
photos:
      - ["https://sysummerblog.club/golang2.jpeg"]
---
总结一下这段时间学习的有关go语言中的锁的知识。
<!--more-->
多线程和多协程编程能够充分利用多核处理器的优势，是系统性能提升。但是有时也会给我们意外的“惊喜”，比如下面的计数器的例子
```go
var count int = 0
func main() {
	wg := sync.WaitGroup{}
	wg.Add(5)

	for i := 1; i <= 5; i++ {
		go func() {
			defer wg.Done()
			count++
		}()
	}

    wg.Wait()
	fmt.Println(count)
}
```
我们期待的结果是5，但事实上结果是不确定的。因为5个协程是并发执行的，当每个协程执行到`count += i`的时候count的值是不确定的，因此结果也是不确定的。

向例子中的count变量可以成为临界资源，当多个线程或协程对临界资源进行修改的时候就必须要“上锁”。使用锁可以保证在对临界资源进行修改的这一动作上各个并行的协程是“串行修改的”。

## Mutex 互斥锁
创建一个互斥锁
```go
mx := new(sync.Mutex)
```

1.mx.Lock()加锁，mx.Unlock()解锁
2. 锁是针对资源的，不是针对协程的，哪个协程加锁，就得哪个协程解锁。
3. 一个协程获得锁后其他协程无论读写只能等待，直到占用锁的协程释放锁
4. Lock()之前Unlock()会导致panic，还未Unlock()之后Lock()会产生死锁

还是上面的例子，我们稍加修改使用互斥锁

```go
var count int = 0
func main() {
    wg := sync.WaitGroup{}
	wg.Add(5)

	mx := sync.Mutex{}

	for i := 1; i <= 5; i++ {
		go func() {
			defer wg.Done()
			mx.Lock()
			count++
			mx.Unlock()
		}()
	}

	wg.Wait()
	fmt.Println(count)
}
```
输出的结果恒为5。

## RWMutex 读写锁
互斥锁的优势是简单，劣势是太粗暴了。假设对于一个临界资源，有的协程想写，有的只想读。如果在某一时刻只能允许一个协程读，其他的协程必须等待的话就显得效率太低了。因此读写锁出现了。

```go
rwmx := sync.RWMutex{}
```
1. rwmx.Lock()加写锁，rwmx.Unlock()解写锁。写锁排斥任何读锁和写锁
2. rwmx.RLock()加读锁，rwmx.RUnlock()解读锁。读锁排斥写锁但是不排斥读锁

下面的例子，主协程上了读锁，子协程也都可以获取读锁，整个程序和没有加锁是一样的
```go
var count int = 0
func main() {
	var wg sync.WaitGroup
	var mx sync.RWMutex

	mx.RLock()
	fmt.Println("主协程获取了读锁")

	wg.Add(2)

	for i := 1; i <= 2; i++ {
		go func(x int) {
			defer wg.Done()
			mx.RLock()
			fmt.Printf("协程%d获取了读锁\n", x)
			fmt.Println(count)
			mx.RUnlock()
			fmt.Printf("协程%d释放了读锁\n", x)
		}(i)
	}

	wg.Wait()
	mx.RUnlock()
	fmt.Println("主协程释放了读锁")
}
```
结果为：
```
主协程获取了读锁
协程2获取了读锁
0
协程2释放了读锁
协程1获取了读锁
0
协程1释放了读锁
主协程释放了读锁
```
下面的示例是主协程获取了写锁想修改count的值，子协程想获取读锁读取count的值
```go
var count int = 0
func PracticeCond5() {
	var wg sync.WaitGroup
	var mx sync.RWMutex

	mx.Lock()
	fmt.Println("主协程获取了写锁")

	wg.Add(2)

	for i := 1; i <= 2; i++ {
		go func(x int) {
			defer wg.Done()
			mx.RLock()
			fmt.Printf("协程%d获取了读锁\n", x)
			fmt.Println(count)
			mx.RUnlock()
			fmt.Printf("协程%d释放了读锁\n", x)
		}(i)
	}

	count++
	mx.Unlock()
	fmt.Println("主协程释放了写锁")
	wg.Wait()
}
```
结果为
```
主协程获取了写锁
主协程释放了写锁
协程2获取了读锁
1
协程2释放了读锁
协程1获取了读锁
1
协程1释放了读锁
```
可以看出子协程都是等到主协程将count值加一然后释放写锁后才获取的读锁。

## Cond 条件变量
条件变量适用于多个协程等待某一个资源就绪的情况。条件变量依赖于互斥锁或者读写锁。

使用互斥锁创建条件变量
```go
    wx   := new(sync.Mutex)
	cond := sync.NewCond(wx)
```
在使用的过程中有以下几个函数

* cond.Wait() 当前协程等待资源就绪，整个过程会阻塞当前协程，直到被通知资源就绪
* cond.Signal() 通知1个等待的协程，使其能够继续工作
* cond.Broadcast() 广播，通知所有的协程继续工作

需要强调以下几点

1. 调用cond.wait()之前一定要先`cond.L.Lock()`因为wait函数是
```go
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}
```
它是先将协程加入到监听队列然后再释放锁。等到资源就绪的时候再尝试获取互斥锁。

来看一个完整的例子
```go
func main() {
	mx   := new(sync.Mutex)
	cond := sync.NewCond(mx)
	wg   := sync.WaitGroup{}
	wg.Add(4)

	for i := 1; i <= 4; i++ {
		go func(x int) {
			defer wg.Done()
			cond.L.Lock()
			cond.Wait()
			fmt.Println(x)
			time.Sleep(time.Second)
			fmt.Printf("%d准备释放锁\n", x)
			cond.L.Unlock()
		}(i)
	}

	fmt.Println("start")
	time.Sleep(time.Second * 3)
	fmt.Println("下发第一个通知")
	cond.Signal()
	time.Sleep(time.Second * 3)
	fmt.Println("下发第二个通知")
	cond.Signal()
	time.Sleep(time.Second * 3)
	fmt.Println("开始广播")
	cond.Broadcast()

	wg.Wait()
	fmt.Println("退出")
}
```
结果为
```
start
下发第一个通知
1
1准备释放锁
下发第二个通知
2
2准备释放锁
开始广播
3
3准备释放锁
4
4准备释放锁
退出

```
