LSM 树是处理大量写入工作负载时通常使用的数据结构，通过仅执行顺序写入来优化写入路径。 LSM 树是许多数据库背后的核心数据结构，包括 [BigTable](https://cloud.google.com/bigtable)、[LevelDB](https://github.com/google/leveldb)、[Cassandra](https://cassandra.apache.org/_/index.html) 和 [RocksDB](https://rocksdb.org/)。

LSM tree 之所以有效是基于以下事实：磁盘或内存的连续读写性能远高于随机读写性能，有时候这种差距可以达到三个数量级之高。
LSM tree 在工作过程中尽可能避免随机读写，充分发挥了磁盘连续读写的性能优势。

## SSTable

## 写入数据

## 读取数据

## 压实（Compaction）

## 删除数据


