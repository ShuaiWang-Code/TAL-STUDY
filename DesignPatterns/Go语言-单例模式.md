# Go语言—单例模式

# 1 背景
在研读业务代码时，我发现自定义类型会采用“懒汉式-非线程安全”和“饿汉式”进行实例化，因此对单例模式产生兴趣，它是什么？原理是怎样的？解决了什么样的问题？应用场景如何？优缺点是什么？本文将借鉴OPP面向对象编程的设计模式思想，探讨Go语言实现单例设计模式。
# 2 什么是单例
**单例设计模式（Singleton Design Pattern）：** 面向对象语言中，一个类只允许创建一个对象（或者实例），那这个类就是一个单例类，这种设计模式就叫作单例设计模式，简称单例模式。

单例模式是最简单的设计模式之一，它提供了创建对象的最佳方式。

这种模式涉及到一个单一的结构体，该结构体负责创建自己的对象，同时确保只有单个对象被创建。

即**多次创建一个结构体的对象，得到的对象的存储地址永远与第一次创建对象的存储地址相同。**

# 3 为什么使用单例
## 3.1 资源访问冲突问题
goroutine1:
```go
 file1 = new(File)
 file1.write("abc")
```
goroutine2:
```go
 file2 = new(File)
 file2.write("defh")
```
多协程对同一文件的实例进行写操作时，就有可能存在写入的数据互相覆盖的情况。
![在这里插入图片描述](https://img-blog.csdnimg.cn/cbe2d09753784125a969e8b5f163d82e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NhdGtpbl93cw==,size_16,color_FFFFFF,t_70)

## 3.2 解决方式
- 1 加锁，同一个文件的所有实例共享一把锁
- 2 分布式锁，但实现安全可靠、无 bug、高性能较难
- 3 并发队列，多个线程同时往并发队列写数据，一个单独线程将并发队列中的数据写入文件
- **4 单例模式 ：**
	- 不用创建那么多实例，节省内存空间，
	- 节省系统文件句柄（对于操作系统来说，文件句柄也是一种资源）

## 3.3 全局唯一&应用场景
从业务概念上，如果有些**数据在系统中只应保存一份，那就比较适合设计为单例**。 

    简单概括应用场景如下：
    
    1.需要频繁实例化然后销毁的对象。 
    2.创建对象时耗时过多或者耗资源过多，但又经常用到的对象。 
    3.有状态的工具类对象。 
    4.频繁访问数据库或文件的对象。 

**具体应用场景如下：**
- **配置信息类**：在系统中，我们只有一个配置文件，当配置文件被加载到内存之后，以对象的形式存在，也理所应当只有一份。
- **日志应用**：共享的日志文件一直处于打开状态，因为只能有一个实例去操作。
- **数据库连接池**：用单例模式来维护频繁打开或者关闭数据库连接所引起的效率损耗
- **多线程的线程池**：采用单例模式对池中的线程进行控制。 
- **操作系统的文件系统**：一个操作系统只能有一个文件系统，也是单例模式实现的例子

## 3.4 设计思考
- 考虑对象创建时的线程安全问题
- 考虑是否支持延迟加载
- 考虑 getInstance() 性能是否高（是否加锁）
# 4 如何创建单例
单例模式是最简单的设计模式之一，它提供了创建对象的最佳方式。
这种模式涉及到一个单一的结构体，该结构体负责创建自己的对象，同时确保只有单个对象被创建。即多次创建一个结构体的对象，得到的对象的存储地址永远与第一次创建对象的存储地址相同。
## 4.1 饿汉式-线程安全
直接创建对象，线程安全。
缺点
- 在导入包/init的同时会创建该对象，并持续占有在内存中。
- 这样的实现方式不支持延迟加载（在真正用到再创建实例）
```go
package singleton

type School struct{}

var (
	instance *School
)

func init(){
　　instance = new(School)
}

func GetInstance() *School {
	return instance
}

```

## 4.2 懒汉式-非线程安全
优点是支持延迟加载
缺点是通过懒汉式-非线程安全方式创建单例，在并发下可能会多次创建
```go
package singleton

type School struct{}

var (
	instance *School
)

func GetInstance() *School {
	if instance == nil {
		instance = new(School)
	}
	return instance
}
```

## 4.3 懒汉式-线程安全
在非线程安全的基础上，利用Sync.Mutex进行加锁,保证线程安全，
但由于每次调用该方法都进行了加锁操作，在性能上相对不高效
```go
package singleton

type School struct {}

var (
	instance *School
	lock     *sync.Mutex = &sync.Mutex{}
)

func GetInstance() *School {
	lock.Lock()
	defer lock.Unlock()
	if instance == nil {
		instance = new(School)
	}
	return instance
}
```
## 4.4 双重检查
在懒汉式（线程安全）的基础上再进行忧化，通过判断来减少加锁的操作。保证线程安全同时不影响性能。这种实现方式解决了懒汉式并发度低的问题。
**设计思路：第一次判断不加锁，第二次加锁保证线程安全，一旦对象建立后,获取对象就不用加锁了**
```go
package singleton

var (
	instance *School
	lock     sync.Mutex
)

func GetInstance() *School {
	if instance == nil {
		lock.Lock() //第一次判断不加锁，第二次加锁保证线程安全，一旦对象建立后,获取对象就不用加锁了
		if instance == nil {
			instance = new(School)
		}
		lock.Unlock()
	}
	return instance
}
```

## 4.5 once写法：推荐采用

利用 sync.Once 方法Do，确保Do中的方法只被执行一次的特性，创建单个结构体实例。
使用Do方法也巧妙的保证了并发线程安全。
```go
package singleton

var (
	instance *School
	once     sync.Once
)

func GetInstance() *School {
	once.Do(func() {
		instance = new(School)
	})
	return instance
}
```

其中Once源码利用atomic操作实现f函数只执行一次
- 进行加锁，再做一次判断，如果没有执行，则进行标志已经执行并调用该方法
- 判断是否执行过该方法，如果执行过则不执行
```go
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 1 {  ////判断是否执行过该方法，如果执行过则不执行
	   return
	}
	// Slow-path.
	o.m.Lock()
	defer o.m.Unlock()   ////进行加锁，再做一次判断，如果没有执行，则进行标志已经执行并调用该方法
	if o.done == 0 {
	   defer atomic.StoreUint32(&o.done, 1)
	   f()
	}
 }
```

# 5 采用Once实现单例模式

```go
package main

import (
	"fmt"
	"strconv"
	"sync"
)

type School struct {
	name string
	id   int
}

var (
	instance *School
	once     sync.Once
	wg       sync.WaitGroup
)

func GetInstance(name string, id int) *School {
	once.Do(func() {
		fmt.Println("------------init-----------")
		instance = new(School)
	})
	instance.name = name
	instance.id = id
	return instance
}

func main() {

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(seq int) {
			defer wg.Done()
			instance = GetInstance("xiaobai"+strconv.Itoa(seq), 25)
			fmt.Printf("gonum: %s, address: %p, name: %s, id: %d\n",
				strconv.Itoa(seq), instance, instance.name, instance.id)
		}(i)
	}
	
	wg.Wait()
	fmt.Println("end!")
}

```
运行结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/fe08441a93174b23b2a33a897d84dd2f.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NhdGtpbl93cw==,size_16,color_FFFFFF,t_70)



# 参考资料

1 [https://blog.csdn.net/TCatTime/article/details/106882600](https://blog.csdn.net/TCatTime/article/details/106882600)

2 [https://blog.csdn.net/qq_37703616/article/details/81989889](https://blog.csdn.net/qq_37703616/article/details/81989889)

3 [https://github.com/tmrts/go-patterns/blob/master/creational/singleton.md](https://github.com/tmrts/go-patterns/blob/master/creational/singleton.md)

