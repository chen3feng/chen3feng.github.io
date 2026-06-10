---
title: "深入二进制系列：线程局部存储演化简史"
date: "2026-06-10 09:00:00 +0800"
categories: [System, Binary]
tags: [TLS, Thread Local, ELF, PE, Mach-O, FS, GS, TPIDR, TLSDESC, thread_local, ThreadLocal, glibc, Java]
layout: post
published: false
---

> 多线程程序里，`errno` 凭什么能让每个线程各看各的？一个 `thread_local` 变量，如何让每个线程有自己的副本？这件看似轻而易举的事，背后有四十年的演化：从手动管理的 API，到编译器原生支持的段寄存器寻址，再到 C++、Java 的语言级抽象。本文按时间顺序，把线程局部存储（TLS）从「为什么需要它」一路拆到今天的 `thread_local`、`ThreadLocal` 和 TLSDESC。

这是「深入二进制」系列的第三篇。前两篇——[《符号导出三国志》]({% post_url 2026-06-07-symbol-export-windows-linux-macos %}) 和 [《在 Wine 里打开并读一个文件》]({% post_url 2026-06-08-wine-pe-call-elf %})——分别拆了动态库的符号访问和 PE/ELF 跨界调用，这篇接着讲一个同样「藏在二进制底层、平时看不见」的东西：每个线程怎么拥有一份自己的变量。

## 一、TLS 是什么：从 `errno` 说起

最早的程序都是单线程的。一个进程从 `main` 开始，一条执行流走到底；全局变量就是名副其实的「全局」——整个程序独此一份，谁都能读写，也从来没人跟你抢。那个年代，很多设计都建立在这个隐含前提上，`errno` 就是典型。

C 标准库的 `errno` 本来就是个全局 `int`：系统调用失败了，把错误码塞进去，你紧接着读它。只要全程只有一条执行流，这毫无问题。

可线程出现后，这个前提塌了。多个线程并发跑、共享同一份全局变量，灾难立刻来了：线程 A 刚把 `errno` 设成 `EAGAIN`，还没来得及读，线程 B 的一个失败调用就把它覆盖成 `EINVAL`。一个本该「我自己的错误码」的东西，被别的线程踩得稀烂——**`errno` 成了 TLS 这整套机制的导火索**。

问题的本质是：**有些状态，语义上就该「每个线程一份」，而不是「整个进程一份」**。`errno` 如此，随机数种子、当前区域设置（locale）、每线程的内存池、日志上下文，都是如此。把它们当普通全局变量，在多线程下必然出错。

解决思路很直白：**让 `errno` 不再是「一个」全局变量，而是「每个线程一份」**。线程 A 读到的是 A 的那份，线程 B 读到的是 B 的那份，互不干扰。今天 glibc 里 `errno` 其实是个宏，展开成 `(*__errno_location())`，而 `__errno_location()` 返回的正是一个**线程局部变量**的地址。

这就是**线程局部存储（Thread-Local Storage, TLS）**：同一个变量名、同一份代码，每个线程看到独立的副本。

把它抽象一下，任何 TLS 方案都要回答两个问题，**后面每一代技术，优化的都是这两步**：

1. **锚点**：怎么快速知道「我是哪个线程」——拿到当前线程的一个基准地址；
2. **定位**：从这个锚点出发，怎么找到具体那个变量的副本。

下面就按时间顺序，看这两步是怎么一步步从「手动查表」演化到「一条指令取址」的。

---

## 二、第一代：纯 API，手动管理

最早没有任何编译器或语言支持，TLS 完全是**库函数 + 手动管理**。两大平台各一套，长得几乎一样：

| | POSIX | Windows |
|---|---|---|
| 申请一个槽 | `pthread_key_create` | `TlsAlloc` |
| 写 | `pthread_setspecific(key, v)` | `TlsSetValue(idx, v)` |
| 读 | `pthread_getspecific(key)` | `TlsGetValue(idx)` |
| 释放 | `pthread_key_delete` | `TlsFree` |

模型很简单：你申请到一个 **key（其实就是个整数下标）**，运行时在每个线程里维护一张「槽位表」，`key` 就是这张表的索引。读写就是「先定位本线程的表，再用 key 索引」。

```c
pthread_key_t key;
pthread_key_create(&key, NULL);
pthread_setspecific(key, malloc(...));   // 本线程的槽
... pthread_getspecific(key) ...         // 各线程各取各的
```

特征也很清楚：

- **灵活**：纯运行时，key 是普通整数，能随便传递、跨模块共享，不受链接边界限制；
- **啰嗦**：变量没法像普通变量那样直接用，处处得 `get`/`set`；生命周期、初始化、清理全靠手动；
- **慢**：每次访问都是一次函数调用 + 查表。

这层看似原始，却是**整个 TLS 大厦的地基**——后面会看到，编译器原生 TLS、macOS 的 TLV、Java 的 `ThreadLocal`，最终都还落在它（或它的等价物）上面。

---

## 三、第二代：编译器原生 TLS

手动 API 太难用，于是大家想：能不能让线程局部变量**像普通全局变量一样声明、一样直接用**？这就有了编译器层面的关键字：

```c
__thread int x;              // GCC/Clang 扩展
__declspec(thread) int x;    // MSVC
```

这两个关键字出现的时间差了快十年，正好反映两个平台线程化的早晚。**`__declspec(thread)` 早得多**——随 Windows NT 一起，约 **1994 年**（Visual C++ 2.x 时代）就能用了；NT 从一开始就把线程、TEB、TLS 当一等公民。**`__thread` 要到 ~2003 年**才有（GCC 3.3 + glibc 2.3），因为 Linux 像样的线程库（NPTL）到那时才落地，编译器原生 TLS 自然跟着才成熟。

要做到「直接用」，编译器、链接器、加载器、乃至 CPU 都得配合。核心是把「锚点」这一步**下沉到硬件**。

### 锚点：线程指针寄存器

几乎每个现代架构都拿出（或约定）一个寄存器，专门存「当前线程的基准地址」，操作系统在上下文切换时维护它。这样取锚点不再是函数调用，而是**读一个寄存器**。

各家的选择：

| 架构 | 线程指针 | 形态 |
|---|---|---|
| x86-64 | **FS**（Linux）/ **GS**（Windows、macOS） | 段基址（MSR / FSGSBASE） |
| x86-32 | GS（Linux）/ FS（Windows TEB） | 段基址（LDT/GDT 项） |
| AArch64 | **TPIDR_EL0**（Linux）/ **TPIDRRO_EL0**（macOS） | 专用寄存器 |
| ARMv7 | TPIDRURO | 专用寄存器 |
| RISC-V | `tp`（x4） | ABI 保留的通用寄存器 |
| SPARC | %g7 | ABI 保留的通用寄存器 |
| PowerPC | r13（ppc64） | ABI 保留的通用寄存器 |
| MIPS | `rdhwr $29`（UserLocal） | 专用 HW 寄存器，老核内核模拟 |
| s390x | %a0/%a1 | 访问寄存器 |

归成三种形态：**专门造的特殊寄存器**（ARM TPIDR、x86 段基址）、**拿一个通用寄存器 ABI 约定保留**（RISC-V `tp`、SPARC `%g7`）、以及硬件没有时的**软件模拟**（老 ARM 的 kuser helper page、老 MIPS 内核模拟 `rdhwr`、编译器的 `-femulated-tls`）。

这里有两个值得记的「反转」梗：x86-64 上 **Linux 用 FS、Windows 和 macOS 用 GS**；到了 arm64，**Linux 用 TPIDR_EL0、macOS 用 TPIDRRO_EL0**。同一块硬件，不同系统各挑一个。正因为这种不一致，Wine 在 PE 和 ELF 之间来回切换时，必须在边界上倒腾 FS 基址——这段我在 [《在 Wine 里打开并读一个文件》]({% post_url 2026-06-08-wine-pe-call-elf %}) 里专门拆过。

顺带一提：连不写任何 TLS 的代码也天天用这个寄存器——**栈保护的 canary** 在 x86-64 Linux 上就是从 `%fs:0x28` 读的。所以几乎每个带栈保护的函数，开头都在碰线程指针。

### 定位：从锚点到变量

锚点统一了，但「从锚点怎么找到变量」这一步，各平台分了岔——而且这个分岔，是本文真正的核心，下一章专门讲。

---

## 四、第三代：动态链接逼出的两种哲学

如果只有一个可执行文件，定位很简单：变量在「本线程 TLS 块」里的偏移是链接期就定死的常量，访问就是 `线程指针 + 常量偏移`，一条指令。

麻烦出在**动态库**：一个 `.so`/`.dll` 可能在程序启动时加载，也可能事后被 `dlopen`/`LoadLibrary` 拉进来；它的 TLS 块在某个线程里**到底分配了没有、在哪**，编译期都不知道。于是三大平台给出了三套不同的取舍，但本质是同一道选择题：

> **惰性分配 + 访问时调函数** ，还是 **加载期预先分配 + 访问时直接取址**？

### Linux/ELF：给你最多的自由

ELF 把它做成**四种访问模型**，从最通用最慢到最快最受限，让你按场景选：

| 模型 | 适用场景 | 访问 | 调函数？ |
|---|---|---|---|
| global-dynamic | 变量可能在 `dlopen` 进来的库里 | `__tls_get_addr({模块,偏移})` | **是** |
| local-dynamic | 同模块多个 TLS 变量 | 调一次拿模块基址，再常量偏移 | **是**（摊薄） |
| initial-exec | 启动时就加载的库，永不 dlopen | 一次 GOT load 拿 tpoff，再 TP 相对 | 否 |
| local-exec | 变量在主程序里 | `TP + 链接期常量` | 否 |

为什么 global-dynamic 非得调 `__tls_get_addr`？因为它要支持 `dlopen` 和**惰性分配**——访问那一刻，这个线程的这块 TLS 可能还没分配。这个函数干的是「检查 DTV（每线程的模块→块映射表）、没分配就现场 `malloc` 并初始化、再返回地址」，有分支、要加锁、可能分配内存，**天然只能是个函数调用**。

用什么模型，通过编译器选项 `-ftls-model` 来设置，它的默认值取决于是否开启 PIC：**编 `.so`（`-fPIC`）默认 global-dynamic，编可执行文件（`-fno-pic`）默认 initial-exec**。这就解释了一个常见困惑——**为什么 `.so` 里的 `thread_local` 默认就慢**：因为它默认是 global-dynamic，每次访问一个函数调用。而它「慢」的根因，是符号默认可被插入（interposition）导致编译器不敢假设变量在本模块——这层我在 [《符号导出三国志》]({% post_url 2026-06-07-symbol-export-windows-linux-macos %}) 里讲过，标 `hidden` 或显式 `-ftls-model=initial-exec` 就能拨快。

### Windows：加载器预分配，零调用

Windows 走另一极端：**加载器在线程创建时就把每个模块的 TLS 块都分配好**，访问时块保证已存在，于是不需要任何「可能要分配」的逻辑。静态 TLS（`__declspec(thread)`）的访问是纯 inline（x86-64）：

```asm
mov rax, gs:[58h]            ; TEB->ThreadLocalStoragePointer
mov ecx, [_tls_index]        ; 本模块的下标，加载器填
mov rax, [rax+rcx*8]         ; 直接拿到本线程、本模块的块
mov eax, [rax + OFFSET]      ; 加常量偏移
```

无分支、无调用，等价于 ELF 的 initial-exec。代价记在别处，而且是更难受的**能力账**：

- **Vista 之前，`LoadLibrary` 动态加载的 DLL 里的静态 TLS 根本不工作**——因为已存在线程的 TLS 数组是创建时定好大小的，没给新模块留位置。Vista 起，加载器学会在 `LoadLibrary` 时给所有已存在线程补建、扩容 TLS 向量，才修好。
- 至今有**静态 TLS 总量预算**，一口气 `LoadLibrary` 太多带 TLS 的 DLL 会失败。

### macOS：描述符 + thunk，统一一条调用路

macOS 既不像 Windows 预分配，也不像 ELF 给你自己选，而是**所有原生 TLS 统一走一条「描述符 + 函数」的路**，相当于「一切都按 global-dynamic 来」。每个线程局部变量对应一个 **TLV**（Thread-Local Variables，macOS/Mach-O 对这套原生线程局部变量机制的叫法）描述符：

```c
struct tlv_descriptor {
    void*         (*thunk)(struct tlv_descriptor*);  // 指向 tlv_get_addr
    unsigned long key;        // 一个 pthread key
    unsigned long offset;     // 变量在本 image TLS 块里的偏移
};
```

访问变量，每次都去调那个 thunk：

```asm
movq   _x$tlv$init@TLVP(%rip), %rdi   ; 描述符地址
callq  *(%rdi)                         ; descriptor->thunk(descriptor)，返回变量地址
```

而 `tlv_get_addr` 内部就是 `pthread_getspecific(key)` 拿到本线程的块、空就惰性分配——**绕了一圈，macOS 的原生 TLS 还是落在第一代的 pthread key 之上**。注意它的锚点链：`TPIDRRO_EL0 → TSD 数组 → key → 块 → +offset`，线程指针只把你带到 TSD 表，后面还得查一次。

### 三家对比

| | Linux/ELF | Windows | macOS |
|---|---|---|---|
| 锚点寄存器（x86-64） | FS | GS | GS |
| 从锚点到变量 | 四模型（DTV / TP+偏移） | `TLSP[_tls_index]+off` | 描述符 → pthread key |
| 分配时机 | 惰性（GD/LD）/ 预分配（IE/LE） | 加载器**预分配** | 惰性（pthread key） |
| 访问是否调函数 | GD/LD 要，IE/LE 不要 | **否** | **总是** |
| 模型可调 | **有**（四档） | 无 | 无 |

由此还牵出两个常被踩的后果：

- **跨模块访问**：Linux 透明——你直接 `extern __thread` 引用别的 `.so` 的 TLS 变量就行，global-dynamic 自动搞定；Windows 不行，得让那个 DLL **导出一个返回地址的访问函数**。有意思的是，两边最后都落到一次函数调用，只是 Linux 让编译器自动生成，Windows 让你手写。
- **dlopen 与静态 TLS 预算**：Windows 的「静态 TLS budget」和 glibc 的「static TLS surplus」（可调项 `glibc.rtld.optional_static_tls`）其实是**同一个问题的两个版本**——预分配换来的零成本访问，总得有个量的上限。

### 三家为何选了不同的路

这三套设计不是谁更聪明，而是各自的**历史包袱和优化目标**不同。

**Windows——一切围着「集成的加载器」转。** NT 从 1993 年起就把线程和加载器作为一套紧密集成的 OS 组件来设计：加载器本就掌管 TEB、本就要遍历所有模块。既然 OS 同时管着加载和线程，它就有条件在加载器里**预先把每线程的 TLS 都铺好**。而且早期 Windows 本就不鼓励、甚至不支持 `LoadLibrary` 式的晚加载 TLS（Vista 才补），所以「必须支持任意 dlopen + 惰性分配」这个压力根本不在设计考量里。它优先**访问速度**，把活儿压给 OS，代价（静态预算、晚加载受限）在当时可以接受。

**Linux/ELF——一切围着「最大化动态链接灵活性」转。** ELF 的整个世界观就是 dlopen、符号插入、平铺命名空间（见[《符号导出三国志》]({% post_url 2026-06-07-symbol-export-windows-linux-macos %})）。TLS 必须塞进这个世界：任何 `.so` 可能在任意时刻被 dlopen 进一个已有大量线程的进程，TLS 变量还可能跨模块、被插入。在这些约束下，**唯一通用正确的方案就是惰性分配 + 运行时 resolver**（`__tls_get_addr`）。因为慢，才又补了 IE/LE 作为「你知道得更多时」的 opt-in 优化。于是 ELF 选了**「默认正确灵活、快靠手动拨」**，和它「最大化通用动态链接」的一贯气质一致。

**macOS——一切围着「复用 + 统一」转。** macOS 直到 2011 年（10.7）才有原生 TLS，那时 **pthread key 这套 TSD 基础设施早已根深蒂固**。与其再造一套 Windows 式的预分配加载器机器，Apple 选了最省事的路：把原生 `__thread`/`thread_local` 做成 **pthread key 之上的一薄层**，用统一的「描述符 + thunk」一条路把 dlopen、C++ 动态初始化/析构全部透明覆盖，不开特例。它优先**实现的简单与统一**，而非峰值访问速度——一条代码路径、永远能用、复用现成的东西。

一句话：**Windows 因为加载器集成早、又赌 load-time 已知，选了预分配；Linux 因为要服务 dlopen 和插入的极致灵活，选了惰性 + 光谱；macOS 因为原生 TLS 来得晚、不愿重造轮子，选了复用 pthread key 的统一 thunk。** 三种取舍，对应三种「在什么时候、为什么做这个决定」。

---

## 五、第四代：语言标准化

`__thread` 是编译器扩展，C11/C++11 把它收进标准：C 的 `_Thread_local`、C++ 的 `thread_local`。但这里有个常被忽略的差异——**`thread_local` 不只是 `__thread` 的标准化名字**。

存储机制两者完全相同（还是上面那套 ELF/PE TLS），差别在**初始化和析构**：

| | `__thread` / `_Thread_local` | C++ `thread_local` |
|---|---|---|
| 初始化器 | 必须编译期常量 | 可以运行期动态值 |
| 初始化时机 | TLS 块分配时拷模板 | 常量：同左；动态：**首次使用时（惰性）** |
| 析构 | 无 | 有，`__cxa_thread_atexit` |
| 访问 | 直接取址 | 常量：直接；动态：**调 wrapper** |

- `thread_local int x = 5;` —— 平凡类型 + 常量初始化，编译器把它**降级得和 `__thread` 一模一样**，零额外开销。
- `thread_local std::string s = f();` —— 动态初始化，多一层：编译器为它生成一个 **TLS wrapper 函数 `_ZTW`**，每次访问都经过它「检查 guard → 没初始化就调 `_ZTH` 跑构造 + 登记析构 → 返回地址」。

```cpp
// 用到 s 时：
//   call _ZTW1s             ; TLS wrapper
//   ;   if (!guard) { guard = true; _ZTH1s(); }   ; 构造 + __cxa_thread_atexit 登记析构
//   ;   return <s 的 TLS 地址>
```

为什么非得套个 wrapper？为了**跨翻译单元 / 跨 DSO 的正确性**：动态初始化的变量定义在一个 TU、被另一个引用时，引用方无法知道它初始化没有，所以 C++ ABI 规定这类访问一律走 wrapper，保证「用之前必已初始化」。（这个 per-thread guard 比函数内 `static` 局部变量的 guard 还便宜——每份副本只有本线程碰，**不存在竞争，不用加锁**。）

---

## 六、另一条支线：托管运行时

前面都是 C/C++ 世界。到了带 GC 的托管运行时，TLS 又有一套**纯库实现、完全不碰寄存器和编译器**的玩法。以 Java 为例。

### Java `ThreadLocal<T>`

Java 的 `ThreadLocal` 是 `java.lang` 里一个普通的类，机制是：

- 每个 `Thread` 对象挂着一张 `ThreadLocalMap`——一个定制的**开放寻址哈希表**；
- 这张表的 **key 是 `ThreadLocal` 实例本身，value 是你存的对象**；
- `get()` 大致就是 `Thread.currentThread().threadLocals.get(this)`。

也就是说，Java 的「锚点」是 `Thread.currentThread()`，「定位」是拿这个 ThreadLocal 对象去哈希表里探查。

```java
ThreadLocal<SimpleDateFormat> fmt =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
fmt.get().format(date);   // 每个线程各拿各的 SimpleDateFormat
```

这里有个**递归的彩蛋**：它的锚点 `Thread.currentThread()`，HotSpot 底层又是用一个原生的 `__thread JavaThread*` 指针实现的——**绕一大圈，Java 的 `ThreadLocal` 最终还是落回 FS/GS/TPIDR 那个线程寄存器**。从硬件寄存器到托管哈希表，整条链就这么叠起来了。

几个要点：

- **经典内存泄漏**：`ThreadLocalMap` 的 key 是**弱引用**（允许没人用的 ThreadLocal 被 GC），但 value 是**强引用**。配上线程池里长生命周期的线程，不 `remove()` 就会泄漏——Java 里最有名的坑之一。
- 散列用 `0x61c88647`（黄金比例的整数形式）做增量，让开放寻址分布均匀。
- `InheritableThreadLocal` 让子线程在创建时继承父线程的值。

### 演化还在继续

`ThreadLocal` 这套到了**虚拟线程**（Project Loom）时代开始吃力：几百万个虚拟线程，每个挂一张 `ThreadLocalMap`，内存和语义都成了负担。于是 Java 21 起引入 **`ScopedValue`**——不可变、绑定到一个动态作用域、天然适配结构化并发的替代品。连托管世界的 TLS 也在重新演化，正好呼应「简史」这个标题。

顺带一句别的取舍：.NET 有 `[ThreadStatic]`（偏原生，JIT/运行时支持）和 `ThreadLocal<T>`（偏库）两条路；而 **Go 干脆不提供 goroutine-local**——`g` 指针放在寄存器里供运行时自己用，但语言层面有意不暴露，逼你显式传 `context`。同一个问题，连「要不要给用户这个能力」都能有不同答案。

---

## 七、当下与未来

最后看几个还在发生的变化。

- **TLSDESC（TLS 描述符）**：对 global/local-dynamic 的重写，用「描述符 + resolver」取代那次又慢又要全局锁的 `__tls_get_addr`。动态链接器能给静态 TLS 区的模块填一个「直接返回常量 tpoff」的快 resolver，只给真正惰性的模块填慢的。于是**「动态模型必慢」这个老叙事被改写**——TLSDESC 下 GD 能快到接近 initial-exec。它在 **AArch64 上已是默认**，x86/ARM 上靠 `-mtls-dialect=gnu2` 选用，RISC-V 2024 年才补上。
- **musl vs glibc**：同一个 ELF ABI，两种实现选择。glibc 惰性分配（首次访问可能 `malloc`，带来著名的信号不安全问题）；**musl 干脆在 `dlopen` 时就为所有线程急切分配好**——本质选了「Windows 式急切」，用 dlopen 多干活换访问期简单和信号安全。
- **TCB 不再只住 `__thread` 变量**：线程指针指向的 TCB 区，如今挤进了越来越多「内核 / libc 也要用」的东西。最典型的是 **rseq（restartable sequences，可重启序列；Linux 4.18 引入，glibc 2.35 起每线程自动注册）**——它在 TLS 里放一个 `struct rseq`，**内核每次返回用户态都把当前 CPU 号写进去**，于是用户代码一条 TLS load 就知道「我跑在哪个 CPU 上」，不用系统调用。再配上「线程在临界区中途被抢占或迁移到别的 CPU，就由内核跳到 abort 处理把这段重跑一遍」的机制，就能写出**无锁的 per-CPU 数据结构**：per-CPU 计数器、内存分配器的 per-CPU 快路径（tcmalloc、jemalloc 都用了它）。加上前面提过的栈 canary、pointer guard，**线程指针区早已不只是「存你的线程局部变量」的地方了**。
- **经典文档该更新了**：Ulrich Drepper 的《ELF Handling for Thread-Local Storage》至今是这领域的骨架标准，但它（最后修订也在十多年前）该补上 TLSDESC、AArch64/RISC-V、`thread_local` 的动态初始化层、静态 TLS 余量这个运维现实，以及上面这些 TCB 新住户。

---

## 八、收尾

绕了一大圈，回到开头的 `errno`。

四十年里，TLS 要解决的核心问题从没变过——**每个线程一份**。变的只是那两步「找到锚点、从锚点定位变量」怎么实现：从第一代手动查表的 `pthread_getspecific`，到第二代下沉进段寄存器、一条指令取址的 `__thread`，到第三代被动态链接逼出的四种模型与三家分歧，到第四代语言级的 `thread_local`，再到托管世界里绕回原生 TLS 的 `ThreadLocal`。

而贯穿始终的，是同一道没有标准答案的选择题：

> **快**（预分配、直接取址）、**灵活**（惰性分配、可 dlopen、可跨模块）、**省心**（托管哈希表、自动 GC）——这三者不可兼得。每一代技术、每个平台、每门语言，都在重新选一次。

Windows 赌「预分配换零调用」，吃了 `LoadLibrary` 和静态预算的亏；Linux 选「惰性换灵活」，于是默认慢、又造出四模型让你手动拨快；macOS 统一成一条调用路，简单但永远多一脚；Java 干脆把整套搬到托管层，换来安全和跨类加载，代价是一次哈希探查和一个泄漏陷阱。

没有免费的午餐——这大概是「深入二进制」这些底层机制反复教给我们的同一件事。
