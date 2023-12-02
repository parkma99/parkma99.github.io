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

## 详细API

以下的 API 均根据 [leveldb docs](https://github.com/google/leveldb/blob/main/doc/index.md) 进行验证



