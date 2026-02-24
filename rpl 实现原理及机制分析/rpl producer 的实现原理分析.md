# producer 的实现原理分析

[toc]

## 1 producer 的本质是什么？

在 rpl 里：

> **producer 本质是一个“可启动的数据生成器”**

它并不是一个线程，也不是一个队列。

它只是：***一个保存了“如何产生数据”的函数对象***

可以理解为：

```c++
template <typename T>
class producer {
    using Start = std::function<void(consumer<T>)>;
    Start _start;
};
```

但 Telegram 实际实现中：

- 不使用 std::function
- 不使用虚函数
- 使用模板 + move-only lambda
- 通过内联优化消除抽象开销

> [!note]
>
> 在 C++ 里，用“模板抵消虚函数开销”的核心思想是：**把运行时多态（virtual dispatch）变成编译期多态（static dispatch）**
>
> **虚函数的开销来自**
>
> 1. 通过 vptr 查 vtable
>
> 2. 间接跳转（无法内联）
>
> 3. 破坏分支预测
>
> 4. 编译器无法优化
>
> **使用模板的优势**
>
> 1. 无 vtable
> 2. 可内联
> 3. 无间接跳转
> 4. 零运行时开销
>
> **使用模板的缺点**
>
> 1. **代码膨胀** 每种类型都会实例化
> 2. **不能运行时替换** 类型必须编译期确定
> 3. **接口不稳定**   `ABI` 不稳定
> 4. **编译时间增加** 模板展开成本高
>
> **以下情况仍必须使用虚函数**
>
> 1. 插件系统
> 2. 动态加载
> 3. 运行时扩展
>
> **Rust 在“用静态多态替代虚函数”这件事上做得更彻底**
>
> Rust 天生就区分两种多态方式：
>
> :one: 静态分发（编译期）—— 零成本
>
> :two: 动态分发（运行时）—— 类似 C++ 虚函数

## 2 producer 的核心构造方式

producer 通常由函数构造：

```C++
rpl::producer<int> p = rpl::make_producer<int>(
    [](auto consumer) {
        consumer.put_next(1);
        consumer.put_done();
    }
);
```

本质等价于：

```c++
producer 持有一个 lambda：
lambda(consumer)
```

当调用：

```C++
p.start(...)
```

时：

```c++
内部执行这个 lambda
```

## 3 producer::start 的运行机制

简化后的逻辑是：

```c++
template <typename OnNext, typename OnError, typename OnDone>
void producer::start(OnNext next, OnError error, OnDone done) {
    consumer c(next, error, done);
    _start(std::move(c));
}
```

执行流程：

:one: 构造 consumer
:two:把 consumer 传给 _start
:three: _start 内部主动 push 数据

也就是说：

> producer 不会主动运行
>
> 它只有在 start() 时才会“执行”

## 4 producer 的核心运行模型

**push 模型（不是 pull）**

rpl 是：

```c++
push-driven
```

而不是：

```c++
pull-driven（如 range、generator）
```

producer 主动调用：

```c++
consumer.put_next(value)
```

consumer 并不会“拉取”数据。

## 5 producer 链式操作的实现原理

例如：

```c++
stream
| rpl::map(...)
| rpl::filter(...)
```

:heavy_exclamation_mark:**关键点：**

每个 operator：

```
输入 producer
输出 producer
```

也就是说：

```
map 返回的是一个新的 producer
```

内部结构类似：

```c++
template <typename F>
producer map(F f) {
    return producer([=](consumer out) {
        original.start(
            [=](auto v) { out.put_next(f(v)); },
            ...
        );
    });
}
```

你可以看到：

```
map 不会立刻执行
map 只是包装了 start
```

------

## 6 真正执行发生在什么时候？

只有当调用：

```c++
producer.start(...)
```

才会：

```
递归展开整个 operator 链
```

执行过程：

```
最外层 start()
    ↓
map.start()
    ↓
filter.start()
    ↓
event_stream.start()
    ↓
真正 push 数据
```

这是一个：

```
递归层层包裹的函数调用链
```

不是循环，不是调度器，而是：

```
编译期展开的函数嵌套
```

## 8 producer 为什么零开销？

Telegram rpl 设计的核心优化：

:one:  **不使用虚函数**

所有 producer 都是模板类

:two:**​ 不使用 std::function**

用 lambda + move-only wrapper

:three: **全部 inline**

每一层 operator 在编译期展开

最终效果类似：

```c++
if (cond(value)) {
    next(f(value));
}
```

编译器可以完全优化掉抽象层。

## 8 producer 和 event_stream 的关系

`event_stream` 是一个特殊 producer。

内部结构类似：

```c++
class event_stream {
    std::vector<consumer<T>> subscribers;
};
```

当调用：

```c++
stream.fire(value);
```

会：

```
遍历 subscribers
调用 put_next(value)
```

所以：

- `producer` 是抽象生成器
- `event_stream` 是手动触发型 `producer`

------

## 9 producer 的生命周期机制

producer 本身没有生命周期控制。

生命周期在：

```c++
consumer + lifetime
```

start 时会绑定 lifetime：

```c++
stream.start(next, lifetime);
```

当 lifetime destroy：

```c++
consumer 自动失效
```

producer 仍然存在，但不再推送。

## 10 完整运行流程图

```
构造 producer
      ↓
链式包装 producer
      ↓
调用 start()
      ↓
构造 consumer
      ↓
递归调用最底层 producer
      ↓
put_next()
      ↓
数据层层向上流动
```

## 11 producer 的核心设计哲学

可以总结为一句话：

> producer 是“延迟执行的函数封装”

更准确地说：

```c++
producer = void(consumer<T>)
```

整个 rpl 就是在做函数组合。

## 12 与 Rx 的区别

对比 RxCpp：

| 特性         | Rx     | rpl    |
| ------------ | ------ | ------ |
| 运行时多态   | 有     | 无     |
| 虚函数       | 有     | 无     |
| scheduler    | 有     | 无     |
| backpressure | 有     | 无     |
| 线程模型     | 多线程 | 单线程 |
| 性能         | 一般   | 极高   |

rpl 的目标是：***为 GUI 单线程场景优化***

这也符合 Telegram Desktop 的架构特点。

------

## 13 总结

> producer 是一个“接受 consumer 的函数对象”

本质等价于：

```c++
using producer<T> = std::function<void(consumer<T>)>;
```

但通过模板展开消除了 std::function 的开销。

> `producer` 通过模板单态化和函数内联，把运行时多态转换为编译期静态调用链，从而用代码体积换取零分发成本的执行效率。
>
> **它甚至不仅仅是空间换时间**
>
> 更深一层是：把`抽象成本`前移到编译期
>
> 也就是说：
>
> - 运行时几乎零成本
> - 成本转移到编译器工作量
>
> 这也是现代 C++ 高性能库的共同特点。