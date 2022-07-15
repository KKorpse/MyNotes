[内存数据库相关论文清单](https://mp.weixin.qq.com/s/4J-DRJohpUgQ-SZ8XqXbww)

• Locking/latching 

• Cache-line misses 

• Pointer chasing

 • Predicate evaluation

 • Data movement and copying 

• Networking (between application and DBMS)

# 0. 基础概念

- [x] `OLTP，OLAP`
  * [OLTP与OLAP的关系](https://www.zhihu.com/question/24110442/answer/851671343)
- [ ] `Undo-Only，Redo-Only，Redo-Undo`（Redo-Only为什么屌）
  * [Aries算法](https://zhuanlan.zhihu.com/p/420777227)（为什么在现在内存数据库用的不多）
- [ ] `Pessimistic or Optimistic`？
  * optimistic approaches perform very well when conflict rates are low, but performance degrades (due to rollback and restart) for higher conflict rates.
  * `2PL`：传统磁盘锁，Pessimistic，要求每个事务持有并维持锁
- [ ] `Fuzzy checkpoints`
  * [Full or sync checkpoints and Fuzzy checkpoints](https://blog.csdn.net/cuanlvchi0974/article/details/100457565?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_aa&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_aa&utm_relevant_index=2)
- [ ] `多版本（Multi-Version）控制`
- [ ] `BW-Tree`
- [ ] `CSB+-Tree`
- [ ] 列索引怎么引的，为啥只能有一个列索引
- [ ] 事务保存点
- [ ] 谓词锁是啥

# 1. 技术突破点

## Data organization and layout

* **索引内直接存储内存指针，避免缓存池的使用**<!--因为主存数据库不需要像传统数据库那样为磁盘分页并为数据分配逻辑页号等，直接使用指针会更快-->
  * *避免了基于页面的间接寻址* （Avoiding page-based indirection）：缓冲池类似于分页<!--跟页表有啥区别？-->每次寻址会有两步操作导致开销；二级索引、页面不在内存内命中也要浪费时间。
  * *避免了页面锁*（Avoiding page latching）：新建一个操作，甚至持有共享锁，都可能会产生锁存变量，这些变量的操作是CPU的原子操作，所以会在一定程度上产生开销。<!--传统数据库有哪些锁-->
* **不同组织方法**
  * `Partition（分区）`：每个数据库分区单独跑一个线程，每个分区事务串行，避免了繁琐的并发处理。索引等内部结构也不需要多线程的支持。（系统处理复杂，线程处理简单）
  * `None-Partition`：每个线程可访问全局数据库，不会产生某个分区过于拥挤的情况（拥挤也有特定的处理方法），也不用处理跨分区的事务（线程处理复杂，系统处理简单）
  * `Multi-Version`：主存数据库允许了并发控制，比如读写互不阻塞，减少了上下文开销。另一个用途是允许对过去的快照读取。
  * `Row/Columnar layout`：row布局适合OLTP，columnar适合分析数据

## Index

* 内存带来的好处：直接在内存时，B+树node节点可以不用固定大小，也不用多个装在一个页中。

* **优化CPU缓存效率**【论】

  * `CSB+-Tree`，合并节点<!--尽量减少搜索指针？-->
  * `Cache line prefetching（预取）`，提速1.5x左右

* **提高并行性**

  * `Partition（分区）`：问题和优势跟上面数据组织同理
  * `Shared（共享）`：管理并发很麻烦（可以用到下方的无锁存）
  * `latch-free 索引结构`（将索引操作封装在基于CPU的原子操作内）

  ```相关研究：无锁存同步好像优势不明显，可以带来高性能，但需要谨慎的内存回收协议。现代cpu上的可伸缩性最重要的因素之一是避免全局内存位置上的争用，无论同步方法是基于锁存还是无锁存。```<!--？-->

## concurrency control（并发）

* `尽量避免实现单独的 lock manager`：避免每次访问都要经过锁管理导致的间接性开销。
* `Multi-Version and Optimistic`：用多版本控制来满足不加锁产生的后果？
  * 优点：高并发性，避免了上下文切换导致的开销（悲观锁）
  * 缺点：高征用导致高中止率，事务回滚，老版本删除等开销。
  * HANA的尝试解决方案：`row-level write locks.`

* `Partition`：上面提到过，根据核心或者机器进行分区，每个分区串行执行。
  * 优点：单个分区下的事务执行会非常快
  * 缺点：跨分区的事务还是会要求锁

* `Apriori knowledge of write/read sets.`：先验读写集？
  * 示例：BOHM决定了事务在执行之前的序列化顺序，然后在内存中为事务将要写入的新版本创建一个记录占位符。这种方法避免了使用集中的时间戳生成器以及乐观多版本方法中常见的验证阶段。<!--这里看不懂-->


## durability and recovery techniques

* Aries恢复算法（disk用它）
  * `physical logging`：记录了被事务修改的数据库元素(例如元组、块或索引的内部数据结构)的前后图像
  * `logical logging`：只记录操作是什么（但是恢复时需要重新执行一遍所有操作）
  * 一般Redo - physical，Undo - logical。
  * 添加检查点，减少恢复开销
* 尽量减少log大小 / 数量
  *  `Redo-only log`：writing the newest record version for only transactions guaranteed to commit <!--?-->
  *  `command logging`：简化版本的`logical log`
  *  不记录索引数据（恢复后重新构建索引就行）
  *  `Parallelize log I/O`：并行化日志流量（如：`Silo recovery prototype`）
* `Checkpointing` <!--为什么主存的检查点比磁盘的检查点大？”因为主存系统通常将所有(或大部分)行写入存储器“这段话看不懂-->
  * Hekaton：将新的记录版本写入版本化的检查点文件。日志上的删除被写入现有检查点文件之上的增量。
  * CALC：非阻塞异步方法，进入一种临时的复制-写模式，只写入自上一个检查点以来发生的记录更新。<!--?-->


## query processing and compilation

<!--感觉是涉及到面向对象编程的一些方法的优缺点？-->

* `iterator model` :disk数据库常用模型，使用get-next等一些基本函数构成各种操作。
  * iterator model 在内存数据库中会导致大量的虚函数调用或者函数指针调用，会产生开销。以及很差的代码局部性
  * `compilation`：内存数据库常常将query处理编译成机器码或者C语言。

* 为了避免虚函数调用的开销、degradation of branch prediction、 byte interpretation，一些系统放弃了“get  next”迭代器处理模型。<!--？-->
* 查询被编译成高度优化的机器码，并直接在内存记录上运行。

# 2. 内存数据库案例

## Heka-ton（OLTP engine）

* **特点**：高并发，不分区，无锁，乐观-多版本，编译关键代码

* **Data Orgnization**
  * **`Hekaton table`**
    * `hash index` 
    * `range index`：范围索引， `Bw-tree`
    * `column store index`：<!--暂时不知道用来干啥-->
  * 更新永远是新建行（多版本），事务完成任务后设置对应行时间戳（红色部分，用begin和end判断行的新鲜性），无事务关联的老版本交给垃圾处理。
  * Multi-Version：读者写着不会互相阻塞，但写者可能会互相阻塞<!--怎么个处理法他没说-->。

![image-20220416143603636](C:\Users\54351\AppData\Roaming\Typora\typora-user-images\image-20220416143603636.png)

* **Index**
  * **`Copy-On-Write`**：通过复制来写而不是原地修改，<u>通过这个策略减少Cache失效次数</u><!--是因为当前row的地址不一定在cache中么，新建的话可以使用连续的地址所以Cache命中率高？-->
  *  **`compare-and-swap (CAS)`**：Bw-Tree中的节点存的是PID，PID通过mapping到具体内存指针：更新的时候可以更改物理指针地址，且更改时不需要将位置信息在树上更新（因为pid没变），改pid-pointer就行。<!--为啥这样一来就可以实现无锁并行？-->
* **Concurrency Control**
  * `Transaction States`：`Active`, `Preparing`, `Committed`, `Aborted`
  * `Validation`：不防止冲突，而是在update事务读操作之前进行验证。（准备阶段的验证范围取决于事务的隔离级别）
    * 只读事务和在快照隔离下运行的更新事务不需要验证；
    * 事务尝试更新并导致事务回滚时，检测到写-写冲突。
    * 在可重复读取或可序列化隔离下运行的事务需要在提交之前进行验证
      * 读是否稳定（有没有事务中途修改）
      * 避免返回`PhantomData`
* **Query Processing**
  * 只访问内存表的存储过程被编译成定制的、高效的机器码
* **Durability and Recovery**
  * 只有用户数据做log，索引等不做。被中止的事务也不做log
  * `Checkpointing`：
    * 三个特质：Continuous checkpointing，Sequential I/O，Parallel recovery
    * `data file`：只增不减，但当过时的部分太多时，可以合并并删除过时的部分。
    * `delta file`：用于恢复时过滤data，只增不减

## H-Store/VoltDB 

* **特点**：
  * 分区，串行
  * 存储过程执行（不是边运行边提交sql，而是通过java提前编写好事务要使用的sql请求）
  * 分布式部署（表被拆分或者复制到不同的分区）
  * 紧凑的日志（日志只记录事务具体是啥）
  * 两部分结构（java的事务管理部分 + C++的事务执行引擎）
* **Data Organization**
  * 执行引擎只能看见和管理本分区，每个分区分为固定长度部分和可变长度部分。
  * **`look-up table`**：4字节的头部能计算出元组的分区，位置。
  * 对于每个表（不是look-up），维护一个四字节偏移表，记录和分配空余的元组位置。（类似于空闲内存表）
* **Database Design**（好像是需要根据公司需求独立设计）
  * Table Partitioning：根据hash或者值进行水平拆分到不同的分区
  * Table Replication：将表复制到不同的分区（对只读或者读很少的表项最有用）
  * Secondary Index Replication：维护一个横跨所有分区的二级索引，当某个操作不是单个分区时，不用广播到每个分区，而是可以分配到对应分区。（但更新操作频繁时效果不明显）
* **Concurrency Control**
  * `One-at-a-time`：每个分区内串行执行，每个事务根据请求时间和顺序分配一个`id`，处理器根据id选择事务运行
  * 在大部分事务都在一个分区下执行时提升最大（那Database Design就很关键了），涉及多个分区的事务需要加锁 。
* **Query Processing**
  * java的事务管理部分 ，C++的事务执行引擎，每个分区单线程管理。
  * 没看懂：<!--在java级别的组件中，执行引擎的线程阻塞在队列上，等待消息执行代表事务的工作。这项工作可以指示引擎调用过程的控制代码来启动一个新事务，或者代表在另一个分区上运行的事务执行一个查询计划片段。注意，对于后者，H-Store的事务协调框架确保不允许任何事务在执行引擎上对查询请求排队，除非事务持有该引擎分区的锁。-->
  * C++引擎只运行在自己的分区，Java引擎通过`JNI`调用C++库的方法并传入参数。
  * Stored Procedures（存储过程）
    * 调用`run()`方法时执行事务
    * 即使使用相同的输入参数，对同一查询的多次调用也会被单独处理。<!--为啥这样-->
    * 存储过程控制代码中禁止的非确定性操作类（防止两次相同的代码产出不同的结果）

* ![image-20220417180205005](C:\Users\54351\AppData\Roaming\Typora\typora-user-images\image-20220417180205005.png)
  * Stored Procedure Routing
    * `routing attribute`：根据这个，每个事务能被分配到最合适的分区（这个事务涉及到的数据大部分都在这个分区）
    * <!--如果需要其他分区的数据，好像是让其他分区的执行器帮忙跑对应的sql语句-->
* **Durability and Recovery**
  * **`command logging`**：简化版本的`logical log`，每个日志记录包含存储过程的名称和从应用程序发送的输入参数（不支持事务保存点）
  * 用一个单独的线程记录log，在事务执行完但还未返回时录入log：避免了abort和回滚（事务运行时需要访问其他分区时会回滚并上锁）的事务。
  * 打包log：将多个事务的命令日志项组合在一起，并将它们批量写入，以分摊写入磁盘的成本。
  * **`Snapshots`：**定期或者主动创建只包含元组的快照，对于快照保存一个检查点，每次恢复从检查点开始。
    1. 产生一个特殊的事务，这个事务告诉所有分区进入`copy-on-write（写时复制）`状态，所有在这个检查点之后的事务对数据库的更改都不会覆盖原有的信息
    2. 此时后台进行快照（复制到磁盘）
    3. 结束后，关闭写时复制状态，清理这个数据结构。
  * **`Crash Recovery`**：
    * 扫描日志，以事务当时运行的顺序执行一次（disk数据库则可以对事务顺序重新排列）<!--为啥可以重新排列啊-->

##  Hy-PeR

* **特点**：**`OLTP+OLAP`**混合的同时满足**`ACID`**（得益于现在的多核和大容量内存服务器）
* **Data Organization**（实现混合模式）
  * **`snapshots（快照）`**：创建虚拟内存快照
  * **`copy-on-wright`**，**`copy-on-update`**：
    * 当数据没有被修改时，新老进程共用同一套指针
    * 当数据更新时，为OLAP进程创建一个快照副本，而修改工作在原地进行（在原地修改是为了数据的连续性）
  * **storage structures**：数据驻留在主存，不使用虚拟内存避免OS参与
  * **row，column存储同时支持**，可以将常用的属性装入一条向量
  * `HyPer engine`查询引擎
    * 部署了一个 成熟的高级查询优化器
    * 、
    * 】用使用**`HyPerScript`**语言编译成`LLVM`汇编代码。消除了C++编译器的耗时
    * 自动并发处理请求
    * 复杂的数据结构和算法：缓存意识，减小线程并行的开销。
  * **Dynamic Storage Allocation**
    * **`direct-mapping（DM）`**：

## SAP HANA 

```
P*time 模型是啥
```



* **特点**：同时满足OLTP&OLAP，压缩方案
* **对于OLTP优化**：列存储但，在大规模OLTP中表现良好，且冗余数据很少
  1. Query plan generation：查询编译器提前准备预备语句，计划捷径 <!--怎么个提前准备没有说-->
  2. Parallelization considered harmful：有时候并行会有害（当对很小的集合进行操作时，并行的上下文开销反而相比来说很大），所以需要对并行策略进行很好的决策
  3. Concurrent OLTP and  OLAP scheduling：1.因为没有很多数据冗余和类似于索引的转换开销（压缩方法），所以很适合大量并发OLTP工作。2.仔细的工作负载监控，防止大量的OLAP导致OLTP停止运行。
* **Data Organization**
  * `Dictionary Compression`
    * 所有列存储的数据都使用字典压缩（节约了空间，支持了高效scan）
    * 两个核心表：`Index Vector`,`Dictionary`（index存储了一行值，其下标就是其行号，字典存储列中出现的所有不同的值）
  * `Delta / Main Concept`
    * 系统对字典排序，这样一来比较对应value时，根据字典序就可以判断其大小。但插入信息时，后继的字典序都要修改，开销极大（相当于顺序存储插入元素）
    * 所以维护两个表，其中Delta中有：一个索引向量，一个无序的字典，一个基于树的索引。
    * 每次查询需要查询两个数据结构，其中delta表速度较慢，故需要在合适的时候进行合并操作。
  * `Compression Techniques`：
    * `domain coding`：列中的每个值分配`0 - n-1`中的一个整数码，压缩成二进制，存入。 <!--这样存储的不就是下标么，下标存起来有啥用？数据不就看不到了么-->
    * `Prefix coding`：删除每列最开始初重复的值，记录下其值和频率。
    * `Sparse coding`：将最平凡的值全部删除，并将他们的位置存储在一个使用前缀编码的位向量中。
    * `Cluster coding`：只具有单个不同值的块通过这个值压缩，另外还需要一个向量表来表明哪些块被压缩了。
    * `Indirect coding`：只对合适的块应用domain编码
    * `Run length encoding` <!--看中文都看不懂-->
  * column ordering problem：列中存储的值下标就是其行号，如果直接进行排序，需要额外引入转换开销。所以只把每列中最常用的值移动到最前方，其他值在不破坏整体顺序的情况下用更高级的办法进行压缩。
  * Data aging：

### 其他：SPARK、Redis、[SQLite（轻量）](https://www.zhihu.com/question/511921936/answer/2313819670)

