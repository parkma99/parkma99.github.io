LevelDB是一个键值对数据库，是一个持久化的有序的Map。LevelDB实际上不是一个功能完备的数据库，而只是一个数据库函数库，提供了接口来操作数据库，但因此可以通过提供的接口来初步使用LevelDB

## 环境配置

1. 从[leveldb release](https://github.com/google/leveldb/releases) 下载最新的稳定版代码，我这里下载的是1.23 版本的

2.  使用cmake 编译安装leveldb

```bash
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release .. && cmake --build .
```
直接运行后报错，查看报错信息后发现是 leveldb 使用的第三方库没有在代码库中，下载 benchmark 和 googletest 的源码并放到third_party 目录下后，重写运行上述命令，即可编译成功

安装到系统
```bash
make install
```

3. 创建cmake 项目使用leveldb

```cmake
add_executable(leveldb_example leveldb_example.cpp)
find_package(Threads)
target_link_libraries (leveldb_example ${CMAKE_THREAD_LIBS_INIT})
find_package(leveldb CONFIG)
find_library(LEVELDB_LIB leveldb REQUIRED)
find_path(LEVELDB_INCLUDE_DIR "leveldb/db.h" REQUIRED)
target_link_libraries(leveldb_example PRIVATE ${LEVELDB_LIB})
target_include_directories(leveldb_example PRIVATE ${LEVELDB_INCLUDE_DIR})
```
## 简单用法

```cpp
#include <assert.h>
#include <string.h>
#include <leveldb/db.h>
#include <iostream>

using namespace leveldb;

using namespace std;

int main()
{
    leveldb::DB *db;
    leveldb::Options options;
    options.create_if_missing = true;

    // open
    leveldb::Status status = leveldb::DB::Open(options, "testdb", &db);
    assert(status.ok());

    string key = "name";
    string value = "shane";

    cout << "write:" "("<< key << ")" "=>" "(" << value << ")" << endl;
    status = db->Put(leveldb::WriteOptions(), key, value);
    assert(status.ok());
    status = db->Get(leveldb::ReadOptions(), key, &value);
    assert(status.ok());
    cout << "read:" "("<< key << ")" "=>" "(" << value << ")" << endl;
    status = db->Delete(leveldb::WriteOptions(), key);
    assert(status.ok());
    cout << "delete:" "("<< key << ")"  << status.ToString() << endl;
    status = db->Get(leveldb::ReadOptions(), key, &value);
    if (!status.ok()) {
        cout << "read error: " << status.ToString() << "("<< key << ")"  << endl;
    } else {
        cout << "read:" "("<< key << ")" "=>" "(" << value << ")" << endl;
    }
    delete db;
    return 0;
}
```

## LevelDB API

以下的 API 均根据 [leveldb docs](https://github.com/google/leveldb/blob/main/doc/index.md) 进行验证

打开数据库
```cpp
    leveldb::DB *db;
    leveldb::Options options;
    options.create_if_missing = true;
    leveldb::Status status = leveldb::DB::Open(options, "testdb", &db);
    assert(status.ok());
```

关闭数据库
```cpp
delete db;
```

读和写

```cpp
    string key = "name";
    string value = "shane";
    status = db->Put(leveldb::WriteOptions(), key, value);
    assert(status.ok());
    status = db->Get(leveldb::ReadOptions(), key, &value);
    assert(status.ok());
```
原子更新

```cpp
#include <leveldb/write_batch.h>
leveldb::Status s = db->Get(leveldb::ReadOptions(), key, &value);
    if (s.ok()) {
        leveldb::WriteBatch batch;
        batch.Delete(key);
        batch.Put("name2", value);
        s = db->Write(leveldb::WriteOptions(), &batch);
    }
```

同步写
```cpp
    leveldb::WriteOptions write_options;
    write_options.sync = true;
    db->Put(write_options, "key1", "value1");
```

并发

当一个进程打开一个LevelDB数据库时，会获取这个数据库的一个文件锁，其它进程就没法获取这个文件锁了。所以一个LevelDB数据库只支持一个进程同时访问，但是这一个进程里面可以同时有多个线程并发访问。对于leveldb::DB里的很多方法，都是线程安全的，在这些方法内都有加锁的步骤。但是对于其它的一些对象，比如WriteBatch，如果多线程并发访问，需要自己同步。

迭代

```cpp
    leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
    for (it->SeekToFirst(); it->Valid(); it->Next()) {
        cout << it->key().ToString() << ": "  << it->value().ToString() << endl;
    }
    assert(it->status().ok());  // Check for any errors found during the scan
    delete it;
// 反向迭代
for (it->SeekToLast(); it->Valid(); it->Prev()) {
}
```
Snapshots

LevelDB支持快照功能。快照是一个一致性视图，当创建一个快照时，就给那个时刻的数据库状态打了个快照，以后的更新插入删除在这个快照下是不可见的，注意快照不再使用时，需要马上释放，防止不需要的数据长久被占用，无法清理。、

```cpp
    leveldb::ReadOptions read_options;
    read_options.snapshot = db->GetSnapshot();
    db->Put(write_options, "key3", "value3");
    leveldb::Iterator* iter = db->NewIterator(read_options);
    for (iter->SeekToFirst(); iter->Valid(); iter->Next()) {
        cout << iter->key().ToString() << ": "  << iter->value().ToString() << endl;
    }
    delete iter;
    db->ReleaseSnapshot(read_options.snapshot);
```

Slice

```cpp
leveldb::Slice s1 = "hello";

std::string str("world");
leveldb::Slice s2 = str;

std::string str = s1.ToString();
assert(str == std::string("hello"));
```

Be careful when using Slices since it is up to the caller to ensure that the external byte array into which the Slice points remains live while the Slice is in use. For example, the following is buggy:

变量的生命周期问题

```cpp
leveldb::Slice slice;
if (...) {
  std::string str = ...;
  slice = str;
}
Use(slice);
```

比较器

```cpp
#include <leveldb/comparator.h>
class TwoPartComparator : public Comparator {
public:
    [[nodiscard]] int Compare(const leveldb::Slice& a, const leveldb::Slice& b) const override {
        if ( a.size() == b.size()) return 0;
        if (a.size() > b.size()) return 1;
        return -1;
    }

    // Ignore the following methods for now:
    [[nodiscard]] const char* Name() const override { return "TwoPartComparator"; }
    void FindShortestSeparator(std::string*, const leveldb::Slice&) const override {}
    void FindShortSuccessor(std::string*) const override {}
};
TwoPartComparator cmp;
    leveldb::DB* db;
    leveldb::Options options;
    options.create_if_missing = true;
    options.comparator = &cmp;
    leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
    assert(status.ok());
```
在我的开发环境下需要添加 `-fno-rtti` 编译参数，不然出现ld错误



