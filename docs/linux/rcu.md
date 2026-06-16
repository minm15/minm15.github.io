# RCU (Read, Copy and Update)

RCU 同步機制是為了要提升 reader 的 scalability，讓多個 reader 可以進到各自的 read-side critical section。當 writer 要對 RCU 保護的物件進行修改或刪除的時候，會先發佈新的版本或是移除舊物件，接著透過 Grace Period (寬限期) 等待所有的 pre-existing reader 都離開 critical section，最後才釋放舊物件。

## 傳統的同步機制

### Mutex

linux 的 Mutex 實作如下，`owner` 表示目前持有這個 Mutex 的 task。當有 task 想要搶 lock，就會呼叫 `mutex_lock`，並進到 `__mutex_trylock_fast` 的 `atomic_long_try_cmpxchg_acquire`。

```c
struct mutex {
    atomic_long_t       owner;
    raw_spinlock_t      wait_lock;
    struct mutex_waiter *first_waiter __guarded_by(&wait_lock);
}
```

```c
void __sched mutex_lock(struct mutex *lock)
{
    might_sleep();

    if (!__mutex_trylock_fast(lock))
        __mutex_lock_slowpath(lock);
}
```

```c
static __always_inline bool __mutex_trylock_fast(struct mutex *lock)
    __cond_acquires(true, lock)
{
    unsigned long curr = (unsigned long)current;
    unsigned long zero = 0UL;

    MUTEX_WARN_ON(lock->magic != lock);

    if (atomic_long_try_cmpxchg_acquire(&lock->owner, &zero, curr))
        return true;

    return false;
}
```

`atomic_long_try_cmpxchg_acquire` 比較 `owner` 的數值並修改。當有多個 CPU 同時呼叫這個函式，嘗試修改目標值的時候，會透過硬體的 cache coherency 讓一個 CPU 取得這個物件的修改權限，並使其他來自同一個 cacheline 的副本失效，因此不同 CPU 的這些操作會被序列化，只有一個 CPU 能夠成功修改目標物件的值。如此一來便能夠確保不會同時有多個 task 認為自己持有這個 lock。

根據 **Intel® 64 and IA-32 Architectures Software Developer’s Manual** 的說明，當 `LOCK_PREFIX` 套用在 RMW 指令上，會確保指令有原子性。也就是其他處理器無法在這次 locked operation 中插入其他指令來對這個記憶體位置進行修改，硬體會透過 cache coherency 協定取得該 cacheline 的獨佔權限。
> In a multiprocessor environment, the LOCK# signal insures that the processor has exclusive use of any shared memory while the signal is asserted.

```c
case __X86_CASE_Q:
{
    volatile u64 *__ptr = (volatile u64 *)(_ptr);

    asm_inline volatile(lock "cmpxchgq %[new], %[ptr]"
        : "=@ccz" (success),
          [ptr] "+m" (*__ptr),
          [old] "+a" (__old)
        : [new] "r" (__new)
        : "memory");
    break;
}
```

假設有三個 CPU 嘗試進行 `cmpxchg`，且最後拿到修改權限的是 CPU 0，在這個情況下，其他兩個 CPU 對於目標資料所在的 cacheline 的副本會被 invalidate，因此當其他兩個 CPU 發現失效並且會嘗試重新取得 cache 最新值，以及當它們之後嘗試取得 exclusive 的權限，會導致這個 cacheline 的權限頻繁轉移，這個現象就稱為 cache bouncing。
然而 mutex 讓 reader 依然要取得 exclusive 的權限，因此 bouncing 在讀寫操作都會發生，成為效能瓶頸。

![cacheline ownership bounce][cacheline-bounce]


### RWLock
Read/Write lock 和 mutex 最大的差異就是允許多個 reader 進入 critical section。rwlock 需要紀錄 reader 的數量，因此也需要 atomic RMW 的操作，這導致 cache bouncing 依然在讀取操作存在。


## RCU 核心想法

RCU 同步機制針對讀取頻繁的情境提出一個權衡，讀取的操作不需要再去修改全域變數，來幾乎消除 atomic RMW 的 cache coherency。引入寬限期來允許讀到比較舊的版本。

![rcu grace period][rcu-grace-period]

一開始受到 rcu 保護的指標指向舊物件 old foo，reader A 和 B 開始讀取的時候看到 old foo，在它們讀取的過程中，writer 將 rcu 保護的指標改成指向新物件，隨後開啟寬限期，並且不會馬上釋放舊物件，會等到 pre-existing reader A 和 B 都完成後並結束寬限期，才能安全釋放掉舊物件。而 reader C 和 D 都是在 writer 修改後才開始讀取，因此它們讀到的都是新物件。

## RCU 基本 API

讀取時，`lock/unlock` 定義出 critical section 的範圍，透過 `rcu_dereference` 安全讀取 rcu 指標。寫入則是透過 `rcu_assign_pointer(p, v)` publish。

```c
rcu_read_lock();
p = rcu_dereference(ptr);
rcu_read_unlock();
```

若是 writer 需要釋放舊物件，有兩種方法

1. 同步等待：`synchronize_rcu() + kfree(old)`，`synchronize_rcu` 阻塞目前的 writer，讓它等待 GP 結束，之後釋放舊物件。
2. 非同步延後回收：`call_rcu(struct rcu_head *head, rcu_callback_t func)`，不會阻塞 writer，而是把 `rcu_head` 加入到一個 queue，當 GP 結束後執行 callback 函式釋放舊物件。

### RCU list API

除了上述針對單一指標的物件指派之外，Linux kernel 中更常見的用途是用來保護 list 的節點。以下是 list 和 pointer 對於相同功能的 API 整理。

| pointer | list |
| --- | --- |
| `rcu_assign_pointer()` | `list_add_rcu()`、`list_replace_rcu()` |
| `rcu_dereference()` | `list_for_each_entry_rcu()` |
| `RCU_INIT_POINTER(ptr, NULL)` | `list_del_rcu()` |

接著考慮加入新節點的順序，在加入節點前，需要先準備好這個節點（在 list 的案例中，就是要先給 prev 和 next 的值），這樣在 release 新節點之後才能確保 reader 看到的是可以正確使用的。

```c
static inline void __list_add_rcu(struct list_head *new,
                                  struct list_head *prev,
                                  struct list_head *next)
{
    if (!__list_add_valid(new, prev, next))
        return;

    new->next = next;
    new->prev = prev;
    rcu_assign_pointer(list_next_rcu(prev), new); // prev->next = new
    next->prev = new;
}

// 插入 new 到 head 後面
// head <-> A <-> B           (before)
// head <-> new <-> A <-> B   (after)
static inline void list_add_rcu(struct list_head *new, struct list_head *head)
{
    __list_add_rcu(new, head, head->next);
}
```

RCU list 有承諾下方這種正向走訪是安全的，並且前面 `list_add_rcu()` 要先 release `head->next` 再更新 `next->prev` 也是針對正向走訪來設計。
而反向走訪不安全是因為過程中若是節點被刪除，`list_del_rcu` 會讓 `prev` 指標失效，若是真的需要雙向走訪，則可以使用 `list_bidir_del_rcu()`。

```c
#define list_for_each_entry_rcu(pos, head, member, cond...)     \
    for (__list_check_rcu(dummy, ## cond, 0),           \
         pos = list_entry_rcu((head)->next, typeof(*pos), member);  \
        &pos->member != (head);                 \
        pos = list_entry_rcu(pos->member.next, typeof(*pos), member))
```

```c
static inline void list_del_rcu(struct list_head *entry)
{
    __list_del_entry(entry);
    entry->prev = LIST_POISON2;
}
```

### Memory Ordering

在一般的 lock 使用上，例如 spinlock 透過 `lock` 和 `unlock` 提供互斥的能力，讓 writer 在做完完整的 update 並釋放 lock 之後，reader 才能看到修改後的完整內容。
然而 RCU 在 reader 端並沒有提供這種能力的 lock，因此 RCU 透過自己實作的 API 來確保寫入和讀取的正確順序。

考慮以下程式碼，直接從 C 語言來看可以知道對 `global_foo` 的賦值是在對 `new_fp` 的成員賦值之後，但是經過編譯器最佳化的指令，有可能 `global_foo = new_fp` 會在 `new_fp` 的成員賦值的之前完成。在這個情況下，reader 在 `fp = global_foo` 讀到不完整的 `fp`，導致後面的使用出現非預期錯誤。

```c
void foo_update_no_rcu_api(struct foo *new_fp)
{
    struct foo *old_fp;
    spin_lock(&foo_mutex);
    old_fp = global_foo;
    new_fp->a = 1;
    new_fp->b = 'b';
    new_fp->c = 100;   
    global_foo = new_fp;
    spin_unlock(&foo_mutex);
    synchronize_rcu();
    kfree(old_fp);
}
```

```c
void foo_read_no_rcu_api(void)
{
    struct foo *fp;
    rcu_read_lock();
    fp = global_foo;
    if (fp)
        do_something(fp->a, fp->b, fp->c);
    rcu_read_unlock();
}
```

因此 RCU 為 writer 和 reader 提供 `rcu_assign_pointer` 以及 `rcu_dereference` 的 API 確保 reader 能夠讀到完整的新物件。接著進一步看這兩個 API 的實作。

```c
#define rcu_assign_pointer(p, v)					      \
context_unsafe(							      \
	uintptr_t _r_a_p__v = (uintptr_t)(v);				      \
	rcu_check_sparse(p, __rcu);					      \
									      \
	if (__builtin_constant_p(v) && (_r_a_p__v) == (uintptr_t)NULL)	      \
		WRITE_ONCE((p), (typeof(p))(_r_a_p__v));		      \
	else								      \
		smp_store_release(&p, RCU_INITIALIZER((typeof(p))_r_a_p__v)); \
)
```

- 首先，當傳入的 `v` 是編譯時期的常數 NULL 代表要清空指標，而不是釋放新物件，因此不需要確保「讓完整的新物件可見」，所以只做 `WRITE_ONCE` 來保證這次的清空一定會執行，這是為了避免編譯器最佳化導致在多核心中的其他執行緒讀到錯誤的內容。例如 `x = NULL; x = new_value`，編譯器可能會認為第一次的清空會在之後馬上被賦值，所以會把 `x = NULL` 優化掉，這時候如果其他執行緒需要依賴清空後的結果，就會出現非預期錯誤。
- `smp_store_release` 透過 memory barrier 讓在這一行之前的寫入指令不會在這一行之後才執行。從上面 `foo_update_no_rcu_api` 的例子來看，就是對 `new_fp` 成員的指派不會被重排到 `global_foo = new_fp` 之後。

接著考慮 `foo_read_no_rcu_api` 這個讀取，在暫存器不夠的情況下，編譯器可能將指令優化成類似下方的程式碼。這樣在讀取過程中，若有 writer 做其他更新導致 `global_foo` 的成員被修改，那麼 reader 就會讀到錯誤的內容。

```c
void foo_read_no_rcu_api(void)
{
    rcu_read_lock();
    if (global_foo)
        do_something(global_foo->a, global_foo->b, global_foo->c);
    rcu_read_unlock();
}
```

把 reader 程式碼改成以下形式：

```c
void foo_read(void)
{
    struct foo *fp;
    rcu_read_lock();
    fp = rcu_dereference(global_foo);
    if (fp)
        do_something(fp->a, fp->b, fp->c);
    rcu_read_unlock();
}
```

目前 linux 核心對 `rcu_dereference` 的實作如下：

```c
#define rcu_dereference_check(p, c) \
    __rcu_dereference_check((p), __UNIQUE_ID(rcu), \
                            (c) || rcu_read_lock_held(), __rcu)

#define __rcu_dereference_check(p, local, c, space) \
({ \
    typeof(*p) *local = (typeof(*p) *__force)READ_ONCE(p); \
    RCU_LOCKDEP_WARN(!(c), "suspicious rcu_dereference_check() usage"); \
    rcu_check_sparse(p, space); \
    ((typeof(*p) __force __kernel *)(local)); \
})

#define rcu_dereference(p) rcu_dereference_check(p, 0)
```

- `typeof(*p) *local = (typeof(*p) *__force)READ_ONCE(p)` 就是在做 `local = READ_ONCE(p)`，透過 `READ_ONCE` 保證會讀取一次 p，使得編譯器不會把指令優化成 `if (global_foo)` 的形式。


### Tree RCU

為了知道所有 CPU 的狀態，若是採用一個全域變數來讓 CPU 回報，會導致 lock contention 和 cache bouncing 的問題，因此透過 RCU Tree 讓 leaf node 管理部分少量的 CPU，讓競爭問題縮小到少量 CPU 的範疇。

![rcu tree][rcu-tree]

1. `struct rcu_state` 表示一個 RCU Tree 的全域狀態，以 dense array 的形式儲存，node[0] 表示 root，是 level 0，接著 node[1] 到 node[m] 是 level 1，以此類推。
    - `rcu_state.gp_seq` 表示最新一代的寬限期序號，這個無號整數除了給寬限期 index 之外，其中低位元隱含了這個寬限期的狀態資訊。

2. `struct rcu_node` 是 RCU Tree 的內部節點結構，管理下面一層的 child node 或是作為 leaf node 直接管理 CPU。
    - `rcu_node.qsmask` 表示哪一個 bit 對應到的 child node 或是 CPU 尚未回報 quiescent state。
    - `rcu_node.grpmask` 表示這一個 node 在 parent 的 mask 中用哪一個 bit 表示。

3. `struct rcu_data` 表示一個 CPU 本身的 RCU 狀態。
    - `rcu_data.mynode` 指向管理這個 CPU data 的 leaf node
    - `rcu_data.grpmask` 表示這一個 node 在 leaf node 的 mask 中用哪一個 bit 表示。
    - `rcu_data.core_needs_qs` 說明這個 CPU 在這次的寬限期是否需要回報。

透過 `synchronize_rcu()` 或 `call_rcu()` 提出等待寬限期的請求後，系統可能會啟動新的寬限期並更新 `rcu_state` 的 `gp_seq`，然後 RCU 會找到需要等待的 CPU 集合，並更新對應的 `leafnode.qsmask`，接著當 CPU 們逐漸完成，透過 `mynode` 找到它們對應的 leaf node，並將 `qsmask` 的 bit 清掉，當一個 node 的 `qsmask` 為 0 的時候就知道底下管理的資源都完成了，接著會去清掉 parent 的 `qsmask` 的對應 bit，直到 root node 的 `qsmask` 為 0，代表這一次寬限期要等待的 CPU 都通過 QS。

### 寬限期請求與等待路徑

前文提及 writer 要釋放舊物件時會有兩種方法，分別是同步等待的 `synchronize_rcu()` 以及非同步延後回收的  `call_rcu(struct rcu_head *head, rcu_callback_t func)`。

```c
void synchronize_rcu(void)
{
    unsigned long flags;
    struct rcu_node *rnp;

    RCU_LOCKDEP_WARN(lock_is_held(&rcu_bh_lock_map) ||
             lock_is_held(&rcu_lock_map) ||
             lock_is_held(&rcu_sched_lock_map),
             "Illegal synchronize_rcu() in RCU read-side critical section");
    if (!rcu_blocking_is_gp()) {
        if (rcu_gp_is_expedited())
            synchronize_rcu_expedited();
        else
            synchronize_rcu_normal();
        return;
    }
    
    rcu_poll_gp_seq_start_unlocked(&rcu_state.gp_seq_polled_snap);
    rcu_poll_gp_seq_end_unlocked(&rcu_state.gp_seq_polled_snap);

    local_irq_save(flags);
    WARN_ON_ONCE(num_online_cpus() > 1);
    rcu_state.gp_seq += (1 << RCU_SEQ_CTR_SHIFT);
    for (rnp = this_cpu_ptr(&rcu_data)->mynode; rnp; rnp = rnp->parent)
        rnp->gp_seq_needed = rnp->gp_seq = rcu_state.gp_seq;
    local_irq_restore(flags);
}
```

- 一開始的 `RCU_LOCKDEP_WARN` 會判斷目前這個同步等待是不是在一個 reader critical section 裡面，如果是的話，那就有可能這次的同步等待需要等到目前所處的 reader 完成，進而導致 deadlock。
- 當 `rcu_blocking_is_gp` 為 true 的時候，代表目前系統處在一個特殊狀態，讓 RCU 可以判斷當有 writer 請求等待寬限期要 blocking 時，目前不可能有 pre-existing reader 存在，因此可以直接結束這個寬限期並返回，讓 writer 不用 blocking，繼續做該做的事。
- `rcu_gp_is_expedited` 表示這次的 RCU 同步等待是否要走加速的版本，`synchronize_rcu_normal` 會被動等待 CPU 走到 quiescent state，而 `synchronize_rcu_expedited` 會主動通知並且製造額外事件來讓 CPU 檢查自己是否還在 critical section 裡面，讓 CPU 提前回報抵達 quiescent state。
- `rcu_poll_gp_seq_start_unlocked` 和 `rcu_poll_gp_seq_end_unlocked` 標記這次寬限期的開始與結束。
- `rcu_state.gp_seq` 是 RCU 用來追蹤這次寬限期的序號，`+= (1 << RCU_SEQ_CTR_SHIFT)` 的操作讓這次的寬限期視為已完成

```c
static void synchronize_rcu_normal(void)
{
    struct rcu_synchronize rs;

    ...

    rcu_sr_normal_add_req(&rs);

    /* Kick a GP and start waiting. */
    (void) start_poll_synchronize_rcu();

    /* Now we can wait. */
    wait_for_completion(&rs.completion);
    destroy_rcu_head_on_stack(&rs.head);

trace_complete_out:
    trace_rcu_sr_normal(rcu_state.name, &rs.head, TPS("complete"));
}
```

在 normal 的實作中，會先用 `rcu_sr_normal_add_req` 將目前這個同步等待的請求加入 queue，讓 RCU 系統知道目前有 writer 請求等待寬限期完成，然後用 `start_poll_synchronize_rcu` 讓 RCU 開始一個新的或是找到已經存在的寬限期來涵蓋這次的等待，最後 `wait_for_completion` 進到 blocking 的狀態，等到 RCU 完成這個寬限期並喚醒目前的 task。

```c linenums="1"
void synchronize_rcu_expedited(void)
{
    unsigned long flags;
    struct rcu_exp_work rew;
    struct rcu_node *rnp;
    unsigned long s;

    RCU_LOCKDEP_WARN(lock_is_held(&rcu_bh_lock_map) ||
             lock_is_held(&rcu_lock_map) ||
             lock_is_held(&rcu_sched_lock_map),
             "Illegal synchronize_rcu_expedited() in RCU read-side critical section");

    /* Is the state is such that the call is a grace period? */
    if (rcu_blocking_is_gp()) {
        // Note well that this code runs with !PREEMPT && !SMP.
        // In addition, all code that advances grace periods runs
        // at process level.  Therefore, this expedited GP overlaps
        // with other expedited GPs only by being fully nested within
        // them, which allows reuse of ->gp_seq_polled_exp_snap.
        rcu_poll_gp_seq_start_unlocked(&rcu_state.gp_seq_polled_exp_snap);
        rcu_poll_gp_seq_end_unlocked(&rcu_state.gp_seq_polled_exp_snap);

        local_irq_save(flags);
        WARN_ON_ONCE(num_online_cpus() > 1);
        rcu_state.expedited_sequence += (1 << RCU_SEQ_CTR_SHIFT);
        local_irq_restore(flags);
        return;  // Context allows vacuous grace periods.
    }

    /* If expedited grace periods are prohibited, fall back to normal. */
    if (rcu_gp_is_normal()) {
        synchronize_rcu_normal();
        return;
    }

    /* Take a snapshot of the sequence number.  */
    s = rcu_exp_gp_seq_snap();
    if (exp_funnel_lock(s))
        return;  /* Someone else did our work for us. */

    /* Ensure that load happens before action based on it. */
    if (unlikely((rcu_scheduler_active == RCU_SCHEDULER_INIT) || !rcu_exp_worker_started())) {
        /* Direct call during scheduler init and early_initcalls(). */
        rcu_exp_sel_wait_wake(s);
    } else {
        /* Marshall arguments & schedule the expedited grace period. */
        rew.rew_s = s;
        synchronize_rcu_expedited_queue_work(&rew);
    }

    /* Wait for expedited grace period to complete. */
    rnp = rcu_get_root();
    wait_event(rnp->exp_wq[rcu_seq_ctr(s) & 0x3],
           sync_exp_work_done(s));

    /* Let the next expedited grace period start. */
    mutex_unlock(&rcu_state.exp_mutex);
}
```

- 一般流程從第 37 行開始，s 是一個 `unsigned long` 型別，表示當前這個 expedited GP 的序號。
- `exp_funnel_lock` 回傳 true 代表有其他已經存在或是完成的 GP 足以涵蓋目前的 GP，表示別人幫忙完成這次的同步等待，可以直接 return。
- 一般流程已經初始化完成且 expedited worker 已經啟動的情況下，會執行 `synchronize_rcu_expedited_queue_work`，把目前的 expedited work 交給 worker。接著透過 `wait_event` 在 `rnp->exp_wq[rcu_seq_ctr(s) & 0x3]` 這個 queue 等待 GP 完成，直到 `sync_exp_work_done(s)` 回傳 true。
- worker 會負責執行 expedited GP 的主要流程，包含走訪 RCU tree、選出需要等待的 CPU、主動讓 CPU 回報 quiescent state、喚醒等待者。

### Normal GP：登記需求與喚醒GP kthread

在 normal 的路徑中，`start_poll_synchronize_rcu` 用來告訴 RCU 系統需要推進寬限期，RCU 透過 `rcu_start_this_gp` 登記這次需求到 RCU tree，並且得到 `needwake` 決定是否要喚醒 RCU GP kthread。不需要喚醒的情況包含：

1. 這個 GP 需求已經被紀錄過
2. 這個要求的 GP 已經開始
3. 目前的 GP 流程會處理這個需求

```c
static void start_poll_synchronize_rcu_common(void)
{
	unsigned long flags;
	bool needwake;
	struct rcu_data *rdp;
	struct rcu_node *rnp;

	local_irq_save(flags);
	rdp = this_cpu_ptr(&rcu_data);
	rnp = rdp->mynode;
	raw_spin_lock_rcu_node(rnp); // irqs already disabled.

	needwake = rcu_start_this_gp(rnp, rdp, rcu_seq_snap(&rcu_state.gp_seq));
	raw_spin_unlock_irqrestore_rcu_node(rnp, flags);
	if (needwake)
		rcu_gp_kthread_wake();
}
```

當 kthread 被喚醒後，會負責初始化這個 GP 需要的 RCU Tree，包含設定 `qsmask` 以及標記 CPU 的 `rcu_data.core_needs_qs` 告訴它們需要回報狀態。接著當 CPU 執行到 RCU 已知的 quiescent state 檢查點，RCU 會檢查 CPU 是否需要回報狀態，要的話就清掉 `rcu_data.core_needs_qs` 和 `qsmask`，接著當 leaf node 完成並往 parent 更新，直到 `root.qs_mask` 為 0，代表這次 GP 完成。

```c
static void
rcu_check_quiescent_state(struct rcu_data *rdp)
{
	note_gp_changes(rdp);
	if (!rdp->core_needs_qs)
		return;
	if (rdp->cpu_no_qs.b.norm)
		return;
	rcu_report_qs_rdp(rdp);
}
```

```c linenums="1"
static void rcu_report_qs_rdp(struct rcu_data *rdp)
{
	unsigned long flags;
	unsigned long mask;
	struct rcu_node *rnp;

	WARN_ON_ONCE(rdp->cpu != smp_processor_id());
	rnp = rdp->mynode;
	raw_spin_lock_irqsave_rcu_node(rnp, flags);
	if (rdp->cpu_no_qs.b.norm || rdp->gp_seq != rnp->gp_seq ||
	    rdp->gpwrap) {

		rdp->cpu_no_qs.b.norm = true;	/* need qs for new gp. */
		raw_spin_unlock_irqrestore_rcu_node(rnp, flags);
		return;
	}
	mask = rdp->grpmask;
	rdp->core_needs_qs = false;
	if ((rnp->qsmask & mask) == 0) {
		raw_spin_unlock_irqrestore_rcu_node(rnp, flags);
	} else {
		if (!rcu_rdp_is_offloaded(rdp)) {
			WARN_ON_ONCE(rcu_accelerate_cbs(rnp, rdp));
		}

		rcu_disable_urgency_upon_qs(rdp);
		rcu_report_qs_rnp(mask, rnp, rnp->gp_seq, flags);
	}
}
```

```c linenums="1"
static void rcu_report_qs_rnp(unsigned long mask, struct rcu_node *rnp,
			      unsigned long gps, unsigned long flags)
	__releases(rnp->lock)
{
	unsigned long oldmask = 0;
	struct rcu_node *rnp_c;

	raw_lockdep_assert_held_rcu_node(rnp);

	/* Walk up the rcu_node hierarchy. */
	for (;;) {
		if ((!(rnp->qsmask & mask) && mask) || rnp->gp_seq != gps) {
			raw_spin_unlock_irqrestore_rcu_node(rnp, flags);
			return;
		}
		WARN_ON_ONCE(oldmask); /* Any child must be all zeroed! */
		WARN_ON_ONCE(!rcu_is_leaf_node(rnp) &&
			     rcu_preempt_blocked_readers_cgp(rnp));
		WRITE_ONCE(rnp->qsmask, rnp->qsmask & ~mask);
		trace_rcu_quiescent_state_report(rcu_state.name, rnp->gp_seq,
						 mask, rnp->qsmask, rnp->level,
						 rnp->grplo, rnp->grphi,
						 !!rnp->gp_tasks);
		if (rnp->qsmask != 0 || rcu_preempt_blocked_readers_cgp(rnp)) {
			raw_spin_unlock_irqrestore_rcu_node(rnp, flags);
			return;
		}
		rnp->completedqs = rnp->gp_seq;
		mask = rnp->grpmask;
		if (rnp->parent == NULL) {
			break;
		}
		raw_spin_unlock_irqrestore_rcu_node(rnp, flags);
		rnp_c = rnp;
		rnp = rnp->parent;
		raw_spin_lock_irqsave_rcu_node(rnp, flags);
		oldmask = READ_ONCE(rnp_c->qsmask);
	}
	rcu_report_qs_rsp(flags); 
}
```

當 CPU 到 RCU 檢查點時，會呼叫 `rcu_check_quiescent_state`，檢查 CPU 是否需要回報以及用 `rdp->cpu_no_qs.b.norm` 知道 RCU 是不是還沒經過 QS，當 CPU 需要回報並且已經經過 QS 的話，就執行 `rcu_report_qs_rdp` 判斷以下兩個情況有沒有發生，沒有的話就執行 `rcu_report_qs_rnp`。

1. 對應到第 10 行的情況，
    - CPU 實際上還沒到 QS
    - CPU 和 leaf node 追蹤的 GP 序號不一致
    - `rcu_data` 目前追蹤的 GP 序號不可靠
2. 對應到第 19 行
    - 檢查這個 CPU 是否需要回報 QS 狀態

`rcu_report_qs_rnp` 從目前的 rnp 開始，清掉 `rnp.qsmask` 的對應 bit，若是 `qsmask = 0`，則繼續往上清。若是這個 CPU 是最後需要回報 QS 的 CPU，則這次的走訪不會提早 return，會結束 for loop 後呼叫 `rcu_report_qs_rsp` 結束這次 GP。


### CPU QS 回報路徑：從檢查點到 RCU tree

這一段會來討論 RCU 提供什麼樣的機制給 CPU，讓它到達 QS 的時候可以進行回報。回報要做的事情就是包含呼叫 `rcu_check_quiescent_state` 並且把 `rdp->cpu_no_qs.b.norm` 設為 false。

以 context switch 為例，`kernel/sched/core.c` 的 `__schedule` 函式的簡略版本如下，會進到這個函式有三種情況，詳細可以看參照檔案內的註解，簡單說就是當目前 CPU 的 task 不能或不該繼續執行的時候會進到這個函式從 CPU 的 runqueue 挑選下一個要執行的 task，並且在必要的時候做 context switch。

在進入這個函式時，會呼叫 `rcu_note_context_switch(preempt)` 來通知 RCU 這個 CPU 進到了 scheduler 做 context switch 的檢查點，讓 RCU 可以檢查目前這個 CPU 的狀態，確認是否有到達 QS。

```c
static void __schedule(int sched_mode)
{
    prev = rq->curr;
    local_irq_disable();
    rcu_note_context_switch(preempt);
    rq_lock(rq);

    if (!preempt && prev->__state != TASK_RUNNING)
        block prev;

    next = pick_next_task(rq);

    if (prev != next) {
        rq->curr = next;
        context_switch(prev, next);
    } else {
        unlock rq;
        continue prev;
    }
}
```

接著 `rcu_note_context_switch` 根據 RCU 是否為 preemptive 的版本，有不同的實作。
首先看 non-preemtive 的版本，主要更新 RCU 狀態的動作在 `rcu_qs` 裡面，透過 `__this_cpu_write(rcu_data.cpu_no_qs.b.norm, false)` 告知這個 CPU 已經經過 QS。

```c
void rcu_note_context_switch(bool preempt)
{
	trace_rcu_utilization(TPS("Start context switch"));
	rcu_qs();
	/* Load rcu_urgent_qs before other flags. */
	if (!smp_load_acquire(this_cpu_ptr(&rcu_data.rcu_urgent_qs)))
		goto out;
	this_cpu_write(rcu_data.rcu_urgent_qs, false);
	if (unlikely(raw_cpu_read(rcu_data.rcu_need_heavy_qs)))
		rcu_momentary_eqs();
out:
	rcu_tasks_qs(current, preempt);
	trace_rcu_utilization(TPS("End context switch"));
}

static void rcu_qs(void)
{
	RCU_LOCKDEP_WARN(preemptible(), "rcu_qs() invoked with preemption enabled!!!");
	if (!__this_cpu_read(rcu_data.cpu_no_qs.s))
		return;
	trace_rcu_grace_period(TPS("rcu_sched"),
			       __this_cpu_read(rcu_data.gp_seq), TPS("cpuqs"));
	__this_cpu_write(rcu_data.cpu_no_qs.b.norm, false);
	if (__this_cpu_read(rcu_data.cpu_no_qs.b.exp))
		rcu_report_exp_rdp(this_cpu_ptr(&rcu_data));
}
```

接著是 preemtive 的實作，這是為了處理當 task 在 read-side critical section 被 context switch 的情況。

當目前的 task 還在 critical section 並且沒有被標記為 blocked reader 的時候會讓第 10 行的條件為真，首先會將目前的 task 標記為 blocked reader，並且指出這個 task 被哪一個 rcu node 管理，再來 `rcu_preempt_ctxt_queue(rnp, rdp)` 把目前的 task 加入 blocked reader queue。最後還是會執行 `rcu_qs`，這代表目前的 CPU 可以被認為抵達 QS，但不代表 GP 會結束，因為 blocked reader list 還有 task 需要完成。

`rcu_qs` 和 non-preemptive 的版本有些微差異，在於 `WRITE_ONCE(current->rcu_read_unlock_special.b.need_qs, false)`。`current->rcu_read_unlock_special` 代表當 preemptive RCU 的 task 執行 `read_unlock` 時要做的額外特殊處理，而設定 `need_qs` 為 false，則表示當目前的 task 在之後呼叫 unlock 的時候，不需要再處理 CPU QS，因為相關的 CPU 已經在這個階段被標記達到 QS 了。 

```c linenums="1"
void rcu_note_context_switch(bool preempt)
{
	struct task_struct *t = current;
	struct rcu_data *rdp = this_cpu_ptr(&rcu_data);
	struct rcu_node *rnp;

	trace_rcu_utilization(TPS("Start context switch"));
	lockdep_assert_irqs_disabled();
	WARN_ONCE(!preempt && rcu_preempt_depth() > 0, "Voluntary context switch within RCU read-side critical section!");
	if (rcu_preempt_depth() > 0 &&
	    !t->rcu_read_unlock_special.b.blocked) {

		rnp = rdp->mynode;
		raw_spin_lock_rcu_node(rnp);
		t->rcu_read_unlock_special.b.blocked = true;
		t->rcu_blocked_node = rnp;

		WARN_ON_ONCE(!rcu_rdp_cpu_online(rdp));
		WARN_ON_ONCE(!list_empty(&t->rcu_node_entry));
		trace_rcu_preempt_task(rcu_state.name,
				       t->pid,
				       (rnp->qsmask & rdp->grpmask)
				       ? rnp->gp_seq
				       : rcu_seq_snap(&rnp->gp_seq));
		rcu_preempt_ctxt_queue(rnp, rdp);
	} else {
		rcu_preempt_deferred_qs(t);
	}

	rcu_qs();
	if (rdp->cpu_no_qs.b.exp)
		rcu_report_exp_rdp(rdp);
	rcu_tasks_qs(current, preempt);
	trace_rcu_utilization(TPS("End context switch"));
}

static void rcu_qs(void)
{
	RCU_LOCKDEP_WARN(preemptible(), "rcu_qs() invoked with preemption enabled!!!\n");
	if (__this_cpu_read(rcu_data.cpu_no_qs.b.norm)) {
		trace_rcu_grace_period(TPS("rcu_preempt"),
				       __this_cpu_read(rcu_data.gp_seq),
				       TPS("cpuqs"));
		__this_cpu_write(rcu_data.cpu_no_qs.b.norm, false);
		barrier(); /* Coordinate with rcu_flavor_sched_clock_irq(). */
		WRITE_ONCE(current->rcu_read_unlock_special.b.need_qs, false);
	}
}
```

[cacheline-bounce]: ../assets/images/linux/rcu/cacheline-ownership-bounce-rmw-diagram.png
[rcu-grace-period]: ../assets/images/linux/rcu/rcu-grace-period-pointer-update.png
[rcu-tree]: ../assets/images/linux/rcu/rcu-hierarchy-struct-layout.png
