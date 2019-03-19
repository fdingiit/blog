---
title: Go slice internal
date: 2017-05-25 12:04:48
tags: golang
---

## 前言：
slice作为golang的一个较为重要，且具有特色的数据结构，其被使用的频率十分之高。本篇文章作为[Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)的补充，介绍了在使用slice时可能会遇到的一些问题，并通过golang源码阅读以及代码实验等方式，对各个问题出现的原因做出了较为详细的解释。

## 第零部分： slice实现及使用
在计算机科学的领域中，所有问题最终都可以归结为1.如何看待数据；2.如何操作数据。golang的slice也不例外，其实质上为一个描述符，描述数据在哪，有什么属性：

``` go
// $GOPATH/src/runtime/slice.go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
slice的目的，是对一块连续的内存空间进行管理。如上述golang slice实现源码所示，`array`作为一个指针，指向内存空间首地址；`len`和`cap`则描述了这块内存现在有多少个数据元素，总共能够存储多少个数据元素。
基于此，我们可以通过内置函数append()，把slice作为一个动态数组来使用，如：  
``` go
s := []int{1}
s = append(s, 10)     // [1, 10]
```
我们也可以通过分片，把slice作为FIFO队列，或者栈来使用：
``` go
s := []int{1, 2, 3}
s = s[1:]           // [2, 3]
s = s[:len(s)-1]    // [2]
```
关于slice的使用多种多样，本文不再过多赘述。

## 第一部分：slice, function and pass-by-value
众所周知，golang函数调用遵循pass-by-value的原则（注：详见[The Go Programming Language Specification: Calls](https://golang.org/ref/spec#Calls)），即所有形参在函数体内都为实参的一个副本。如果不了解这个原则，那么对形参的修改及操作可能不会得到期待的效果，简单的例子：
``` go
func foo(i int) {
    i = 0
}

func main() {
    x := 1
    foo(x)
    fmt.Println(x)  // still 1
}
```
那么，当使用slice作为函数参数时，会发生什么呢？
``` go
func foo(s []int) {
    s = append(s, 10)
}

func main() {
    s = []int{1}
    foo(s)
    fmt.Println(s)  // still [1]
}
```
程序的最终输出为`[1]`，看起来十分符合pass-by-value的函数调用原则：实参`s`在函数`foo()`中被修改的只是其一个内部副本，因此`main()`函数中slice并没有任何改变。但，事实真的如此吗？  
考虑golang提供的排序接口：  
``` go
// A type, typically a collection, that satisfies sort.Interface can be
// sorted by the routines in this package. The methods require that the
// elements of the collection be enumerated by an integer index.
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int
	// Less reports whether the element with
	// index i should sort before the element with index j.
	Less(i, j int) bool
	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```
`sort`包为我们提供了一个能够对任何数据结构，进行任意自定义排序的功能。一个简单的应用为对“学生”按“年龄“进行排序：
``` go
type Student struct {
	name string
	age int
}

type Students []Student

func (s Students) Len() int {
	return len(s)
}

func (s Students) Less(i, j int) bool {
	return s[i].age < s[j].age
}

func (s Students) Swap(i, j int) {
	s[i], s[j] = s[j], s[i]
}
```
注意，`func (s Students) Swap(i, j int)`中，`Swap()`方法的接收者`s Students`，事实上就是一个数据类型为`Student`的slice，其将会被作为`Swap()`的默认首参参与函数调用，即`Swap(s []Student, i, j int)`（注：这部分内容会在后文中详细讨论）。可以看到，如果slice在函数中仅仅作为一个副本的形式而存在，那么`Swap()`无法正确更改slice内部数据内容，继而导致`sort()`包所提供的接口永远不会产生作用。很显然，slice在参与函数调用时，并不是仅仅遵循pass-by-value的原则那么简单，难道slice是一个特例？  
为了搞清楚这个令人疑惑的现象，并结合我们之前的例子，现提出以下问题：  
:question: **函数foo()具体对s做了什么？为什么输出为[1]？10到底有没有被添加到slice中？**  
### **原理分析：**  
回顾第零部分的内容，slice实质上是一个由三元素组成的描述符，即`foo(i []int)`可以看做`foo(p unsafe.Pointer, length, capa int)`。很显然，函数参数中出现了指针，而指针的加入使得遵循pass-by-value原则的函数调用过程能够实际修改参数的内容。而`foo()`函数体中`s = append(s, 10)`的执行会对`s`产生两种影响：
- 若s仍有空余位置，即`capa - length >= 1`，则此时无需开辟新的内存空间，直接将10写入p[length+1]的位置，此时三个形参的值将会被更新为{p, length+1, capa}
- 若s没有空余空间，即`capa - length == 0`，则需要开辟新的内存空间，其起始地址设为q，新的capa设为capaq，此时三个形参的值将会被更新为{q, length+1, capaq}  

由此可见，`foo()`函数的确将`10`写到了某个内存位置，但是由于pass-by-value原则，在其调用返回以后，`main()`中slice描述符`s`的值依旧为`{p, length, capa}`，而标准输出函数在处理slice时需要用到这个描述符来确定输出的内容，其中较为重要的信息包括元素的个数，即`len(s)`的值。因此，其在打印时，要么slice的长度“不对”——其length值没有被+1（对应无需申请新的内存空间的情况），要么打印的内存位置和slice长度都“不对”——其依旧是旧的内存内容（对应需申请新的内存空间的情况），所以`10`的确被添加到slice中，但不会出现在标准输出中，最终得到的输出为`[1]`。
### **代码show：**  
为了证明上述结论的正确性，在这里我们不使用`fmt`包所提供的函数打印slice，转而自己编写一个针对性的输出函数`prettyPrint()`：
``` go
func prettyPrint(s []int) {
	start := (*uintptr)(unsafe.Pointer(&s))      // 获取内存首地址的地址
	_, capa := len(s), cap(s)
	fmt.Print("prettyPrint: [")
	for i := 0; i < capa; i++ {
		if i != 0 {
			fmt.Print(" ")
		}
		addr := (*int)(unsafe.Pointer((uintptr(*start)) + uintptr(i) * (unsafe.Sizeof(int(0)))))    // 计算第i个元素的地址
		fmt.Print(*addr)
	}
	fmt.Println("]")
}
```
如上代码所示，`prettyPrint()`通过使用`unsafe`包计算各个元素地址的方式，将slice的`cap(s)`个元素全部打印出来，即包含了`s[len(s):cap(s)]`的部分。假如我们使用`prettyPrint()`来替代`fmt.Println()`，整个slice的具体内容便一览无余：
``` go
func foo(s []int) {
    s = append(s, 100, 200)
}

func main() {
    s := make([]int, 1, 10)   // 一个足够大的slice
    s[0] = 1
    foo(s)
    prettyPrint(s)           // [1 100 200 0 0 0 0 0 0 0]
                             // 注意！ 100和200被写进了slice

    s = []int{1}             // 不够大
    foo(s)
    prettyPrint(s)           // [1]
}
```
以上，可以看出，当使用slice作为参数，并且函数会对其进行某些操作，如`append()`时，虽然在函数返回之后slice**看起来**没有变，但实质上其内容已经被修改了。 
### **结论：**
- slice在参与函数调用时依旧遵循pass-by-value原则；
- slice实质上一个描述符，其数据域包含了一个指向具体内存位置的指针。正是由于这个指针的存在，才使得函数可以对参数中的slice进行修改，如`sort`接口所做的那样；
- 在经过函数调用后的slice，看上去没有发生改变，可能是因为其`len`和`cap`域因pass-by-value原则没有得到更新。

下面是一个典型的错误例子，源码试图在一个二叉树里找出所有从根节点到叶子节点的和为目标值的路径。要如何对其进行修改呢？  
源码：[Go playgroud](https://play.golang.org/p/VTyXoljzLb)
``` go
package main

import "fmt"
import "strconv"

type Node struct {
	val int
	left *Node
	right *Node
}

func (n Node) String () string {
	return strconv.Itoa(n.val)
}

func newNode(val int) *Node {
	return &Node{val, nil, nil}
}

// Given a binary tree and a target sum, find all root-to-leaf paths
// where each path's sum equals the given sum
func bs(root *Node, path []*Node, now int, target int, ret *[][]*Node) {
	if root == nil {
		return
	}
	path = append(path, root)
	now += root.val
	if now == target && root.left == nil && root.right == nil {
		*ret = append(*ret, path)
		return
	}
	bs(root.left, path, now, target, ret)
	bs(root.right, path, now, target, ret)
}

func main() {
	// a simple tree like:
	//    0
	//   / \
	//  1   2
	root := newNode(0)
	left := newNode(1)
	right := newNode(2)
	root.left = left
	root.right = right

	ret := [][]*Node{}
	bs(root, make([]*Node, 0, 10), 0, 1, &ret)
  // what about change it to
  // bs(root, make([]*Node, 0, 1), 0, 1, &ret) ?
	fmt.Println(ret)
}
```

# 第二部分：slice, for-range and go routine
[Effecive Go](https://golang.org/doc/effective_go.html#channels) Channels这一小节描述了对channel进行for range的一个错误使用：
``` go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req) // Buggy; see explanation below.
            <-sem
        }()
    }
}
```
文章中谈到：  
> The bug is that in a Go for loop, the loop variable is reused for each iteration, so the req variable is shared across all goroutines  

作者明确指出了，在以上for loop中， `req`被当做公共变量在for循环的每一次迭代中被所有goroutine共享使用，因此其可能会导致错误的结果。**在对slice进行for range迭代时，也可能出现同样的问题：**
``` go
package main

import "fmt"

func main() {
	slice := []int{1, 2, 3}

	for _, n := range slice {
		go func() {fmt.Println(n)}()
	}


	for _, n := range slice {
		go func(n int) {fmt.Println(n)}(n)
	}
}
```
实际上，使用race detector可以直观地发现错误出现在`go func() {fmt.Println(n)}()`：一个routine所读取的变量n在另一个routine中被改写了：  
`$> go run -race 1.go`

```
==================
WARNING: DATA RACE
Read by goroutine 6:
  main.main.func1()
      /home/fd/Test/1.go:9 +0x32

Previous write by main goroutine:
  main.main()
      /home/fd/Test/1.go:8 +0x106

Goroutine 6 (running) created at:
  main.main()
      /home/fd/Test/1.go:9 +0x130
==================
```
为了解决这个问题，我们给`go`语句所要执行的函数添加一个参数，并把for-loop中的公共变量n传递进去，这样每次执行函数调用时，其n的值都会被确定，从而消除了读取脏数据的可能。但是，如果我们把代码稍微改一下：
``` go
package main

import "fmt"

type Int struct {
	val int
}

func (n Int) print1() {
	fmt.Println(n.val)
}

func (n *Int) print2() {
	fmt.Println(n.val)
}

func main() {
	slice := []Int{{1}, {2}, {3}}
	for _, n := range slice {
		go n.print1()
	}

	for _, n := range slice {
		go n.print2()
	}
}
```
以上代码会出现data race的情况吗？在哪里出现？为什么？  
为了理解这个问题，需要知道一点编译器对于OOP的实现方法：简单来说，对于一个程序，编译器可以通过词法/语法分析以生成一张符号表（symbol table），这张表记录了程序中出现的所有identifiers的性质。类表（class table）是符号表的一部分，它罗列了运行程序所需要的各个类的信息，而method table是类表的一部分，其维护了包括类方法名及参数列表等在内的各种信息（虚函数表在此不考虑），最终，链接器会将函数名与函数入口地址联系起来。因此，如果假设在method table里已经计算好了各个函数的地址，那么函数调用可以看做成一个通过查表获取函数地址`addr`，然后按序压参并`call addr`的过程。需要注意的是，类的每个方法都有一个指向该类具体实例的指针(可以理解为this指针)作为默认首参。代码：
``` cpp
class A {
    foo(int a) {}
    bar(int a, float b) {}
}

A a, b;
a.foo(10);
b.bar(10, 20.0);
```
`a.foo(10)`从编译到调用的过程大致为：
- 编译器分析出`class A {xxx}`是类的定义，其将id`A`记录在symbol table的class table中，并维护id为`foo`和`bar`的两个方法于类`A`的method table中；
- 编译器分析出`A a, b;`是一条变量声明语句，于是将`a, b`两个id记录在symbol table中，并将它们的类型标记为`A`；
- 编译器分析出`a.foo(10)`是一条call statement，其body为`a`，function为`foo`，参数为`10`；
- 编译器通过查symbol table得知`a`的类型为`A`，`A`的类型为class；
- 编译器查找class table以获取类`A`的各个信息，并在其method table中寻找函数`foo(int)`的相关信息，其入口地址由linker给出；
- 编译器将指向`a`的指针作为`foo(A*, int)`的默认首参参与函数调用；
- codegen...
- execute binary...

从伪代码的角度，可以将这个过程看做：
``` cpp
A->mtable->foo(&a, 10);
A->mtable->bar(&b, 10, 20.0);
```

```
|-----------|
|class table|
|-----------|
|     A     | ===>   |-------------------------|    
|-----------|        |      method table       |
|     B     |        |-------------------------|  
|-----------|        |foo(int a)               |
|     C     |        |-------------------------|
|-----------|        |bar(int a, float b)      |
                     |-------------------------|
```
（注：这种查表的过程只是编译器在类型检查阶段的一种具体实现方法，并不是编译器的标准，不同编译器可能会有不同的处理方式）

由于golang区分pointer receiver和value receiver，对于`func (t T) method()`来说，在调用时即是`T.method(t)`:
``` go
for _, n := range slice {
    go n.print1()
}
// 等价于：
for _, n := range slice {
    go Int.print1(n)
}
```
对于`func (t *T) method()`来说，在调用时即是`(*T).method(t)`:
``` go
for _, n := range slice {
    go n.print2()
}
// 等价于：
for _, n := range slice {
    go (*Int).print2(&n)
}
```
可以看到，`go (*Int).print2(&n)`中所有routine访问`&n`这个公共地址且没有进行同步处理，因此引入了data race。

#### 附：  
一.
golang对于receiver funtion的处理方式，为维护一个type->functions（类型->类型方法）的表，源码详见[$GOPATH/src/go/types/decl.go](https://github.com/golang/go/blob/master/src/go/types/decl.go)中的`addMethodDecls()`，与上文所述OOP对于method table的处理类似，本文不做详述，待日后填坑；  


二.
证明：
> 对于`func (t T) method()`来说，在调用时即是`T.method(t)`:

**从文档的角度：**
请阅读以下小节：[The Go Programming Language Specification: Method expressions](https://golang.org/ref/spec#Method_expressions)。

**从源码的角度：**
最根本的方法是分析golang源码的parser/type checker/codegen等模块，待日后填坑。 

**从实验的角度：**
看可执行文件的binary。  
第一步：随便写个例子
``` go
// 1.go
package main

import "fmt"

type Int struct {
	val int
}

func (n Int) print() {
	fmt.Println(n.val)
}

func main() {
    n := Int{10}
    n.print()
}
```
第二步：build成x86 executable，并dump出binary （x86_64也一样）
``` shell
$> env GOOS=linux GOARCH=386 go build -v 1.go
$> objdump -d 1 > dmp
```  

第三步：分析binary
``` nasm
080490d0 <main.main>:
 80490d0:	65 8b 0d 00 00 00 00 	mov    %gs:0x0,%ecx
 80490d7:	8b 89 fc ff ff ff    	mov    -0x4(%ecx),%ecx
 80490dd:	3b 61 08             	cmp    0x8(%ecx),%esp
 80490e0:	76 1a                	jbe    80490fc <main.main+0x2c>
 80490e2:	83 ec 08             	sub    $0x8,%esp
 80490e5:	31 db                	xor    %ebx,%ebx
 80490e7:	b8 0a 00 00 00       	mov    $0xa,%eax
 80490ec:	89 44 24 04          	mov    %eax,0x4(%esp)
 80490f0:	89 04 24             	mov    %eax,(%esp)
 80490f3:	e8 08 ff ff ff       	call   8049000 <main.Int.print>
 80490f8:	83 c4 08             	add    $0x8,%esp
 80490fb:	c3                   	ret    
 80490fc:	e8 df 7a 04 00       	call   8090be0 <runtime.morestack_noctxt>
 8049101:	eb cd                	jmp    80490d0 <main.main>
```

注意：
``` nasm
80490e7:	b8 0a 00 00 00       	mov    $0xa,%eax
80490ec:	89 44 24 04          	mov    %eax,0x4(%esp)
80490f0:	89 04 24             	mov    %eax,(%esp)
80490f3:	e8 08 ff ff ff       	call   8049000 <main.Int.print>
```
`0xa`，即源码中的`Int{10}`被作为参数压栈。`func (t *T) method()`同理。

终。

### 引用及相关资料：
- [effective_go](https://golang.org/doc/effective_go.html)
- [Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)
- [Introducing the Go Race Detector](https://blog.golang.org/race-detector)
- [The Go Programming Language Specification](https://golang.org/ref/spec)
- [Cross compilation with Go 1.5](http://dave.cheney.net/2015/08/22/cross-compilation-with-go-1-5)
- [objdump - GNU Binary Utilities](https://sourceware.org/binutils/docs/binutils/objdump.html)
- [x86 calling conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)
