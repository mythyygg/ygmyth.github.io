---
title: "[总结] Golang学习总结"
date: 2020-10-04T20:44:02+08:00
draft: false
---
最近学习了Go语言。由于它的语法是C-like的，加上基本不存在依赖问题，有问题随时查官方文档，所以学习起来进度还是挺快的。学习结束，总结一些Tips：

## 程序架构

- 在Go中，程序是以包为单位组织的，每个文件夹为一个包，每个程序的开头会有package语句声明自己属于哪个包。会编译成二进制文件的程序会声明自己为package main

- 包级公开变量以首字母大写作为标识，而私有变量以小写字母开头。不同于一般语言的下划线声明私有变量

- init函数用来对程序进行初始化，它在main函数之前执行

- 接口的方法集：

    类型*S的可调用方法集包含接受者为*S或S的所有方法集

    类型S的可调用方法集只包含接受者为S的方法集，不包含接受者为*S的方法集

- 内嵌对象的方法集提升：

    如果S包含一个匿名字段K，S和*S的方法集都包含接受者为K的方法提升

    如果S包含一个匿名字段*T，S和*S的方法集都包含接受者为T或者*T的方法提升

- Go语言函数返回局部变量指针，局部变量内存不会随函数栈帧弹出而销毁，不同于C/Cpp会造成野指针。原因是Go编译器会自动决定把一个变量放在栈还是放在堆，编译器会做逃逸分析escape analysis，当发现变量的作用域没有跑出函数范围，就可以在栈上，反之则必须分配在堆。所以不用担心会不会导致memory leak，因为GO语言有强大的垃圾回收机制
```go
 //go
 package main

 import (
 	 "fmt"
 )

 func foo() *int {
 	 var i int64 = 199000000000
 	 return &i
 }
 func main() {
 	 pointer := foo()
 	 fmt.Println(*pointer)
 }
 ```
- 异常处理：
```go
defer func() {
	if r := recover(); r != nil {
		log.Printf("Runtime error caught: %v", r)
	}
}()
```
## 语法

- Go中的指针与值的使用非常不严格，大多时候不需要明确进行解引操作，编译器会自动生成正确的语句。重要的是引用类型和值类型（严格来讲Go全部是值类型，引用类型只是底层值类型指针的封装），引用类型：hashmap, channel, interface, slice, function

- 对于可迭代对象，range返回的是元素的副本

- 对于花括号包裹的数据类型的定义，单行时只需在每个元素间用逗号分割，多行时强制每个元素后加拖尾逗号
```go
a := map[int]int{1: 1, 2: 2}
  
a := map[int]int{
    1: 1,
    2: 2,
}
```
- Go中的闭包内层函数一定是匿名函数，JavaScript也是这样：
```go
package main 
  
import (
	"fmt"
)
  
func test() {
    test_v := 10
    func() {
        test_v = 20
    }()
    fmt.Println(test_v)
}
  
func main() {
    test()
}
```
  
- Go语言函数声明处指定函数返回值后，函数体内部无需声明该返回值

- Go语言中语法很多与C/Cpp相反如，据说这是Thompson总结C语言经验得出的更优设计：

    变量声明：var a int

    指针声明：var a *int，指针解引时相同：*a = 1

    别名：type myint int，C/Cpp是本名在前，别名在后

- make函数用于切片，哈希表，通道的初始化，并返回对象；new函数也是初始化上述三个类型，但返回指针，相当于C/Cpp的malloc/new

## 数据类型/结构

- Go的数组完全基于值语义，C/Cpp数组传递时基于引用语义，结构体中定义时基于引用语义

- Go的切片行为：
```go
import (
      "fmt"
)

type stu struct {
    tslice []int
    sk     int
}

func (this stu) testfunc() {
    //this.tslice = append(this.tslice, 10)		
    this.tslice[0] = 10	
}

func main() {
    a := stu{[]int{1, 2, 3, 4}, 1}
    a.testfunc()
    fmt.Println(a)
}
  
//append函数对切片的修改外部不可见
```
原因：

切片的内部实现：
```go
struct Slice {
      byte*    array; 
      uintgo    len; 
      uintgo    cap; 
};
```
函数内调用append修改了len/cap，但是结构体类型中的值为拷贝，故仅局部可见。不同于slice[0] = 1的赋值是使用引用类型结构体中指针变量赋值

- Go的原生字符串用飘号包裹，Json/Xml标签也用飘号包裹

- 数组与切片的声明区别，中括号内是否有值：`array := [...]string{"a", "b", "c"} , slice := []string{"a", "b"}`。如果两个切片共享同一个底层数组的话，一个修改元素会影响到另一个

- 切片可以用make函数进行初始化make([]int, 3)，也可以以数组进行切片array[1, 2]。以数组声明切片时，可以指定两个或三个索引，分别是“起始，长度” && “起始，长度，容量”。为了减少内存分配次数以提升性能，你需要思考如何传递make的参数

- Go不允许为简单的内置类型添加方法，可用type创建别名实现方法重载

- type T Cls创建了全新的类型，除了结构相同，和原类型无任何关系（无原类型的方法集、无法相互类型断言…etc.），当然，结构相同是可以做强制类型转换的

- Go1.9引入了type alias，type T = Cls，除了名字不同，和原类型无任何区别（拥有原类型的方法集、可以在任何地方将Cls替换为T…etc.）

- Go语言的OOP与一般语言不同，它的多态是以接口实现的，继承是以内嵌实现的

- 嵌入后外部类型相当于子类，内部类型相当于父类，内部类型的方法实现会被提升到外部类型

- 结构体中类型的后面可使用raw string声明标签，它是提供每个字段元信息的一种机制，如使用encoding/json库解析json时，将json文档与结构体字段进行映射：
```go
type jsonResult{
    URL    string  `json:"url"`
    title  string  `json:"title"`
}
```
- 将对象实例赋值给接口时，要求实例实现了接口的所有方法。编译器会根据需求生成新的方法，当使用下面一种方法赋值时，编译器无法通过，原因是当将int实例的指针赋值给接口时，编译器会自动生成新的对象方法func (a *myint) Less(b myint) bool {...}，而假如将int实例的值赋值给接口，编译器不会为Add方法生成值方法，因为这与需求违背，修改值的副本无法达成需求，故编译器会认为实例没有实现接口的所有方法集，将会报错：
```go
type myint int
type mynum interface{
    Less(b myint) bool
      Add(b myint)
}
func (a myint) Less(b myint) bool {
    return a < b
}
func (a *myint) Add(b myint) {
    (*a) += b
}
//将实例赋值给接口时：
var a myint = 3
var b mynum = &a
//var b mynum = a  /*this is wrong*/
```
- 将接口赋值给接口时，可将父集赋值给子集

## 并发/并行

- 当一个函数创建为goroutine时，编译器会将其视为一个独立的工作单元。这个单元会被调度到可用的逻辑处理器上执行。调度器会将操作系统的线程与逻辑处理器绑定，调度器在任何时间，都会控制goroutine在哪个逻辑处理器上运行。Go语言运行时默认会为每个可用的物理处理器分配一个逻辑处理器。调度器对可创建的逻辑处理器数量没有限制

- Go的并发模型来自一个叫做通信顺序进程（Communicating Sequential Process, CSP）的范型。它是一种消息传递模型，故Go的并发模型是建立在信号通信而不是互斥锁

- 当goroutine阻塞时，调度器会创建一个新线程，并将其绑定到该逻辑处理器上，接着从本地队列里选择另一个goroutine来运行

- 如果希望Go程序并行运行，需要创建多于一个的逻辑处理器。这会使用多个线程，达到真正的并行效果

- runtime.GOMAXPROCS(runtime.NumCPU())分配CPU核数个逻辑处理器，runtime.NumCPU相当于Python中multiprocessing库的cpu_count函数获取的CPU核数

- runtime.Gosched()函数强制退出当前进程，返回队列

- go build -race竞争检测器

- 保证线程安全：

    原子函数sync/atomic库的LoadInt64, AddInt64及StoreInt64函数（在x86下使用XXInt64时需要开发者自己保证内存对齐，否则panic）

    互斥锁sync.Mutex类型，对象方法为Lock()/Unlock()，类似Python中的acquire()/release()

    通道channel

        无缓冲通道make(chan int)，必需sender和reciever都准备好，否则阻塞

        有缓冲通道make(chan int, 8)，原生的并发安全队列。对有缓冲通道，当通道关闭后依然可以取出其中数据，但不可以再写入

- channel的三种方向：ch := make([chan | <- chan | chan <-] type)

    chan：双向

    <- chan int: 单向发送：<- ch

    chan <- int: 单向接收：ch <- a[type:int]

- select I/O多路复用：
```go
//定时
select {
    case data := <-ch:
        process(data)
    case <-time.After(time.Second * 3):
        fmt.Println("time out")
    }
    
//轮询
select {
    case i1 = <- c1:
        fmt.Printf("received ", i1)
    case c2 <- i2:
        fmt.Printf("sent ", i2)
    case i3, ok := (<-c3):
        if ok {
            fmt.Printf("received ", i3)
        } else {
            fmt.Printf("c3 is closed\n")
        }
    default:
        fmt.Printf("no communication\n")
}   
``` 

