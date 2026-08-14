# Stormpot 全景技术架构与组件设计深度白皮书

## 摘要 (Abstract)

Stormpot 是 Java 生态中极致优化的无锁/低竞争对象池（High-Performance Lock-Free Object Pool）实现。其核心目标是在极高的并发借用（`claim`）与归还（`release`）场景下提供极低且可预测的延迟（Predictable Tail Latency）。本文档作为 Stormpot 的全景技术架构白皮书，全面剖析其分层设计、核心组件、四状态 CAS 状态机、Wait-Free 归集栈 `RefillPile`、缓存行填充（Cache Line Padding）、虚拟线程（Project Loom）的 128-Stripe 适配架构以及后台异步解耦生命周期管理机制。

---

## 1. 全景技术架构分层 (System Architecture Layers)

Stormpot 采用**分层解耦、快慢路径分离、生产者-消费者异步化**的设计架构。系统划分为 4 个核心层级：

```
+-----------------------------------------------------------------------------------+
|                            1. 业务接入层 (Public API Layer)                       |
|   Pool<T> / PoolTap<T>      PoolBuilder<T>        Poolable / BasePoolable        |
|  (池管理与借还入口)         (配置与建造者)         (池化资源包装接口)              |
+-----------------------------------------------------------------------------------+
                                      │
                                      ▼
+-----------------------------------------------------------------------------------+
|                        2. 接入适配与水龙头层 (Tap Adaptation Layer)               |
|  ThreadSafeTap              SingleThreadedTap          VirtualThreadSafeTap       |
| (平台线程 ThreadLocal)       (单线程无竞争)            (虚拟线程 128-Stripe)      |
+-----------------------------------------------------------------------------------+
                                      │
                                      ▼
+-----------------------------------------------------------------------------------+
|                        3. 核心并发调度引擎 (Core Engine Layer)                    |
|                                 BlazePool<T>                                      |
|   ┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐   |
|   │ 4-State BSlot       │    │ Wait-Free Stack     │    │ Global Lock-Free Q  │   |
|   │ (CAS 状态机与钥匙盒) │    │ (RefillPile 归集栈) │    │ LinkedTransferQueue │   |
|   └─────────────────────┘    └─────────────────────┘    └─────────────────────┘   |
+-----------------------------------------------------------------------------------+
                                      │
                                      ▼
+-----------------------------------------------------------------------------------+
|                        4. 异步生命周期管理层 (Background Lifecycle Layer)         |
|  BAllocThread (后台分配守护线程)   Allocator<T> (物理创建/销毁)   Expiration<T> (失效策略) |
+-----------------------------------------------------------------------------------+
```

---

## 2. 核心组件剖析与比喻映射 (Component Deep-Dive)

| 核心组件 | 生活化比喻 | 技术职责与作用含义 |
| :--- | :--- | :--- |
| **`Pool<T>`** | **共享汽车公司总部** | 对象池核心主接口，负责对外暴露 `claim()`、`tryClaim()`、`setTargetSize()`、`shutdown()` 等全局调度与管理能力。 |
| **`PoolTap<T>`** | **无人取车闸机** | `Pool` 的轻量级只读访问 Facade。只保留借用接口，隔离池容量动态修改等管理方法。 |
| **`Poolable` / `BasePoolable`** | **带有智能车匙的汽车** | 被池管理的资源包装接口，内置归还（`release()`）与失效（`expire()`）逻辑。 |
| **`Slot` / `BSlot<T>`** | **智能钥匙盒** | 槽位节点，内部维护基于 `VarHandle` 的 4 状态（`LIVING`, `CLAIMED`, `TLR_CLAIMED`, `DEAD`）原子状态机。 |
| **`BSlotPadded<T>`** | **带隔音墙的独立车位** | 继承 `BSlot`，填入 72 字节 Padding 变量，使 `state` 独占 CPU 64 字节 Cache Line，彻底消除伪共享。 |
| **`BlazePool<T>`** | **AI 调度中央引擎** | 核心池实现，协调 ThreadLocal 快滑道、全局无锁队列（`LinkedTransferQueue`）与后台分配线程。 |
| **`RefillPile<T>`** | **极速投递箱** | 基于 `getAndSet` 硬件原语实现的 Wait-Free 栈，用于并发尖峰时的槽位归集与批量转运。 |
| **`BlazePoolThreadSafeTap`** | **私人专属车位** | 平台线程默认的 Tap，维护 `ThreadLocal<BSlotCache>`，实现零锁零竞争快滑道借还。 |
| **`BlazePoolVirtualThreadSafeTap`**| **128 通道智能闸机** | 为 Java 21+ 虚拟线程设计的 Tap，改用 128-Stripe 固定数组，将内存控制在 $O(1)$ 且无 `synchronized` 锁。 |
| **`BAllocThread<T>`** | **后勤维修车队** | 后台专属守护线程，解耦物理对象的创建（`allocate`）、重构与销毁（`deallocate`）。 |
| **`Allocator<T>`** | **汽车制造厂/拆解厂** | 用户定义的物理资源创建与销毁接口。 |
| **`Expiration<T>`** | **车辆报废标准** | 判定对象生命周期过期的规则接口（如按时间、离散时间等）。 |
| **`PreciseLeakDetector`** | **GPS 防盗追踪器** | 基于虚引用（`PhantomReference`）监控未显式 `release()` 即被 GC 的泄漏槽位并报警。 |

---

## 3. 底层并发与内存控制机制

### 3.1 4 状态原子状态机 (`BSlot`)

[`BSlot.java`](file:///d:/yanyun/stormpot/src/main/java/stormpot/internal/BSlot.java#L37-L40) 定义了 4 种精细的状态：

```java
private static final int CLAIMED     = 1; // 已被正常借出（来自全局队列）
private static final int TLR_CLAIMED = 2; // 已被特定线程通过 ThreadLocal 快滑道借出
private static final int LIVING      = 3; // 闲置中，可供借用
private static final int DEAD        = 4; // 已废弃/失效，等待后台分配线程重构
```

#### 内存序优化与转移控制：
- **快滑道 CAS 抢占**：`STATE.compareAndSet(this, LIVING, TLR_CLAIMED)`。
- **慢速路径 CAS 抢占**：`STATE.compareAndSet(this, LIVING, CLAIMED)`。
- **宽松写归还 (Opaque Store)**：`STATE.setOpaque(this, LIVING)`。仅发出 Store-Store 屏障，避免昂贵的 Store-Load 全屏障（`LOCK` 前缀），极大降低写入开销。
- **单次销毁原则**：只有成功将状态转换为 `CLAIMED`（即从全局队列拉取的线程）才有资格执行 `claim2dead()` 并提交至 `dead` 队列，彻底杜绝重复入队与内存泄漏。

### 3.2 `RefillPile` 的 Wait-Free 算法证明

[`RefillPile.java`](file:///d:/yanyun/stormpot/src/main/java/stormpot/internal/RefillPile.java#L58-L61) 实现了基于 `getAndSet` 的无等待压栈：

```java
public void push(BSlot<T> slot) {
  RefillSlot<T> element = new RefillSlot<>(slot);
  element.next = getAndSet(element); // 硬件级原子交换指令
}
```

- **Wait-Free 证明**：在 x86 架构下映射为单条 `XCHG` 原语。任何线程执行 `push` 仅包含精准的 1 次硬件原语 + 1 次指针赋值，步骤数恒定（Bounded Steps），无 `while(CAS)` 死循环重试，符合 Wait-Free 严格定义。
- **批量转运 (`refill`)**：通过 `getAndSet(STACK_END)` 1 步切断整条链表，将分散的并发归还转化为单线程批量推回主队列，消除了主队列尾节点的原子冲突。

---

## 4. 虚拟线程（Project Loom / Java 21+）深度适配架构

虚拟线程场景下，传统对象池面临 **ThreadLocal 内存爆炸** 与 **Carrier Thread Pinning 锁死** 两大痛点。Stormpot 的应对方案如下：

### 4.1 128-Stripe 带状数组（消除 ThreadLocal）

在 [`BlazePoolVirtualThreadSafeTap.java`](file:///d:/yanyun/stormpot/src/main/java/stormpot/internal/BlazePoolVirtualThreadSafeTap.java) 中：
- 废弃 `ThreadLocal`，固定分配 `BSlotCache[128]` 数组。
- 基于 `VirtualThread.threadId()` 进行 4-Stripe 哈希探测。
- 将空间复杂度从 $O(N_{\text{virtual\_threads}})$ 降至 **$O(1)$ 常数级**。

### 4.2 零 `synchronized` 与 `LockSupport` 卸载（防止 Pinning）

- **零 `synchronized`**：核心路径全采用 `VarHandle` CAS 与 Opaque Store，绝对不包含 `synchronized` 关键字。
- **Unmount 自动卸载**：慢速路径阻塞等待采用 `LinkedTransferQueue.poll()` 和 `LockSupport.parkNanos()`。在 Java 21 中，虚拟线程陷入 `LockSupport.park` 时会自动保存堆栈并从 Carrier Thread 上 **Unmount（卸载）**，释放 Carrier Thread 去处理其他任务，**彻底消除 Carrier Thread Pinning**。

---

## 5. 对象全生命周期流转序时图 (End-to-End Sequence)

```
[业务线程 claim()] 
       │
       ▼
 [PoolTap 挑选模式]
       ├─► 平台线程 ────► [ThreadLocal Fast-Path] ──(命中)──► [读取 BSlot (LIVING)] ──► 变色为 TLR_CLAIMED ──► 返回对象!
       └─► 虚拟线程 ────► [128-Stripe Probe]      ──(未命中)─┐
                                                            │
                                                            ▼
                                                [退化至 Slow-Path 慢速路径]
                                                            │
                                                            ├─► 从 [RefillPile 投递箱] pop 槽位
                                                            └─► 从 [LinkedTransferQueue 主库] poll 槽位
                                                                        │
                                                                 (发现对象已过期 Expiration)
                                                                        │
                                                                        ▼
                                                            [BSlot 变色为 DEAD]
                                                                        │
                                                                        ▼
                                                            [推入 tasks 异步队列]
                                                                        │
                                                                        ▼
                                                            [BAllocThread 后台线程唤醒]
                                                                        ├─► Allocator.deallocate(旧对象)
                                                                        └─► Allocator.allocate(新对象)
                                                                        │
                                                                        ▼
                                                            [Wait-Free 压回 newAllocations 栈]
                                                                        │
                                                                        ▼
                                                            [业务线程成功拿到新对象!]
```

---

## 6. 总结 (Conclusion)

Stormpot 通过极具前瞻性的架构设计，将理论上的无锁（Lock-Free）与无等待（Wait-Free）算法成功落实到了工程实践中。其优雅的分层、对 CPU 缓存行的微观控制、生产者-消费者解耦以及对虚拟线程（Loom）的原生适配，使其成为 Java 高并发对象池领域的工业级典范。
