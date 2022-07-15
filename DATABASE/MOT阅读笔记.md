#### 问题

* MOT**组同步重做日志记录**中：事务在持久化到磁盘之后才会被标记为成功，那每个事务都必须要访问磁盘，会不会形成性能瓶颈（是不是因为耗时最多的是查询等的处理，持久化需要的时间并不多）
* 日志的内容是啥

[MOT关键技术](https://opengauss.org/zh/docs/2.0.0/docs/Developerguide/MOT%E5%85%B3%E9%94%AE%E6%8A%80%E6%9C%AF.html)

* 数据和索引都在内存中
* 免锁事务管理，OCC
* 免锁索引：Masstree，优化锁方法、高速缓存感知和内存预取。
* 高效持久性：
  * 并行日志，所有openGauss磁盘表都支持。
  * 每个事务的日志缓冲和无锁事务准备。
  * 增量更新记录，即只记录变化。
  * 除了同步和异步之外，创新的NUMA感知组提交日志记录。
  * 最先进的数据库检查点（CALC）使内存和计算开销降到最低。

[MOT的概念](https://opengauss.org/zh/docs/2.0.0/docs/Developerguide/MOT%E7%9A%84%E6%A6%82%E5%BF%B5.html)（这章跟代码关联性最大）

#### 并发控制：OCC

* [MOT本地内存和全局内存](https://opengauss.org/zh/docs/2.0.0/docs/Developerguide/MOT%E6%9C%AC%E5%9C%B0%E5%86%85%E5%AD%98%E5%92%8C%E5%85%A8%E5%B1%80%E5%86%85%E5%AD%98.html)
  * 全局内存：是所有核共享的长期内存，主要用于存储所有的表数据和索引。
  * 本地内存：短期内存，主要由会话使用，用于处理事务及将数据更改存储到事务内存中，直到提交阶段。
  * 当事务需要更改时，SILO将该事务的所有数据从全局内存复制到本地内存。使用OCC方法，全局内存中放置的是最小的锁，因此争用时间极短。事务更改完成后，该数据从本地内存回推到全局内存中。
  * ![mot内存分配流程](C:\Users\54351\Desktop\MOT阅读笔记.assets\mot内存分配流程.png)
* [MOT SILO增强特性](https://opengauss.org/zh/docs/2.0.0/docs/Developerguide/MOT-SILO%E5%A2%9E%E5%BC%BA%E7%89%B9%E6%80%A7.html)（SILO是啥）
* [MOT 隔离级别](https://opengauss.org/zh/docs/2.0.0/docs/Developerguide/MOT%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB.html)
* [MOT乐观并发控制](https://opengauss.org/zh/docs/2.0.0/docs/Developerguide/MOT%E4%B9%90%E8%A7%82%E5%B9%B6%E5%8F%91%E6%8E%A7%E5%88%B6.html)

#### 扩展FDW与其他openGauss特性

* FDW是MOT与openGauss的
* 创建表和索引
* 索引规划与执行的使用方法
* 持久性、复制性和高可用性（存储引擎不处理日志、检查点和恢复，特别是因为某些事务包含多个不同存储引擎的表）
* VACUUM和DROP（没看懂）

#### NUMA-aware分配和亲和性

* NUMA：非统一内存访问，处理器访问本地内存的速度更快。
* NUMA感知：MOT意识到内存不是统一的，而是通过访问最快和最本地的内存来获得最佳性能。
* 总结：MOT有一个智能内存控制模块，它预先为各种类型的内存对象分配了内存池。这种智能内存控制可以提高性能，减少锁并保证稳定性。事务的内存对象分配始终是NUMA-local，从而保证了CPU内存访问的最佳性能，降低时延和争用。被释放的对象返回到内存池中。在事务期间最小化使用操作系统的malloc函数可以避免不必要的锁。

#### MOT索引

* 基于Masstree（Trie前缀树&B+树）
  * https://zhuanlan.zhihu.com/p/271740123
  * https://www.modb.pro/db/103351
  * 支持主索引，辅助索引，无键索引

#### MOT持久性概念

* 将事务性更改持久化到磁盘，并保持频繁的定期[MOT检查点](https://opengauss.org/zh/docs/2.0.0/docs/Developerguide/MOT持久性.html#section182761535131617)
* MOT日志记录：WAL（预写日志记录）重做日志概念（事务的缓冲区被写入日志之后，才更改内存中的数据，然后提交事务）
* 通过openGauss的XLOG接口持久化WAL记录（代码里面可能有？）
* 三种日志记录
  * 同步重做日志记录
  * 组同步重做日志记录
  * 异步重做日志记录
* 检查点算法：CALC

数据恢复：平行重做日志（最后单位为核）

面锁索引

[索引](https://opengauss.org/zh/docs/2.0.0/docs/Developerguide/MOT%E7%B4%A2%E5%BC%95.html)：关键数据结构：mastree

预缓存对象池（按需预先访问操作系统中较大的内存块，然后按需将内存分配给MOT，使内存分配更加高效。）

外部数据封装（FDW）：FDW在MOT引擎和openGauss组件之间进行中介。





