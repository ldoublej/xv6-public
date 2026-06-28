# xv6 项目结构全览与学习路线图

## 项目简介

xv6 是 MIT 为教学目的重新实现的 **Unix V6**，运行在 **x86 多处理器**架构上，代码量约 **9000 行**（极为精简）。它是理解操作系统内核设计的绝佳起点。

---

## 整体架构分层

xv6 的系统可以分成 **6 个层次**，从底层硬件到上层用户程序：

```
┌─────────────────────────────────────────────────────────────┐
│                    用户程序层                                 │
│   sh.c  cat.c  echo.c  ls.c  grep.c  wc.c                   │
│   mkdir.c  rm.c  ln.c  kill.c  ...                          │
│   用户库: ulib.c  printf.c  umalloc.c                        │
├─────────────────────────────────────────────────────────────┤
│                    系统调用层 (接口)                          │
│   syscall.c  sysfile.c  sysproc.c                           │
│   usys.S  syscall.h  user.h                                 │
├──────────────────┬──────────────────┬───────────────────────┤
│    文件系统层     │    进程管理层      │       锁/同步          │
│   fs.c/bio.c     │   proc.c          │   spinlock.c         │
│   log.c          │   exec.c          │   sleeplock.c        │
│   file.c         │   swtch.S         │                       │
│   ide.c          │   pipe.c          │                       │
├──────────────────┴──────────────────┴───────────────────────┤
│                    内存管理层                                 │
│   vm.c  kalloc.c                                             │
├─────────────────────────────────────────────────────────────┤
│                 硬件/中断/外设                                 │
│   trap.c  trapasm.S  lapic.c  ioapic.c                      │
│   picirq.c  uart.c  console.c  kbd.c                        │
│   mp.c  vectors.S                                            │
├─────────────────────────────────────────────────────────────┤
│                   启动层 (boot)                               │
│   bootasm.S  bootmain.c  entry.S                             │
│   main.c  init.c                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心文件解说（按学习顺序）

### 第一站：启动与初始化（理解系统如何从零启动）

| 文件 | 职责 | 代码量 |
|------|------|--------|
| [bootasm.S](bootasm.S) | CPU 上电第一条指令：从实模式切换到保护模式 | ~100行 |
| [bootmain.c](bootmain.c) | 从磁盘加载内核 ELF 到内存，跳转执行 | ~90行 |
| [entry.S](entry.S) | 内核入口，设置页表、开启分页 | ~80行 |
| [main.c](main.c) | **内核主初始化函数**，依次初始化所有子系统 | ~117行 |
| [init.c](init.c) | 第一个用户态进程（`init`），启动 shell | ~30行 |

> **学习入口：[main.c:17-38](main.c#L17-L38)** — `main()` 函数就是内核初始化的完整流程图。

### 第二站：内存管理（一切数据结构的基础）

| 文件 | 职责 | 关键点 |
|------|------|--------|
| [kalloc.c](kalloc.c) | 物理页分配器（`kalloc`/`kfree`） | 链表管理空闲页 |
| [vm.c](vm.c) | 虚拟内存：页表创建、切换、用户地址空间管理 | 核心：`setupkvm`, `inituvm`, `switchuvm` |
| [memlayout.h](memlayout.h) | 物理/虚拟地址映射宏（`V2P`/`P2V`） | 理解地址空间布局 |

### 第三站：锁与并发（多处理器安全的基础）

| 文件 | 职责 |
|------|------|
| [spinlock.c](spinlock.c) | 自旋锁：忙等待，用于短临界区 |
| [sleeplock.c](sleeplock.c) | 睡眠锁：阻塞等待，用于长临界区（如磁盘I/O） |

### 第四站：进程管理（操作系统的核心）

| 文件 | 职责 | 关键点 |
|------|------|--------|
| [proc.h](proc.h) | `struct proc` 进程结构体、`struct cpu` CPU结构体、进程状态枚举 | 理解进程在内存中的样子 |
| [proc.c](proc.c) | 进程调度（`scheduler`）、`fork`、`exit`、`wait`、`sleep`/`wakeup` | 核心约500行 |
| [swtch.S](swtch.S) | 上下文切换的汇编实现 | 理解寄存器保存/恢复 |
| [exec.c](exec.c) | `exec()` 加载并执行新程序 | ELF 解析 |
| [pipe.c](pipe.c) | 管道实现 | 进程间通信 |

### 第五站：中断与陷阱

| 文件 | 职责 |
|------|------|
| [trap.c](trap.c) | 设置IDT，处理系统调用/中断/异常 |
| [trapasm.S](trapasm.S) | 陷阱入口/出口的汇编代码 |
| [vectors.S](vectors.S) | 中断向量表（自动生成） |
| [lapic.c](lapic.c) | 本地 APIC 中断控制器 |
| [ioapic.c](ioapic.c) | I/O APIC 中断路由 |
| [picirq.c](picirq.c) | 旧 8259A PIC（兼容） |

### 第六站：文件系统（数据持久化）

| 文件 | 职责 | 关键点 |
|------|------|--------|
| [fs.h](fs.h) | 磁盘布局：superblock、inode、dirent 结构 | 理解磁盘格式 |
| [file.h](file.h) | 内存中的 `struct file`、`struct inode` | 理解文件描述符 |
| [fs.c](fs.c) | 文件的创建、读写、目录操作 | 约 600行，核心逻辑 |
| [bio.c](bio.c) | 块缓冲层（buffer cache） | 磁盘块缓存 |
| [log.c](log.c) | 写日志（write-ahead logging），保证崩溃一致性 | 文件系统事务 |
| [ide.c](ide.c) | IDE 磁盘驱动 | 实际 I/O |
| [file.c](file.c) | 文件描述符表管理 | `open`/`close`/`read`/`write` |

### 第七站：系统调用

| 文件 | 职责 |
|------|------|
| [syscall.c](syscall.c) | 系统调用分发器，参数提取 |
| [syscall.h](syscall.h) | 系统调用号定义 |
| [sysfile.c](sysfile.c) | 文件相关系统调用（`sys_open`, `sys_read`, `sys_write` 等） |
| [sysproc.c](sysproc.c) | 进程相关系统调用（`sys_fork`, `sys_exit`, `sys_wait` 等） |
| [usys.S](usys.S) | 用户态进入系统调用的跳板代码 |
| [user.h](user.h) | 用户程序可用的系统调用声明 |

### 第八站：设备驱动

| 文件 | 职责 |
|------|------|
| [console.c](console.c) | 控制台输入/输出（显示器+键盘抽象） |
| [kbd.c](kbd.c) | PS/2 键盘驱动 |
| [uart.c](uart.c) | 串口输出（用于调试） |
| [mp.c](mp.c) | 多处理器检测与配置 |

### 第九站：用户程序（在 xv6 上运行的应用程序）

| 文件 | 用途 |
|------|------|
| [sh.c](sh.c) | shell（命令解释器） |
| [cat.c](cat.c), [echo.c](echo.c), [ls.c](ls.c), [wc.c](wc.c) | 标准 Unix 工具 |
| [grep.c](grep.c), [kill.c](kill.c), [ln.c](ln.c) | 工具命令 |
| [mkdir.c](mkdir.c), [rm.c](rm.c) | 文件操作 |
| [usertests.c](usertests.c) | 用户态回归测试 |

### 公共头文件

| 文件 | 内容 |
|------|------|
| [types.h](types.h) | `uint`, `ushort`, `pde_t` 等类型定义 |
| [defs.h](defs.h) | **全内核函数声明**（理解各模块接口必读） |
| [param.h](param.h) | 系统参数常量 |
| [x86.h](x86.h) | x86 内联汇编辅助宏（`inb`, `outb`, `lgdt`, `lcr3` 等） |
| [mmu.h](mmu.h) | x86 MMU 页表相关常量 |
| [elf.h](elf.h) | ELF 可执行文件格式定义 |

---

## 推荐学习路线

```
第1步  → main.c        (看懂内核初始化全流程，每一行调用对应一个子系统)
第2步  → proc.h        (理解核心数据结构：进程、CPU、上下文)
第3步  → proc.c        (理解进程创建、调度、切换)
第4步  → swtch.S       (上下文切换的本质)
第5步  → vm.c          (虚拟内存与页表)
第6步  → kalloc.c      (物理内存分配)
第7步  → trap.c        (中断处理与系统调用入口)
第8步  → syscall.c     (系统调用如何分发)
第9步  → fs.c + bio.c  (文件系统)
第10步 → spinlock.c    (锁)
第11步 → exec.c        (程序加载执行)
第12步 → pipe.c        (管道)
第13步 → init.c → sh.c (从内核态进入第一个用户进程，到 shell)
第14步 → 其他用户程序   (了解系统调用如何使用)
```

**核心原则**：始终以 [main.c](main.c) 的 `main()` 函数为主线的调用顺序为索引，读到哪个函数就去对应的源文件里看，这样能始终把握"系统正在做什么"，不会迷失在细节里。

---

## 关键数据流

1. **系统调用路径**：用户程序 → [usys.S](usys.S) → `int 64` 陷入内核 → [trapasm.S](trapasm.S) → [trap.c](trap.c) → [syscall.c](syscall.c) → [sysfile.c](sysfile.c) / [sysproc.c](sysproc.c)

2. **磁盘读写路径**：系统调用 → [fs.c](fs.c) → [bio.c](bio.c) → [log.c](log.c) → [ide.c](ide.c)

3. **进程切换路径**：[proc.c](proc.c) `scheduler()` ↔ `sched()` → [swtch.S](swtch.S)
