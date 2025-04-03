流程理解：

---

### 一、符号提取阶段（构建时）
1. **正则匹配异步函数**
   ```rust
   grep -E "_<.*Future>::poll"
   ```
   - 匹配所有实现`Future` trait的`poll`方法
   - 捕获`async fn`编译生成的生成器结构体

2. **符号Demangle处理**
   ```bash
   rustfilt < zcore-mangled.sym > zcore.sym
   ```
   - 将`_RNvMNtNtCs...`形式的符号转换为可读的Rust函数名
   - 示例转换：
     ```
     0x8020b200 _RNvXs_NtNtCsfn4sx... // 转换为
     0x8020b200 <async_task::Executor::run::{{closure}}>
     ```

3. **生成专用符号表**
   - 输出文件：`zcore-async-fn.sym`
   - 内容示例：
     ```
     0x8020b200 async_std::task::Builder::spawn
     0x8020c440 core::future::from_generator::GenFuture<T>::poll
     ```

---

### 二、探针加载阶段（运行时）
1. **eBPF字节码加载**
   ```c
   mmap + read => bpf_prog_load_ex()
   ```
   - 将`async-fn-context.o`加载到内核验证通过后返回fd

2. **动态附加探针**
   ```c
   for each symbol in zcore-async-fn.sym:
       entry_event = "kretprobe@entry$" + symbol_name
       exit_event = "kretprobe@exit$" + symbol_name
       bpf_prog_attach(entry_event, prog_fd)
       bpf_prog_attach(exit_event, prog_fd)
   ```
   - 为每个异步函数创建两个探测点：
     - **Entry**：函数入口（参数准备完成）
     - **Exit**：函数返回（上下文销毁前）

---

### 三、数据采集阶段（运行时）
1. **入口探测（kretprobe@entry）**
   ```c
   // async-fn-context.c
   case 1: // ptype=1表示entry事件
       depth = bpf_get_depth(pc+4, fp_reg);
       bpf_trace_printk("[entry] 0x%llx depth=%d", pc, depth);
   ```
   - 获取程序计数器+4（跳过call指令）
   - 通过RISC-V的s0/fp寄存器回溯调用栈

2. **出口探测（kretprobe@exit）**
   ```c
   case 2: // ptype=2表示exit事件
       depth = bpf_get_depth(pc, fp_reg);
       bpf_trace_printk("[exit] 0x%llx depth=%d", pc, depth);
   ```
   - 记录返回地址
   - 此时栈帧尚未销毁，可获取完整上下文

3. **调用栈深度计算**
   ```rust
   while current_fp > sstack {
       trace_pc.push(return_address);
       current_fp = *(current_fp - 16) // RISC-V栈帧布局
       current_pc = *(current_fp - 8)
   }
   ```
   - RISC-V特定栈结构：
     ```
     | ...          |
     +--------------+
     | Return Addr  |  -8
     +--------------+
     | Prev FP      |  -16
     +--------------+
     ```

---

### 四、数据分析阶段
1. **日志样例**
   ```
   [entry] 0x8020b200 depth=3 // 进入三级嵌套的异步任务
   [exit] 0x8020c440 depth=2 // 退出后深度减少
   ```

2. **关键数据分析维度**
   - **生命周期可视化**：通过entry/exit时间戳计算函数执行时长
   - **调用链重构**：结合深度变化与PC值绘制异步任务树
   - **资源竞争检测**：同一深度下多个异步任务的并发情况
---

- 构建过程出现
  ```
  copy zcore failed: No such file or directory (os error 2)
  copy zcore-mangled.sym failed: No such file or directory (os error 2)
  ```
  错误（找到相关代码，还未解决）
  
- 内核异步函数分类有疑问（和用户态一样吗）
  
