### 从` spdlog::info("Welcome to spdlog!");` 开始阅读

通过查找 ` spdlog::info` 函数定义，可以找到其定义在 `spdlog.h` 下

```cpp
template<typename... Args>
inline void info(format_string_t<Args...> fmt, Args &&...args)
{
    default_logger_raw()->info(fmt, std::forward<Args>(args)...);
}
```

其中使用了 ` default_logger_raw()` 应该是获取到了系统默认的记录日志的对象，然后调用这个对象的info 方法， 函数的参数是模版类型，以后在进行分析。

`default_logger_raw()` 函数的定义在`spdlog-inl.h` 下
```cpp
SPDLOG_INLINE spdlog::logger *default_logger_raw()
{
    return details::registry::instance().get_default_raw();
}
```

通过`details::registry::instance()`函数就进入了今天要阅读了 `registry` 类

## `registry` 类的成员

