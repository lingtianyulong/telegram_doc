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

**角色与定位**

在 RPL 的「生产者-消费者」模型里：

- Producer：按需向某个「接收端」推送值/错误/完成，它不直接持有回调，而是接收一个 consumer。

- Consumer：这个「接收端」的抽象。它封装三个回调（next / error / done），并负责生命周期（lifetime）和状态（state）。Producer 的 generator 被调用时，会拿到一个 consumer，通过它把事件推下去。

也就是说：consumer 是订阅端接口 + 回调容器 + 生命周期管理的统一体。

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

`telegram` 中 `consumer` 的核心类型结构，如下：

```c++
consumer<Value, Error, Handlers>
    └── 继承 consumer_base<Value, Error, Handlers>
            └── 持有一个 std::shared_ptr<Handlers> _handlers
```

- Handlers 默认是 details::type_erased_handlers<Value, Error>（类型擦除的接口）。

- 具体实现是 consumer_handlers<Value, Error, OnNext, OnError, OnDone>，内部保存三个可调用对象：

  - _next：处理下一个值

  - _error：处理错误

  - _done：处理完成

因此，consumer 的「实现」就是：一个 `shared` 的 `handlers` 对象，上面挂着 `next/error/done` 三个回调。

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

> [!note]
>
> **向 Consumer 推送事件的 API（consumer_base）**
>
> | 方法                                              | 作用                                                         |
> | :------------------------------------------------ | :----------------------------------------------------------- |
> | put_next(Value &&) / put_next_copy(const Value &) | 推送一个「下一个值」，会调用 _next。返回 bool 表示是否还接受（未 terminate 时为 true）。 |
> | put_error / put_error_copy                        | 推送错误，调用 _error，并终止该 consumer（见下）。           |
> | put_done()                                        | 推送完成，调用 _done，并终止该 consumer。                    |
> | put_next_forward / put_error_forward              | 对 move 或 copy 版本的简单转发。                             |
>
> 要点：
>
> - 单次终止：一旦调用了 put_error 或 put_done，handlers 会 terminate()，之后 _handlers 被置空或失效，再调用 put_* 会直接返回/不做事，避免重复调用或 use-after-free。
>
> - put_next 的返回值：上游（如 operator 或 producer 实现）可以根据返回值决定是否继续发、是否要 copy 给多个 consumer 等（例如 event_stream 里对多个 consumer 的 copy/move 策略）。

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

> [!note]
>
> ### 终止与生命周期（terminate / lifetime）
>
> - terminate()：
>
>   - 在 consumer_base 里：若还有 _handlers，则 take_handlers() 后调用 handlers->terminate()。
>
>   - 在 type_erased_handlers::terminate() 里：设 _terminated = true 并 _lifetime.destroy()。
>
> 即：终止 = 标记已终止 + 销毁该 consumer 所持有的所有 lifetime（取消订阅、释放 state 等）。
>
> - add_lifetime(lifetime &&)：
>   - 把一段生命周期挂到当前 consumer 的 handlers 上。若已 terminate，则直接 lifetime.destroy() 并返回 false；否则加入 handlers 的 _lifetime，在 terminate() 时一起被 destroy。
>   - Producer 的 generator 返回的 lifetime 通常就是通过 consumer.add_lifetime(...) 挂上去的，所以取消订阅只要让该 lifetime 被 destroy（或先 terminate consumer）即可。
>
> - terminator()：
>
>   - 返回一个可调用对象，调用时执行 self.terminate()。常用于「当某个外部 lifetime 结束时，顺带取消这次订阅」，例如：`producer.h` Lines351-352
>
>     ```c++
>     alive_while.add(consumer.terminator());
>     consumer.add_lifetime(std::move(_generator)(consumer));
>     ```
>
>     即：先把「终止该 consumer」注册到 alive_while 上，再把 generator 返回的 lifetime 交给 consumer；这样只要 alive_while 被 destroy，consumer 会先被 terminate，其上的 lifetime（包括 generator 的）也会被一起清理。

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

> [!note]
>
> **与 Producer 的衔接（数据流机制）**
>
> 1. 用户对 producer 调用 start(next, error, done) 或 start_existing(consumer)。
>
> 2. start 内部用 make_consumer(next, error, done) 得到一个 consumer，再调用 start_existing(consumer, lifetime)。
>
> 3. start_existing 中：
>
>    - 把 consumer.terminator() 加到 alive_while，保证外部生命周期结束时能终止订阅。
>
>    - 调用 _generator(consumer)，即 producer 的 lambda，传入当前 consumer。
>
> 4. Producer 的 generator 实现（包括各种 | operator 链）会：
>
>    - 使用传入的 consumer 调用 put_next / put_error / put_done 向下游推送；
>
>    - 需要时用 consumer.add_lifetime(...) 注册子订阅的 lifetime；
>
>    - 需要时用 consumer.make_state<...>(...) 在订阅生命周期内保存状态。
>
> 5. Generator 返回的 lifetime 被 consumer.add_lifetime(...) 收纳，因此当 consumer 被 terminate（或该 lifetime 被显式 destroy）时，整个订阅链一起结束。
>
> 所以：consumer 既是「下游接口」（供 producer/operator 推送事件），又是「订阅生命周期与状态的持有者」。

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