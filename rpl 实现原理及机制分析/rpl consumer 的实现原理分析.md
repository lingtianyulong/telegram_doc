# rpl consumer 的实现原理分析

[toc]

## 1 整体结构回顾 Producer $\leftrightarrow$ Consumer

在 rpl 中，一个典型链路是：

```c++
producer
   ↓
operator(变换)
   ↓
consumer
```

本质上：

- **producer**：保存“如何产生数据”的逻辑（延迟执行）
- **consumer**：保存“如何处理数据”的逻辑（回调封装）
- **operator**：返回新的 producer，本质是 producer 的包装

------

## 2 consumer 的核心结构

在 Telegram rpl 中，consumer 本质上是一个 **回调函数集合的类型擦除容器**，通常包含：

```c++
OnNext(Value)
OnError(Error)
OnDone()
```

可以理解为：

```c++
template <typename Value, typename Error>
struct consumer {
    void next(Value);
    void error(Error);
    void done();
};
```

但实际实现并不是简单的虚函数接口，而是：

- 使用模板 + 小对象优化 + 类型擦除
- 避免虚函数带来的额外开销

## 3 consumer 的核心设计思想

**:one: ​无虚函数（Zero virtual dispatch）**

rpl 不使用传统：

```c++
struct IConsumer {
    virtual void next(...) = 0;
};
```

原因：

- vtable 间接跳转
- cache 不友好
- 无法 inline

而是采用：

```c++
template <typename OnNext, typename OnError, typename OnDone>
class consumer_impl;
```

在编译期生成具体类型。

这意味着：

- next 是 **直接函数调用**
- 可以 inline
- 没有 vtable

**:two: 类型擦除包装**

虽然内部是模板类型，但对外必须：

```c++
producer.start(...)
```

因此需要“统一存储”。

rpl 通过：

- 小对象优化（Small Buffer Optimization）
- 手写 type-erasure 容器
- placement new
- 手动析构

来实现：

```c++
class consumer {
    void (*_next)(void*, Value);
    void (*_error)(void*, Error);
    void (*_done)(void*);
    void* _storage;
};
```

效果：

- 外部类型统一
- 内部仍然是模板实例
- 无虚函数
- 调用仍然是静态分发

## 4 consumer 的运行机制

当你写：

```c++
producer.start(
    [](auto v){ ... },
    [](auto e){ ... },
    []{ ... }
);
```

流程如下：

**步骤 1：创建 consumer**

模板实例化：

```c++
consumer_impl<Lambda1, Lambda2, Lambda3>
```

将 lambda 存入内部 buffer。

**步骤 2：producer 持有 consumer**

producer 的 start：

```c++
void start(Consumer&& c) {
    _logic(std::move(c));
}
```

producer 并不产生数据，它只保存“如何调用 consumer”。

**步骤 3：数据产生**

当 producer 触发：

```c++
consumer.next(value);
```

实际执行：

```c++
_fn_next(_storage, value)
```

其中：

- `_fn_next` 是模板生成的静态函数
- `_storage` 指向真实 lambda 对象

所以调用路径是：

```c++
producer → static function pointer → lambda
```

:warning:不是虚函数
:warning:不是 std::function
:warning:没有堆分配（多数情况）

## 5 consumer 与 lifetime 的关系

在 Telegram Desktop 中，rpl 强依赖 `lifetime`。

典型使用：

```c++
producer.start_with_next(callback, lifetime);
```

核心机制：

- `consumer` 被注册到 `lifetime`
- `lifetime` 销毁时
- `consumer` 自动取消订阅

实现方式：

- `consumer` 内部保存 `lifetime` 指针
- `lifetime` 保存 `destroy` 回调列表
- `destroy` 时执行

这保证：

> 不会出现 Qt 那种悬空回调问题

## 6 consumer 的执行模型

rpl 是：

- 同步执行模型
- 无调度器
- 无线程切换
- 无事件循环依赖

这点和：

- RxCpp
- RxJava

不同。

rpl 更像：

> “编译期优化的函数组合库”

consumer 只是一个回调容器。

## 7 和 producer 的配合机制

producer 本质是：

```c++
std::function<void(consumer)>
```

（真实实现更高效）

consumer 是：

```
数据终点
```

operator 实现方式：

```c++
new producer( [upstream] (consumer c) {
    upstream.start( new consumer(
        [c](value v){
            c.next(transform(v));
        }
    ));
});
```

可以看到：

- consumer 被包一层
- 数据逐层向下传递
- 所有操作在编译期展开

## 8 性能来源分析

为什么 Telegram 的 rpl 极其高效？

原因：

:white_check_mark:**无虚函数**

:white_check_mark: **无 std::function**

:white_check_mark: **模板 inline**

:white_check_mark: **小对象优化**

:white_check_mark:**无 scheduler**

:white_check_mark:**无动态多态**

consumer 是整个链路性能的关键。

它让：

```c++
producer → operator → operator → consumer
```

最终退化成：

```
函数嵌套调用
```

几乎等价于手写代码。

## 9 consumer 的本质总结

可以把 rpl consumer 理解为：

> 一个“可移动的、类型擦除的、零虚函数的回调三元组”

它的特征：

| 特性         | 实现方式          |
| ------------ | ----------------- |
| 多态         | 模板实例化        |
| 统一存储     | 手写 type erasure |
| 生命周期管理 | lifetime          |
| 性能优化     | inline + 无虚函数 |
| 取消订阅     | lifetime destroy  |

## 10 回答你一个关键问题

你之前问：

> rpl 是不是用空间换时间？

是的。

consumer 和 producer 都：

- 生成大量模板实例
- 编译期膨胀
- 代码体积增大

换来的结果：

- 运行时几乎无抽象成本
- 和手写代码接近

这就是典型：

> 用编译期空间换运行期时间

## 11 一句话总结

在 Telegram Desktop 的 rpl 中：

> consumer 不是“接口对象”，
>  而是“编译期生成的回调封装体”，
>  通过手写类型擦除实现统一存储，
>  通过 lifetime 实现安全销毁，
>  最终把整个响应式链退化成一组可 inline 的函数调用。