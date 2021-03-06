# DataBase知识点
## 1.事务处理与恢复
理解数据库事务的前提：缓存管理、锁管理、恢复；  
在节点（站点）内部：  
事务管理器  
锁管理器  
页缓存  
日志管理器  
分布式  
### 1.1ACID
原子性Atomicity，一致性Consistency，隔离性Isolation，持久性Durability  
### 1.2缓冲区管理
数据库大多采用双存储结构：内存+磁盘  
#### 概念：  
虚拟磁盘/页缓存/缓冲池  一个意思  
换入page in，换出evict，刷写flush，脏页  
#### 页缓存的主要功能：  
1）在内存保留被缓存的页的内容 2）将对磁盘页的修改缓存，修改的是缓存的版本 3）当请求页不再内存且空间足够，会将磁盘也换入到内存4）如果内存不够，将会选择一页换出  
#### 绕过内核的页缓存：
数据库使用O_DIRECT打开文件，此标准允许IO绕过内核的页缓存直接访问磁盘，使用数据库专用的缓冲区管理。此过程不是异步的。  
#### 页缓存
直接访问块存储设备，对磁盘抽象访问，将逻辑写与物理写分离。若不想某个页被换出，可以固定pin，若缓存页被修改，则被标记为脏页，表示磁盘与内存不同步，必须要刷写回磁盘。  
后台刷写器：数据库单独的后台进程，循环检查可能被换出的脏页，更新磁盘。  
检查点经常：为了保证数据的持久性，检查点进程会协调刷写进程，控制预写日志（WAL）和页缓存。  
#### 权衡：
1）推迟刷写减少磁盘IO，2）提前刷写让页能快速换出， 3）选择最优的换出页 4)将缓存大小保持在内存范围内 5）避免因没有持久化到磁盘而丢失  
#### 页面置换
FIFO、LRU、CLOCK（LRU的变体），LFU（最小使用频率） 
### 1.2恢复
预写日志（WAL，Write-Ahead Log，也叫提交日志），一种仅追加的辅助磁盘数据结构，用于崩溃和事务的恢复。  
1）在内存中的缓存页将修改缓存时，保证数据库的持久性
2）每个修改数据库状态的操作，必须先写日志到磁盘，然后才能修改相关页的内容  
3）崩溃时，可以从日志恢复  
WAL每条记录都有LSN（日志序列号，单调递增），WAL保存食事务完成的情况，仅当事务完成刷盘，才能视为提交。  
为了防止回滚与撤回中发生崩溃，使用补偿日志记录（CLR）  
steel和force策略https://blog.csdn.net/Singularinty/article/details/80747290 
### 1.3并发控制
乐观并发控制OCC、多版本并发控制MVCC、悲观并发控制PCC https://blog.csdn.net/m0_37519490/article/details/82824371   
#### 可串行化
调度：  
串行调度：一个接一个的调度  
可串行化调度：并发的执行事务，同时保证串行调度的正确性和简单性  
#### 事务隔离
协调与同步
#### 读异常写异常
脏读、不可重复读、丢失更新、脏写、写偏斜
#### 隔离级别
读未提交、读已提交、可重复读、可串行化。ACID中的隔离性意味着“可串行化”，但是可串行化牺牲了一定的并发性（冲突的时候必须串行）。  
快照隔离：一个事务可以看到在该事务开始前已提交的数据库状态，每个事务都有一个快照    在快照上查询  
#### 基于锁的并发控制
基于锁的并发控制是悲观并发控制
###### 两阶段锁2PL：
增长阶段：获取事务所需的所有锁，且不释放  
收缩阶段：释放增长阶段获得的锁
##### 死锁：
2PL保守避免死锁，一次性获取所有的锁。  
引入超时终止运行机制  
死锁检测  
#### 锁
锁（lock）：逻辑完整性  
闩（latch）：物理完整性，如树索引的合并分裂的保护。在访问任何的页前必须加闩锁，即使无锁并发控制（OCC，MVCC）也需要闩锁    
#### 读写锁
闩锁不论读还是写都是排他的，因此需要将读和写分开，因为读可以并发。我们需要保证并发的读-写、写-写之间的隔离性。  
读写锁：允许多个读，只有写需要获得对象的排他性访问。  
#### 闩锁耦合
在从根节点的搜索路径上，不必要对所有的节点上锁，当确保不会对父节点产生影响（分裂合并）时，可以及时释放父节点的闩锁。
