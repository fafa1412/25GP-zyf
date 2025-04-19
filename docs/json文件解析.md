
#### **Executor的核心原理**
**Executor（调度器）**是zCore的“任务管理器”，负责协调所有异步任务的执行。它的设计灵感来源于用户态的异步运行时（如tokio），但针对内核环境做了优化。

##### **1. 协作式调度**
- **核心思想**：任务主动让出CPU，而不是被强制打断。
- **任务队列**：Executor维护一个任务队列，每个任务是一个`Future`对象。
- **调度流程**：
  1. 从队列头部取出任务。
  2. 调用任务的`poll()`方法推进执行。
  3. 若任务返回`Pending`（需等待），标记为“睡眠”并放回队列尾部。
  4. 循环处理下一个任务。

##### **2. 唤醒机制（Waker）**
- 当任务因等待事件（如IO完成）挂起时，会注册一个**Waker**。
- 事件完成后，Waker通过`wake()`方法将任务重新标记为“活跃”，触发Executor重新调度。

**类比理解**：  
Executor就像餐厅的服务员，任务（Future）是顾客的点餐流程。服务员按顺序处理每位顾客的需求，如果某位顾客需要等待（比如等菜做好），服务员先去服务其他顾客，等菜做好后（Waker触发），再回来继续处理。

---

#### **Rust Future与Executor的结合**
##### **1. Future的本质**
- **状态机**：Rust的`Future`是一个异步操作的状态机，通过`poll()`方法逐步推进。
- **层级结构**：zCore将Future分为三层：
  - **顶层Future**：封装用户线程，处理内核与用户态的切换。
  - **中层Future**：实现系统调用（如带超时的阻塞调用）。
  - **底层Future**：直接操作硬件或等待事件（如时钟中断）。

##### **2. Executor如何驱动Future**
- **顶层Future的执行流程**：
  ```rust
  loop {
      // 切换到用户态执行
      let mut context = thread.wait_for_run().await;
      kernel_hal::context_run(&mut context);
      // 处理中断或系统调用
      match context.trap_num {
          0x100 => handle_syscall().await,
          // ...
      }
  }
  ```
  - `context_run()`切换用户态，返回时表示发生中断或系统调用。
  - 通过`await`挂起当前任务，等待处理完成。

- **中层Future的协作**：
  ```rust
  // 带超时的系统调用实现
  select_biased! {
      ret = future.await => ret,
      _ = sleep_until(deadline).await => Timeout,
  }
  ```
  - 使用`select!`宏同时等待多个事件（如操作完成或超时），任一事件触发即返回。

##### **3. 底层Future与硬件交互**
- **示例：实现Sleep**  
  ```rust
  struct SleepFuture { deadline: Duration }
  impl Future for SleepFuture {
      fn poll() -> Poll<()> {
          if 当前时间 >= deadline {
              Poll::Ready(())
          } else {
              注册定时器回调（触发Waker）；
              Poll::Pending
          }
      }
  }
  ```
  - 当定时器到期时，回调函数调用`Waker.wake()`，唤醒任务。

---

