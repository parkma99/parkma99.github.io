LSM 树是处理大量写入工作负载时通常使用的数据结构，通过仅执行顺序写入来优化写入路径。 LSM 树是许多数据库背后的核心数据结构，包括 [BigTable](https://cloud.google.com/bigtable)、[LevelDB](https://github.com/google/leveldb)、[Cassandra](https://cassandra.apache.org/_/index.html) 和 [RocksDB](https://rocksdb.org/)。

LSM tree 之所以有效是基于以下事实：磁盘或内存的连续读写性能远高于随机读写性能，有时候这种差距可以达到三个数量级之高。
LSM tree 在工作过程中尽可能避免随机读写，充分发挥了磁盘连续读写的性能优势。

## SSTable

LSM 树使用Sorted Strings Table(SSTable) 格式将数据持久保存到磁盘。SSTable 是一种存储键值对的格式，其中键按排序顺序排列。 SSTable 将由多个称为segment的排序文件组成。这些segment一旦写入磁盘就不可更改。

![image](https://github.com/parkma99/parkma99.github.io/assets/84610851/c0c00fc3-0b41-4328-ad97-b9a31c4b843e)

## 写入数据
LSM tree 的所有写操作均为连续写，因此效率非常高。但由于外部数据是无序到来的，如果无脑连续写入到 segment，显然是不能保证顺序的。对此，LSM tree 会在内存中构造一个有序数据结构（称为 memtable），例如红黑树。每条新到达的数据都插入到该红黑树中，从而始终保持数据有序。当写入的数据量达到一定阈值时，将触发红黑树的 flush 操作，把所有排好序的数据一次性写入到硬盘中（该过程为连续写），生成一个新的 segment。而之后红黑树便从零开始下一轮积攒数据的过程。

![image](https://github.com/parkma99/parkma99.github.io/assets/84610851/89e28c16-455c-4ab2-9c53-7a81ebf6d065)

![image](https://github.com/parkma99/parkma99.github.io/assets/84610851/2c78b596-9aa5-4af1-8f98-4bc882ca336a)

## 读取数据

那么我们如何在 SSTable 中找到一个值呢？一种简单的方法是扫描segment以查找所需的key。我们将从最新的segment开始查找到最旧的segment，直到找到我们正在寻找的key。这意味着我们能够更快地检索最近写入的key value。

当扫描某个特定的 segment 时，由于该 segment 内部的数据是有序的，因此可以使用二分查找的方式，在O(log n)的时间内得到查询结果。但对于二分查找来说，要么一次性把数据全部读入内存，要么在每次二分时都消耗一次磁盘 IO，当 segment 非常大时（这种情况在大数据场景下司空见惯），这两种情况的代价都非常高。
一个简单的优化是保留内存中的[稀疏索引(sparse index)](https://yetanotherdevblog.com/dense-vs-sparse-indexes/)

![image](https://github.com/parkma99/parkma99.github.io/assets/84610851/85a7dc4b-5f25-4e6f-bfb5-90d578522446)

有了稀疏索引之后，可以先在索引表中使用二分查找快速定位某个 key 位于哪一小块数据中，然后仅从磁盘中读取这一块数据即可获得最终查询结果，此时加载的数据量仅仅是整个 segment 的一小部分，因此 IO 代价较小。以上图为例，假设我们要查询 dollar 所对应的 value。首先在稀疏索引表中进行二分查找，定位到 dollar 应该位于 dog 和 downgrade 之间，对应的 offset 为 17208~19504。之后去磁盘中读取该范围内的全部数据，然后再次进行二分查找即可找到结果，或确定结果不存在。

稀疏索引极大地提高了查询性能，然而有一种极端情况却会造成查询性能骤降：当要查询的结果在 SSTable 中不存在时，我们将不得不依次扫描完所有的 segment，这是最差的一种情况。有一种称为 **布隆过滤器（bloom filter）** 的数据结构天然适合解决该问题。布隆过滤器是一种空间效率极高的算法，能够快速地检测一条数据是否在数据集中存在。我们只需要在写入每条数据之前先在布隆过滤器中登记一下，在查询时即可断定某条数据是否缺失。

## 删除数据

LSM tree 所采用的做法是设计一个特殊的标志位，称为 tombstone（墓碑），删除一条数据就是把它的 value 置为墓碑，如下图所示：

![image](https://github.com/parkma99/parkma99.github.io/assets/84610851/7a2bb334-d256-4a55-a2a2-a2f8ec5c6492)

这个例子展示了删除 segment 2 中的 dog 之后的效果。注意，此时 segment 1 中仍然保留着 dog 的旧数据，如果我们查询 dog，那么应该返回空，而不是 52。因此，删除操作的本质是覆盖写，而不是清除一条数据，这一点初看起来不太符合常识。墓碑会在 compact 操作中被清理掉，于是置为墓碑的数据在新的 segment 中将不复存在。

## 压实（Compaction）

随着数据的不断积累，SSTable 将会产生越来越多的 segment，导致查询时扫描文件的 IO 次数增多，效率降低，因此需要有一种机制来控制 segment 的数量。对此，LSM tree 会定期执行文件合并（compaction）操作，将多个 segment 合并成一个较大的 segment，随后将旧的 segment 清理掉。由于每个 segment 内部的数据都是有序的，合并过程类似于归并排序。

![image](https://github.com/parkma99/parkma99.github.io/assets/84610851/31ec315c-f2dc-42c0-9c52-dfa10439c361)
在上图的示例中，segment 1 和 2 中都存在 key 为 dog 的数据，这时应该以最新的 segment 为准，因此合并后的值取 84 而不是 52，这实现了类似于字典/HashMap 中“覆盖写”的语义。因此，在新的segment被标记为删除的key， 在新的segment 将不会出现
至于旧的segment何时被删除，具体要看具体的实现

## 总结

LSM tree 存储引擎的工作原理包含以下几个要点：

- 写数据时，首先将数据缓存到内存中的一个有序树结构中（称为 memtable）。同时触发相关结构的更新，例如布隆过滤器、稀疏索引。
- 当 memtable 积累到足够大时，会一次性写入磁盘中，生成一个内部有序的 segment 文件。该过程为连续写，因此效率极高。
- 进行查询时，首先检查布隆过滤器。如果布隆过滤器报告数据不存在，则直接返回不存在。否则，按照从新到老的顺序依次查询每个 segment。
- 在查询每个 segment 时，首先使用二分搜索检索对应的稀疏索引，找到数据所在的 offset 范围。然后读取磁盘上该范围内的数据，再次进行二分查找并获得结果。
- 对于大量的 segment 文件，定期在后台执行 compaction 操作，将多个文件合并为更大的文件，以保证查询效率不衰减。


