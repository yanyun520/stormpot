# Stormpot 对象池底层运行原理 —— 通俗解读与 Demo 演示

> 本文通过一个**数据库连接池**的 Demo，结合**生活类比**、**源码级执行步骤**与 **ASCII 流程图**，带你彻底搞懂 Stormpot 底层到底在做什么、各个线程之间如何协作。

---

## 1. 先建立直觉：对象池 = 共享单车停放点

在深入代码之前，先记住一个生活模型，后面的所有机制都能对号入座：

| Stormpot 概念 | 生活类比 | 一句话解释 |
| --- | --- | --- |
| `Pool`（池） | 共享单车停放点 | 一个存放"现成对象"的集合 |
| `BSlot`（槽） | 一个停车位 + 一辆车 | 池的最小管理单元：车位状态 + 车（对象） |
| `claim()`（借用） | 扫码取车 | 从池里取走一辆车（独占使用） |
| `release()`（归还） | 还车 | 把车放回池里（其他线程可再借） |
| `Allocator`（分配器） | 造车厂 | 负责造新车 / 报废旧车 |
| `BAllocThread`（后台线程） | 调度员 | 在后台不停地"补车、修车、换车"，不让取车的人等太久 |
| 线程本地缓存（TLR） | 自家门口的车位 | 每个用户把常用的一辆车停在自己家门口，取还都快 |
| `Expiration`（过期） | 车辆保养期限 | 车太旧了就得送回厂里换新的 |
| `poison`（毒化） | 故障车 | 分配失败/被显式过期的对象，标记后回收 |

**核心理念一句话：把"创建对象的昂贵开销"转移到后台线程，让业务线程永远快速取用。**

---

## 2. Demo：一个 5 线程并发借还数据库连接的场景

```java
import stormpot.*;
import java.util.concurrent.TimeUnit;

// ---------- ① 池化对象：一个数据库连接 ----------
class DbConnection extends BasePoolable {
    final String name;
    DbConnection(Slot slot, String name) {
        super(slot);
        this.name = name;
        System.out.println("[造车厂] 制造新连接: " + name);
    }
    void query(String sql) {
        System.out.println("[" + Thread.currentThread().getName()
            + "] 用连接 " + name + " 执行: " + sql);
    }
}

// ---------- ② 分配器：造车厂（创建/销毁连接） ----------
class DbConnectionAllocator implements Allocator<DbConnection> {
    private int seq; // 给连接编号，便于观察
    @Override
    public DbConnection allocate(Slot slot) throws Exception {
        return new DbConnection(slot, "conn-" + (++seq));
    }
    @Override
    public void deallocate(DbConnection obj) {
        System.out.println("[报废厂] 销毁连接: " + obj.name);
    }
}

// ---------- ③ 主程序 ----------
public class Demo {
    public static void main(String[] args) throws Exception {
        // 创建"停放点"：后台线程模式，容量 4 辆，连接 2 秒过期
        Pool<DbConnection> pool = Pool.fromThreaded(new DbConnectionAllocator())
                .size(4)
                .expiration(Expiration.after(2, TimeUnit.SECONDS))
                .build();

        // 5 个业务线程同时"借车-用-还车"
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                try {
                    for (int round = 0; round < 3; round++) {
                        DbConnection conn = pool.claim(new Timeout(1, TimeUnit.SECONDS));
                        if (conn == null) {
                            System.out.println("[" + Thread.currentThread().getName()
                                + "] 等 1 秒没借到车，放弃");
                        } else {
                            conn.query("SELECT * FROM users");   // 业务使用
                            conn.release();                       // 归还
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }, "顾客线程-" + i).start();
        }

        Thread.sleep(10000);          // 让业务跑一会儿
        pool.shutdown().await();      // 关停：把所有连接销毁
        System.out.println("池已关闭，演示结束");
    }
}
```

> 池容量是 4，但顾客线程有 5 个——这就制造了"池耗尽"的场景，逼出慢路径和等待逻辑。下面逐步拆解这个程序运行时，底层到底发生了什么。

---

## 3. 阶段一：`build()` —— 池的"开业筹备"

调用 `Pool.fromThreaded(allocator).size(4).build()` 时，幕后发生：

```
用户线程
   │
   ▼
PoolBuilderImpl.build()
   │  ① 校验配置（size=4、过期策略、线程工厂…）
   │  ② AllocationProcess.threaded()  →  ThreadedAllocationController
   │  ③ new BlazePool(builder, factory)
   │        ├── live          = new LinkedTransferQueue<>()   ← 主停车场（就绪队列）
   │        ├── newAllocations= new RefillPile<>(live)        ← 新车待上架区（无锁栈）
   │        ├── disregardPile = new RefillPile<>(live)        ← 故障车暂存区（无锁栈）
   │        ├── tlr           = new ThreadLocalBSlotCache<>() ← 每线程一个"家门口车位"
   │        └── poisonPill    = 一个特殊 BSlot（标记 SHUTDOWN_POISON） ← 关停信号弹
   │  ④ 启动后台线程 BAllocThread（调度员上线）
   ▼
池对外可用了，但此时池里还是空的！
```

**此时系统里有 2 类线程：**

```
┌─────────────────┐          ┌──────────────────────────────┐
│  用户业务线程 ×N  │          │  BAllocThread（唯一的调度员）  │
│  只管 claim/use/ │          │  run():                       │
│  release         │          │    while (!shutdown):         │
│                  │          │      replenishPool();  ← 死循环│
└─────────────────┘          └──────────────────────────────┘
```

---

## 4. 阶段二：后台线程"预填充"——池是怎么被装满的

`BAllocThread` 上线后立刻进入 `replenishPool()` 循环，第一次循环就发现 **size(0) < targetSize(4)**，于是开始造车：

```
BAllocThread 第 1 次循环
   │
   ├── ① 从任务队列取任务（tasks.poll()）→ 空
   │
   ├── ② size(0) < targetSize(4) ? 是！
   │       └── increaseSizeByAllocating()
   │              ├── 新建一个 BSlot（状态 DEAD，相当于"空车位"）
   │              ├── size++（0 → 1）
   │              └── allocSingle(slot)
   │                     ├── slot.allocator = 适配后的分配器
   │                     ├── 优先走异步: allocator.allocateAsync(slot)
   │                     │        └── 造车完成 → 回调 onCompleteAsyncAllocation
   │                     │              └── publishSlot()
   │                     │                    ├── resetSlot(): createdNanos=now; 状态 DEAD→LIVING
   │                     │                    └── live 没有等待者 → newAllocations.push(slot) ← 新车放"待上架区"
   │                     └── （若分配失败）slot.poison = 异常 → live.offer(slot)（故障车直接丢停车场）
   │
   └── ③ 回到循环顶部，size(1) < 4，继续造…直到 size == 4
```

**注意这个细节：** 新造好的车不是直接进 `live` 队列，而是先压进 `newAllocations` 这个无锁栈。为什么？因为 **claim 线程会先来这个栈里取（后进先出，最新鲜的车），避免了和"归还的车"争抢同一个队列**。

4 辆车造好后，池的布局：

```
                      ┌──────────────────────────────────────────────┐
                      │                BAllocThread                   │
                      │   造车 → newAllocations.push()  循环等待任务    │
                      └──────────────┬───────────────────────────────┘
                                     │
      newAllocations(无锁栈,LIFO)     │          live(队列,FIFO)
      ┌───────────────┐             │         ┌─────────────────┐
      │ conn-4 (LIVING)│            │         │   (空)           │
      │ conn-3 (LIVING)│            │         │                 │
      │ conn-2 (LIVING)│            │         │                 │
      │ conn-1 (LIVING)│            │         │                 │
      └───────┬───────┘            │         └─────────────────┘
              │                     │
      disregardPile(故障暂存,空)      │
      ┌─────────────────┐           │
      │   (空)          │           │
      └─────────────────┘           │
                                     │
              每线程的 TLR 家门口车位（都是空的）
```

---

## 5. 阶段三：`claim()` —— 借车的完整决策树

某个顾客线程调用 `pool.claim(timeout)`，内部流程（结合 `BlazePool.claim` 源码）：

```
claim(Timeout)
   │
   ├── ① checkShutDown()         池已关停？→ 抛 IllegalStateException
   │
   ├── ② 快路径 tlrClaim(cache)   先看自家门口有没有车
   │       ├── cache.slot 不为空 且  CAS(LIVING→TLR_CLAIMED) 成功？
   │       │      ├── 是 → isValid() 检查（毒化? 过期?）
   │       │      │        ├── 有效 → return obj          ← ★ 无锁！无队列！最快！
   │       │      │        └── 无效 → kill()：TLR_CLAIMED→LIVING，清空缓存
   │       │      └── 否（可能被别的线程发现过期杀掉了）→ 走慢路径
   │       └── 缓存为空 → 走慢路径
   │
   └── ③ 慢路径 slowClaim(timeout, cache)
           └── 循环：
                 ├── newAllocations.pop()      ← 先抢"新车上架区"
                 │      ├── 有 → live2claim() CAS(LIVING→CLAIMED)？
                 │      │        ├── 成功 → isValid()？ → 有效 → cache.slot=slot; return obj
                 │      │        └── 失败 → 塞回 disregardPile
                 │      └── 没有 → live.poll(剩余超时)  ← 在停车场上等（可中断）
                 │              ├── 等到 → 同上判断
                 │              └── 没等到 → disregardPile.refill() 把故障暂存区的车倒回停车场
                 │                       → 超时 → return null
                 └── 循环直到超时
```

**快路径 vs 慢路径的 ASCII 对比：**

```
★ 快路径（TLR 命中）：            ★ 慢路径（TLR 未命中）：
  线程T1                         线程T2
    │                              │
    │ 家门口有车？                  │ 家门口没车
    │  CAS 改状态（1次原子操作）      │ 从 newAllocations 栈取车
    │  检查有效？                    │ 或从 live 队列取车
    │  返回对象                      │ 或阻塞等待后台线程造车
    │  ▼ 无锁，纳秒级                │ ▼ 涉及队列，微秒级
```

**为什么慢路径要先取 `newAllocations` 而不是 `live`？** 因为 `newAllocations` 是无锁栈（`getAndSet` 入栈、CAS 出栈），比 `LinkedTransferQueue` 的并发入队出队更省；而且新车不会过期，检查成本最低。

---

## 6. 阶段四：5 个线程抢 4 辆车——线程交互实录

按时间线演示 `Demo` 中 5 个顾客线程的行为（假设 BAllocThread 已造好 4 辆车）：

```
时间 ──────────────────────────────────────────────────────────────►

[顾客线程-0] claim ──► tlrClaim: 家门口空 → slowClaim: newAllocations.pop()
                       → 拿到 conn-4 → cache.slot=conn-4 → 执行 SQL → release
                                                                       │
[顾客线程-1] claim ──► tlrClaim: 空 → slowClaim: newAllocations.pop()
                       → 拿到 conn-3 → … → release                     │
                                                                       │
[顾客线程-2] claim ──► 拿到 conn-2 → … → release                        │
                                                                       │
[顾客线程-3] claim ──► 拿到 conn-1 → … → release                        │
                                                                       │
[顾客线程-4] claim ──► tlrClaim: 空
                       → newAllocations.pop(): 空！（4 辆全被借走）
                       → live.poll(1秒): 阻塞等待 ←┐
                                                  │ 此时池中：
                       ┌──────────────────────────┴───────────────┐
                       │ live:      (空)                          │
                       │ newAllocations: (空)                      │
                       │ 4 辆车全部 CLAIMED（被线程0~3 占用）        │
                       │ BAllocThread: 发现 size==targetSize(4)    │
                       │   → 没有造车任务，循环空转等任务            │
                       └──────────────────────────────────────────┘

[顾客线程-0] release ──► BSlot.release():
                          ├── 状态 CLAIMED → LIVING（lazySet，无锁）
                          └── live.offer(conn-4)   ← 车回到停车场

[顾客线程-4] live.poll 等到 conn-4
                       ├── live2claim() CAS 成功
                       ├── isValid: 未过期 → 有效！
                       ├── cache.slot = conn-4   ← 从此 conn-4 是它家门口的车
                       └── 执行 SQL → release
```

**关键线程交互点：**

| 交互 | 谁→谁 | 机制 | 源码位置 |
| --- | --- | --- | --- |
| 取车 | 业务线程 → 池 | `live2claim()` CAS 状态转换 | `BSlot.java` |
| 还车 | 业务线程 → 池 | `lazySet(LIVING)` + `live.offer()` | `BSlot.release()` |
| 等车 | 业务线程 → 池 | `live.poll(timeout)` 阻塞（可中断） | `BlazePool.slowClaim` |
| 造车补给 | 调度员 → 池 | `publishSlot()` → `newAllocations.push` 或 `live.offer` | `BAllocThread.allocSingle` |
| 故障回收 | 业务线程 → 调度员 | `allocator.offerDeadSlot(slot)` → 调度员任务队列 | `BlazePool.kill` |

---

## 7. 阶段五：`release()` —— 还车的内部细节

`conn.release()` 最终调用 `BSlot.release()`，注意它做了两件"看不见"的事：

```java
public void release(Poolable obj) {
    if (poison == EXPLICIT_EXPIRE_POISON) {   // ① 若被显式 expire 过
        poisonedSlots.getAndIncrement();      //    记一笔"毒化槽"账
    }
    int slotState = getClaimState();          // ② 读当前状态
    lazySet(LIVING);                          // ③ 状态 → LIVING（无锁写）
    if (slotState == CLAIMED) {               // ④ 只有"正式借出"状态才回队列
        live.offer(this);                     //    （TLR_CLAIMED 的车回"家门口"，不进队列）
    }
}
```

**为什么 TLR 状态的车 release 时**不**回 live 队列？** 因为那辆车本来就在该线程"家门口"（TLR 缓存里），release 后直接留在原地，下次 claim 立刻能用——这就是快路径的秘密：**同一个线程反复借还同一辆车，永远不碰共享队列**。

**错误用法检测也在这：** 如果对一辆 LIVING 状态的车重复 release（`state > TLR_CLAIMED`），直接抛 `PoolException("Slot release from bad state... You most likely called release() twice")`。

---

## 8. 阶段六：过期与回收 —— 调度员的一天

Demo 中连接 2 秒过期（`Expiration.after(2, SECONDS)`）。对象过期有**两条检查路径**：

### 路径 A：借出时顺检（业务线程执行）

```
顾客线程 claim 到车
   └── isValid(slot)
          ├── slot.poison != null ? → 毒化，kill()
          └── deallocRule.hasExpired(slot)?
                 ├── 否 → 有效，放行
                 └── 是 → kill(slot, cache)
                             ├── 状态 CLAIMED → claim2dead()
                             └── allocator.offerDeadSlot(slot)
                                    │
                                    ▼
                          BAllocThread 任务队列
                            收到一个 BSlot 任务
                            └── reallocateDeadSlot() → realloc()
                                   ├── 老对象 deallocate（报废）
                                   ├── 新对象 allocate（再造一辆）
                                   └── publishSlot → 回到池中
```

### 路径 B：后台巡检（BAllocThread 执行，默认开启）

```
BAllocThread 循环空闲时
   └── backgroundCheck()
          ├── disregardPile.refill()  ← 先把暂存区的车倒回停车场
          ├── live.poll() 取一辆车
          ├── 状态 LIVING → live2claim() CAS 抢到手（假装借出）
          ├── expiration.hasExpired(slot) ?
          │      ├── 是 → claim2dead() → tasks.offer(slot) → 下轮 realloc
          │      └── 否 → claim2live() → live.offer(slot) 放回去
          └── 注意：若车是 CLAIMED（正被使用），检查会跳过
              （live 队列里不会有 CLAIMED 的车，所以天然安全）
```

**`TimeSpreadExpiration` 的妙处（防惊群）：** 4 辆车如果都在"第 2 秒整"过期，就会同时报废、同时重建，造成 CPU 尖峰。Stormpot 用 `after(2, 2, SECONDS)` 风格的散布策略时，每辆车在 `[下界, 上界]` 内**随机**选一个过期时刻存进 `BSlot.stamp`，把过期间隔均匀铺开：

```
不用散布策略（惊群）：             用散布策略（均匀）：
  过过期过期过期                      过   期   过   期
  │   │   │   │                      │   │   │   │
  2s  2s  2s  2s                      1.9s 2.3s 2.6s 3.0s
  （4 辆同时报废+重建）              （报废重建分散开，负载平稳）
```

### 失败补偿：分配失败的退避

调度员每次分配失败都会 `consecutiveAllocationFailures++`，失败越多，任务轮询超时越长（`computeTaskPollTimeout` 里的退避公式），**避免在分配器持续故障时疯狂重试烧 CPU**；一旦恢复成功，计数清零。

---

## 9. 阶段七：`shutdown()` —— 关停大扫除

调用 `pool.shutdown()` 的完整时序：

```
顾客线程（调用方）                        BAllocThread                   其他业务线程
      │                                       │                              │
      │ pool.shutdown()                       │                              │
      │   ├── shutdown标志=true               │                              │
      │   └── allocator.shutdown()           │                              │
      │        └── 通知调度员退出循环          │                              │
      │                                       │                              │
      │                                       │ while(!shutdown) 退出        │
      │                                       │   ├── poisonPill.dead2live() │
      │                                       │   └── live.offer(poisonPill) │
      │                                       │        │                     │
      │                                       │        │  ← 信号弹进入停车场  │
      │                                       │        ▼                     │
      │                                       │ shutPoolDown()               │
      │                                       │   while(size>0):             │
      │                                       │     ├── live.poll() 取车      │
      │                                       │     ├── 状态→DEAD             │
      │                                       │     └── dealloc() 销毁        │
      │                                       │   shutdownCompletion.complete()│
      │  shutdown().await() 唤醒 ◄────────────┴───────────────────────────────┘
      │
      │ 与此同时，其他线程若 claim：
      │    ├── checkShutDown() → 抛 IllegalStateException
      │    └── 若已进入慢路径阻塞等待，会碰到 poisonPill：
      │         ├── 拿到 pill → live2claimTlr() 成功 → isValid()
      │         ├── poison == SHUTDOWN_POISON
      │         └── claim2live() 放回队列 → 抛 IllegalStateException
      └── 所有对象销毁完毕，await() 返回，程序干净退出
```

**poisonPill（信号弹）为什么不用简单的 shutdown 标志？** 因为可能有一个线程正阻塞在 `live.poll(超时)` 上等车——只设标志它不知道。把"毒药丸"投进队列，等待者一旦拿到它就知道"池关了"，从阻塞中立刻醒来并抛异常。**这是用队列本身传递关停信号，避免漏唤醒。**

---

## 10. 线程职责总览：谁在什么线程上做什么

| 动作 | 执行线程 | 是否阻塞/耗时 | 说明 |
| --- | --- | --- | --- |
| 快速取还车（TLR 命中） | 业务线程 | 无锁，纳秒级 | `CAS` + 有效性检查 |
| 从队列取车/还车 | 业务线程 | 轻微队列争用 | `LinkedTransferQueue` |
| 等车（池空） | 业务线程 | 阻塞，可超时/可中断 | `live.poll(timeout)` |
| **创建/销毁对象** | **仅 BAllocThread**（或并行分配线程池） | 可能很慢 | 这就是"把开销移出业务线程" |
| 过期巡检 | BAllocThread | 后台空闲时 | 每秒最多一次（可配） |
| 换分配器（`switchAllocator`） | BAllocThread | 逐代替换 | 见下节 |
| 泄漏检测计数 | BAllocThread | 后台 | `PreciseLeakDetector` |
| 关停大扫除 | BAllocThread | 阻塞直到清空 | 完成后通知 `await()` 者 |

**三种模式对比（同一个 BlazePool 内核，不同控制器）：**

```
THREADED（Demo 用的）          INLINE（无后台线程）         DIRECT（预分配）
┌───────────────┐             ┌───────────────┐           ┌───────────────┐
│ 业务线程        │             │ 业务线程        │           │ 业务线程        │
│ claim/release │             │ claim 时顺带   │           │ 直接取用       │
└───────┬───────┘             │ 造车/报废/巡检 │           └───────┬───────┘
        │                     └───────┬───────┘                   │
┌───────▼───────┐                     │                           │
│ BAllocThread  │                     │                           │
│ 后台造车/回收  │                     │                           │
└───────────────┘                     │                           │
                                      │  没有线程，也没有队列，     │
                                      │  全在 claim 调用栈里同步做  │
                                      │                           │
                                      │                           │
┌─────────────────────────────────────────────────────────────────┐
│ 三种模式共享：BlazePool + BSlot 状态机 + TLR + RefillPile        │
│ 区别只在 AllocationController 的实现                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. 进阶彩蛋：热切换分配器（`switchAllocator`）

这是 Stormpot 4.x 的招牌功能——**运行中更换造车厂，业务线程无感知**：

```
顾客线程                                   BAllocThread
   │                                        │
   │ pool.switchAllocator(newFactory)       │
   │   └── 投递 AllocatorSwitch 请求         │
   │       （内含新分配器 + StackCompletion）  │
   │                                        │ 循环中发现 switchRequests 非空
   │                                        │   ├── nextAllocator = 新分配器
   │                                        │   └── priorGenerationObjectsToReplace = size
   │                                        │
   │                                        │ 逐辆处理：
   │                                        │   backgroundCheck()/巡检时发现
   │                                        │   slot.allocator != allocator（旧代车）
   │                                        │   → 旧车用【原分配器】deallocate（关键！）
   │                                        │   → 新车用【新分配器】allocate
   │                                        │   → priorGenerationObjectsToReplace--
   │                                        │
   │                                        │ 全部替换完 → completion.complete()
   │ switchAllocator(...).await() ◄─────────┴──────────┘ 返回
   │
   │ 期间业务线程正常 claim/release，完全无感！
```

**关键设计**：`BSlot.allocator` 字段记录了**每辆车由哪个分配器造的**，所以旧车绝不会被新分配器错误销毁——这是"逐代替换"正确性的基石。

---

## 12. 全景图：Demo 程序从生到死的完整时间线

```
时间 ──────────────────────────────────────────────────────────────────►

Pool.fromThreaded().build()
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 开业筹备：建队列/栈/TLR，BAllocThread 上线                                │
├─────────────────────────────────────────────────────────────────────────┤
│ 预填充：BAllocThread 造 4 辆 → newAllocations 栈                          │
├─────────────────────────────────────────────────────────────────────────┤
│ 业务高峰：5 线程并发 claim/release                                        │
│   ├─ 快路径命中（同一线程反复用同一辆车）★ 无锁                            │
│   ├─ 慢路径取车（newAllocations → live 队列）                             │
│   ├─ 池耗尽 → live.poll() 阻塞 → 等不到 → 超时返回 null                    │
│   └─ 2 秒后：借出时顺检 + 后台巡检 → 过期车 realloc（报废→再造）             │
├─────────────────────────────────────────────────────────────────────────┤
│ 平稳期：调度员循环空转（poll 带超时，省 CPU）                               │
├─────────────────────────────────────────────────────────────────────────┤
│ shutdown()：信号弹入池 → 调度员清空所有车 → completion.complete()           │
│             → await() 返回 → 程序退出                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 13. 灵魂问题速答

**Q1：为什么 Stormpot 快？**
答：① 快路径**完全无锁**（一个 CAS 就完成状态转换，连队列都不碰）；② 昂贵的对象创建被移到后台线程，业务线程永不等待创建；③ 伪共享防护（`BSlotPadded` 缓存行填充）；④ `RefillPile` 批量入队降低队列争用；⑤ 新车走无锁栈、还车走队列，各取所长。

**Q2：claim 一定会遵守超时吗？**
答：THREADED 模式**一定**。因为创建对象在后台线程，业务线程最多只阻塞在 `live.poll(timeout)` 上；INLINE 模式不保证（慢分配会占用 claim 调用）；DIRECT 模式没有分配，天然不超时。

**Q3：release 错用会怎样？**
答：重复 release 会抛 `PoolException`（"You most likely called release() twice"）；用别的线程的 Poolable release 也没事（Slot 协议不要求同线程归还）。

**Q4：池里的对象会不会越借越少？**
答：不会。对象要么在池（LIVING）、要么在用户手里（CLAIMED/TLR_CLAIMED）、要么在回收流程中（DEAD→重造）。`size` 计数器保证**已分配对象总数恒定 = targetSize**（DIRECT 模式除外），除非你主动 `setTargetSize`。

**Q5：`tryClaim()` 和 `claim(Timeout.ZERO)` 区别？**
答：`tryClaim()` 内部就是 `claim(ZERO_TIMEOUT)` 再捕获中断，本质相同——都不等待，拿不到就返回 `null`。

---

*本文档基于对 `BlazePool`、`BSlot`、`BAllocThread`、`RefillPile`、`StackCompletion` 等源码的逐行研读整理，生成日期：2026-08-11。*
