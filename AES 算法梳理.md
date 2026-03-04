# AES 算法梳理

[toc]

## **1 定义**

AES(Advanced Enctyption Standard) 本质上是：**一个分组密码算法(Block Cipher)。**

它只定义：

- 块大小：128 bit
- 密钥长度：128/192/256 bit
- 轮函数 (SubBytes、ShiftRows、MixColumns、AddRoundKey)

它**没有定义如何加密长数据流。**

## 2 什么是 AES-CTR？

AES-CTR 是：

> AES 的一种“工作模式（Mode of Operation）”

即：

```c++
AES + Counter Mode
```

所以：

```c++
AES ≠ AES-CTR
AES-CTR = AES + 计数器模式
```

## 3 AES 的本质原理（算法内部）

AES 属于：

```c++
SPN（Substitution–Permutation Network）
```

核心结构：

```mermaid
graph LR
A("明文块(128 bit)") --> B(轮函数 x 10/12/14)
B --> C(密文块)
```

**每一轮包括：**

1. SubBytes（S-Box）
2. ShiftRows
3. MixColumns
4. AddRoundKey

**特点：**

- 对称加密
- 不可逆混淆扩散
- 强抗差分、线性分析

**安全性：**

- AES-128/256 目前未被实用攻击破解
- 被认为是安全标准

## 4 AES 工作模式（Modes of Operation）

AES 是分组算法，只能加密 128bit 数据。

现实中数据通常很长，因此需要模式：

常见模式：

| 模式 |            是否安全            | 是否推荐 |
| :--: | :----------------------------: | :------: |
| ECB  |           :x:不安全            |   禁止   |
| CBC  |         :warning: 旧式         |  不推荐  |
| CTR  |    :white_check_mark: 安全     |   推荐   |
| GCM  | :white_check_mark: ​安全 + 认证 | 强烈推荐 |
| XTS  |            磁盘专用            |   推荐   |
| CCM  |           嵌入式常用           |   推荐   |

## 5 AES-CTR 原理详解

CTR = Counter Mode

**核心思想：**

> 不直接加密明文，而是加密计数器，然后与明文 XOR。

**流程：**

```c++
keystream_i = AES(key, nonce || counter_i)
cipher_i = plaintext_i XOR keystream_i
```

**本质：**

```c++
AES-CTR = 流密码
```

它把 AES 变成了“伪随机数生成器”。

## 6 AES-CTR 与“普通 AES”的区别

|       项目       | AES（算法）  |         AES-CTR         |
| :--------------: | :----------: | :---------------------: |
|       类型       |   分组密码   |  分组密码 + 计数器模式  |
| 是否能加密长数据 |   :x:不能    | :white_check_mark: 可以 |
|   是否像流密码   |     :x:      |   :white_check_mark:    |
|   是否支持并行   | 算法本身支持 |        完全支持         |
|   是否需要 IV    |    不涉及    |       需要 nonce        |
|    是否有认证    |     :x:      |           :x:           |

注意：

> AES-CTR 只提供加密，不提供完整性认证。

## 7 CTR 的优缺点分析

**优点**

1. 可以并行加密
2. 不需要填充（padding）
3. 性能高
4. 实现简单
5. 适合高吞吐场景

**致命缺点**

:heavy_exclamation_mark: ​**nonce 不能重复**

如果：

```c++
同一 key + 相同 nonce
```

攻击者可以：

```c++
C1 XOR C2 = P1 XOR P2
```

直接泄露信息。

这叫：

> two-time pad attack

### 7.1 避免重复 nonce 的主流方案

**方案 1  随机 nonce（推荐简单系统）**

生成：`nonce = SecureRandom(96 bit)`

优点：

- 实现简单
- 不需要状态存储

风险分析：

96bit 随机数碰撞概率：$2^{48}$ 次使用才接近生日攻击风险。

现实系统完全足够。

适用场景：

- Web 会话
- 短生命周期连接
- 单进程系统

**方案 2  单调递增计数器（强安全系统）**

设计：`nonce = global_counter++`

:heavy_exclamation_mark: ​**关键问题：**

**必须持久化**

否则：

- 程序重启
- counter 重置
- 产生重复 nonce

**正确做法：**

- 将 counter 存入磁盘
- 或使用数据库自增
- 或使用原子写入文件

**适用场景：**

- 文件加密
- 长期运行服务
- 分布式系统

**方案 3：分段结构（工业级常用）**

推荐结构：`nonce = machine_id || timestamp || local_counter`

例如：

- 32 bit machine id
- 32 bit timestamp
- 32 bit counter

**优点：**

- 多机器不冲突
- 重启安全
- 分布式可用

这是大规模系统常用方案。

**方案 4：每次加密使用新 key（终极安全）**

做法：`session_key = HKDF(master_key, unique_id)`

即：

- 主密钥固定
- 每个文件 / 消息派生子密钥
- nonce 从 0 开始

这样：**key 不同 $\rightarrow$ nonce 可以重复**

**优点：**

- 不需要全局 nonce 管理
- 简洁安全

很多现代协议使用这种方法。

### 7.2 工程中最推荐的做法

:one: **文件存储系统**

**推荐：**

- 每个文件生成随机 file_id
- key = HKDF(master_key, file_id)
- nonce 从 0 开始

**优点：**

- 永远不会重复
- 不需要全局状态
- 支持并行

:two: **网络通信（TLS 类场景）**

像 AES-GCM 在 TLS 中：`nonce = fixed_iv XOR sequence_number` 

sequence_number 是单调递增的。

安全保证：

- 每个包唯一
- 不需要随机数

### 7.3 绝对不要做的事情

:x: ​使用时间戳作为唯一 nonce（时间精度不够会重复）

:x: ​重启后计数器归零

:x: 多线程共享 nonce 但未加锁

:x: ​多进程使用同一 key 却没有协调机制

### 7.4 2026 年现代建议

其实今天更推荐：

:x: ​不要自己管理 CTR nonce
 :white_check_mark: 使用 AEAD 模式

推荐：

- AES-GCM
- ChaCha20-Poly1305

原因：

- 内置认证
- nonce 管理规范
- 库封装成熟
- 更不容易犯错

> [!important]
>
> AEDA(Authenticated Encryption with Associated Data) 带关联数据的认证加密
>
> 它是一种 **现代对称加密设计范式**，核心目标是：
>
> 同时保证：
> :one: ​机密性（Confidentiality）
> :two: ​完整性（Integrity）
> :three:认证性（Authentication）
>
> **主流 AEAD 算法**
>
> :one: **​AES-GCM**
>
> 结构：AES-CTR + GHASH
>
> **特点：**
>
> - 非常快（有硬件加速）
> - TLS 默认算法
> - 广泛使用
>
> :two:**​ ChaCha20-Poly1305**
>
> 结构：ChaCha20 加密 + Poly1305 认证
>
> 特点：
>
> - 不依赖硬件
> - 在 ARM / 手机上更快
> - TLS 1.3 标准之一
>
> :three: **AES-CCM**
>
> 常用于：
>
> - 嵌入式
> - 蓝牙
> - IoT
>
> **AEAD 的本质思想**
>
> **不允许开发者分开处理“加密”和“认证”，必须作为一个不可分割的整体。**这是现代密码工程的核心原则。

## 8 AES-CBC vs AES-CTR

`AES-CBC`

**CBC 特点：**

- 串行
- 需要 padding
- 易受 padding oracle 攻击

**优点：**

- 实现简单
- 旧系统广泛使用

**缺点：**

- 不可并行
- 安全性依赖实现
- 易被侧信道攻击

## 9 现代主流：AES-GCM

`AES-GCM`

`GCM = CTR + GHASH`

它不仅加密，还提供：

- 完整性校验（认证）
- AEAD（Authenticated Encryption with Associated Data）

优点：

- 并行
- 有认证
- 硬件支持
- TLS 标准

目前：

- HTTPS
- VPN
- TLS 1.2+
- QUIC（快速 UDP 互联网连接）

都默认使用 AES-GCM。

## 10 AES-XTS（磁盘专用）

`AES-XTS`

专为：

- 磁盘加密
- 文件系统

特点：

- 防止块重排攻击
- 双密钥

## 11 AES 系列当前现状（2026 视角）

:one: **AES 算法本身**

- 依然安全
- 无实用攻击
- 被 NIST 认证

:two: **​AES-128 vs AES-256**

| 版本 | 安全性 | 性能 |
| :--: | :----: | :--: |
| 128  |  足够  |  快  |
| 256  |  更强  | 略慢 |

现实建议：

- 普通应用：`AES-128`
- 高安全场景：`AES-256`

:three: ​硬件加速

现代 CPU 都支持：

- AES-NI（x86）
- ARMv8 Crypto Extension

AES 在硬件支持下：

> 比 ChaCha20 还快（x86 上）

## 12 AES vs ChaCha20

`ChaCha20`

|   项目   |   AES-GCM   | ChaCha20-Poly1305 |
| :------: | :---------: | :---------------: |
| 硬件依赖 | 需要 AES-NI |      不需要       |
| ARM 性能 |    较慢     |       很快        |
| 软件实现 |    复杂     |       简单        |
|   TLS    |    主流     |       主流        |

> [!note]
>
> Google 在移动端更偏向：ChaCha20-Poly1305

## 13 安全推荐（现代建议）

:x:**​不建议：**

- AES-ECB
- AES-CBC（无认证）
- AES-CTR（无认证）

:white_check_mark: **​推荐：**

- AES-GCM
- ChaCha20-Poly1305

核心原因：

> 现代密码学强调 AEAD（加密 + 认证）

**总结**

> AES 是核心分组算法，CTR 是其工作模式；AES-CTR 具有高性能和可并行优势，但缺乏认证机制，现代系统更推荐使用 AES-GCM 或 ChaCha20-Poly1305 等 AEAD 方案。