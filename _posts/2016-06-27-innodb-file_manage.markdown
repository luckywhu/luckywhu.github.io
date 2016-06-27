---
layout: post
title:  "Innodb物理文件格式综述"
date:   2016-06-27 01:09:37 +0800
categories: mysql Innodb
---

### 关于该综述

关于Innodb物理文件格式的文献综述. 基本资料来源于 [Jeremy Cole](https://blog.jcole.us/) 的博客以及 mysql5.1.63 的代码, 文章中的画图全部来自Jeremy Cole如下的两篇博客文章：

1. [The basics of InnoDB space file layout](https://blog.jcole.us/2013/01/03/the-basics-of-innodb-space-file-layout/)
2. [The physical structure of InnoDB index pages](https://blog.jcole.us/2013/01/07/the-physical-structure-of-innodb-index-pages/)

我所做的工作只是阅读mysql源代码，并对每一幅图做注解，然后按照从宏观到微观的形式重新组织了一下材料。该篇综述覆盖了Innodb物理文件组织的大部分知识点，剩下的第六章是Innodb数据字典格式的内容，留待以后补充。

综述写于2013年12月份，一直放在everynote里面没发表，现在拿出来，用marke down格式重写了一下，

## 1.物理页面结构

  Innodb通过表空间来管理所有的物理文件，当使用共享表空间时，表空间只有一个，但可能包含多个物理文件，这些物理文件组成一个表空间，启用独立表空间时，除了共享表空间之外，每个表读都独享一个物理文件，但独立表空间只能包含一个物理文件。Innnodb使用一个4字节的整数对物理表空间进行编号，系统表空间作为第一个文件编号为0，其他表空间使用的编号依次递增。
  
  所有的表空间都会划分成UNIV_PAGE_SIZE 大小的页，这个大小是一个编译时确定的长度，默认是16K（2*8192）字节。页是innodb文件分配的基本单位。页可以组成区（extent），段等。
每个页初始化之后，元信息占据头部38字节和尾部8字节。 剩下的16384-46=16338字节存储数据内容。如下图所示

![innodb page manage](/img/innodb_file_manage_post/innodb_file_page_structure.jpg)

头部38字节和尾部8字节的内容定义如下：
![innodb file header/trailer](/img/innodb_file_manage_post/innodb_file_header_trailer.png)

该部分的代码定义在storage/innobase/include/Fil0fil.h文件中。其中各个字段的含义如下表所示

**物理页内容**

---

相对头部偏移  | 长度（字节）    |	内容	 | 解释
:----------- | :------------- | :----- | :---
0 |	4 | checksum | 该页的校验和
4 | 4 |	Page Number	| 页号，表空间中的所有页都会进行编号，是一个32位整数，总偏移0开始，依次递增。
8	|4|	Previous page|	在一个连成双向链表的页组织结构中，指向其前一页的地址。这些页物理上可能不是相邻的。靠指针链接起来
12|	4	|Next page|	同Previous page，指向后一页的地址。
16	|8	|Last LSN	|最后一次更新本页的事务日志编号，在Innodb的事务系统中有用
24	|2	|Page Type|	页类型，下表解释
26|	8|	Flush LSN	|该数据库最新flush的LSN，注意这是整个数据库的checkpoint点，只在系统表空间的第一个page中有效，在其他所有的page中是都填0
34	|4|	Space ID	|表空间编号	
16376	|4	|Old_style checksum|	老式checksum，作用同头部4字节。后续可能废弃。
16484	|4	|LSN的低4字节|	同Last LSN的低4字节

---

Innodb的表page类型总共有如下11种，如下表所示

**innodb的page类型**

---

类型	|数字编号|	意义
----|-------|------
FIL_PAGE_INDEX	|17855	|索引页
FIL_PAGE_UNDO_LOG	|2	|undo页
FILE_PAGE_INODE	|3	|
FIL_PAGE_IBUF_REEE_LIST|	4|	插入缓冲空闲页链表
FIL_PAGE_TYPE_ALLOCATED|	0|	新分配的页
FIL_PAGE_IBUF_BITMAP	|5	|插入缓冲位图
FIL_PAGE_TYPE_SYS|	6	|系统页（如数据字典等）
FIL_PAGE_TYPE_TRX_SYS	|7	|事务系统页
FIL_PAGE_TYPE_FSP_HDR	|8|	表空间头页，每个表空间中的第一个页
FIL_PAGE_TYPE_XDES	|9	|区描述页，表空间中，每16384个page需要一个区描述页，描述256MB大小的extent的使用情况
FIL_PAGE_TYPE_BLOB	|10	|未压缩的BLOB页

---

## 2.表空间组织

Innodb中文件分两种类型，日志文件和数据文件，数据文件由表空间组成，而表空间则由不同的页组成。启用innodb_file_per_table之后，表空间分为两种：系统表空间（编号0）和独立表空间。无论哪种表空间其大的架构都如下图所示。

![innodb file header/trailer](/img/innodb_file_manage_post/space_file_overview.png)

Innodb物理文件增长以extent为单位，每个extent为1MB（64个page）。对于每一个extent，innodb使用40字节的空间（XDES entry）来描述该extent中的页面使用情况。

**XDES entry**

![innodb file header/trailer](/img/innodb_file_manage_post/xdes_entry.png)

**xdes entry各个字段含义**

---

偏移|	长度|	内容|	释义
---|-------|-------|-------
N+0|	8	|File Segment ID|	该extent所属的段编号，文件段用来区分不同的内容，使用数字编号，例如一个表的索引叶子节点和非叶子节点分为属不同的段。
N+8	|12	|List node for XDES List|	一个双向链表结构，指明在该extent所属的段的extent entry链表中，前向节点页和后向节点的位置。
N+20	|4	|STAT	|该extent的状态，有四种类型FREE，FREE_FRAG， FULL_FRAG，FSEG。 前三种类型表明其属于该表空间中同类型的区域。而FSEG表明该page已经被分配，其File Segment ID指明了其所属的段编号
N+24|	16	|Page Stat Bitmap	|页位图，每2个bit描述一个页。第一个比特表示该页是否是空闲，第二个bit表示该页是否有更新没有被刷盘，目前第二个bit保留未用。

---

XDES entry被组织在类型为FIL_PAGE_TYPE_XDES的页中。每个类型为FIL_PAGE_TYPE_XDES的页存放256个Xdes entry，即能够描述256个extent区域（总共256MB）。这样每增加一个FIL_PAGE_TYPE_XDES页，就可以管理包括该页在内的连续256MB的空间。Innodb将256MB空间开始的第一个page作为extent描述页。

**XDES page结构**

![innodb file header/trailer](/img/innodb_file_manage_post/fsp_hdr_xdes_overview.png)

在每个FIL_PAGE_TYPE_XDES 类型的page中有5986字节的空间被浪费了，除了xdes entry ,页头和页尾，还有一个112字节的保留区域，叫做表空间头部，其作用是记录整个表空间的使用信息。这个FSP header在每一个FIL_PAGE_TYPE_XDES 类型的页当中都存在，但只有一个表空间的第一个page（0号page)中的内容是有效的。对于其他FIL_PAGE_TYPE_XDES  页，这112字节都被置为0.

FSP_HDER区域的112个字节内容布局如下

**fsp header格式**

![innodb file header/trailer](/img/innodb_file_manage_post/fsp_header.png)

---

Innodb文件组织中使用的ListNode
ListNode是一个双向链表，用于管理表空间，处在链表中的可能是page，也可能是inode，或者xde entry。链表base节点主要包含两个部分，分别指向双向链表的头结点和尾节点，此外还包含一个长度字段，其结构如下 
![innodb file header/trailer](/img/innodb_file_manage_post/list_base_node.png)

而list node节点格式如下
![innodb file header/trailer](/img/innodb_file_manage_post/list_node.png)

通过一个页号+页面内偏移，一个List Node可以将同一个表空间内的结构连接起来。

---

FSP HEADER中各字段的意义：

---

偏移	|长度|	内容|	意义
----|----|-----|------
38	|4|	SPACE ID	|表空间编号
	|4|	unused |
	|4	|Highest page number|	当前表空间最大的页号，表明了该表空间的大小（以页为单位）
	|4	|Highest page number initialized|	当前初始化完成的页数目。总是小于最大页号
    |4	|    FLAGS	|
	|4	|Number of pages used in FREE_FRAG list|碎片extent链表中，已经使用的page的数目，方便计算空闲空间的
	|	|List base node for FREE LIST|	空闲区链表，指向的 是XDES entry
	||	List base node  for FREE_FRAG list	|碎片区链表，该链表中的extent其中部分页被分配，部分空闲
    ||List base node for FULL_FRAG	|该区中所有的page，因为不同的目的被分配完毕。
	||	Next unused Segment ID	|该表空间中下一个未使用的段号。
	||	List base node for FULL_INODES list	|索引描述节点页链表，在该链表中的索引描述节点页，其所有描述节点都被使用,该部分后续描述..=====>该list指向的是inode page中的node节点
	||	List base node for FREE_INODES list|	同上，该链表中的页，只有部分inode描述节点被使用。

---

以上描述了系统表空间和独立表空间共有的一些空间布局信息，在Innodb的文件管理中，系统表空间（ibdata1）中由于存有一些事务，数据字典等相关的元信息，其表空间布局和独立表空间又有所不同。除了头部同样拥有的Extent 描述页，插入缓冲位图页，INODE页之外，系统表空间还有其他一些固定的页，其布局如下

**共享表空间**
![innodb file header/trailer](/img/innodb_file_manage_post/ibdata1_file_overview.png)

除了相同的头部3个page之外，系统表空间中，固定的age还有如下5个,其保存的内容分别如下。

---

页号	|类型|	内容	|保存的内容
----|----|----|----|
3	|FIL_PAGE_TYPE_SYS	|Insert Buffer Header	|插入缓冲统计信息
4	|FIL_PAGE_TYE_INDEX	|Insert Buffer root	|插入缓冲所用的索引的根节点
5	|FIL_PAGE_TYPE_TRX_SYS|	Transaction System Header|	保存事务信息相关的内容，如Transaction ID，用于两阶段提交的 MySQL binlog 信息，doublewrite 区的位置等
6	|FIL_PAGE_TYPE_SYS	|Fisrt Rollback Segment	|第一个回滚段页
7	|FIL_PAGE_TYPE_SYS|	Data Dictionary Header|	数据字典的头页
	|||		
64~127||		第一个double write 区	|
128~191	||	第二个double write 区	|

---

对于独立的表空间，其中只保存了一个表的内容，而表的各个索引将该表空间分为不同的段（segment）。独立表空间的布局如下：

**独立表空间格式**

![innodb file header/trailer](/img/innodb_file_manage_post/ibd_file_overview.png)

从page 4开始，依次存放不同索引的根节点，这些根节点的页号保存在系统表空间的数据字典当中。


## 3.索引和表空间段组织

   Innodb中的page都会归属于不同的段（Segment），不同的Segment存储同一类型的Page, 例如一个索引的信息会保存在两个Segment中，一个叶节点Segment，一个非页节点Segment。而表空间的物理区域，分别按照Page单位和Extent单位分配给不同的Segment。File  Segment的信息记录在FILE_PAGE_INODE类型的页中，表空间中第三个 page是该表空间的第一个FILE_PAGE_INODE，所有的FILE_PAGE_INODE页组织成一个双向链表。一个FILE_PAGE_INODE的布局如下。

**FILE_PAGE_INODE**
 
 ![innodb file header/trailer](/img/innodb_file_manage_post/inode_overview.png)

FILE_PAGE_INODE页结构比较简单，除去page header tailer,双向链表节点信息之外，剩下的就是总计85个Inode entry，每个表的每个Index会占用两个Segment inode（一个页节点inode，一个非页节点inode）。每个Inode Entry占据192字节。这192字节的内容布局如下：
 ![innodb file header/trailer](/img/innodb_file_manage_post/inode_entry.png)
 
 其中各部分的作用如下表所示
 
 **inode_entry结构**
 
 ---
 
偏移	|长度	|内容|意义
----|----|----|----
0	|8|	FSG ID	|该inode指向的File Segment的ID，这里的ID来自 FSP Header中的next unused segment ID
8	|4|	Number of Used pages in NOT_FULL list|	非空extent链表中，已经使用的page数目
12|	16	|List base node for FREE LIST	|空闲extent 链表描述结构，头结点和尾节点指向的是XDES page中的xdes entry，而不是实际的extent位置
28	|16|	List base node for NOT_FULL list|	部分使用的extent 链表描述
44|	16	|List base node for FULL list	|全部使用的extent链表描述结构
60	|6|	Magic Number	|魔术数字，以确认该inode entry是否被初始化
64	|	|Fragment Array  entry(0~31)|	碎片页指针数组，0~31总计32个页号。Innodb初始给一个Fil Segment分配空间时，每次一个页，而这些页的页号就记录在Fragment Array Entry数组中，当该File Segment使用的空间超过32个page时，Innodb将每次至少分配一个extent（64页）。因此每个File Segment的空间都是32个page+完整的extent链表组成。
 
 ---
 
 inode管理各个File Segment，File Segment分别属于不同的索引，在每个索引根节点page中，包含了一个FSEG Header，该部分包含了指针指向描述该索引文件组织的Inode entry ，FSEG Header的布局如下
 
 **fsg header formate**
 ![innodb file header/trailer](/img/innodb_file_manage_post/fsg_header.png)

其中各字段的内容如下表所示

---

偏移|	长度|	内容|	解释
----|----|----|----|
74	|4|	Leaf Page Inode Space ID |	Inode所在的表空间的ID，该字段总是其所在的表空间的值
78	|4|	Leaf Page Inode page number	|描述叶节点的inode的页号。
82	|2|	Leaf Pages Inode Offset |	Inode  entry在FIL_PAGE_TYPE_INODE页中的偏移
84|	4	|Non-leaf Inode Space ID	|
88|4	|Non-leaf Inode Page Number	|
92|2	|Non-leaf Inode Offset 	|

---

结合起来，Innodb管理一个表空间采用如下方式。从数据字典中可以得到一个索引的根节点的页号，通过该页号找到该root page，在root page中的 FSEG header字段，可以找到该索引的Inode信息指针，FSEG Header同时包含了指向叶节点Segment和非页节点Segment的 inode信息的指针。Inode entry中，可以得到该Segment分配得到的 pages和extent。这样Innodb将所有的索引和物理page组织起来。
**index file segment structure**
 ![innodb file header/trailer](/img/innodb_file_manage_post/index_file_segment_structure.png)

## 4.索引结构

 Innodb中所有的数据Page都被组织在索引当中，每一个表都会有一个主键索引，所有的数据内容都会保存在主键索引的叶子节点中，而对于非主键索引，其叶子节点中保存了主键索引的值。Index page是page的一种，index page中和数据page有着相似的结构，例如包含System Records（包含当前页中数据上界和下界），该page中的索引记录也通过双向链表连接起来，页的尾部也包含了用于加速查找的页字典。其页面布局如下图所示：
**index overview**
 ![innodb file header/trailer](/img/innodb_file_manage_post/index_overview.png)
 
除了标准的FIL Header 和FIL Trailer外，其各个部分的内容如下表所示：

**index overview**

---

偏移	|长度	|内容	|
----|----|----|----
38	|36	|Index Header|	索引页头，包含索引页和记录的管理信息，下面描述
74	|24|	FSEG Header|	在Index的root page中，FSEG包含了指向该索引的叶节点和非叶节点的inode的指针。在非root page中，该段内容填充为0
94	|26	|System Records	Innodb 页中的两条系统自带的记录行，固定在此位置，分别代表 infimum 和 supremum记录。在该页的记录链表中，infimu 和supremum分别充当了链表的头结点和尾节点。
120	|不定|	User  Records|	索引记录，在page中分布，以指针的形式按key从小到大连接起来。 被删除之后  会连接在空闲区块链表当中
	|不定|	Free Space	|空闲空间
	|不定|	Page Direcory|	单位长度为2的整形数组，每个数据元素指向一行在页面中的偏移地址。Page directory从FIL Trailer开始向前生长。每4~8行记录会在Page dicrecotry中有一条记录，形成一个稀疏索引。

---

Index Page中第一个重要的字段Index Header其布局如下：

**index header**
 ![innodb file header/trailer](/img/innodb_file_manage_post/index_header.png)
 
 各个字段的内容如下表所示
 
 ---
 
偏移	|长度|内容|意义
----|----|----|----
38	|2	|Number of Directory Slots	|页稀疏表的长度
40	|2	|Heap Top Position|	当前页当中free空间的开始地址
42	|2	|Number of Heap Records/Formate flag|	当前页中所有记录的条数，包括 Infimum和Supemum记录以及被删除记录.该字段的头位bit，是一个标志位，表明当前的页格式是COMPAC还是REDUNDANT
44 |2	|Fisrt Garbage record Offset|	第一个被删除空间地址偏移，Index page中，被删除的记录空间也以双向链表的形式连接起来
46	|2	|Garbage Space	|被删除空间占据的大小
48	|2	|Laster Insert Postion	|上一次插入的该page中的记录的偏移
50	|2	|Page Diretion	|该页的插入方向，当一个记录被插入是会和Laster Insert postion位置的记录大小作比较，判断其实LEFT RIGHT还是NO_DIRECTION的插入，并记录在该字段中
52	|2|	Numbers of Insert in Page Direction	|在Page的插入方向上执行的插入次数，注意 每次方向改变时，该值会被清零。
54	|2	|Number of Records|	有效记录的条数（不包括infimum,supremum,deleted records)
56	|8	|Maximum Transaction ID|	修改该page的最大事务ID
64	|2	|Page Level	|页所在的层级，叶子节点层级为0，向上一次递增一个层级
66	|2	|Index ID	|该Index page所属的索引的ID
 
 ---

FSG Header部分包含20字节，在前面（3）部分中已经有解释。
紧跟着FSG header部分的是System Records部分，包含26个字节。这个部分包含两条系统记录，分别叫做infimum和supremum。分别固定在系统的99字节和112字节的偏移处。
System Records的布局如下所示：

**index system records**
 ![innodb file header/trailer](/img/innodb_file_manage_post/index_system_records.png)

部分几点的解释如下：
Number Recorders Owned：如果该行记录在page的 page direction 表中，该字段表示包括该记录在内，到表中记录的下一样记录为止，总共有多少条记录。该字段的意义是告诉搜索逻辑
在通过页表找到该行之后，其后面还有多少行可以通过next record offset指针找到。


给出了索引页的物理结构之后，一个Index的 leaf  page的逻辑结构如下图所示：

**btree simplified leaf page**
 ![innodb file header/trailer](/img/innodb_file_manage_post/btree_simplified_leaf_page.png)

注意在leaf Page的 value中保存的是非索引相关的字段信息。对于primary key，该部分包含的是整行数据，而对于secondary key，该部分的value则是主键内容。
对于一个非叶子节点 ，value部分保存的则是，该行记录对应的叶子节点的page 号，而该行记录的key是其叶子节点中记录的最小的key的内容。如下图所示：

**btree simplified noleaf page**
 ![innodb file header/trailer](/img/innodb_file_manage_post/btree_simplified_noleaf_page.png)

B+树中所有相同层级（Index header中的page level字段）的节点，通过双向链表连接起来，方便同一级之间的遍历。如下图所示。

**btree simplified level**
 ![innodb file header/trailer](/img/innodb_file_manage_post/btree_simplified_level.png)

对于一个以下列sql创建起来的表：

{% highlight sql %}
CREATE TABLE t (  i INT NOT NULL,  s CHAR(10) NOT NULL,  PRIMARY KEY(i)) ENGINE=InnoDB;
INSERT INTO t (i, s) VALUES (0, "A"), (1, "B"), (2, "C");
{% endhighlight %}

其页面结构如下图所示：

**btree detailed_apge_structure**
 ![innodb file header/trailer](/img/innodb_file_manage_post/btree_detailed_page_structure.png)


## 5.记录行结构（barracuda compact format）

Innodb的行主要由列值（Filed）和行记录头部组成。列值按照create table时定义顺序紧凑排列，中间没有分隔，在行记录头部，保存了一些信息，以表明该行相关信息，如各个字段的实际长度，是否被删除，是否是最小行，列数，下一行指针等。需要注意的是在Innodb的格式中，next Record offset指针指向的下一行位置，代表的是该行实际的列值开始的地方，并不包含行头部。如果要得到该行的一些元信息，需要从指针位置向前回溯。由于列值长度的不同，一个行的头部长度也是变长的。一个行的头部布局如下所示，包含6个字节定长部分和变长部分。

**record formate header**
 ![innodb file header/trailer](/img/innodb_file_manage_post/record_format_header.png)
 
 
 各个字段的内容如下所示
 
 ---
 
 相对于列值原点的偏移|	长度|	内容	|解释
 ----|----|----|----
N~|	取决于可变长度列的数目|	包含每一个可变长度列的长度|	一个整形数组，单位长度为一字节或者两字节，取决于最大可变长度的长度，如果没有可变列，该字段为空。
N-5	|不定长，正bytes长度|可能为NULL的列的位图，标志该行那些字段为NULL|	表定义时，可能为NULL的列的数目决定了段占用的bit数，取整到8，如包含7个可能为NULL的列时，则该字段为一个bytes。如果一个Filed为NULL，它的值将不会在key或者 行额rowdata中出现。
	|4bits	|Info FLags	|头两个bit未用，第三那个bit标志该行是否deleted，第四个bit标志该记录是否是飞叶子节点一层中的最小记录
	|4bit	|Number of Records Owned	|该行拥有的记录数目，用于page directory的查询，值未4~8。对于Infimum记录，最小为1
	|13bits	|Order	|插入顺序编号，Infimum记录为0，Supremum为1 ，用户记录从2开始编号。
	|3bits	|Record Type	|记录类型，目前只有四种 conventinal(0), node pointer(1) infimum(2) supremum (3）
	|2bytes	|Next redcords	|一个pge中所有有效的行记录按照key递增循序连成一个单项链表，该字段指向下一个记录的 远点位置。

 ---
 
 
Innodb中的行，除了记录自己的内容外，还会记录一些额外的事务相关的字段信息。例如事务ID，回滚段指针等。对一个聚簇索引中的叶子节点。其页面布局如下：

**cluster key leaf pages**
 ![innodb file header/trailer](/img/innodb_file_manage_post/clustered_key_leaf_pages.png)
 
 从记录原点开始， 最开始的三个字段分别是聚簇索引值，事务ID，回滚段指针。接下来才是实际的列内容。在Innodb中，primary key可能是一个多列索引，由于其组合不可能是一个NULL的值，Innodb简单的将他们按照内部存储的原始自己拼接在一起，形成一个大的字节串。回滚段指针指向的是回滚段中记录修改该记录的事务的回滚信息。该7字节分成四部分  （1）1bit代表是否是插入（2）7bit 回滚段的segment ID，(3)4字节的回滚段页号 （4） 2字节的页内偏移。

聚簇索引的非叶子节点中除了key外，没有列值。其行记录布局如下：

**cluster key nonleaf pages**

 ![innodb file header/trailer](/img/innodb_file_manage_post/clustered_key_nonleaf_pages.png)
 
 对于非聚簇索引，其记录中保存的是聚簇索引的值，在Innodb中，如果一个Secondary key和一个primary key有重合的列，则在Secondary key的记录中，有重合的列只会记录一次，记在Secondary key Fileds中，在Cluster key Filed中则不会记录。一个Secondary key的叶子 page布局如下：
 
**secondary key leaf pages**
 
 ![innodb file header/trailer](/img/innodb_file_manage_post/secondary_key_leaf_pages.png)

非叶子节点的布局如下，需要注意的是在Secondary key的非叶子节点中，聚簇索引包含在记录中，作为key的一部分，和Secondary key组合在一起，保证唯一性。这样在Secondary key中，非叶子节点比叶子节点多了四个字节的page number。

**secondary key nonleaf pages**
 ![innodb file header/trailer](/img/innodb_file_manage_post/secondary_key_nonleaf_pages.png)
 
## 6. 数据字典Data Dictinary

该部分未完待续。




