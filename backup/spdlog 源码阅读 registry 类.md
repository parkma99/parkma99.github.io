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

`registry` 类的定义在 `registry.h`， 实现定义在 `registry-inl.h` 下

```cpp
class SPDLOG_API registry
{
public:
    using log_levels = std::unordered_map<std::string, level::level_enum>;
    registry(const registry &) = delete;
    registry &operator=(const registry &) = delete;

    void register_logger(std::shared_ptr<logger> new_logger);
    void initialize_logger(std::shared_ptr<logger> new_logger);
    std::shared_ptr<logger> get(const std::string &logger_name);
    std::shared_ptr<logger> default_logger();

    // Return raw ptr to the default logger.
    // To be used directly by the spdlog default api (e.g. spdlog::info)
    // This make the default API faster, but cannot be used concurrently with set_default_logger().
    // e.g do not call set_default_logger() from one thread while calling spdlog::info() from another.
    logger *get_default_raw();

    // set default logger.
    // default logger is stored in default_logger_ (for faster retrieval) and in the loggers_ map.
    void set_default_logger(std::shared_ptr<logger> new_default_logger);

    void set_tp(std::shared_ptr<thread_pool> tp);

    std::shared_ptr<thread_pool> get_tp();

    // Set global formatter. Each sink in each logger will get a clone of this object
    void set_formatter(std::unique_ptr<formatter> formatter);

    void enable_backtrace(size_t n_messages);

    void disable_backtrace();

    void set_level(level::level_enum log_level);

    void flush_on(level::level_enum log_level);

    template<typename Rep, typename Period>
    void flush_every(std::chrono::duration<Rep, Period> interval)
    {
        std::lock_guard<std::mutex> lock(flusher_mutex_);
        auto clbk = [this]() { this->flush_all(); };
        periodic_flusher_ = details::make_unique<periodic_worker>(clbk, interval);
    }

    void set_error_handler(err_handler handler);

    void apply_all(const std::function<void(const std::shared_ptr<logger>)> &fun);

    void flush_all();

    void drop(const std::string &logger_name);

    void drop_all();

    // clean all resources and threads started by the registry
    void shutdown();

    std::recursive_mutex &tp_mutex();

    void set_automatic_registration(bool automatic_registration);

    // set levels for all existing/future loggers. global_level can be null if should not set.
    void set_levels(log_levels levels, level::level_enum *global_level);

    static registry &instance();

    void apply_logger_env_levels(std::shared_ptr<logger> new_logger);

private:
    registry();
    ~registry();

    void throw_if_exists_(const std::string &logger_name);
    void register_logger_(std::shared_ptr<logger> new_logger);
    bool set_level_from_cfg_(logger *logger);
    std::mutex logger_map_mutex_, flusher_mutex_;
    std::recursive_mutex tp_mutex_;
    std::unordered_map<std::string, std::shared_ptr<logger>> loggers_;
    log_levels log_levels_;
    std::unique_ptr<formatter> formatter_;
    spdlog::level::level_enum global_log_level_ = level::info;
    level::level_enum flush_level_ = level::off;
    err_handler err_handler_;
    std::shared_ptr<thread_pool> tp_;
    std::unique_ptr<periodic_worker> periodic_flusher_;
    std::shared_ptr<logger> default_logger_;
    bool automatic_registration_ = true;
    size_t backtrace_n_messages_ = 0;
};
```

无参数构造函数和析构函数都位于`private`，有参数的构造函数和赋值重载函数设置为`delete`，说明不允许用户创建 register 类的实例，register类的实例由spdlog库进行管理，register是实现了一个单例类， `instance()` 每次返回同一个register 实例，spdlog 库通过register类来管理日志输出使用的全部资源
 
```cpp
using log_levels = std::unordered_map<std::string, level::level_enum>;
std::mutex logger_map_mutex_, flusher_mutex_;
std::recursive_mutex tp_mutex_;
std::unordered_map<std::string, std::shared_ptr<logger>> loggers_;
log_levels log_levels_;
```

register 类的管理的核心内容是 `loggers_` ,保存用户定义的 logger 名称和logger 实例的对应关系
logger_map_mutex_ 用于在更改 `loggers_` 进行上锁，避免多线程情况下的错误

```cpp
SPDLOG_INLINE void registry::register_logger(std::shared_ptr<logger> new_logger)
{
    std::lock_guard<std::mutex> lock(logger_map_mutex_);
    register_logger_(std::move(new_logger));
}
```
Todo: lock_guard 的用法

