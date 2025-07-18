---
layout:     post
title:      "GoLangMap 详解"
subtitle:   " GoLang "
date:       2025-05-11 12:00:00
author:     "Lils"
header-img: "img/post-bg-2015.jpg"
tags:
    - GoLang
---

## 1 基本用法
### 1.1 概述

`map` 又称字典，是一种常用的数据结构，核心特征包含下述三点：
1. 存储基于 `key-value` 对映射的模式；
2. 基于 `key` 维度实现存储数据的去重；
3. 读、写、删操作控制，时间复杂度 `O(1)`.

### 1.2 初始化
#### 1.2.1 几种初始化方法

`golang` 中，对 `map` 的初始化分为以下几种方式：

```go
myMap1 := make(map[int]int, 2)
```

通过 `make` 关键字进行初始化，同时指定 `map` 预分配的容量.

```go
myMap2 := make(map[int]int)
```

通过 `make` 关键字进行初始化，不显式声明容量，因此默认容量为 0.

```go
myMap3 := map[int]int {
	1:2,
	3:4,
}
```

初始化操作连带赋值，一气呵成.

#### 1.2.2 key 的类型要求

`map` 中，`value` 并没有严格约束，`key` 的数据类型必须为可比较的类型，slice、map、func 不可比较
### 1.3 读

读 `map` 分为下面两种方式：

```go
v1 := myMap[10]
```

第一种方式是直接读，倘若 `key` 存在，则获取到对应的 `val`，倘若 `key` 不存在或者 `map` 未初始化，会返回 `val` 类型的**零值**作为兜底.

```go
v2, ok := myMap[10]
```

第二种方式是读的同时添加一个 `bool` 类型的 `flag` 标识是否读取成功. 倘若 `ok == false`，说明读取失败， `key` 不存在，或者 `map` 未初始化.

此处同一种语法能够实现不同返回值类型的适配，是由于代码在汇编时，会根据返回参数类型的区别，映射到不同的实现方法.
### 1.4 写

```go
myMap[5] = 6
```

写操作的语法如上. 须注意的一点是，倘若 `map` 未初始化，直接执行写操作会导致 `panic`：

```go
const plainError string
panic(plainError("assignment to entry in nil map"))
```

### 1.5 删

```go
delete(myMap, 5)
```

执行 `delete` 方法时，倘若 `key` 存在，则会从 `map` 中将对应的 `key-value` 对删除；倘若 `key` 不存在或 `map` 未初始化，则方法**直接结束**，不会产生显式提示.

### 1.6 遍历

遍历分为下面两种方式：

```go
for k, v := range myMap {  
	// ...
}
```

基于 `k,v` 依次承接 `map` 中的 `key-value` 对；

```go
for k := range myMap {
	// ...
}
```

基于 k 依次承接 `map` 中的 `key`，不关注 `val` 的取值.

需要注意的是，在执行 `map` 遍历操作时，获取的 `key-value` 对并没有一个固定的顺序，**因此前后两次遍历顺序可能存在差异**.

### 1.7 并发冲突

`map` **不是并发安全的数据结构**，倘若存在并发读写行为，会抛出 `fatal error`.

具体规则是：
1. 并发读没有问题；
2. 并发读写中的“写”是广义上的，包含写入、更新、删除等操作；
3. 读的时候发现其他 `goroutine` 在并发写，抛出 `fatal error`；
4. 写的时候发现其他 `goroutine` 在并发写，抛出 `fatal error`.

```go
fatal("concurrent map read and map write")
fatal("concurrent map writes")
```

需要关注，此处并发读写会引发 `fatal error`，是一种比 `panic` 更严重的错误，无法使用 `recover` 操作捕获.

## 2 核心原理

`map` 又称为 `hash map`，在算法上基于 `hash` 实现 `key` 的映射和寻址；在数据结构上基于桶数组实现 `key-value` 对的存储.

以一组 `key-value` 对写入 `map` 的流程为例进行简述：

1. 通过哈希方法取得 `key` 的 `hash` 值；
2. `hash` 值对桶数组长度取模，确定其所属的桶；
3. 在桶中插入 `key-value` 对.

`hash` 的性质，保证了相同的 `key` 必然产生相同的 `hash` 值，因此能映射到相同的桶中，通过桶内遍历的方式锁定对应的 `key-value` 对.

因此，只要在宏观流程上，控制每个桶中 `key-value` 对的数量，就能保证 `map` 的几项操作都限制为常数级别的时间复杂度.
### 2.1 `hash`

`hash` 译作**散列**，是一种将任意长度的输入压缩到某一固定长度的输出摘要的过程，由于这种转换属于**压缩映射**，输入空间远大于输出空间，因此不同输入可能会映射成相同的输出结果.

>  此外，**hash在压缩过程中会存在部分信息的遗失**，因此这种映射关系具有**不可逆**的特质.

1. `hash` 的**可重入性**：相同的 `key`，必然产生相同 的 `hash` 值；
2. `hash` 的**离散性**：只要两个 `key` 不相同，不论其相似度的高低，产生的 `hash` 值会在整个输出域内均匀地离散化；

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-11-20.29.25.jpeg?raw=true)

3. `hash` 的**单向性**：企图通过 `hash` 值反向映射回 `key` 是无迹可寻的.

> 上文我们提到过**压缩映射**，在映射的过程当中我们不要求把原始的输入数据源完整的信息都保留下来，可能中间会存在一部分的信息丢失，在哈希映射的过程中是可以接受的，所以在映射的过程中可能有一部分原始数据就已经丢弃了，最终得到了一个对应的输出。可能这个输出本身的信息是不齐全的，所以还原不出来输入的内容。

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-11-20.30.20.jpeg?raw=true)

4. `hash` 冲突：由于输入域（`key`）无穷大，输出域（`hash` 值）有限，因此必然存在**不同 `key` 映射到相同 `hash` 值**的情况，称之为 `hash` 冲突.

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-11-20.31.15.jpeg?raw=true)

### 2.2 桶数组

`map` 中，会通过长度为 2 的整数次幂的桶数组进行 `key-value` 对的存储：
1. 每个桶固定可以存放 8 个 `key-value` 对；
2. 倘若超过 8 个 `key-value` 对打到桶数组的同一个索引当中，此时会通过创建桶链表的方式来化解这一问题.

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-11-20.32.50.jpeg?raw=true)

> 假设现在有长度为4的桶数组，接下来我要去插入一个 `{key1: value1}` 数据，我们来梳理一遍具体流程：
> 1. 首先 `map` 又称为 `hash map`，是重度依赖哈希算法的，所以我们先要把 `key` 值取出来，然后根据哈希算法来取到该 `key` 值的哈希结果`h1`，`h1 = hash(key1)`，这个 `h1` 可能是一个很大的整数，可能是64位或者128位的。那这个 `key-value` 对应该放在哪一个桶当中呢？
> 2. 我们希望尽可能的每一个桶都能均匀的承载 `key-value` 对数据，不出现所有的数据都聚集在同一个桶上的这种情况，这就是我们上面说过的 `hash` 的**离散性**。在得到 `key1` 的哈希值 `h1` 之后，我们会对桶做取余的操作。假设 `h1 % 4 == 1` ，所以这个 `{key1: value1}` 就属于1号桶。时间复杂度是常数级别的。
> 
> ![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-11-21.20.45.jpeg?raw=true)

> 当我们在进行查询操作的时候跟上面一样。
> 1. 首先通过 `key1` 去取到对应的哈希值 `h1`，只要我们输入的数据源是相同的并且使用的哈希映射的函数也是相同的，那我们得到的结果 `h1` 就一定是相同的。
> 2. 接下来通过 `h1` 对桶数组取余得到1号桶，再到1号桶中去找对应的数据。
> 

### 2.3 解决 `hash` 冲突

首先，由于 `hash` 冲突的存在，不同 `key` 可能存在相同的 `hash` 值；
再者，`hash` 值会对桶数组长度取模，因此不同 `hash` 值可能被打到同一个桶中.
综上，不同的 `key-value` 可能被映射到 `map` 的同一个桶当中.
此时最经典的解决手段分为两种：**拉链法**和**开放寻址法**.

（1）拉链法

拉链法中，将命中同一个桶的元素通过链表的形式进行链接，共享这个桶空间，因此很便于动态扩展.

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-11-20.37.29.jpeg?raw=true)

（2）开放寻址法

开放寻址法中，明确了每一个桶只能存放一条数据，先到来的会把这个桶的空间占据。在插入新条目时，会基于一定的探测策略持续寻找，直到找到一个可用于存放数据的空位为止.

> 比如说下图中 `key3` 进来之后本应该属于 `index=1` 的桶，但是发现这个桶已经被别人占满了，所以 `key3` 会从 `index=1` 的基础上从小到大遍历，尝试看 `index=2` 这个桶是不是空的，知道找到下一个空桶的索引位置并且把它占为己有。

> 在查询的时候也是一样，比如对于的 `key` 映射到这个桶 `index=1`，然后我会先去 `index=1` 的桶的位置去对应数据的情况，发现里面存储的 `key-value` 对的 `key` 和此时我在查询的时候使用的 `key` 是不一样的，那我就会根据索引策略，从 `index=1` 出发，去看 `index=2` 桶中的 `key-value` 对对应的 `key` 是否是我查询使用的 `key`，如果不是的话再去看 `index=3`，以此类推知道找到结果为止。

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-11-20.38.35.jpeg?raw=true)

对标拉链还有开放寻址法，两者的优劣对比：

| **方法** | **优点**                                              |
| ------ | --------------------------------------------------- |
| 拉链法    | 简单常用；<br>无需预先为元素分配内存.                               |
| 开放寻址法  | 无需额外的指针用于链接元素；<br>内存地址完全连续，可以基于局部性原理，充分利用 CPU 高速缓存. |

在 `map` 解决 hash /分桶 冲突问题时，实际上结合了拉链法和开放寻址法两种思路. 以 `map` 的插入写流程为例，进行思路阐述：

1. 桶数组中的每个桶，严格意义上是一个单向桶链表，以桶为节点进行串联；
2. 每个桶固定可以存放 8 个 `key-value` 对（内存是连续的）；
3. 当 `key` 命中一个桶时，首先根据开放寻址法，在桶的 8 个位置中寻找空位进行插入；
4. 倘若桶的 8 个位置都已被占满，则基于桶的溢出桶指针，找到下一个桶，重复第（3）步；
5. 倘若遍历到链表尾部，仍未找到空位，则基于拉链法，**在桶链表尾部续接新桶**，并插入 `key-value` 对

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-11-20.39.46.jpeg?raw=true)

### 2.4 扩容优化性能

倘若 `map` 的桶数组长度固定不变，那么随着 `key-value` 对数量的增长，当一个桶下挂载的 `key-value` 达到一定的量级，此时操作的时间复杂度会趋于线性，无法满足诉求.

因此在实现上，`map` 桶数组的长度会随着 `key-value` 对数量的变化而实时调整，以保证每个桶内的 `key-value` 对数量始终控制在常量级别，满足各项操作为 `O(1)` 时间复杂度的要求.

`map` 扩容机制的核心点包括：
1. 扩容分为**增量扩容**和**等量扩容**；
	- 增量扩容：为了保证每一个桶链表对应的数据规模能维持在一个常数量级，最终它会不断地随着数据规模的扩大而去翻倍的扩增这个桶数组的长度，去维持住每一个桶分配到的数据量级。
	- 等量扩容：
		- 触发时机：当一个桶链表不断的进行插入和删除操作之后我们发现一个又一个桶节点中间会出现一些空洞。于是每一个桶节点其实是不完整的，可能这个链表会被拉的很长，但是内部的数据填充率很低，数据饱和度不够高，最终可能会演化成一种内存泄漏问题。看上去 `key-value` 对的数量不多，但是链表长度很长，链表的节点数量已经很多了。此时就会触发一次等量扩容。
		- 触发机制：是看当前桶链表中的桶节点的数量和我们原本桶数组的长度之间的比值是否达到了阈值，如果发现这个桶链表的长度特别长，比值已经超过了阈值，就会触发一次等量扩容。
		- 等量扩容的目标就是把这些数据迁移一遍，在迁移的过程当中把空洞给补上，同时通过这种弥补间隙的方式去提高数据的利用率，减少每一个桶链表的长度。
2. 当桶内 `key-value` 总数 / 桶数组长度 > 6.5 时发生**增量扩容**，桶数组长度增长为原值的两倍；
3. 当桶内溢出桶（溢出桶：指的是桶链表当中从第一个节点之后开始去拉链拉的来的节点，上图有）数量大于等于 2^B（B 为桶数组长度的指数，B 最大取 15）时有可能存在数据饱和度过低的问题，发生**等量扩容**，桶的长度保持为原值；
4. 采用**渐进扩容**的方式，当桶被实际操作到时，由使用者负责完成数据迁移，避免因为一次性的全量数据迁移引发性能抖动.

> 如果在扩容的时候一次性把所有的数据进行迁移的话那迁移的成本太高了，时间复杂度要到O(n)，所以要采用渐进扩容，把一次性扩容的压力分摊给后续一序列到来的写操作。

> 渐进扩容会同时存在一个老桶数组和新桶数组，两个桶数组并行的情况，此时新写入的数据通通都通过新的桶数组来承载，包括后续一系列的写操作，我们会去先看一眼新桶数组中有没有满足对应的要求如果满足的话我们会优先用新的桶数组，如果没有的话再去用老桶数组兜底。

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-11-20.41.39.jpeg?raw=true)

> 首先上图中有一个老桶数组，长度是4，每个桶都挂载了一些节点，翻倍扩容之后桶数组长度是8，那么老桶数组中的数据节点应该放在新桶数组的哪些位置呢？
> 对于数据的 key ，我们使用的哈希函数是不变的，在得到了哈希结果之后进行模运算的桶数组的长度发生了变化，要么该数据还在原本的索引位置，要么就在原本的桶索引的基础之上加上一倍老桶数组的长度的索引位置，并且应该在老的位置，还在新的位置。（一个写操作迁移一个桶一个写操作迁移一个桶，愚公移山说是）
> 

## 3 数据结构
### 3.1 hmap

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-12-17.52.22.jpeg?raw=true)

```go
type hmap struct {
	count      int            // map 中的 key-value 总数；
	flags      uint8          // map 状态标识，可以标识出 map 是否被 goroutine 并发读写；
	B          uint8          // B：桶数组长度的指数，桶数组长度为 2^B；
	noverflow  uint16         // map 中溢出桶的数量；
	hash0      uint32         // hash 随机因子，生成 key 的 hash 值时会使用到；
	buckets    unsafe.Pointer // 桶数组；
	oldbuckets unsafe.Pointer // 扩容过程中老的桶数组；
	nevacuate  uintptr        // 扩容时的进度标识，index 小于 nevacuate 的桶都已经由老桶转移到新桶中；
	extra      *mapextra      // 预申请的溢出桶.
}
```

### 3.2 mapextra

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-12-18.03.32.jpeg?raw=true)

```go
type mapextra struct {
    overflow    *[]*bmap
    oldoverflow *[]*bmap

    nextOverflow *bmap
}
```

在 map 初始化时，倘若容量过大，会提前申请好一批溢出桶，以供后续使用，这部分溢出桶存放在 hmap.mapextra 当中：

（1）mapextra.overflow：供桶数组 buckets 使用的溢出桶；

（2）mapextra.oldoverFlow: 扩容流程中，供老桶数组 oldBuckets 使用的溢出桶；

（3）mapextra.nextOverflow：下一个可用的溢出桶.

### 3.3 bmap

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-12-18.04.05.jpeg?raw=true)

```go
const bucketCnt = 8
type bmap struct {
    tophash [bucketCnt]uint8
}
```

（1）bmap 就是 map 中的桶，可以存储 8 组 key-value 对的数据，以及一个指向下一个溢出桶的指针；

（2）每组 key-value 对数据包含 key 高 8 位 hash 值 tophash，key 和 val 三部分；

（3）在代码层面只展示了 tophash 部分，但由于 tophash、key 和 val 的数据长度固定，因此可以通过内存地址偏移的方式寻找到后续的 key 数组、val 数组以及溢出桶指针；

（4）为方便理解，把完整的 bmap 类声明代码补充如下：

```go
type bmap struct {
    tophash [bucketCnt]uint8
    keys [bucketCnt]T
    values [bucketCnt]T
    overflow uint8
}
```

## 4 构造方法

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-12-19.46.21.jpeg?raw=true)

## 5 读流程

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-12-19.49.43.jpeg?raw=true)

### 5.1 读流程梳理

map 读流程主要分为以下几步：
1. 根据 `key` 取 `hash` 值；
2. 根据 `hash` 值对桶数组取模，确定所在的桶；
3. 沿着桶链表依次遍历各个桶内的 `key-value` 对；
4. 命中相同的 `key`，则返回 `value`；倘若 `key` 不存在，则返回零值.

`map` 读操作最终会走进 `runtime/map.go` 的 `mapaccess` 方法中，下面开始阅读源码：

### 5.2 `mapaccess` 方法源码走读

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    if h == nil || h.count == 0 {
        return unsafe.Pointer(&zeroVal[0]) // 不用看了直接返回0值
    }
    if h.flags&hashWriting != 0 { // 此时有goroutine在并发写
        fatal("concurrent map read and map write")
    }
    hash := t.hasher(key, uintptr(h.hash0))
    m := bucketMask(h.B)
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
    if c := h.oldbuckets; c != nil {
        if !h.sameSizeGrow() { // 如果不是等量扩容，那就是增量扩容喽
            m >>= 1
        }
        oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        if !evacuated(oldb) {
            b = oldb
        }
    }
    top := tophash(hash)
bucketloop:
    for ; b != nil; b = b.overflow(t) { // 遍历桶节点本身
        for i := uintptr(0); i < bucketCnt; i++ { // 在桶节点内部遍历8个凹槽
            if b.tophash[i] != top {
                if b.tophash[i] == emptyRest { // 如果被标示位emptyRest，代表从这个凹槽往后的所有位置都是空的
                    break bucketloop
                }
                continue
            }
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey() {
                k = *((*unsafe.Pointer)(k))
            }
            if t.key.equal(key, k) {
                e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
                if t.indirectelem() {
                    e = *((*unsafe.Pointer)(e))
                }
                return e
            }
        }
    }
    return unsafe.Pointer(&zeroVal[0])
}


func (h *hmap) sameSizeGrow() bool {
    return h.flags&sameSizeGrow != 0
}


func evacuated(b *bmap) bool {
    h := b.tophash[0]
    return h > emptyOne && h < minTopHash
}
```

## 6 写流程

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-12-20.09.24.jpeg?raw=true)

### 6.1 写流程梳理

map 写流程主要分为以下几步：

1. 根据 `key` 取 `hash` 值；
2. 根据 `hash` 值对桶数组取模，确定所在的桶；
3. 倘若 `map` 处于扩容，则迁移命中的桶，帮助推进**渐进式扩容**；
4. 沿着桶链表依次遍历各个桶内的 `key-value` 对；
5. 倘若命中相同的 `key`，则对 `value` 中进行更新；
6. 倘若 `key` 不存在，优先去看桶链表的前置位有没有空的凹槽，填补这部分的空洞，插入 `key-value` 对；
7. 倘若发现 `map` 达成扩容条件，则会开启扩容模式，并重新返回第（2）步.
8. 最后在退出之前还要再去确认一遍 `flags` 标识，也就是并发写标识是否被破坏，如果被破坏了的话那代表一定有其他 `goroutine` 也在尝试执行并发操作，抛出 `fatal`。

## 7 删流程

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-12-21.24.05.jpeg?raw=true)

### 7.1 删除 kv 对流程梳理

map 删除 kv 对流程主要分为以下几步：

1. 根据 `key` 取 `hash` 值；
2. 根据 `hash` 值对桶数组取模，确定所在的桶；
3. 倘若 `map` 处于扩容，则迁移命中的桶，帮助推进**渐进式扩容**；
4. 沿着桶链表依次遍历各个桶内的 `key-value` 对；
5. 倘若命中相同的 `key`，删除对应的 `key-value` 对；并将当前位置的 `tophash` 置为 `emptyOne`，表示为空；
6. 倘若当前位置为末位，或者下一个位置的 `tophash` 为 `emptyRest`，则沿当前位置**向前遍历**，将毗邻的 `emptyOne` 统一更新为 `emptyRest`.

> 每一个凹槽 `tophash` 都会有两个额外的标识位，一个叫 `emptyOne` 代表当前这个凹槽的数据已经不存在了，另一个叫 `emptyRest`，代表从当前这个凹槽往后一直到这个链表结尾的后置位的所有凹槽都是空的。

## 8 遍历流程

### 8.1 遍历流程梳理

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-12-21.35.57.jpeg?raw=true)

`map` 的遍历流程首先会走进 `runtime/map.go` 的 `mapiterinit()` 方法当中，初始化用于遍历的迭代器 `hiter`；接着会调用 `runtime/map.go` 的 `mapiternext()` 方法开启遍历流程.

> 首先会生成一个随机数，这个随机数决定了两个东西：决定了遍历的起点，从哪个桶开始；决定了是从哪一个 kv 对的凹槽开始。假设是从3号桶节点开始遍历，遍历玩3号桶之后再去遍历4号桶，遍历到末尾之后重新回到头部，从0开始遍历到2号桶节点。
> 
> ![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-12-21.52.32.jpeg?raw=true)
>
> 那从上面就能看出，从哪个桶开始遍历是不固定的，从哪个凹槽开始也是不固定的，最终体现出来的整体的遍历顺序也是不固定的。

> 倘若遍历到的桶属于旧桶数组未迁移完成的桶，需要按照其**在新桶中的顺序**完成遍历. 比如，增量扩容流程中，旧桶中的 `key-value` 对最终应该被分散迁移到新桶数组的 x、y 两个区域，则此时遍历时，哪怕 `key-value` 对仍存留在旧桶中未完成迁移，遍历时也应该严格按照其在新桶数组中的顺序来执行.

## 9 扩容流程

### 9.1 扩容类型

![](https://github.com/akalils/akalils.github.io/blob/master/img/FlowChart/20250511/截屏2025-05-12-21.58.38.jpeg?raw=true)

map 的扩容类型分为两类，一类叫做**增量扩容**，一类叫做**等量扩容**。在2.4讲的比较清楚

（1）增量扩容
表现：扩容后，桶数组的长度增长为原长度的 2 倍；
目的：降低每个桶中 `key-value` 对的数量，优化 `map` 操作的时间复杂度.

（2）等量扩容
表现：扩容后，桶数组的长度和之前保持一致；但是溢出桶的数量会下降.
目的：提高桶主体结构的数据填充率，减少溢出桶数量，避免发生内存泄漏.

### 9.2 何时扩容

1. 只有 `map` 的**写流程**可能开启扩容模式；
2. 写 `map` 新插入 `key-value` 对之前，会发起是否需要扩容的逻辑判断：

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // ...
    
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)
        goto again
    }


    // ...
}
```

## 参考资料
本文核心思路参考了 **小徐先生** 的博客文章，在此表示感谢：  
- **作者**：小徐先生  
- **原文标题**：[Golang map 实现原理](https://mp.weixin.qq.com/s/PT1zpv3bvJiIJweN3mvX7g)
