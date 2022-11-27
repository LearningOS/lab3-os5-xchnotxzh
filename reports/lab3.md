# 功能总结

## 进程创建

- 实现功能 -- 全新创建进程而非基于复制父进程地址空间
- 实现策略
  - 获得程序elf数据 -- 根据名字获得elf 数据
  - 创建进程 -- 根据elf 数据创建TCB
  - 调度进程 -- 将TCB放入调度队列

## stride 调度算法

- 实现功能 -- stride调度，通过让低优先级任务能够跟进高优先级任务，来是保障公平性（均匀分配时间）。
- 实现策略 -- 在TaskManager中实现stride调度算法
  - 数据 -- 加入了优先级priority和行程pass的TCB的队列。
  - 操作
    - add -- 将TCB放入队尾
    - fetch -- 选出行程最小的TCB，并给它的行程增加步长（BigStride/priority），超过 BigStride 则做取模的溢出处理。

# 问答题

## 1

### 问题

stride 算法原理非常简单，但是有一个比较大的问题。例如两个 stride = 10 的进程，使用 8bit 无符号整形储存 pass， p1.pass = 255, p2.pass = 250，在 p2 执行一个时间片后，理论上下一次应该 p1 执行。

实际情况是轮到 p1 执行吗？为什么？

### 解答

 不是。pass的类型是8bit无符号整数，超过255会溢出，p2.pass+10 = 5 < p1.pass。

## 2

### 问题

我们之前要求进程优先级 >= 2 其实就是为了解决这个问题。可以证明， **在不考虑溢出的情况下** , 在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 PASS_MAX – PASS_MIN <= BigStride / 2。

为什么？尝试简单说明（不要求严格证明）。

### 解答

1. 已知因为 stride=BigStride/priority，而 priority>=2，所以 stride<=BigStride/2。
2. 假设某个时刻，p1.pass=PASS_MAX, p2.pass=PASS_MIN，PASS_MAX-PASS_MIN > BigStride/2。
3. 则在p1.pass上一次被执行时，p2.pass <= PASS_MIN < PASS_MAX - BigStride/2 <= PASS_MAX - p1.stride = p1.pass，即此时p2.pass < p1.pass, p1不可能作为pass最小的进程被选择出来执行，矛盾。
4. 所以一定有PASS_MAX - PASS_MIN <= BigStride/2

### 问题

已知以上结论，**考虑溢出的情况下**，可以为 pass 设计特别的比较器，让 BinaryHeap<Pass> 的 pop 方法能返回真正最小的 Pass。补全下列代码中的 `partial_cmp` 函数，假设两个 Pass 永远不会相等。

```rust
use core::cmp::Ordering;

struct Pass(u64);

impl PartialOrd for Pass {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        // ...
    }
}

impl PartialEq for Pass {
    fn eq(&self, other: &Self) -> bool {
        false
    }
}
```

### 解答

```rust
use core::cmp::Ordering;

struct Pass(u64);

impl PartialOrd for Pass {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some((self.0 % (core::u64::MAX / 2)).cmp(&(other.0 % (core::u64::MAX / 2))))
    }
}

impl PartialEq for Pass {
    fn eq(&self, other: &Self) -> bool {
        false
    }
}
```

# 建议

需要简化实验方法，因为需要复制大量之前实验。