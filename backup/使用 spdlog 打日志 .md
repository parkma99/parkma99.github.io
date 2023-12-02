## spdlog 是什么

> [spdlog](https://github.com/gabime/spdlog) Very fast, header-only/compiled, C++ logging library

官方仓库介绍是可以只包含头文件的日志库， 具体使用下来可以只包含头文件，（由于本人的Cmake 的知识太少），使用头文件的方式必须指定在 include 目录下， 后面就将整个 spdlog 仓库作为第三方库放在third_party 下。

## 简单用法

```cpp
#define SPDLOG_ACTIVE_LEVEL SPDLOG_LEVEL_TRACE
#include "spdlog/spdlog.h"

int main()
{
    spdlog::info("Welcome to spdlog!");
    spdlog::error("Some error message with arg: {}", 1);

    spdlog::warn("Easy padding in numbers like {:08d}", 12);
    spdlog::critical("Support for int: {0:d};  hex: {0:x};  oct: {0:o}; bin: {0:b}", 42);
    spdlog::info("Support for floats {:03.2f}", 1.23456);
    spdlog::info("Positional args are {1} {0}..", "too", "supported");
    spdlog::info("{:<30}", "left aligned");

    spdlog::set_level(spdlog::level::debug); // Set global log level to debug
    spdlog::debug("This message should be displayed..");
    spdlog::trace("This message should not be displayed..");

    // change log pattern
    spdlog::set_pattern("[%H:%M:%S %z] [%n] [%^---%L---%$] [thread %t] %v");

    // Compile time log levels
    // define SPDLOG_ACTIVE_LEVEL to desired level
    SPDLOG_TRACE("Some trace message with param {}", 42);
    SPDLOG_DEBUG("Some debug message");
}
```

spdlog 的日志类型分为 7种
```
trace = SPDLOG_LEVEL_TRACE,
debug = SPDLOG_LEVEL_DEBUG,
info = SPDLOG_LEVEL_INFO,
warn = SPDLOG_LEVEL_WARN,
err = SPDLOG_LEVEL_ERROR,
critical = SPDLOG_LEVEL_CRITICAL,
off = SPDLOG_LEVEL_OFF,
```
## 自定义Logger

