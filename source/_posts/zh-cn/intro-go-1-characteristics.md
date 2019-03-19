---
title: 《Intro Go》第一部分：特性
date: 2018-11-27 21:20:37
tags: golang 
---

Go是一个年轻的编程语言，只有九岁[<sup>[1]</sup>](https://blog.golang.org/9years)。但事实上，在2013年docker宣布开源，并在随后几年逐渐成为主流容器解决方案之前，它都没有获得太多的关注[<sup>[2]</sup>](https://www.tiobe.com/tiobe-index/go/)，一直到2016年左右才真正吸引到大众的眼球<a href="#ref-1"><sup>[注1]</sup></a> <a name="bref-1"></a>。  

Go的作者是三位来自Google的工程师：Robert Griesemer, [Rob Pike](https://en.wikipedia.org/wiki/Rob_Pike "Rob Pike"), 和 [Ken Thompson](https://en.wikipedia.org/wiki/Ken_Thompson)。其中后两位比较有说头：前者参与开发了著名的[Plan 9](https://en.wikipedia.org/wiki/Plan_9_from_Bell_Labs)；后者参与编码了Unix内核的早期版本，并发明了[B语言](https://en.wikipedia.org/wiki/B_%28programming_language%29)。可以看到，他们两位的最大共同点在于：都是ANSI C语言的重度使用者。因此，golang有着浓厚的C语言痕迹。

在Go的核心竞争力：并发编程方面，它深受[CSP](https://en.wikipedia.org/wiki/Communicating_sequential_processes)理论影响，通过**消息**（`channel-message`），而不是**共享内存**（`shared-memory`）处理协程（goroutine）之间的同步/异步问题。在这方面，它与最近发展势头很猛的[Rust](https://www.rust-lang.org/en-US/)算是“同宗同族”[<sup>[3]</sup>](https://doc.rust-lang.org/book/2018-edition/ch16-02-message-passing.html#using-message-passing-to-transfer-data-between-threads)。这部分内容将会在《Intro Go》的后续三篇文字中做详细阐述。

本篇文字将试图通过使用与其他常见编程语言对比，并连带一些简单的代码示例的方式，介绍Go与它们最主要的相同点及不同点，从而明确这个新语言的特性。

### 1. 基础属性
由于受到C语言的深刻影响，Go被设计成一个（1）**静态**（2）**强数据类型**，（3）**编译型**编程语言；但同时，他又包含新时期编程语言通常所带有的功能：内存安全、垃圾回收。

|  | statically | strong | compiled | memeory safe | GC |
|:--:|:--:|:--:|:--:|:--:|:--:|
| C/C++ | ✔ | ✖ | ✔ | ✖ | ✖ |
| Java | ✔ | ✔ | ✔ | ✔ | ✔ |
| Python | ✖ | ✔ | ✖ | ✔ | ✔ |
| **Go** | **✔** | **✔** | **✔** | **✔** | **✔** |

```go
// Example 1-1. Statically and strong type-system
var x int		// statically typed
var y uint		// statically typed

if x == y {		// will NOT compiled
	// ...
}
```
如`Example 1-1`所示，在比较`x`和`y`的值是否相等时，由于Go是一个强数据类型语言，而这两个变量并不是同一个数据类型，因此二者无法进行比较。事实上，这段代码无法编译，编译器会给出如下错误提示：

```
➜ go run typed.go 
# command-line-arguments
./typed.go:7:7: invalid operation: x == y (mismatched types int and uint)
```

`Example 1-2`给出了一个正确的和一个错误的示例。

```go
// Example 1-2. Statically and strong type-system, case 2
func sum(a, b int) int {
	return a + b
}

// right way to call
var x, y int = 1, 2
sum(x, y)

// wrong way to call
var m int = 1
var n uint = 2
sum(m, n)			// will NOT compile
```

### 2. 目标代码和运行环境
不同于基于VM或者解释器的语言，对于Go来说，它的目标代码即为目标平台的natvie二进制指令<a href="#ref-2"><sup>[注2]</sup></a> <a name="bref-2"></a>；它的目标代码运行环境即为目标平台的native环境。当在Unix环境中 <a href="#ref-3"><sup>[注3]</sup></a> <a name="bref-3"></a> 使用`file`命令对一个使用golang编码的可执行文件——*elector*——进行查验时，可以得到如下结果：

```
➜ file elector
elector: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, stripped
```

|  | obj-code | environment |
|:--:|:--:|:--:|
| C/C++ | asm | native |
| Java | bytecode | VM |
| Python | no/pyc | interpreter | 
| **Go** | **asm** | **native** |

### 3. 极简主义
Go语言最关键的设计原则及目标之一，即是**简单**。Go语言的创造者通过精简语法和编程模型，尤其是并发模型，以控制语言的学习成本，并提高开发效率。较少的关键字数量便是极简原则的一个最为直观的体现：

```go
// 25 reserved key words of Go v1.10
break        default      func         interface    select
case         defer        go           map          struct
chan         else         goto         package      switch
const        fallthrough  if           range        type
continue     for          import       return       var
```

| language | key words |
|:--:|:--:|
| ANSI C | 32 |
| C++11 | 95 |
| Java | 51 |
| Python | 34 |
| **Go** | **25** |

由上表可以看到，在大规模使用的工业级编程语言中，Go的保留关键字数量可能是最少的。因此，从语法糖的角度来看，Go语言几乎没有可以玩花活儿的余地。程序员没有办法像编写Python那样将几行甚至几十行代码，用一到两行经过精心考量的，~~优雅的~~代码替代。但是，golang语法上的简化，使得学习过Go的程序员都能够以较快的速度读懂几乎所有Go语言项目，而Python则不一定。举个例子，看懂`Example 1-3`的代码你需要多久[<sup>[4]</sup>](https://wiki.python.org/moin/Powerful%20Python%20One-Liners)？

```python
# Example 1-3. One line Python code
f = lambda x: [[y for j, y in enumerate(set(x)) if (i >> j) & 1] for i in range(2**len(set(x)))]
```

`Example 1-4`呢？

```python
# Example 1-4. One line Python code, case 2
f = lambda l: reduce(lambda z, x: z + [y + [x] for y in z], l, [[]])
```

你能看出来`Example 1-3`和`Example 1-4`其实是在算同一个东西吗？可真是[见了鬼了](https://github.com/satwikkansal/wtfpython)。

言归正传，再来看Go的保留关键字。相信任何学过一到两门编程语言的读者都能读懂这25个关键字的20个或更多，除了以下三位可能比较陌生：

```go
chan
select
go
```

这三个关键字便是Go语言核心竞争力之所在，也是《Intro Go》后续部分所要着重介绍的内容。

像很多编译器一样，Go语言是一个[self-hosting](https://en.wikipedia.org/wiki/Self-hosting)（抱歉不知道该如何翻译成中文）的项目。学习Go最直接的方法便是直接阅读使用Go语言编写的，其自身的源码。通过阅读这些代码，相信读者能够对**简单**这个Go语言最重要的设计原则有更具体的体会——因为没有哪一行代码是看不懂的。

### 4. Classic OOPL?
Q: Go语言是一个经典的面向对象编程语言吗？[<sup>[5]</sup>](https://golang.org/doc/faq#Is_Go_an_object-oriented_language)

A: 不，Go语言没有继承（Inheritance）的概念，也不支持多态（Polymorphism），它的接口（Interface）与其他OOPL语言的接口也稍有区别，但你可以用Go实现部分OOPL可以实现的功能 。

#### 4.1 封装
你可以通过使用`struct`关键字及其语法来实现OOPL中的`class`：

```go
// Example 1-5. A Go's `class'
type MyType struct {
	a int
	b float64
}
```

是不是好像哪里见过？

```c
// Example 1-6. A C's `class'
struct MyType {
	int a;
	float b;
};
```

对于每个“class”，你也可以为其定义“方法”：

```go
// Example 1-7. Go `class' methods
func (mt MyType) foo() {
	// ...
}

// another method (pointer receiver)
// Q: what is a pointer receiver?
// A: checkout gopl
func (mt *MyType) bar() {
	// ...
}
```

你可以这样使用Go的“类”：

```go
// Example 1-8. Go sytle OOP
inst := new(MyType) 					// new an instance, get a pointer
inst.a, inst.b = int(1), float64(3.14)
inst.foo()
inst.bar()

// another way to get and init a Go `class'
inst := MyType{a: 1, b: 3.14}
// or
inst := &MyType{1, 3.14}
```

#### 4.2 继承
Go没有传统OOPL意义上的“继承”的概念，你需要把自己当做编译器的一部分，手动实现“继承”。方法很简单：

```go
// Example 1-9. Go way to inhere
type Base struct {
}

type Child struct {
	b Base
}
```

#### 4.3 多态和泛型
这可能是Go语言被社区最为诟病的“问题”之一了（另一个是错误处理）。简单来说，Go 1.x中我们将不可能看到对于多态/泛型的支持，但相关讨论早已展开，我们有希望在Go 2中看到它们。  

那么，我们现在该如何处理多态呢？对不起，只能再一次地把编译器的活儿给干了：

```go
// Example 1-10. One way to implement Go sytle polymorphism by type assertion
type Animal interface {
	Hi() string			// all animals must could say Hi() 
}

// Cat that meow
type Cat struct {
}

func (c Cat) Hi() string {
	return "meow"
}

// Dog that wooooow
type Dog struct {
}

func (d Dog) Hi() string {
	return "wooooow"
}

func Hi(a Animal) string {
	switch a.(type) {
	// type assertion
	case Cat:
		return fmt.Sprintf("a cat says: %s", a.Hi())
	case Dog:
		return fmt.Sprintf("a dog says: %s", a.Hi())
	default:
		panic("unexcepted animal")
	}
}
```

`Example 1-10`声明了一个[接口](https://golang.org/ref/spec#Interface_types)：`Animal`，规定所有`Animal`都会say`Hi()`，并定义了两种动物（`类`实现`接口`）：`Cat`和`Dog`。`func Hi(a Animal) string`通过对传入的参数进行类型检测实现OOPL的多态。

`Example 1-10`的可运行代码示例请点[这里](https://play.golang.org/p/M54JXmXddou)。

**需要强调的是，如果读者在使用Go语言开发程序的时候，不断地重复编写类似`Example 1-10`中的代码，同时明显感觉到了Go在此方面的不便，甚至因此极大地影响到了开发进度，那么则需要研究是否更改开发语言。**

最后，如果你对Go 2可能存在的泛型设计感兴趣，请看[这里](https://github.com/golang/go/issues/15292)，[这里](https://go.googlesource.com/proposal/+/master/design/go2draft-generics-overview.md)，[这里](https://go.googlesource.com/proposal/+/master/design/go2draft-contracts.md)，和[这里](https://github.com/golang/go/wiki/Go2GenericsFeedback)。

### 5. 指针
惊了，都8102年了，能不能不要开历史的倒车，不要写指针，信不信我用**SIGSEGV**糊你脸。

放心，Go是内存安全的，除非你死了心要搞事情。但同时，Go确实有指针的概念。而且，很不幸，还不止一种。

#### 5.1 *T
我个人把这种星号指针称之为high-level-pointer：

```go
// Example 1-11. * pointers
*T

// eg.
*int
*float64
```

这种指针的特点是：**不可参与算术运算**。

```c
// Example 1-11. Pointer arithmetic in C code
int* array = (*int)malloc(sizeof(int) * 10);
int* p;

p = array;
*(p+1) = 5;			// 2nd element turns to 5
```

`Example 1-11`的C代码，通过进行指针运算来操作内存数据。而在Go语言中使用`*`指针无法实现类似功能。原因？早在`Example 1-1`就说明了。

```go
// Example 1-12. Wrong golang code of high-level pointer arithmetic
var p *int
var q *int

// CANNOT complie, mismatched types *int and int
q = p + 1 
```

#### 5.2 unsafe.Pointer
相应的，我个人把使用`unsafe.Pointer`类型的指针称之为low-level-pointer，它可以用来进行具体内存地址的运算：

```go
// Example 1-13. Low-level pointer arithmetic
func add(p unsafe.Pointer, x uintptr) unsafe.Pointer {  
	return unsafe.Pointer(uintptr(p) + x)  
}
```

光是`unsafe`这个包的名字，就足够提醒你，没事不要来用low-level-pointer。更重要的是，根据Go语言文档：

> **Packages that import unsafe may be non-portable and are not protected by the Go 1 compatibility guidelines.**

所以，知道有`unsafe.Pointer`这个东西目前就足够了。

可以看到，Go语言既保留了指针最基础的功能——地址引用；又“删去”了最容易出错的地方——指针运算，简化了指针的使用。

### ~~6. 主观评价~~
这一小节是作者个人对Go这个语言，与几个常见编程语言在某些维度上的比较，具有很强的主观性及特殊性，因此标题被~~横线~~了，仅作参考。

#### 6.1 学习成本
**C++ > Python > Java > Go > C**

毫无疑问，C++的学习成本是最高的。有一位研究生同学是C++大佬，他曾经说过：

> 只要市面上能买得到的有关C++的书，我全看过，且至少一遍。

于是我去某电商搜了下C++这个关键字，然后发现并不能买得起所有的这些书。

某一天（2015年）我开始学习Go的时候，又想起了他的这句话，并暗暗发誓：

> 只要市面上能买得到的有关Go的书，我全看过，且至少一遍。

然后我就成功了。

至于Python，想跑个demo简单，想做个大项目，要学的东西可就真的太多了。

#### 6.2 编码/开发成本
**C++ > Python > Java > C > Go**

对我来说，Python并不简单。而Go由于其极简的语法，代码写起来很快速、顺手。

#### 6.3 调试成本
**Python > Go  > C = C++ = Java**

编译语言比解释语言好调试；  
静态语言比动态语言好调试；
强类型比弱类型语言好调试。

Go调试成本略微比C/C++/Java高是因为调试工具集没有另外三个语言丰富，而且有经验的Go程序员相对来说少得多得多得多。

#### 6.4 Line of Code
**C > C++ > Java > Go > Python**

Go语言的代码量控制的非常出色，如果不是因为丑陋的错误处理和缺乏对多态/泛型的支持，代码行数会更少。

#### 6.5 对计算机的控制
**C = C++ > Go > Java = Python**

毫无疑问，C和C++是最强大的。Go和其他两个语言一样几乎无法控制指令，但Go能够在一定程度上操作内存，因此略省一筹。

**(正文完)**

---

#### Recommandations：

为了更好理解《Intro Go》，推荐阅读：
1. [The Go Programming Language](https://www.amazon.cn/dp/B01ASI3154/ref=sr_1_1?ie=UTF8&qid=1540360238&sr=8-1&keywords=the+go+programming+language) 
	1.1 Chapter 4. Composite Types. P99-P107
	1.2 Chapter 7. Interfaces. P171-P216
	1.3 Chapter 8. Goroutines and Channels. P217-P253
	1.4 Chapter 13. Low-Level Programming. P353-P358
2. The Go doc:
	2.1 [The Go Programming Language Specification](https://golang.org/ref/spec)
	2.2 [Go Concurrency Patterns: Timing out, moving on](https://blog.golang.org/go-concurrency-patterns-timing-out-and)

两个非常有助于理解Go语言设计理念，以及熟悉Go具体语法的好去处：
- [faq](https://golang.org/doc/faq)
- [tour](https://tour.golang.org/list)

其中，faq**非常重要**，建议先读完，再尝试HelloWord。

---
#### Notes：

1. <a name="ref-1"></a> <a href="#bref-1">^</a> 指中国大陆地区。

2. <a name="ref-2"></a> <a href="#bref-2">^</a> 从广义上说，Go的runtime等价于一个与具体用户代码编译在一块的虚拟机。但在这里，我们将其与用户程序独立编译，并在预编译好的虚拟机（如Java Hotspot VM）上执行的这种虚拟机模型区分开来，认为其不是一个通常意义上的基于虚拟机的语言。

3. <a name="ref-3"></a> <a href="#bref-3">^</a>《Intro Go》中所涉及的所有操作默认在Unix环境下执行，后略。

---

#### References:

[1] [Nine years of Go](https://blog.golang.org/9years)  
[2] [The Go Programming Language TIOBE Index](https://www.tiobe.com/tiobe-index/go/)  
[3] [Using Message Passing To Transfer Data Between Threads](https://doc.rust-lang.org/book/2018-edition/ch16-02-message-passing.html#using-message-passing-to-transfer-data-between-threads)  
[4] [Python One Liner](https://wiki.python.org/moin/Powerful%20Python%20One-Liners)  
[5] [Is Go an object-oriented language?](https://golang.org/doc/faq#Is_Go_an_object-oriented_language)  

---

如果您喜欢这篇文章，请支持我，谢谢。

😉

~~唯一指定支付宝~~  
<img src="/about/index/ali_pay.jpeg" style="width:120px;height:120px;">

~~唯一指定微信~~  
<img src="/about/index/wechat_pay.jpeg" style="width:120px;height:120px;">
