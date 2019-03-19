---
title: 《Intro Go》序
date: 2018-11-27 12:04:48
tags: golang
---

《Intro Go》系列文字来源于公司内部技术培训手稿的进一步整理，其致力于从Go语言源码/实现——主要是runtime——的角度向大家介绍这门受面越来越广[<sup>[1]</sup>](https://blog.github.com/2018-11-15-state-of-the-octoverse-top-programming-languages/)的新技术。
  
与大多数介绍编程语言的文字/书籍不同，《Intro Go》的初衷不在于告诉大家如何写出正确优雅的Go代码，亦不是介绍使用golang开发的各种主流第三方库和开源项目（例如[docker](https://github.com/moby/moby)），更不会竭力向读者证明golang有多棒，“golang天下第一”。《Intro Go》的目标是希望读者通过阅读这一系列文章，能够理解这门语言的设计理念，及其三个最核心的关键字——`chan`/`select`/`go`——的具体实现。  

可能有读者会问：我觉得只要学会如何写Go就足够了，进一步研究那几个关键字看起来也能扩展视野，但为什么还要去理解这门**语言**的**设计思路**？相比而言，我更想快速掌握golang看懂代码，然后去研究docker的具体实现。

如果你有经典OOPL的基础，或者是一个纯粹的面向过程编程的C程序员，日常工作不是在设计各种数据结构及其相互关系，就是在使用**共享内存**处理**并发**，那么充分理解这门语言的设计理念是写好这门语言的**重中之重**。根据我在公司培训期间得到的反馈，大家通常在理解 

>**Do not communicate by sharing memory; instead, share memory by communicating.**

这句话[<sup>[2]</sup>](https://blog.golang.org/share-memory-by-communicating)时存在一定困难，这种困难便是源于无法理解Go语言的设计理念（更准确的说，是无法理解[CSP](https://en.wikipedia.org/wiki/Communicating_sequential_processes)理论），无法从经典OOPL、共享内存的思路中脱离出来，快速转向golang的编程模型所导致的。我在学习的初期也无可避免地遇到过类似的问题。因此，《Intro Go》的目标及内容，便与常见的编程语言资料不太一致。《Intro Go》希望能够一方面通过与常见语言进行对比，以向大家介绍Go的**特殊性**；另一方面通过详细解剖Go的核心竞争力的具体实现，以加深读者对于Go语言设计理念的理解，帮助大家快速上手这门语言。 

因此，《Intro Go》系列文字的主要内容包括：
- golang**特性**介绍，及其与部分其他编程语言的横向比较；
- golang核心功能的实现：
	- `chan`
	- `select`
	- `go`

希望在阅读完《Intro Go》系列文字后，大家能够回答以下几个问题：
- what：什么是Go？
- why：为什么选用Go？
- when：什么时候选用Go？
- how：Go的核心功能是如何工作的？

《Intro Go》中所有引用的源代码来自golang的[github repo](https://github.com/golang/go/tree/release-branch.go1.10)，版本1.10。除此之外，我个人极力推荐大家阅读[这篇论文](http://www.usingcsp.com/cspbook.pdf)。

---
#### Ref:

[1] [The State of the Octoverse: top programming languages of 2018](https://blog.github.com/2018-11-15-state-of-the-octoverse-top-programming-languages)  
[2] [Share Memory By Communicating](https://blog.golang.org/share-memory-by-communicating)
