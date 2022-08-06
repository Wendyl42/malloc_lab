
# 简单的通用动态内存分配器

## 描述

本项目基于CS:APP Malloc lab，实现了一个简单的通用内存分配器，包括
- `void *malloc(size_t size)`
- `void free(void *bp)`
- `void *realloc(void *oldptr, size_t size)`
- `void *calloc(size_t nmemb, size_t size)`

它们与标准库的同名函数有相同的功能，可以在堆上动态地分配与释放内存。并且，我们希望尽可能地优化吞吐量（即一段时间内可以完成的请求数）与空间利用率

## 设计 
我们假定块的地址按照8字节对齐，意味着块的大小一定是8的倍数


### 1. 块的结构是什么样的？
我们将块分为三个部分：头部、有效载荷、脚部
- 头部：占用4个字节。头部的最低3个bit置零得到的是该块的大小，最低一个bit标记块是已分配还是空闲
- 有效载荷：用于存储应用程序数据。我们设计这部分不少于8个字节，原因会在稍后说明
- 脚部：占用4个字节，和头部的内容相同。设置脚部的意义在于方便其地址上后继的块可以读取该块的信息

我们要求块指针必须指向有效载荷部分的起点，并设计若干宏来读取块的不同信息

此外，我们设计两个特殊的块，以便我们后续对块的操作：

- 序言块：只有头部和脚部，且头部信息中标记该块大小为8字节、已分配
- 结束块：只有头部，且头部信息标记该块为0字节，已分配。之后每次堆发生增长（设置为4KB），旧的结束块自动成为新的4KB块的头部，同时我们在堆的末尾设置新的结束块


### 2. 用分离空闲链表组织空闲块

所谓分离空闲链表，就是维护多个空闲链表，同一链表内的块大小接近。在这里，我们维护10个空闲链表，其大小范围分别是{16}, {17~32}, {33~64}, {65~128}, {129~256}, {257~512}, {513~1024}, {1025~2048}, {2049~4096}, {4097~INF}

每一个链表都设计为显式空闲链表，即空闲块在它的数据载荷部分存放链表中前后块的“地址”。这一点要求即便是最小的空闲块，其有效载荷部分也必须足以存放这些信息。不过，我们实际存放的并非前后块的完整地址，而是偏移量，这一点会在优化策略一节中分析。下文中所有打引号的地址，实际都是偏移量


在分配内存时，我们遍历每一个可行的链表寻找空闲块，并将空闲块切分为两部分，前者返回给用户，后者作为新的空闲块重新插入链表。在释放内存时，我们尝试将新的空闲块与其地址上的前后块（如果空闲的话）进行合并，并插入空闲链表中。由于性能瓶颈更多在于空间而非时间，因此我们宁可多一些复杂的边界情况讨论，也不引入头尾的哨兵结点


### 3. 维护堆与空闲链表

当我们初始化一个堆时，首先需要分配一些空间存放元数据。堆的起始部分用于存放10个空闲链表的头节点“地址”，然后是一个对齐用的填充4字节word。接下来，分配初始的序言块和结束块

之后，我们让堆进行第一次大的增长，并将新增长的部分组织成第一个空闲块

#### a. 堆的增长

我们默认堆每次增长的大小是4KB。它首先通过增大brk指针扩充空间。之后，它将增长部分的最后4字节作为新的结束块，而其余部分和旧的结束块则作为一个新的4KB空闲块（注意到旧的结束块恰好称为了该空闲块的头部），并填写其头部和脚部。最后，它尝试将该空闲块尝试与地址上的前一个块（如果空闲的话）进行合并

#### b. 将空闲块插入列表
我们根据空闲块的大小，容易确定要将其导向哪一个空闲链表。通过堆起始部分存放的链表“地址”，我们可以定向到该链表，并插入空闲块

所谓的“插入”空闲块，就是在该块（以及链表中近邻的块）的有效载荷部分写入“地址”。倘若插入的是头节点，还需要更新堆起始部分的链表“地址”

#### c. 将块从空闲链表移出
与插入的情况类似，我们根据块有效载荷部分存放的“地址”信息，找到它在链表中的前驱与后继，并对其进行修改。同样地，这里需要比较复杂的边界情况处理

#### d. 空闲块的切分

这里的重点是切分出给用户的部分后，剩余部分的大小。16字节是我们设置的最小块大小（4字节头部+8字节载荷+4字节脚部），倘若剩余部分达不到16字节，那么就不做切分，直接将完整的空闲块做好标记后返回

#### e. 空闲块的合并

我们需要检查与该块**地址上相邻**的两个块是否已分配。倘若有空闲块，那么我们需要将待合并的空闲块从链表中移出，待合并完毕后再将新块插入链表。


### 4. 若干优化策略

#### a. 用基址+偏移量代替地址
在64位系统中，一个地址的长度为8个字节。然而，我们可以只设置1个基址指针，它也是序言块的块指针。之后，其它所有的空闲块地址都可以用一个4字节的，相对于基址的偏移量代替

这样设计的最直接效果在于，最小的块大小从24字节缩减为16字节，一定程度上缓解内部碎片的问题。这一优化策略也可以兼容32位系统和64位系统

#### b. 链表内空闲块按大小排序

空闲链表内的空闲块如何排序？如果我们专注于时间吞吐量，那么应该采用的是后进先出，即我们总将块插入链表的头部。在搜寻块的时候，也采用首次匹配策略来优化时间表现。然而，这样做的空间利用率非常差

我们也可以将块按照地址高低排序，实践表明即便仍然使用首次匹配策略，这样做的空间利用率仍然比LIFO策略更佳。

但是，既然选择对空闲块排序，那么更好的选择是直接按照块的大小排序。在这种情况下，首次匹配等同于最佳匹配，空间利用率可以进一步提升


## 编译与测试

首先运行`make clean`清除已编译的目标文件，然后运行`make`重新编译项目

如果希望运行所有测试样例，键入`./mdriver`即可。如果希望与C标准库的`malloc`系列函数对比性能，可以指定参数`-l`

有关测试器的更多用法，参见[CS:APP3e, Bryant and O'Hallaron (cmu.edu)](http://csapp.cs.cmu.edu/3e/labs.html)中Malloc lab的writeup文件

## 其它

项目中其它文件的信息概要如下

- `mm-naive.c`与`mm-textbook.c`：均为CS:APP书中提供的简单实现，后者基于隐式空闲链表。尽管可以保证正确性，但时空性能不佳
- `clock.{c,h}`：用于获取x86-64机器的CPU周期计数
- `fcyc.{c,h}`：基于CPU周期的计时函数
- `ftimer.{c,h}`：基于间隔计时器与`gettimeofday()`的计时函数
- `fsecs.{c,h}`：对若干计时函数的包装
- `memlib.{c,h}` ：对堆进行抽象，并包装了`sbrk`等函数
