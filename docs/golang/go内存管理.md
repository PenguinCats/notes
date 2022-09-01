# Go 内存管理

> 图解Go语言内存分配 - Stefno的文章 - 知乎
> https://zhuanlan.zhihu.com/p/59125443
> 
> Go runtime剖析系列（一）：内存管理 - 腾讯技术工程的文章 - 知乎
> https://zhuanlan.zhihu.com/p/323915446
> 
> 10分钟掌握golang内存管理机制 - 令狐冲的文章 - 知乎 
> https://zhuanlan.zhihu.com/p/523215127
> 
> Golang 内存管理 - 尼不要逗了的文章 - 知乎
> https://zhuanlan.zhihu.com/p/27807169

Golang 内存分配器的核心设计思想是：

+ 多级内存分配模块
+ 减少内存分配时锁的使用与系统调用
+ 多尺度内存单元，减少内存分配产生碎片

## overview

Go在程序启动时，会向操作系统申请一大块内存，之后自行管理（也可能会再要一点）。

golang 的内存分配器借鉴了 TCMalloc 现代分配器的设计思想：一次性或者提前分配多级内存模块，**减少内存分配时锁的使用和与操作系统的沟通**；多尺度内存单元，**减少内存分配产生碎片**。

![preview](https://pic3.zhimg.com/v2-d61456cb3b6c350da24b3a440301cbde_r.jpg)

申请到的内存块被分配了三个区域，在 X64 上分别是 512MB，16GB，512GB大小。

![preview](https://pic4.zhimg.com/v2-d5f5de4d6d22e67887ab4861ba5e721f_r.jpg)

+ `arena区域`就是我们所谓的堆区，Go 动态分配的内存都是在这个区域，它把内存分割成`8KB`大小的页，一些页组合起来称为`mspan`。
  
  需要注意的是，**堆区不一定的连续的。所以，Golang 的堆由很多个 arena 组成**，每个 arena 在 64 位机器上是 64MB，且起始地址与 arena 的大小对齐，然后再细分为 page。

+ `bitmap区域`标识`arena`区域哪些地址保存了对象、对象是否包含指针、`GC`标记信息。

+ `spans区域`存放`mspan`的指针，记录了相应的页 (span)对应的 mspan 的地址。

![preview](https://pic2.zhimg.com/v2-9410a51dfd5e7399714a7bc757bfdc29_r.jpg)

## mspan

**mspan 是堆上内存管理的基本单元，由一连串 8kb 的页构成**。

> 对于 mspan 来说，它的`Size Class`会**决定它所能分到的页数，这也是写死在代码里的**。

+ golang根据内存大小将mspan 分成了67个等级 `Size Class`，**每个 mspan 根据等级被切分成了很多小的对象**，用于不对尺度的对象分配。

+ 使用一个位图来标记其尚未使用的 object

+ 数组里最大的数是 32KB，超过此大小就是大对象了，实际上直接由堆内存分配。
  
  而对于微小对象（小于16B），分配器会将其进行合并，将几个对象分配到同一个object 中。

![](https://pic4.zhimg.com/80/v2-38c31cf60d1514beb5be81d465ea36af_720w.jpg)

## mcache

每个工作**线程（其实对应的是 GMP 中绑定了 P 的 M，可以当成是绑核的？）** 有一个独立的堆内存 mcache，在 mcache 上申请内存**不需要加锁**，每个 mache 有67种尺度内存单元，每个尺度的内存又由 2 个 mspan 构成，**分别存储指针类型和非指针类型**（这样 GC 的时候可以区分出是否需要进一步扫描下去）。当程序在堆上创建对象时，分配器会根据对象的大小（小于32kb）和类型（是否为指针）从 67\*2 种 mspan 中分配内存。

```go
type mcache struct {
    alloc [numSpanClasses]*mspan
    ......
}
numSpanClasses = _NumSizeClasses << 1
```

`mcache` 在初始化的时候是没有任何 `mspan` 资源的，在使用过程中会动态地从`mcentral`申请，之后会缓存下来。当对象小于等于32KB大小时，使用 `mcache` 的相应规格的 `mspan` 进行分配。

## mcentral

如果 mache 内存不足，会从所有线程共享的缓存 mcentral 中申请内存。一共有 67\*2 种mcentral， mcache 中空间不足的 mspan 会从对应的 mcentral 申请内存，**申请内存时需要加锁**，但是锁冲突的概率并不高。

> 当一个`mcache`从`mcentral`申请`mspan`时，只需要在独立的`mcentral`中使用锁，并不会影响申请其他规格

为了进一步提升内存分配效率，每一个 mcentral 由2个链表构成：

+ `empty`表示这条链表里的`mspan`都被分配了`object`，或者是已经被`cache`取走了的`mspan`，这个`mspan`就被那个工作线程独占了。

+ `nonempty`则表示有空闲对象的`mspan`列表。

> `mcache`从`mcentral`获取和归还`mspan`的流程：
> 
> - 获取 加锁；从`nonempty`链表找到一个可用的`mspan`；并将其从`nonempty`链表删除；将取出的`mspan`加入到`empty`链表；将`mspan`返回给工作线程；解锁。  
> 
> - 归还 加锁；将`mspan`从`empty`链表删除；将`mspan`加入到`nonempty`链表；解锁。

## mheap

**当 mcentral 上 mspan 不足时** 或者 **对象大于 32kb 时**会从 mheap上 申请内存，全局仅有一个 mheap，**也需要加锁访问**。

`mheap`主要用于大对象的内存分配，以及管理未切割的`mspan`，用于给`mcentral`切割成小对象。

![preview](https://pic4.zhimg.com/v2-0f51158704083f36561a15edc89d6f27_r.jpg)

## 分配流程

大体上的分配流程：

- 32KB 的对象，直接从mheap上分配；  

- <=16B 的对象使用mcache的tiny分配器分配；
- (16B,32KB] 的对象，首先计算对象的规格大小，然后使用mcache中相应规格大小的mspan分配；
- 如果mcache没有相应规格大小的mspan，则向mcentral申请
- 如果mcentral没有相应规格大小的mspan，则向mheap申请
- 如果mheap中也没有合适大小的mspan，则向操作系统申请
