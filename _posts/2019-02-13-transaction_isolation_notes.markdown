---
layout: post
title:  "事务隔离级别综述"
date:   2019-02-13 01:09:37 +0800
categories: transaction database
---

### 事务隔离级别梳理

*数据库的不同事务隔离级别并没有一个比较明确的分类方法, 由于历史原因, 很多事务隔离级别是先有了工程实现(Oracle,DB2,MySQL)及其对应的事务隔离表现. 然后才有学术界上的一些梳理和规范化, 有些商业数据库的特定事务隔离级别的行为表现是无法纳入一个学术理论分类中的任何一个类别的, 所以很多商业数据库的事务隔离级别表现和理论上的分类不一致的.这是非常需要注意的地方* 

*另一个需要注意的点是,一般的数据库并发控制使用加锁的实现, 可以使用对读写行为不同加锁(锁类型, 加锁目标)方式来区分不同的隔离级别, 然而locking base的隔离级别实现和学术上基于phenomenological definitions的隔离级别有些不一致的地方. 在杜绝诸如P0,P1,P2,P3等三个现象时,二者是一致的,但是针对P4(lost update), A5(Data Item Constraint Violation)两个异常. 基于加锁的实现和基于异常(phenomenon)的实现无法维持一致* 

注:下文中的很多术语如P0,P1,P2,P3,P4, A5A,A5B等,含义同参考文献[1].

### 1.无隔离

**定义**: 不做任何控制, 任何事情都可能发生, 例如事务的修改被其他事务覆盖(P0:Dirty Write).

基于加锁的实现中,读操作不加锁, 写操作加短周期锁(只在写操做发生时,而不是事务存续期间). 其行为即为此隔离级别.

### 2.读未提交(read uncommitted)

**定义**: 可以还未提交的事务对数据所做的修改.

在加锁的实现中, **读不加锁, 写操作加长锁**(持续到事务结束), 可以实现读未提交隔离级别. 因为对写操作加长锁,所以可以避免P0(Dirty Write). 但是由于读不加锁, 可能读到未提交事务的修改(P1: Drity Read)

### 3.读已提交(read committed)

**定义**: 读到的数据都是事务已经提交的数据.

在加锁的实现中, **读加短锁, 写操作加长锁**, 可以实现读已提交隔离级别. 但是由于未加predicate lock. 可能发生读多次返回的数据不一致的情况(P2:fuzzy read),即模糊读取.

### 4.可重复读(repeatable read)

**定义**: ?

在基于锁的实现中, **读加长锁, 写加长锁, 同时加短predicate lock**. 可以实现可重复读. 

可重复读可以阻止P4(lost update)现象的发生.因为读加长锁, 但是不能阻止幻读(P3:phantom read). 因为不满足predicate 条件的记录可能被更新(或者新写入符合predicate条件的记录),导致符合条件.

*需要注意的是,MySQL的InnoDB引擎,在RR隔离级别下,不会发生幻读现象. 因为InnoDB的是基于MVCC+2PC来实现并发控制自己知的, 而其RR隔离级别是基于snapshot来实现的.所以不会发生幻读现象.*

基于加锁的可重复读实现是能够阻止RadSkew(A5A)和WriteSkew(A5B)的. 因为读写都持有长锁. 但是基于ANSI SQL定义的可重复读隔离级别(基于Anomaly定义,杜绝了A2:R<sub>1</sub>[X]…W<sub>2</sub>[X]...c2...R<sub>1</sub>[X]…c1现象的发生,可以达到可重复读隔离级别), 却不能阻止ReadSkew(i.e:R<sub>1</sub>[X]…W<sub>2</sub>[x]...W<sub>2</sub>[Y]...c2…R<sub>1</sub>[Y]...(c1 or a1) ) 和WriteSkew(i.e:R<sub>1</sub>[X]…R<sub>2</sub>[Y]...W<sub>1</sub>[Y]...W<sub>2</sub>[X]...(c1 and c2) )的发生.  所以对于可重复读隔离级别是否会发生WriteSkew. 这个取决于在何种语境来讲. **在ANSI SQL隔离级别下,Repeatable Read可能发生write skew.  而locking based的Isolation却不会出现write skew.**

*为了有更普适的意义,我认为应该使用基于phenomena定义的事务隔离级别*

### 5.串行化(serializability)

**定义:**多个事务的执行结果和它们的某个串行化执行(无并发,事务的执行时间区间没有重叠)的结果是一致的

串行化隔离级别的描述里面强调的是**结果**, 而不是强调事务之间的先后顺序,  一组并发执行的事务,他们的可串行化调度方法有很多种. 只要他们的结果是一致的就可以. 就都满足可串行化的要求.

在加锁的实现中, **读加长锁, 写加长锁, 同时对predicates条件覆盖的范围加加长锁**.

### 6.严格串行化(strict serializability)

**定义**:满足串行化要求执行的事务,同时保证事务的执行顺序(提交时间戳)和事务实际运行中的前后关系保持一致.

理解严格串行化隔离级别关键的两点, 1. 首先要满足串行化的要求  2. 在满足串行化隔离级别的前提下, 事务的提交时间戳(在代码实现中一般用Commit_ID, LSN等表示)和实际物理时间戳先后关系一致. 打个比方,两个事务A和B,  如果实际提交时间T<sub>A</sub> < T<sub>B</sub>. 则一般有commit_id<sub>A</sub> < commit_id<sub>B</sub> 或者  LSN<sub>A</sub>< LSN<sub>B</sub>
需要注意的一点是对这句话: "**事务的执行顺序(提交时间戳)和事务实际运行中的前后关系保持一致**"的理解. 事务实际运行中**有前后关系**特指: 一个事务的结束时间早于另一个事务的开始时间.  即两个事务的执行时间区间在物理上没有重叠. 否则认为两个事务是并发的. 没有前后关系.   严格串行化只要求有先后关系的事务之间其提交时间戳和先后关系一致. 如果两个事务没有先后关系(即并发事务), 则对其先后顺序不做要求.这一点区别于接下来需要讲到的外部一致性. 

```
(1)如下的场景, 事务A和B之间是并发的, 在严格串行化要求下, 他们的提交顺序是不确定的, A,B谁先提交都可以


           A_start          A_end   |
A-------------[----------------]----+------------------------------->
                   B_start          |  B_end
B--------------------[--------------+----]-------------------------->
                                    |


(2)如下的场景, 事务A在事务B之前提交, 则在严格串行化的要求下, A的提交时间戳必须小于B的提交时间戳.
    
                                   | 
            A_start          A_end |
A-------------[----------------]---+-------------------------------->
                                   | B_start            B_end
B----------------------------------+---[------------------]--------->
                                   |
                                   
                                   
                                  图-1
```

​                              

### 7.外部一致性(external consistency)

**定义**: 多个事务的执行结果和他们的某个串行化执行结果一致, 并且这些事务的提交时间戳和他们实际完成的时间顺序一致.

外部一致性比串行化和严格串行化更进一步,  关注它和严格串行化的区别, 在严格串行化要求下, 事务的提交时间戳和事务实际运行的前后关系保持一致,  如果事务的执行时间之间有重叠, 且它们之间没有依赖(如读写相同行的数据) , 则这些并发事务的提交时间戳是不确定的.   而在满足外部一致性的要求下, 事务的提交时间戳顺序和完成时间(物理上的提交时间)一致.  这样即使并发事务的执行时间之间有重叠, 但他们的提交时间有先后, 则提交时间戳和实际提交时间需要保持一致. 如下图所示,  虽然事务A和事务B的执行时间范围之间有重叠, 他们没有先后关系, 在严格串行化隔离级别下, 事务A和事务B的提交时间戳是不确定的,  谁在前面都可以

但是在满足外部一致性的要求下,  因为T<sub>A_end</sub> < T <sub>B_end</sub>  则必须有A的提交时间戳小于B的提交时间戳.

```
                                   T1
           A_start          A_end   |
A-------------[----------------]----+------------------------------->
                   B_start          |  B_end
B--------------------[--------------+----]-------------------------->
                                    |
                                   图-2
```

如果我们以一个外部观察者的视角来看待A和B两个事务的执行, 如果A和B之间没有依赖关系(读写同一行数据), 在严格串行化的隔离级别下, 因为事务A和B没有先后关系,在系统内,事务B有可能先提交,事务A后提交(比如B获得提交时间戳100, 事务A获得提交时间戳200). 则在事务B提交之后事务A提交之前的某个时刻, 这个观察者获取一个snapshot(比如基于时间戳150),  这个snapshot会包含事务B的修改, 但是不包含事务A的修改. 这和观察者看到的客观事实是违背的. 而在外部一致性隔离级别的要求下,  事务A和事务B的提交时间戳必须和他们实际完成的时间一致(也就是和外部观察者观察到的时间先后顺序一致).  必然要求事务A的提交时间戳小于事务B的提交时间戳. 这样就能保证,外部观察者看到的事实(A先于B提交),和数据库中实际数据的一致(如果能看到B的修改,则一定能看到A的修改).

*一个疑问:对比严格串行化和外部一致性的定义, 会感觉严格串行化的定义没有实际用处. 它不能指导出一个明确的决策事务先后顺序的调度方法. 而只是描述一种现象*



### 8.快照隔离级别(snapshot)

**定义**: snapshot隔离级别下, 事务的读写操作都是作用在事务开始那一刻产生的一个快照上. 

对比其他几种隔离级别, 快照读更像是一种实现(implementation),而不是一种定义.  snapshot是一种多版本并发控制方法.在此实现下,每个事务T<sub>1</sub>都在自己的快照上进行读写操作, 在最后提交时,事务会获取一个提交时间戳Commit-Timestamp,这个时间戳必须要要比系统当前分配的任何Start-Timestamp或者Commit-Timestamp要大.事务能成功提交的前提是:没有任何其他事务T<sub>2</sub>的提交时间戳Commit-Timestamp处于T<sub>1</sub>的执行时间范围[Start-timestamp, Commit-Timestamp]中间.并且T<sub>2</sub>修改了T<sub>1</sub>所修改的记录.  否则的话,T<sub>1</sub>将会回滚.  如果T<sub>1</sub>提交成功, 则任何Start-Timestamp时间大于T1的Commit-Timestamp时间的事务,都可以看到事务T<sub>1</sub>的提交.

从SI的隔离级别描述可以看到, 它可以防P4(lost update)现象的发生.  同时可以防止A5A(Read Skew), 但是不能防止A5B(WriteSkew). 

snapshot隔离级别比ReadCommitted要严格,但是其与RepeatableRead隔离级别之间的关系并不明确.这个取决于你是否认为可重复读隔离级别下会否发生WriteSkew. 

### 附录

#### A.事务隔离级别对照表

1. 基于加锁实现的事务隔离级别对照表

| **Isolation Level**                         | **Read Locks on** **Data Items and Predicates** **(the same unless noted)** | **Write Locks on** **Data Items and Predicates** **(always the same)** |
| ------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Degree 0**                                | None required                                                | Well-formed Writes                                           |
| **Degree 1 = Locking** **READ UNCOMMITTED** | None required                                                | Well-formed Writes Long duration Write Locks                 |
| **Degree 2 = Locking** **READ COMMITTED**   | Well-formed Reads Short duration Read locks(both)            | Well-formed Writes Long duration Write locks                 |
| **Cursor Stability**                        | Well-formed Reads Read locks held on current of cursor Short duration Read locks | Well-formed Writes Long duration Write locks                 |
| **Locking** **REPEATABLE READ**             | Well-formed Reads Long duration data-item Read locks Short duration Read locks | Well-formed Writes Long duration Write locks                 |
| **Degree 3 = Locking** **SERIALIZABLE**     | Well-formed Reads Long duration Read locks(both)             | Well-formed Writes Long duration Write locks                 |

2. 事务隔离级别及表现

| **Isolation** **Level** | **P0** **Dirty Write** | **P1** **Dirty**  **Read** | **P4C** **Cursor Lost Update** | **P4** **Lost Update** | **P2** **Fuzzy Read** | **P3** **Phantom** | **A5A** **Read Skew** | **A5B** **Write Skew** |
| :---------------------- | ---------------------- | -------------------------- | ------------------------------ | ---------------------- | --------------------- | ------------------ | --------------------- | ---------------------- |
| **Read UNCOMMITTED**    | N                      | Y                          | Y                              | Y                      | Y                     | Y                  | Y                     | Y                      |
| **READ COMMITTED**      | N                      | N                          | Y                              | Y                      | Y                     | Y                  | Y                     | Y                      |
| **Cursor Stability**    | N                      | N                          | N                              | N                      | Sometimes Y           | Y                  | Y                     | Sometimes Y            |
| **REPEATABLE READ**     | N                      | N                          | N                              | N                      | N                     | Y                  | N                     | N                      |
| **Snapshot**            | N                      | N                          | N                              | N                      | N                     | N                  | N                     | Y                      |
| **SERIALIZABLE**        | N                      | N                          | N                              | N                      | N                     | N                  | N                     | N                      |


#### B.隔离级别中Phenomenon的分类描述

- P0: W<sub>1</sub>[X]...W<sub>2</sub>[X]...(c<sub>1</sub> or a<sub>1</sub>)                                                 (Drity Write)
- P1: W<sub>1</sub>[X]...R<sub>2</sub>[X]...(c<sub>1</sub> or a<sub>1</sub>)                                                  (Dirty Read)
- P2: R<sub>1</sub>[X]...W<sub>2</sub>[X]...(c<sub>1</sub> or a<sub>1</sub>)                                                   (Fuzzy Read or Non-Repeatable Read)
- P3: R<sub>1</sub>[P]...W<sub>2</sub>[Y in P]...(c<sub>1</sub> or a<sub>1</sub>)                                           (Phantom)
- P4: R<sub>1</sub>[X]…W<sub>2</sub>[X]...W<sub>1</sub>[X]...c1                                                 (Lost Update)
- A5A: R<sub>1</sub>[X]…W<sub>2</sub>[x]...W<sub>2</sub>[Y]...c2…R<sub>1</sub>[Y]...(c1 or a1)                (Read Skew)
- A5B: R<sub>1</sub>[X]…R<sub>2</sub>[Y]...W<sub>1</sub>[Y]...W<sub>2</sub>[X]...(c1 and c2)                    (Write Skew)

需要注意其与ANSI SQL中Anomaly的不同, Phenomena对比Anomaly是一种更普适的表述.

### 参考文献

- [1] A Critique of ANSI SQL Isolation Levels
- [2] Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed Transactions
- [3] Generalized Isolation Level Definitions



