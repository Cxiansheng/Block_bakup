---
title: redis学习
date: 2021-01-17 23:28:21
tags: 学习
---

[TOC]

# 数据结构章节

* 数据库键总是一个字符串对象
* 数据库键的值可以是：
  * 字符串对象
  * 列表对象 list object 
  * 哈希对象 hash object
  * 集合对象  set object
  * 有序集合对象 sorted  set object

## SDS 数据结构

* redis 使用的一种名为简单动态字符串（simple dynamic string ， SDS）的抽象类型，

  用作redis的默认字符串标识。

```c++
struct sdshdr {
    
    //记录buf 数组总已使用的字节的数量
    //等于SDS所保存的字符串的长度
    int len;
    
    // 记录buf数组中未使用的字节的数量
    int free;
    
    // 字节数组，用于保存字符串（字节数组，字符数组，勿混淆）
    char buf[]
}
```



* ![image-20210108135408987](C:\Users\Cxiansheng\AppData\Roaming\Typora\typora-user-images\image-20210108135408987.png)

  如果执行``` sdscat(s, " Cluster");```

  目前s的空间不足以拼接，遂sdscat 会先扩展s的空间，然后再执行拼接操作。

  拼接完成后的SDS如下图：

  ![image-20210108135429257](C:\Users\Cxiansheng\AppData\Roaming\Typora\typora-user-images\image-20210108135429257.png)

  **sdscat 不仅对这个SDS 进行了拼接的操作，还为SDS分配了13字节的未使用空间，并且拼接之后的字符串也正好是13字节长， 这不是BUG也不是巧合，她和SDS的空间分配策略有关**

  

* 普通的C字符串的缺陷：

  因为C字符串并不记录自身的长度，所以对于一个包含了N个字符的C字符串来说，底层实现总是一个N+1个字符长的数组。

  因此，每次增长或缩短一个C字符串，程序都总要对保存这个C字符串的数组进行一次内存分重分配的操作：

  - 如果程序执行的是增长字符串的操作，比如拼接操作，那么程序需要在执行操作之前先通过内存重分配来  **扩展底层数组的空间大小**——如果忘记这一步，						就会产生 **缓冲区溢出**
  - 如果程序执行的是缩短字符串的操作，比如截断操作，那么程序需要再执行操作之前先通过内存重分配来  **释放字符串不再使用的那部分空间**——如果忘记这一步，就会产生 **内存泄漏**

**因为内存重分配涉及复杂的算法*（具体是什么呢？待了解）***并且可能需要执行系统调用，所以它通常是一个比较耗时的操作。

所以，为了避免C字符串的这种缺陷，SDS通过未使用空间解除了字符串长度和低层数组长度之间的关联：**在SDS中，buf数组的长度不一定就是字符数量加一，数组里面可以包含没使用的字节，由SDS的free属性记录**

通过未使用空间，SDS实现了空间预分配和惰性空间释放的两种优化策略。

###	1、空间预分配

* 空间预分配用于优化SDS的字符串增长操作：当SDS的API 对一个SDS进行修改，并且需要对SDS进行空间扩展的时候，程序不仅会为SDS分配修改所必须要的空间，还会为SDS分配额外的未使用空间。

  其中，额外分配的未使用空间数量由一下公式决定：

  * 如果对SDS进行修改后，SDS的长度（len属性的值）将小于1MB,那么程序分配和len属性同样大小的未使用空间，即free 和 len 值相同，eg：如果进行修改后SDS的len将变成13字节，那么free 也将是13 ，那SDS的buf数组的实际长度将变为：

    13+13+1=27字节（**不要忘记最后一个字节的空字符**）

  * 如果对SDS进行修改之后，SDS的长度将大于等于1MB，那么程序将会分配1MB的未使用空间

  * **总结：free 的大小在SDS进行空间扩展时，将会变化成不超过1MB的数值**

* 案例：

  ![image-20210108135452772](C:\Users\Cxiansheng\AppData\Roaming\Typora\typora-user-images\image-20210108135452772.png)

  执行```sdscat(s, " Cluster");```

  ![image-20210108135505730](C:\Users\Cxiansheng\AppData\Roaming\Typora\typora-user-images\image-20210108135505730.png)

  执行```sdscat(s, " Tutorial");```

  ![image-20210108135514445](C:\Users\Cxiansheng\AppData\Roaming\Typora\typora-user-images\image-20210108135514445.png)

**注意三张图中，free，len，buf的内容变化！**

在扩展SDS空间之前，SDS API 会先检查未使用空间是否足够，如果足够，API就会直接使用未使用空间，而无需执行内存重分配

通过这种预分配策略，SDS将连续增长N次字符串所需的内存重分配次数从 **必定N次降低为最多N次**

### 2、惰性空间释放

* 惰性空间释放用于优化SDS的字符串缩短操作：当SDS 的API需要缩短SDS保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节，而是使用free属性将这些字节的数量记录起来，并等待将来使用。

* 案例：

  sdstrim 函数接收一个SDS和一个C 字符串作为参数，移出SDS中所有在C 字符串中出现过的字符。

  ![image-20210108135538868](C:\Users\Cxiansheng\AppData\Roaming\Typora\typora-user-images\image-20210108135538868.png)

  执行```sdstrim(s, "XY"); //移出SDS字符串中所有的'X'和'Y'```

  ![image-20210108135552579](C:\Users\Cxiansheng\AppData\Roaming\Typora\typora-user-images\image-20210108135552579.png)

  观察free的值变化，执行了sdstrim之后SDS并没有释放多出来的8字节口径建，而是将这8字节空间作为未使用空间保留在了SDS中，之后如果要对SDS进行增长操作的话，就能用上了。

  

  接着执行```sdscat(s, " Redis")；```

  ![image-20210108135602380](C:\Users\Cxiansheng\AppData\Roaming\Typora\typora-user-images\image-20210108135602380.png)

  通过惰性空间释放策略，SDS避免了缩短字符串时所需的内存你分配操作，并为将来可能有的正常操作提供了优化。

  

  当然，SDS也提供了响应的API（**学习源码时留意一下**），让我们可以真正的释放SDS的未使用空间， 所以不用担心策略造成的内存浪费

### 3、二进制安全

注意到SDS 数据结构中的buf数组吗？注释写的是**字节数组**而非**字符数组**

因为len属性的存在，SDS是使用len的值而不是使用空字符来判断字符串是否结束，

所以buf数组里可以存放二进制数据，而不用担心像C字符串那样，读取遇到空字符就结束了。

并且，SDS的API都是二进制安全的，所有的SDS API都会以处理二进制的方式来处理buf数组里的数据

### 4、兼容部分C字符串函数

SDS的API同样遵循C字符串以空字符结尾的惯例：这些API总会将SDS保存的数据的末尾设置为空字符，并且总会在为buf数组分配空间时多分配一个字节来容纳这个空字符，这是为了让那些保存了文本数据的SDS可以重用一部分<string.h> 库定义的函数。

