## 从` spdlog::info("Welcome to spdlog!");` 开始阅读

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
    static registry &instance();
private:
    registry();
    ~registry();
};
```
### 构造函数与析构函数
```cpp
std::shared_ptr<logger> default_logger_; // 单例 
```
无参数构造函数和析构函数都位于`private`，有参数的构造函数和赋值重载函数设置为`delete`，说明不允许用户创建 register 类的实例，register类的实例由spdlog库进行管理，register是实现了一个单例类， `instance()` 每次返回同一个register 实例，spdlog 库通过register类来管理日志输出使用的全部资源
 
```cpp
using log_levels = std::unordered_map<std::string, level::level_enum>;
std::mutex logger_map_mutex_, flusher_mutex_;
std::unordered_map<std::string, std::shared_ptr<logger>> loggers_;
log_levels log_levels_;
```

### loggers_ 成员与线程安全

register 类的管理的核心内容是 `loggers_` ,保存用户定义的 logger 名称和logger 实例的对应关系
logger_map_mutex_ 用于在更改 `loggers_` 进行上锁，避免多线程情况下的错误

```cpp
SPDLOG_INLINE void registry::register_logger(std::shared_ptr<logger> new_logger)
{
    std::lock_guard<std::mutex> lock(logger_map_mutex_);
    register_logger_(std::move(new_logger));
}
```
Todo: `lock_guard` `mutex` 的用法

### 线程池与线程安全

```cpp
std::recursive_mutex tp_mutex_;
std::shared_ptr<thread_pool> tp_;
```
 全局的线程池，以及全局的线程池的递归锁

Todo: `recursive_mutex` 的用法

spdlog 的线程池是自己定义的，下面是具体的定义

```cpp
class SPDLOG_API thread_pool
{
public:
    using item_type = async_msg;
    using q_type = details::mpmc_blocking_queue<item_type>;

    thread_pool(size_t q_max_items, size_t threads_n, std::function<void()> on_thread_start, std::function<void()> on_thread_stop);
    thread_pool(size_t q_max_items, size_t threads_n, std::function<void()> on_thread_start);
    thread_pool(size_t q_max_items, size_t threads_n);

    // message all threads to terminate gracefully and join them
    ~thread_pool();

    thread_pool(const thread_pool &) = delete;
    thread_pool &operator=(thread_pool &&) = delete;

    void post_log(async_logger_ptr &&worker_ptr, const details::log_msg &msg, async_overflow_policy overflow_policy);
    void post_flush(async_logger_ptr &&worker_ptr, async_overflow_policy overflow_policy);
    size_t overrun_counter();
    void reset_overrun_counter();
    size_t queue_size();

private:
    q_type q_;

    std::vector<std::thread> threads_;

    void post_async_msg_(async_msg &&new_msg, async_overflow_policy overflow_policy);
    void worker_loop_();

    // process next message in the queue
    // return true if this thread should still be active (while no terminate msg
    // was received)
    bool process_next_msg_();
};
```
后续在具体阅读其代码

全局的线程池对象 `tp_` 用在创建async_logger
```cpp
static std::shared_ptr<async_logger> create(std::string logger_name, SinkArgs &&...args)
    {
        auto &registry_inst = details::registry::instance();

        // create global thread pool if not already exists..

        auto &mutex = registry_inst.tp_mutex();
        std::lock_guard<std::recursive_mutex> tp_lock(mutex);
        auto tp = registry_inst.get_tp();
        if (tp == nullptr)
        {
            tp = std::make_shared<details::thread_pool>(details::default_async_q_size, 1U);
            registry_inst.set_tp(tp);
        }

        auto sink = std::make_shared<Sink>(std::forward<SinkArgs>(args)...);
        auto new_logger = std::make_shared<async_logger>(std::move(logger_name), std::move(sink), std::move(tp), OverflowPolicy);
        registry_inst.initialize_logger(new_logger);
        return new_logger;
    }
```
从上述代码可以看出全局线程池不是由register类创建的，而是在创建async_logger时无法获得到全局的线程池时创建的

### formatter_ 成员

``` cpp
SPDLOG_INLINE registry::registry()
    : formatter_(new pattern_formatter()){}
SPDLOG_INLINE void registry::set_formatter(std::unique_ptr<formatter> formatter)
{
    std::lock_guard<std::mutex> lock(logger_map_mutex_);
    formatter_ = std::move(formatter);
    for (auto &l : loggers_)
    {
        l.second->set_formatter(formatter_->clone());
    }
}
```
formatter_ 成员默认是 pattern_formatter， 同时提供了更换 formatter 的接口，register管理的logger使用同一个formatter_。

formatter_ 是一个基类，定义如下
```cpp
class formatter
{
public:
    virtual ~formatter() = default;
    virtual void format(const details::log_msg &msg, memory_buf_t &dest) = 0;
    virtual std::unique_ptr<formatter> clone() const = 0;
};
```
用于提供一种格式化的方法

### 


