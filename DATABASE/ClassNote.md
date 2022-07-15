## L1-Instruction

Relation Data Model

Data Manipulate Language（DML）

* Procedural：The query specifies the (high-level) strategy the DBMS should use to find the desired result.
* Non-Procedural：The query specifies only what data is wanted and not how to find it.

关系模型语法<!--（这个有时间再看，反正看了就忘）-->

## L3-storage

Buffer Pool

* 跟虚拟内存很像

Why not use the OS

* 因为如果交给操作系统，数据库会丧失对整个结构的管理控制，在数据库眼里只有虚拟内存（分页什么的都是OS在管）
* use mmap（内存映射文件）：相当于让操作系统当管家，自己发需求（very bad idea）

在磁盘上的存储管理

* 大部分都是在已有的文件系统上处理（写自己的文件系统太没必要）



## L7-Trees

**B+Tree**

* 特点，插入删除等基本知识

* `Clustered Indexes`：按照主键排序，堆存储<!--指的是在内存连续？-->或者是索引存储。
* `Non-Unique Indexes`
  1. key重复复制存储
  2. key唯一，但不同value存在其下的一个链表内
* `Index Scan Page Sorting`：对需要查询的tuple排序后再查询，会快很多

**B+Tree Design Choices**

* Node size
  * 硬件：比如内存按缓存大小分，磁盘按块分
  * 需求：表的大小，数据大小，OLTP小，OLAP大
* Merge Threshold
  * 减少分支次数，一次删除、插入多个节点，或者按组来。
* Variable Lenght Keys
  1. `Pointers`：每次都要寻址耗时，一般是小嵌入式设备用
  2. `Variable Length Nodes`：不常用，因为开销大
  3. `Padding`：让key的大小保持一致，太浪费空间
  4. `Key Map/Indeirection`：用index取代key-value配对：比如取较少的key的前缀构成一个表，映射到完整的key-value（类似于硬盘里面的slot table）
* Intra-Node Search
  1. linear
  2. Binary
  3. Interpolation

**Optimization**

* Prefix Compression：将相同前缀的key取出相同部分合并起来，其他部分保存在后面。
* Deduplication：把相同key的不同value保存在一个key后面。
* Bulk Insert：刚开始插入数据时，可以先在叶节点排好序，再自底向上构建B+Tree，减少分割开销。

## L8-Index  Concurrency

**两个层面的并发控制（这节课是Logical）**

1. `Logical Correctness`：逻辑上的写，读并发
2. `Physical Correctness`：具体数据存储结构，内存，指针

**两种锁**

1. `Locks`：高层次，事务之间的锁，允许roll back 操作
2. `Latches`：低层次，关于内部数据结构的锁，线程之间的锁，不可回退。

**Lache Implementation**

1. `Blocking OS Mutex`：
   1. `user-space`，`OS-level`两种锁，user-space锁对于申请者（如DBOS）表面上是一个，但实际上可能同其他锁绑定 <!--为啥-->
   2. 用起来很简单，但开销大且不能控制锁的时间。
2. `Test-And-Set`：完全由自己管理，但对Cache不友好 <!--为啥-->，且进程等待时其实是在死循环
3. `Reader-Writer Latches`：允许多个读者，单个写者，缺点是要注意死锁。 <!--上面那俩锁不也能用来实现这个功能么-->

**Hash Table Latching** <!--暂时不管-->

**B+Tree Latching**

* Search：从根部出发，对孩子上锁，如果孩子安全，父母解锁（孩子安全指的是它不会被可能的分支或者合并变动方向，所以不一定只要叶子节点安全）
* Insert/Delet
  1. 从根部出发，一路上上`READ`锁，到达叶子后上`WRITE`锁
  2. 如果叶子不安全，撤销所有锁，从根部开始，一路上锁，如果孩子安全，给所有祖先解锁。

* Leaf Node Scan：可能会造成死锁

## L9-Sorting & Aggregation

`ORDER BY`, `GROUP BY`, `JOIN`,`DISTINCT` 

**Sorting**（就是数据结构里面的）

* `Two-way Merge Sort`：两路归并
* `K-way Merge Sort`
* Double Buffering Optimization：利用多线程，当一个线程在合并时，另一个线程把合并好的分区装入磁盘，高效利用I/O
* `B+Trees`：clustered index in trees are already sorted

**Aggregation**（聚合操作）

* `sorting`
  1. Use either an in-memory sorting algorithm or the external merge sort algorithm （if the size of thedata exceeds memory）
  2. Scan and compute
  3. If there are filter，use fileter before sorting
* Hashing <!--这里也也没学，到时候和上面的hash一起搞-->

## L10-joins

只讲了 `inner equijoin`，其他join稍作修改就能实现

**Joins’ Operator Output**

* `early materialization（Data）`：直接复制一份元组结果输出，好处是后面有关的操作可以直接在这份数据上操作，坏处是空间开销大。
*  `late materialization（Record Ids）`：这种对于列存储比较好，因为不用浪费空间存那些用不着的数据。

 **Nested Loop Join**（下面的开销指的是I/O开销）

![image-20220420105610172](C:\Users\54351\AppData\Roaming\Typora\typora-user-images\image-20220420105610172.png)

The table in the outer for loop is called the `outer table`, while the table in the inner for loop is called the `inner table`.

1. `Simple Nested Loop Join`

   * 两层循环，让tuple直接对比。

   * csot：M + (m × N)

2. `Block Nested Loop Join`
   * 都以block为单位比较，这样不用每个tuple都要单独调用一个block
   * cost：M + (M × N)
   * 如果用多个缓冲池可以并行处理
3. `Index Nested Loop Join`
   * 用现成的或者构建一个临时的`index`（外部表用index，内表用tuple），C是index占用的block数
   * Cost: M + (m × C)

**Sort-Merge Join**（就是排序加合并）

![image-20220420111303044](C:\Users\54351\Desktop\ClassNote.assets\image-20220420111303044.png)

**Hash Join**

1. **`Basic Hash Join`**
   1. `Build`：First, scan the outer relation and populate a hash table using the hash function h1 on the join attributes.
   2. `Probe`：Scan the inner relation and use the hash function h1 on each tuple’s join attributes to jump to the corresponding location in the hash table and find a matching tuple.
2. **`Grace Hash Join / Hybrid Hash Join`**
   * 就是多次哈希（每次函数不同），且内表外表都哈希，直到某个bucket的大小适合内存，在内存操作会快很多。
   * Partitioning Phase Cost: 2 × (M + N)
   * Probe Phase Cost: (M + N)

## L11-query execution-1

**Processing Models**

1. **`Iterator Model（Volcano / Pipline）`**
   * 每个操作符通过 `child.next()`获取子操作符的下一个`tuple`
   * 获取之后对其进行操作的时候，子操作也可以同时I/O，这样达到流水线的作用。
   * `pipeline breakers`：`joins`，`subqueries`，`ordering`，需要拿到所有tuple才能进行计算
2. **`Materialization Model`**
   * child每次返回所有的数据，对`OLTP`友好，但对大量数据的`OLAP`不友好
3. **`Vectorization Model`**
   * 每次返回一个向量（相当于上面俩的折中）
   * 好处还有能利用`SIMD`向量处理指令。

**Access Methods**（访问数据的方式，table或者index）

1. **`Sequential Scan`**，直接顺序遍历（遍历时做匹配）
   * 优化方法：
   * `Prefetching`：提前读入后几个pages
   * `Buffer Pool Bypass`：单独的数据通道
   * `Parallelization`
   *  `Zone Map`：记录了每个page的各种信息（如最大值等），单独存储，先访问它可能会减少总访问数。
   *  `Heap Clustering`：顺序存储？
2.  **`Index Scan`**
   * 选取最合适的index做匹配（有的index不能把原有的数据很好区分，用了之后数量可能跟原有数据差距不大）
   *  The DBMS can use bitmaps, hash tables, or Bloom filters to compute record IDs through set intersection.

**Modification Queries**

两种方法应对修改的请求：

1. Materialize tuples inside of the operator
2. Operator inserts any tuple passed in from child operators.

`Halloween Problem`：修改数据时改变了数据的位置，导致一个扫描操作两次扫描到同一个tuple。

**Expression Evaluation**

1. 构建表达式树，DBOS维护一个上下文句柄，运行时遍历树来计算结果（很慢）
2. 或者直接计算表达式的值

## L12-query execution-2

**Parallel vs Distributed Databases**

* 平行：在同一个系统，消息传递快
* 分布式：消息传递慢

**Process Models**

defines how the system supports concurrent requests from a multi-user application / environment

1. **`Process per Worker`**
   * 依赖OS scheduler
   * 对同一份数据可能会导致多份复制（解决办法：使用共享内存）
2. **`Process Pool`**
   * 依赖OS scheduler
   * 进程间可共享queries
   * query-process没有一一绑定，造成较差的局部性
3. **`Thread per Worker`**
   * DBOS全权管理
   * 上下文切换开销小
   * 不需要维护共享内存

**Inter-Query Parallelism**

`Inter`： executes different queries are concurrently

**Intra-Query parallelism**

`Intra`： executes the operations of a single query in parallel

1. 在同一个数据结构上运行
2. 或者数据分区

* **`exchange operator`**：将子分支数据处理完后再提交给上级（这里就可以并行处理）
* **`Horizontal`**：同一个operator把数据分开用多个线程计算结果。
* **`Vertical`**：每一个operator分配一个线程

**Bushy Parallelism**

a hybrid of intra-operator and inter-operator parallelism 

**I/O Parallelism**

为了减小I/O开销的瓶颈， split installation across multiple devices.

1. **`Multi-Disk Parallelism`**：操作系统把数据分开存储到多个设备，对DBOS透明。

2. **`Database Partitioning`**： database is split up into disjoint subsets that can be assigned to discrete disks.Some DBMSs allow for specification of the disk location of each individual database.
   * `logical partitioning`：the application should be able to access logical tables without caring how things are stored.
   * `vertical partitioning`：类似列存储
   * horizontal partitioning：根据key值存储

## L13-Query Planning & Optimization 

 **Overview**

*  two high-level strategies for query optimization.
  1. 
