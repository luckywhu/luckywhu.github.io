---
layout: post
title:  "Rocksdb Compaction实现逻辑"
date:   2017-08-03 01:09:37 +0800
categories: kv rocksdb
---

**1. Rocksdb文件组织整体视图**

Rocksdb磁盘上的文件组织成一个多层结构，以level-1，level-2,level-3（或者L1,L2)区分，其中

A, L0层可能有多个文件，由于L0层的文件来自内存中的memtable直接写盘，因此多个L0文件之间可能存在key重叠，

B, L1-Ln层，每一层之内，每个文件都包含不同的key区间，而不同层之内，key区间会存在重叠部分

![img](https://gw.alicdn.com/tfscom/TB1LYewRFXXXXXSXpXXXXXXXXXX.png)

在每一层中，key根据所处的区间，保存在不同的SST文件中，同时每个SST文件内部也是有序的

![img](https://gw.alicdn.com/tfscom/TB1RANORFXXXXchapXXXXXXXXXX.png)

每一层的文件大小之和都有个限定值，如果超过限制，Compaction将会将本层的文件与下层的文件合并，减少本层文件数目及大小。

![img](https://gw.alicdn.com/tfscom/TB1WQXVRFXXXXcOaXXXXXXXXXXX.png)

**2，Compaction逻辑**

如果L0层的文件数目超过一个限制值level0_file_num_compaction_trigger，则L0层的文件会被compaction，并写入到L1层文件中，由于L0层的文件之间Key是可能重叠的，因此合并L0的时候，需要所有文件一起合并，**这是一个多路归并过程** 如下图所示：

![img](https://gw.alicdn.com/tfscom/TB1trVVRFXXXXXOapXXXXXXXXXX.png)

合并完L0层文件之后，合并之后的内容会写入L1层，此时可能导致L1层文件整体size变大，并超过某限制值，此时会触发L1层与L2层的合并

![img](https://gw.alicdn.com/tfscom/TB1fqBYRFXXXXcHaXXXXXXXXXXX.png)

对于L1-L2层的compaction，可能会选中1~n个同一层的sst文件，并和其下一层进行合并，如下图所示

![img](https://gw.alicdn.com/tfscom/TB1RXKnRFXXXXXeXFXXXXXXXXXX.png)

合并的结果是增加L2层文件的数目，如下图所示

![img](https://gw.alicdn.com/tfscom/TB1jpGlRFXXXXaXXFXXXXXXXXXX.png)

依次类推，L2层与L3层之间的合并如下图：

![img](https://gw.alicdn.com/tfscom/TB1zY1wRFXXXXXWXpXXXXXXXXXX.png)

L2与L3之间的合并结果如下图所示：

![img](https://gw.alicdn.com/tfscom/TB1sjBRRFXXXXaoapXXXXXXXXXX.png)

考虑到多层之间的compaction并不冲突，因此多层之间的compaction过程可以并行执行，如下图所示

![img](https://gw.alicdn.com/tfscom/TB1T94QRFXXXXaZapXXXXXXXXXX.png)

**3，特殊的L0层的compaction**

由于L0层的特殊性，其多个文件之间的key可能是重叠的，因此原本不能通过并行compaciton来实现文件的，但如果多个L0之间Range如果可以划分开，形成不同的group，则可以开启多个subcompaction任务，进行并行执行，如下图所示：

![img](https://gw.alicdn.com/tfscom/TB1xLqsRFXXXXa_XpXXXXXXXXXX.png)

**4，sst文件的格式**

一个sst文件根据用户的配置，会有不同的大小，如4MB，16MB，256MB等，sst文件由block以及一些meta信息组成, 大体的架构如下图所示

```
<beginning_of_file>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1: filter block]                  (see section: "filter" Meta Block)
[meta block 2: stats block]                   (see section: "properties" Meta Block)
[meta block 3: compression dictionary block]  (see section: "compression dictionary" Meta Block)
...
[meta block K: future extended block]  (we may add more meta blocks in the future)
[metaindex block]
[index block]
[Footer]                               (fixed size; starts at file_size - sizeof(Footer))
<end_of_file>
```

1. 一个sst文件中的key和value会按key的大小顺序排好序存放，然后存放在不同的block中，每个block也是从文件开始逐个存放，
2. 数据block内容之后，存放了一些meta block，这些meta  block的内容用于快速定位key属于那个block。例如index block中保存了每个data block中记录的最大key大小，这样通过在index block中查找可以迅速定位到一个key具体位于那个data block

**5, data block的格式**

datablock中所有key-value对紧凑排列，同时使用前缀压缩以节省空间。格式如下

![img](https://gw.alicdn.com/tfscom/TB1HjF9RFXXXXcvXVXXXXXXXXXX.png)

1. 每一个key-value pair是一组记录，每组记录包含key和value，
2. 由于相同区间的key前缀重复较多，采用前缀压缩方式，如果一个key-value和前一个key-value记录的key重合可以只保存后缀部分，每隔一定记录条目，会有一个restar特点，在restart点上，即使key的前缀和前一条记录有相同也会保存完整记录，因此在restart点上key前缀长度字段长度为0

**6，compaction的最小子问题**

简单来讲compaction动作拆分为最小单元就是上一小节中所讲的两个data  block之间的合并。初步设想的接口如下

```C
//合并block1  block2，输出到一个output_block
void single_block_merge(BLOCK* block_1, /*in*/
                 BLOCK* block_2, /*in*/
                 BLOCK* output_block /*out*/)
//合并一组block，并输出到指定空间位置
void multi_block_merge(BLOCK** input_block_array, /*in*/
                       size_t input_blokc_array_size,/*in*/
                       BLOCK** output_block_array, /*out 此处有足够内存保存merge的输出*/
                       size_t& output_block_array_size,/*out*/)
```

merge过程基本操作

1，两个block中记录的归并排序，输出一个完整有序数组，并也按原有格式输出合并之后的BLOCK

2，两个block中可能存在重复记录，则根据key字段的特定属性进行删除。

**7, 交互过程**

Rocksdb的compaction主控逻辑会将comaction拆解成6节中描述的一个个子任务并post给 FPGA执行，根据FPGA的处理速度可以采用同步或者异步响应的模式与主逻辑进行交互。
