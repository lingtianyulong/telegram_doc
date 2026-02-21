# rpl 的主要实现原理与机制

[toc]

##  1 什么是 RPL（Reactive Programming Library）

在 Telegram Desktop 的源码中（特别是 UI、数据更新和本地化等模块），经常可以看到 `rpl::producer`, `rpl::variable` 这样的类型。
 RPL 是一个 **轻量级的响应式编程库（Reactive Programming Library）**，用于**描述数据随时间变化的流**，并能在这些流上建立变换、组合、订阅等操作。

简单来说：

- **Producer** 表示一个值流（value stream），可以随时间发出多个值。
- **operator| 和 on_next** 提供管道式订阅和响应处理。
- 代码中的 UI 和数据层往往使用它来构建响应式绑定，使得数据变化自动驱动界面更新。

RPL 的设计理念与现代响应式库（例如 RxCpp、RxJS、Reactive Swift 等）类似，但更轻量、更适合嵌入到 Qt 和 Telegram Desktop 这类 C++ 程序中。

### 1.1 主要实现原理与机制

RPL 是拉取式（pull-based）的响应式库，核心是「生成器 + 消费者」：

- Producer（生产者）：内部持有一个 Generator（可调用对象），不主动发数据，只有被订阅时才执行。

- Consumer（消费者）：订阅时由上层传入，Generator 通过它把 value/error/done 推下去。

- Lifetime：表示订阅的存活期，析构或 destroy() 时执行所有已注册的清理回调，用于取消订阅和释放资源。

数据流是：有人订阅 → 调用 Generator(consumer) → Generator 在合适时机对 consumer 调用 put_next/put_error/put_done。因此是「按需拉取」触发计算，再通过 consumer 把结果推给下游。

## 2 RPL 的核心组成和抽象

### 2.1 Producer（生产者）

- 核心是一个 **可发出值的流**。
- 它并非静态值，而是随时间动态产生多个值。
- 通过 `rpl::single(value)` 可以创建只发出一个值的流。

 :large_blue_diamond:**Producer 的作用：**

- 封装一段状态变化或事件序列。
- 提供订阅接口，当值变化时触发回调。

> ### producer<Value, Error, Generator>
>
> - 本质：包装一个 Generator。
>
> - Generator 的约定：lifetime operator()(const consumer<Value, Error> &consumer)
>
> - 入参：一个 consumer（只读引用）。
>
> - 返回：一个 lifetime，用于在这次订阅存活期间注册清理逻辑（取消、释放资源等）。
>
> - 懒执行：不订阅就不会调用 Generator，因此不会产生任何数据流。
>
> ```c++
> template <typename Value, typename Error, typename Generator>
> template <typename Handlers>
> inline void producer_base<Value, Error, Generator>::start_existing(
>     const consumer_type<Handlers> &consumer, lifetime &alive_while) && {
>   alive_while.add(consumer.terminator());
>   consumer.add_lifetime(std::move(_generator)(consumer));
> }
> ```
>
> 含义：
>
> - alive_while.add(consumer.terminator())：当外部 lifetime 结束时，会调用 consumer 的 terminate()，终止这条订阅。
>
> - consumer.add_lifetime(std::move(_generator)(consumer))：真正发起订阅——调用 Generator，并把 Generator 返回的 lifetime 接到 consumer 上；这样上游的取消/清理会在 consumer 销毁时一起执行。
>
> 也就是说：订阅 = 用 consumer 调用 Generator，并把两边的 lifetime 绑在一起。

### 2.2 Consumer(消费者)

**consumer<Value, Error, Handlers>**

- 本质：对「next / error / done」三个回调的封装，用 std::shared_ptr\<Handlers\> 持有，便于拷贝、共享和与 lifetime 绑定。

- 接口（上游 Producer/Generator 调用）：

- put_next / put_next_copy：发下一个值。

- put_error / put_error_copy：发错误。

- put_done()：结束流。

- 终止语义：一旦调用了 put_error 或 put_done，内部会 terminate()，之后 put_next 不再生效，并执行 _lifetime.destroy()，从而取消这次订阅并跑完所有清理回调。

:large_orange_diamond:因此, Consumer 既是 **\[接收事件的接口\]**，也是**\[订阅生命周期\]**的载体（通过内部的 _lifetime）。

> [!note]
>
> **类型擦除机制**
>
> - Producer：默认或某些编译配置下使用 details::type_erased_generator<Value, Error>，即用 std::function<lifetime(const consumer<...>&)> 保存 Generator，这样不同 lambda/具体 Generator 类型可以统一成 producer<Value, Error>，便于存储、传参。
>
> - Consumer：在 GCC Debug 等配置下使用 type_erased_handlers（虚接口），用 put_next/put_error/put_done 的虚函数调用掩盖具体回调类型，避免模板爆炸。
>
> 这样既保留编译期类型安全（Value/Error），又能在需要时隐藏具体 Generator/Handlers 类型，减少编译量和 ABI 依赖。

### 2.3 Subscription / Lifetime（订阅管理）

- RPL 使用 **lifetime 对象管理订阅的生命周期**，防止内存泄露或者回调失效。
- 当 lifetime 结束时，生产者会自动停止通知消费者。

> ### lifetime
>
> - 实现：持有一个 std::vector\<base::unique_function<void()>\>callbacks。
>
> - 作用：
>
> - add(callback)：注册在生命周期结束时执行的清理函数（例如取消订阅、释放资源）。
>
> - destroy()：按注册顺序的逆序依次执行所有 callback，然后清空；析构时也会调用 destroy()。
>
> - 用法：
>
> - 订阅时：要么传入一个 lifetime&（订阅绑定到该 lifetime），要么接收 start() 返回的 lifetime 并保存，否则订阅会立刻被析构掉。
>
> - Generator 返回的 lifetime 会通过 consumer.add_lifetime(...) 与 consumer 绑定，这样上游的清理会在 consumer 终止时一起执行。
>
> ```c++
> inline void lifetime::destroy() {
>   auto callbacks = details::take(_callbacks);
>   for (auto i = callbacks.rbegin(), e = callbacks.rend(); i != e; ++i) {
>   	(*i)();
>   }
> }
> ```

### 2.4 Transformations（变换操作）

RPL 提供常用的响应式操作，例如：

| 操作      | 作用                 |
| --------- | -------------------- |
| `map`     | 对流中的值执行转换   |
| `filter`  | 过滤掉不满足条件的值 |
| `combine` | 合并多个 producer    |
| `on_next` | 订阅最新值的处理     |

示例：

```c++
auto strings = std::move(ints) | rpl::map([](int value) {
    return QString::number(value * 2);
});
auto evenInts = std::move(ints) | rpl::filter([](int value) {
    return (value % 2 == 0);
});
```

:small_blue_diamond: 这些操作返回新的 producer，形成可链式组合的响应式拓扑。

### 2.5 Combine / Composition（组合）

RPL 支持组合多个 producer，例如：

```C++
auto combined = rpl::combine(countProducer, textProducer);

std::move(combined) | rpl::on_next([=](int count, const QString &text) {
    // 两个值的最新组合
});
```

:large_blue_diamond: 这可以把多个异步变化的 Producer 合并成一个复合流，从而以更高抽象来处理事件。

## 3 核心实现机制（大致设计思想）

虽然源码复杂且模板化较多，但核心机制可总结如下：

### 1. 推模型（Push-based Stream）

RPL 基于 **推送（push）模型**：

- 当一个 producer 发出新的值时，会通知所有订阅者。
- RPL 内部维护订阅列表和回调机制，负责触发回调。

### 2. Observable 构建

Producer 底层类似于一个 **Observable / Publisher**：

- 它封装具体事件源（UI 事件、数据变化、值存储等）。
- 订阅者通过 `on_next` 注册回调。
- lifetime 确保订阅在适当时机取消。

### 3. 变量 & 状态包装

RPL 提供 `rpl::variable` 这样的类型用于封装 **可变状态的响应式变量**，类似其他响应式框架的 BehaviorSubject：

```
rpl::variable<bool> _adaptiveForWide;
```

当变量值变化时，会自动触发与其关联的 Producer。

## 四、RPL 在 Telegram Desktop 中的实际作用

###  1. **UI 状态驱动**

许多界面元素使用 producer 来动态驱动文本、颜色等。例如：

```
auto titleProducer = tr::lng_settings_title();
```

这里 titleProducer 是一个 rpl::producer，意味着当语言变化时 UI 会自动更新文本。

### 2. **本地化响应式**

字符串、本地化标签等使用 producer 自动跟踪语言变化，避免手动刷新 UI。

### 3. **数据层事件传播**

数据层（data changes）使用 producer 推送更新事件，例如：

```c++
rpl::producer<PeerUpdate> realtimePeerUpdates(...)
```

这些事件流会被监听并驱动 UI、缓存和本地状态更新。

## 五、RPL 与其它响应式库比较

| 特性             | RPL                 | RxCpp / ReactiveX |
| ---------------- | ------------------- | ----------------- |
| 语言             | C++                 | 多语言            |
| 依赖             | 自实现，无外部依赖  | 通用库            |
| 目标             | 轻量嵌入式响应式    | 全面响应式        |
| 自动 memory 管理 | 是（with lifetime） | 需手动管理订阅    |
| 变换操作         | 提供基本流变换      | 支持更复杂操作符  |

:arrow_right: RPL 的设计在 Telegram Desktop 场景下更轻、更专注 UI 和数据变化驱动。

## 六、实现机制总结

| 机制                     | 作用                          |
| ------------------------ | ----------------------------- |
| Push-based streams       | producer 推送新值触发订阅     |
| Pipelines（管道式链接）  | 连续变换流、高阶组合          |
| Lifetime 管理            | 订阅生命周期控制              |
| Transform operators      | map / filter / combine 等变换 |
| Integration with UI/data | UI 自动订阅数据更新           |

