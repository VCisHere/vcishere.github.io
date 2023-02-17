---
layout: post
title: "brpc学习-bthread"
date: 2022-10-17 16:03:15 +0800
categories: 计算机 其他
math: false
typora-root-url: ..

---
# brpc

## 简介

brpc 是一个 C++ 编写的高性能分布式 RPC 框架，最初由百度于 2014 年创建，在 2017 年开源，于 2018 年进入 Apache 孵化器。如今 brpc 已经被广泛地应用于各大公司，包括百度、字节跳动、滴滴、bilibili、网易等。



## 整体架构

一个RPC的基本流程：

1. Client 通过 Channel（通道，可视为命名服务的 Client）进行 RPC 调用

2. 通过命名服务中的 Server 列表和某种负载均衡算法找到访问的 Server

3. 使用某种协议进行序列化请求通过套接字发送

4. Sever 反序列化通过方法名找到对应服务

5. Sever 执行方法调用后将结果返回给 Client



## bthread 线程库

### 介绍

bthread 是 brpc 使用的 M:N 线程库，M:N 是指 M 个用户级线程（bthread）会映射至 N 个内核级线程（pthread），一般M远大于N。

> 设计目标
> 
> - 用户可以延续同步的编程模式，能在数百纳秒内建立 bthread，可以用多种原语同步。
> 
> - bthread 所有接口可在 pthread 中被调用并有合理的行为，使用 bthread 的代码可以在 pthread 中正常执行。
> 
> - 能充分利用多核。



### 整体设计

**TaskMeta**

- 每个 bthread 对应一个 TaskMeta，TaskMeta 即 bthread 下 task 的元信息，包括任务处理的上下文，bthread 的 id 等

**TaskGroup**

- 意思为 Thread-Local group of tasks，一个 TaskGroup 是线程的单例，是一个 Thread Local Storage 变量

- 负责调度 bthread 和管理调度队列 rq 和 remote_rq

**TaskControl**

- 是进程的单例，负责创建和管理 TaskGroup



### 执行流程

开启 bthread

bthread 的启动入口函数有两个：

1. bthread_start_urgent()：让出当前 worker 立即执行新的 bthread

2. bthread_start_background()：将要启动的 bthread 放入队列等待调度

使用示例

```cpp
// 用于保存 bthread 的 ids
std::vector<bthread_t> tids(3);
for (size_t i = 0; i < tids.size(); ++i) {
    // 开启后台 bthread，开启的 bthread id 保存在 tid[i] 中
    bthread_start_background(&tids[i], nullptr, fn, arg);
}
std::vector<void*> res(tids.size());
for (size_t i = 0; i < tids.size(); ++i) {
    // 等待执行结束，返回值保存在 res[i] 中
    bthread_join(tids[i], &res[i]);
}
```

- 如果 caller 不是 worker 线程，任务加入 remote_rq

- 如果 caller 是 worker 线程，任务加入 rq



调度

常见的调度策略

1. 星切调度
   main task -> task1 -> main task -> task2 -> main task -> task3 -> ... -> main task

2. 环切调度
   main task -> task1 -> task2 -> task3 -> ... -> main task

对比两种调度策略，环切调度次数是星切调度的一半，但同时实现难度会更大，task 运行完成后要主动调用切换方法，在被切换后如何回收内存等，bthread 采用了环切调度。



**TaskGroup**

```cpp
class TaskGroup {
    // ...
    size_t _steal_seed; // work stealing 算法随机数种子
    size_t _steal_offset; // work stealing 算法随机数偏移量
    
    // main bthread，调度线程
    ContextualStack* _main_stack;
    bthread_t _main_tid;
    
    WorkStealingQueue<bthread_t> _rq; // 通过 worker 添加
    RemoteTaskQueue _remote_rq; // 非 worker 添加
};
```



**Work Sharing**

在介绍 Work Stealing 之前，先介绍一下 Work sharing 工作共享算法。

在该算法中，所有的线程共享一个全局的任务队列，虽然实现起来会简单些，但是很明显的是队列的竞争会很严重，因此，让每个 Thread 有自己的调度队列，同时为了负载均衡采取 Work Stealing 算法是更优的方法。



**Work Stealing**

Work Stealing 即工作窃取算法，是一种实现 CPU 负载均衡的方式，使 CPU 在处理任务时，每个核的负载均衡。在 bthread 中，这里的工作窃取是某一个 worker 线程通过调度器从其他 worker 线程的任务队列中获取任务。



**具体流程分析**

前面讲到每个 worker 线程开启的时候就会运行 run_main_task()

```cpp
void TaskGroup::run_main_task() {
    TaskGroup* dummy = this;
    bthread_t tid;
    while (wait_task(&tid)) {
        TaskGroup::sched_to(&dummy, tid);
        // ...
        if (_cur_meta->tid != _main_tid) {
            TaskGroup::task_runner(1);
        }
        // ...
    }
}
```

从代码可以看出 run_main_task() 执行的是一个无限循环，三个关键函数：

- wait_task：等待获取一个任务，其中包括从自身队列中获取和从其他 worker 线程中获取

- sched_to：进行调度，包括进行栈、寄存器等上下文的切换

- task_runner：执行 bthread



**wait_task()**

伪代码如下：

```cpp
bool TaskGroup::wait_task(bthread_t* tid) {
    // 1. 等待 parking lot，可以理解为一个 bthread 到来的条件变量
    // 2. 执行 steal_task(tid)，成功返回 true
}

bool TaskGroup::steal_task(bthread_t* tid) {
    // 1. 优先从本 _remote_rq 中获取
    if (_remote_rq.pop(tid)) {
        return true;
    }
    // ...
    // _control 是本 TaskGroup 所属的 TaskControl 指针
    return _control->steal_task(tid, &_steal_seed, _steal_offset);
}

bool TaskControl::steal_task(bthread_t* tid, size_t* seed, size_t offset) {
    bool stolen = false;
    size_t s = *seed;
    // ngroup 是所有 TaskGroup 的数量
    for (size_t i = 0; i < ngroup; ++i, s += offset) {
        // _groups 数组保存所有该 TaskControl 下的 TaskGroup 指针
        TaskGroup* g = _groups[s % ngroup];
        if (g) {
            // 2. 从随机 TaskGroup 的 _rq 队列中取
            if (g->_rq.steal(tid)) {
                stolen = true;
                break;
            }
            // 3. 从随机 TaskGroup 的 _remote_rq 中取
            if (g->_remote_rq.pop(tid)) {
                stolen = true;
                break;
            }
        }
    }
    // ...
    // 更新随机数种
    *seed = s;
    return stolen;
}
```

可以看出一个 worker 线程执行 bthread 的调度策略的优先级是：

1. 从自己的 remote_rq 中获取

2. 从随机 worker 的 rq 中获取

3. 从随机 worker 的 remote_rq 中获取

设计原因：

本质上是为了更好的实现 bthread 的分配和减少 worker 线程之间 steal 时的竞争

1. 区分 remote_rq 和 rq 可以使 worker 能在自己 remote_rq 获取时，不必和其他 worker的 rq 窃取竞争

2. 从自己的 remote_rq 中获取到从随机 worker 的 remote_rq 中隔了一层随机 worker 的rq 获取来减小从自己remote_rq的竞争

3. 在自己 remote_rq 中获取不到时不直接从自己的 rq 中获取是为了避免直接和其他前来窃取的 worker 竞争



**sched_to()**

```cpp
void TaskGroup::sched_to(TaskGroup** pg, TaskMeta* next_meta) {
    TaskGroup* g = *pg;
    ...
    // Switch to the task
    // cur_meta 是当前执行的 TaskMeta
    // __builtin_expect(next_meta != cur_meta, 1) 的意思是极大可能 next_meta != cur_meta，在此处是绝不可能
    // __builtin_expect 用于编译器优化条件分支语句
    if (__builtin_expect(next_meta != cur_meta, 1)) {
        ...
        if (cur_meta->stack != NULL) {
            if (next_meta->stack != cur_meta->stack) {
                // 切换栈
                jump_stack(cur_meta->stack, next_meta->stack);
            }
            ...
        }
    } else {
        LOG(FATAL) << "bthread=" << g->current_tid() << " sched_to itself!";
    }
    ...
}
```

在函数 jump_stack() 中，进行了栈的切换



**task_runner()**

```cpp
void TaskGroup::task_runner(intptr_t skip_remained) {
    TaskGroup* g = tls_task_group;
    do {
        // 当前运行 bthread 上下文
        TaskMeta* const m = g->_cur_meta;
        ...
        执行 bthread 函数
        ...
        // 执行 ending_sched
        ending_sched(&g);
    } while (g->_cur_meta->tid != g->_main_tid);
}

void TaskGroup::ending_sched(TaskGroup** pg) {
    TaskGroup* g = *pg;
    bthread_t next_tid = 0;
    ...
    const bool popped = g->_rq.steal(&next_tid);
    if (!popped && !g->steal_task(&next_tid)) {
        // _rq 已经没有任务并且其他 TaskGroup 也没有任务时
        // 重置为 main bthread
        next_tid = g->_main_tid;
    }
    TaskMeta* const cur_meta = g->_cur_meta;
    TaskMeta* next_meta = address_meta(next_tid);
    ...
    sched_to(pg, next_meta);
}
```

可以看出 task_runner() 是一个循环，先是执行之前获取到的 bthread，执行完后进入 ending_sched()，先是尝试从 rq 队列中取任务，如果失败再次从其他 TaskGroup 中取任务，当两者都失败时才会把当前运行的 bthread 设置为 main bthread，用来退出 task_runner 的循环，重新回到 wait_task()，这样设计能够在大量 bthread 时减少 wait_task 的调用次数，从而提高效率。



### 思考

**与 goruntine 的异同**

| **特性**   | **bthread**   | **goruntine** |
| -------- | ------------- | ------------- |
| 本地存储     | yes           | no            |
| 优先级      | no            | no            |
| 并发效率     | 高             | 高             |
| 创建时延     | 0.2us         | -             |
| 抢占       | no            | yes           |
| 易用性      | 还行            | 强             |
| 最大数量     | 少于 goruntine  | 超级多           |
| 调度时延     | 3-20us (越忙越快) | 3us-10ms (gc) |
| 线程池调整线程数 | no            | yes           |

1. bthread 和 goruntine 都是基于线程池技术实现的 M:N 线程模型，在 CPU 负载均衡上都采用了 Work Stealing 算法

2. bthread 的栈大小并不是动态的，但是提供了 3 种大小的栈（32KB、1MB、8MB），goruntine 使用的栈是可动态变化的（初始2KB）

3. bthread 并不是一个完备的线程库，并没有支持优先级和抢占，主要原因是它纯粹为了 rpc 这种场景而设计的，主要目的在于高效地创建和调度，但是还是提供了bthread_start_urgent() 来保证优先执行，还有 bthread_join() 等原语同步

4. goruntine是一个系统级完备的 M:N 线程库，在设计上要比 bthread 复杂的多

通过学习 bthread，已经可以了解到 M:N 线程模型的基本实现。


