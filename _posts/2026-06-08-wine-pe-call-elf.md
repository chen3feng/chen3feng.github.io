---
title: "在 Wine 里打开并读一个文件，到底发生了什么"
date: "2026-06-08 09:00:00 +0800"
categories: [System, Wine]
tags: [Wine, PE, ELF, Syscall, FSGSBASE, TLS, ntdll, wineserver, IPC]
layout: post
published: true
---

> 一个 Windows 程序编译成 PE，Linux 的库编译成 ELF——两种格式天差地别，本来连放进同一个可执行文件都做不到。可 Wine 跑起来后，Windows 程序里一句 `CreateFile` + `ReadFile`，最终却落到了 ELF 里的 `open()`、`read()`。这中间到底经过了什么？拆开来看，背后藏着三种不同的"跨界"机制：进程内的 NT 系统调用门、进程内的 unixlib 门，以及一道跨进程通到"用户态微内核"的门。本文用打开并读一个文件这条线，把三者串起来讲清楚。顺带还有个隐藏难题：Linux 用 FS、Windows 用 GS 做线程本地存储，同一个线程怎么同时伺候两套约定？

前一阵子盘点各种二进制跨平台技术时，写了篇文章，简单提到了 Wine，它是一个可以在 Linux 或者 MacOS 上运行 Windows 可执行文件的工具。不过其实我只是大概了解它的基本原理，也就是 API 翻译，并没有深入研究过它的代码。

但是今天在上班路上时，我突然想到了两个问题：

- Wine 会把 Windows API 调用转为 Linux 中的翻译层，但是 ReadFile 不可能直接调到到 Linux 中的 read，因为两者存在的可执行文件格式都不一样，根本没法链接到一起。
- 在 x86-64 上实现 TLS 时，Windows 和 Linux 的寄存器是反过来的，一个进程中怎么同时兼容两套机制？

带着这两个问题，挖掘了一下源代码，都搞清楚了，记下来以免以后忘记 

先简单介绍下 [Wine](https://www.winehq.org/)。它的名字是 "Wine Is Not an Emulator" 的递归缩写——刻意强调自己**不是模拟器**。它不模拟 CPU 指令，也不像虚拟机那样跑一整个 Windows 内核；它做的是**兼容层**：把 Windows 程序用到的那套 API（`kernel32`、`user32`、`ntdll` 等成百上千个 DLL）在 Linux/macOS 上**重新实现一遍**，让 Windows 的 `.exe` 直接当作本地进程跑起来。因为不带模拟，原生指令直接在 CPU 上执行，性能损失很小——代价是 Wine 必须自己把 Windows 那一整套运行时和加载机制在用户态搭出来。

正因为"不模拟、不带内核"，今天这个朴素的好奇才有意思：Windows 程序最终还是得落到 Linux 的系统调用上才能干活。可问题是，Windows 的 `.exe` 是 **PE** 格式、调用的是 NT API；Linux 的 `.so` 是 **ELF** 格式、调用的是 Linux 系统调用。这两种东西**编译完格式完全不同，本来就不该出现在同一个 image 里**。

那一句最普通不过的"打开文件、读文件"，在 Wine 里到底是怎么走通的？我顺着源码把这条线拆了一遍，记在这里。

## 一、背景：两种格式，本就不该在一起

先说清楚"为什么这是个问题"。

PE（Windows 的 `.exe` / `.dll`）和 ELF（Linux 的可执行文件和 `.so`）是两套完全独立的二进制容器格式：文件头不同、节区布局不同、加载和重定位规则不同、连导入符号的方式都不同（PE 走 IAT，ELF 走 GOT/PLT）。**宿主的加载器 `ld.so` 只认 ELF，根本不知道怎么解析 PE。** 反过来 Windows 的加载器只认 PE。

所以你没法像平时那样，把一个 Windows DLL 和一个 Linux `.so` 链接进同一个文件、互相 `call` 过去——它们各自的加载器谁也不认识对方。

可 Wine 就是得让 Windows 程序在 Linux 进程里跑起来，最终还得调到 glibc 的 `read`、`write`、`open`。于是问题拆成几层：

1. 怎么让 PE 模块和 ELF 模块**同时待在一个进程的地址空间里**？
2. 待在一起之后，PE 里的代码怎么**调用**到 ELF 里的函数？
3. 像"句柄"这种跨进程共享的东西，又是谁在管？

一个看似简单的打开文件，涉及到了这三层面的东西。

## 二、先让它们共处一室

对 Windows 系统稍有了解的都知道，kernel32 里的 API 只是个包装翻译层，真实的系统调用都是调 ntdll 中的 native API 实现的，因此我们绕过 kernel32，直接看 ntdll 是怎么实现的。

挖掘源代码发现，Wine 的 `ntdll` 不是一个文件，而是**一分为二**的。打开 [dlls/ntdll/Makefile.in][mk] 头两行就点题了：

```make
MODULE    = ntdll.dll      # PE 侧：交叉编译成真正的 PE，跑在 Windows 程序的地址空间里
UNIXLIB   = ntdll.so       # Unix 侧：用本机编译器编成宿主原生 ELF，真正调系统调用
```

| | `ntdll.dll`（PE 侧） | `ntdll.so`（Unix 侧） |
|---|---|---|
| 格式 | PE | 宿主原生（Linux→ELF，macOS→Mach-O） |
| 编译器 | MinGW 风格交叉编译器 | 本机 gcc / clang |
| 职责 | 对 Windows 程序暴露 NT API | 真正调用 `open`/`read`/`mmap` |
| 谁来加载 | Wine 自己写的 PE loader | 宿主的 `dlopen` / `ld.so` |

加载顺序是**先 ELF 后 PE**：

整个 Wine 进程的起点是一个原生可执行文件 `wine`。它的 `main()` 第一件正经事，就是用**宿主的 `dlopen`** 把 `ntdll.so` 加载进来——见 [loader/main.c][main]：

```c
handle = dlopen( ".../ntdll.so", RTLD_NOW );          // 宿主 ld.so 加载 ELF
void (*init_func)(int, char**) = dlsym( handle, "__wine_main" );
init_func( argc, argv );                              // 控制权交给 Unix 侧
```

`ntdll.so` 是宿主原生 ELF，所以这一步走的是**操作系统自己的加载器**，规规矩矩。这也是整个进程里**唯一**一个由宿主 loader 加载的 Wine 模块。

接下来 `__wine_main` 初始化时，再反过来把 PE 侧的 `ntdll.dll` 拉进来——见 [load_ntdll()][load_ntdll]：

```c
status = open_builtin_pe_file( name, &attr, &module, &size, &info, ... );
```

注意这里**没有再调 `dlopen`**——宿主加载器不认 PE。Wine 自己实现了一个 PE 加载器：打开 `.dll`，按 PE 头把各节 `mmap` 到约定基址（x86-64 上是 `0x170000000`），做好重定位。

到这一步，**同一个进程的地址空间里就同时装下了两种格式**：高地址那块是宿主 loader 正常加载的 ELF（`ntdll.so` 和 glibc），低地址那块是 Wine 自己 mmap 进来的 PE 模块。物理上它们在一起了——但"在一起"不等于"能互相调用"，这才是核心。

## 三、先看 ReadFile，再深入 CreateFile

倒过来讲最直观：先看"读"——它最能体现 PE→ELF 这条线；再回头看文件当初是怎么"打开"的，那一步藏着更深的东西。

一个 Windows 程序调 `ReadFile`，在 Wine 里大致这样走：

```
  Windows 程序 (PE)
      │  ReadFile → NtReadFile            ← ntdll.dll 里的桩，只有一个系统调用号
      ▼
 ① NT 系统调用门  __wine_syscall_dispatcher  ← 存上下文、切栈、切 FS 段寄存器
      ▼
  unix/file.c: NtReadFile (ELF)           ← 真正的实现，跑在原生世界
      │
      ├─ 第一次：③ wine_server_call(get_handle_fd) 把真 fd 经 socket 取回来，并缓存
      │
      ▼
  对这个 unix fd 直接 pread()              ← 之后再读就不打扰 wineserver 了
```

读到最后，就是一句最普通的 `pread`。这条线已经踩到两道门：一道**进程内的 PE→ELF 门**（① NT 系统调用门），一道**跨进程门**（③ wineserver）。

但这里留了个悬念：`pread` 要的那个 unix fd、以及 `ReadFile` 一开始拿到的那个 `HANDLE`，到底是哪来的？顺着往回找，就到了更早的 `CreateFile`：

```
  Windows 程序 (PE)
      │  CreateFileW → NtCreateFile
      ▼
 ① NT 系统调用门 → unix/file.c: NtCreateFile
      ▼
 ③ wine_server_call(create_file)           ← 把 unix 路径发给 wineserver
      ▼
  wineserver 进程：亲自 open() 真文件，登记句柄，返回一个 Windows HANDLE
```

关键就藏在这一步：**打开文件这个动作，并不是在当前进程里 `open()` 的，而是发生在另一个进程 `wineserver` 里**——它替所有 Windows 进程统一开文件、统一发句柄。为什么要这么绕，第六节讲。

这样，"打开 + 读一个文件"就把三种机制点齐了。其中 ① 和下面要讲的 ② `__wine_unix_call` 是一对孪生（都是进程内 PE→ELF），③ 是唯一跨进程的。下面按"先讲最清楚的 ②，再讲它的孪生 ①，最后讲跨进程的 ③"展开。

## 四、机制二：`__wine_unix_call` —— 进程内按表调用

为什么不能让 PE 代码直接 `call` ELF 里的函数地址了事？两个原因：PE 侧是交叉编译产物，**编译期根本不知道** ELF 那边函数的地址；更重要的是，PE 代码跑在 Windows 的运行环境约定下（段寄存器、栈、TEB 全是 Windows 那套），**不能就地切进 glibc 的世界**，否则 glibc 一用线程本地存储就崩（第七节细说）。

所以 Wine 设了一道**门**：PE 侧不直接调 ELF 函数，而是"报一个编号，让门后面的派发器去查表调用"。

入口朴素得超出预期，在 [include/wine/unixlib.h][unixlib]：

```c
extern NTSTATUS (WINAPI *__wine_unix_call_dispatcher)( unixlib_handle_t, unsigned int, void * );

static inline NTSTATUS __wine_unix_call( unixlib_handle_t handle, unsigned int code, void *args )
{
    return __wine_unix_call_dispatcher( handle, code, args );
}
```

三个参数：`handle` 是这个模块的 unixlib 句柄——**它其实就是 Unix 侧那张函数表的地址**；`code` 是要调第几个函数，也就是表里的**下标**；`args` 是参数结构体指针。

门后面要调的"ELF 函数"，被排成一张数组。`ntdll` 自己的表在 [loader.c][funcs]：

```c
static const unixlib_entry_t unix_call_funcs[] =
{
    load_so_dll,
    unwind_builtin_dll,
    unixcall_wine_dbg_write,
    unixcall_wine_server_call,
    ...
};
```

每个元素就是一个原生函数指针，类型 [`NTSTATUS (*)(void *args)`][typedef]。所谓 `code` 就是这张表的下标。**PE 调 ELF 的本质到这里就清楚了：不是按地址调，是"按编号查表调"。**

真正干活的派发器是一段手写汇编，在 [signal_x86_64.c][udisp]。删掉无关细节，骨架是：

```c
__ASM_GLOBAL_FUNC( __wine_unix_call_dispatcher,
    "movq %rcx,%r10\n\t"            /* r10 = handle，也就是函数表基址 */
    "movq %gs:0x378,%rcx\n\t"       /* 取出本线程预留的 frame */
    /* ... 把通用寄存器、xmm6-15、MxCsr 全部存进 frame ... */
    "movq %rcx,%rsp\n\t"            /* 切换到内核栈 */
    /* ... 切 FS 基址，见第七节 ... */
    "movq %r8,%rdi\n\t"            /* args 作为第一个参数 */
    "callq *(%r10,%rdx,8)\n\t"     /* 关键这一句：调用 funcs[code](args) */
    /* ... 恢复寄存器，切回用户栈，返回 PE 侧 ... */
)
```

大半篇幅都在**保存和恢复寄存器**——因为接下来要执行原生 C 代码，回来时 PE 侧上下文必须一字不差。真正"调用"只有一句 `callq *(%r10,%rdx,8)`：`%r10` 是函数表基址，`%rdx` 是下标，合起来就是 `funcs[code](args)`——一句话从 PE 跨进 ELF。

挑个最干净的例子：Wine 里 PE 代码打 `TRACE`/`ERR` 日志，最后要写到 stderr，而 `write()` 是 glibc 的活。它走的就是这道门，落点在 [debug.c][dbg]：

```c
NTSTATUS unixcall_wine_dbg_write( void *args )
{
    struct wine_dbg_write_params *params = args;
    return write( 2, params->str, params->len );   // 真正的 libc write
}
```

PE 侧攒好字符串 → 报编号进门 → 汇编派发器存上下文、切栈、切 FS → `callq` 到 `unixcall_wine_dbg_write` → ELF 侧一句 `write(2, ...)` 落到内核。这就是一次完整的 PE→ELF 调用。

## 五、机制一：`__wine_syscall_dispatcher` —— NT 系统调用门

回到文件这条线。`NtCreateFile`、`NtReadFile` 这些 `Nt*` 函数，走的不是上面那道 `__wine_unix_call`，而是一道**孪生的门**。

区别在 PE 侧：`Nt*` 在 `ntdll.dll` 里**不是普通函数，而是一段只带"系统调用号"的汇编桩**，由 [signal_x86_64.c][sysent] 里的宏批量生成：

```c
#define SYSCALL_ENTRY(id,name,args) __ASM_SYSCALL_FUNC( id, name )
```

这和真实 Windows 上 `Nt*` 是 `syscall` 指令陷入内核的形态一模一样——只不过在 Wine 里，它"陷入"的不是内核，而是 Unix 侧的派发器 [`__wine_syscall_dispatcher`][sysdisp]。这道门和第四节那道几乎逐行相同：同样从 `%gs` 取 frame、同样保存上下文、同样切 FS、同样切进 ELF，唯一的差别是——

- `__wine_unix_call` 按 `code` **查函数表**调用；
- `__wine_syscall_dispatcher` 按**系统调用号查 NT 系统调用表**调用。

派发的终点，就是 Unix 侧真正的实现。`NtCreateFile` / `NtReadFile` 都在 [unix/file.c][ntcreate] 里。换句话说：**Wine 把 Windows 的"系统调用"在用户态重新实现了一遍**——PE 侧负责发起，ELF 侧负责落地。

所以理解了机制二，机制一也就懂了：它俩是同一个思路（进程内 PE→ELF 跨界）的两个版本，一个查表、一个查号，共用同一套"存上下文 + 切 FS"的机器。

## 六、机制三：`wine_server_call` —— 跨进程的"用户态微内核"

**文件句柄**这种东西，得能在多个 Windows 进程之间共享、能被统一管理生命周期和权限。这种状态不能各进程自己存，于是 Wine 交给一个**独立的进程** `wineserver` 统一持有——其实就是个跑在用户态的"微内核"，管着句柄、注册表、同步对象、窗口等等。要请它办事，就得跨进程通信，这道门叫 `wine_server_call`，底层是一次**同步的请求/应答 IPC**：每个线程和 server 之间一对管道，请求用 `writev` 发过去，然后**阻塞 `read`** 等应答。这就是它比前两道门贵得多（µs 级 vs ns 级）的原因——是真正的跨进程往返。

先接着 ReadFile 讲。Unix 侧的 `NtReadFile` 要读，得先把抽象的 `HANDLE` 换成真正的 unix fd，靠 [server_get_unix_fd()][getfd]：

```c
ret = get_cached_fd( handle, &fd, ... );           // ① 先查本地缓存
if (ret != STATUS_INVALID_HANDLE) goto done;       //    命中就直接用，不打扰 server

SERVER_START_REQ( get_handle_fd )                  // ② 没缓存才问 server
{
    req->handle = wine_server_obj_handle( handle );
    if (!(ret = wine_server_call( req )))
    {
        fd = wine_server_receive_fd( &fd_handle );  //    经 SCM_RIGHTS 收下真 fd
        *needs_close = !add_fd_to_cache( handle, fd, ... );  // ③ 缓存起来
    }
}
SERVER_END_REQ;
```

管道只能传字节、传不了文件描述符，所以这里用一条专门的 **Unix 域 socket + `SCM_RIGHTS`** 把 fd 从 server 进程"递"过来。注意这个 **fd 缓存**：第一次读文件要跨进程问 server 要 fd，拿到后就缓存在本进程；之后 `NtReadFile` 直接对这个 fd `pread`，**不再打扰 wineserver**——见 [file.c][readfile]：

```c
result = virtual_locked_pread( unix_handle, buffer, length, offset->QuadPart );
```

这正是 Wine 性能优化的核心思路：**把热路径上的跨进程往返尽量省掉**。

现在往深一层，回到第三节那个悬念：这个 `HANDLE`、这个 fd，最初是谁造的？是 `CreateFile`。Unix 侧的 `NtCreateFile` 并不自己 `open()`，而是把解析好的 unix 路径**发给 wineserver，让它去开**——见 [open_unix_file()][openfile]：

```c
SERVER_START_REQ( create_file )
{
    req->access  = access;
    ...
    wine_server_add_data( req, unix_name, strlen(unix_name) );
    status = wine_server_call( req );                       // 跨进程：请 server open()
    *handle = wine_server_ptr_handle( reply->handle );      // 拿回一个 Windows HANDLE
}
SERVER_END_REQ;
```

由 wineserver 统一 `open()`、统一发 `HANDLE`，正是为了让句柄能跨进程共享、生命周期可控——这就是它"微内核"身份的由来。开文件这一下绕不开 server，但开完之后的成千上万次读写，靠 fd 缓存都在进程内 `pread` 解决。

## 七、贯穿两道门的难题：TLS 段寄存器

我早就知道**x86-64 上实现线程本地存储（TLS）时，Windows 和 Linux 选的段寄存器正好是反过来的**——一个用 GS，一个用 FS。

| | 线程指针用哪个段寄存器 | 指向什么 |
|---|---|---|
| Linux x86-64 | **FS** | glibc 的 TCB（`errno`、`pthread`、`malloc` 锁都靠它） |
| Windows x64 | **GS** | TEB（Thread Environment Block） |

（顺带一段历史：32 位时代正好反过来，Win32 用 FS 指 TEB，Linux 用 GS 做 TLS；到 64 位两边都"换了一只手"，最后恰好交叉。）

这在一个系统里显然没问题，让我好奇的是：Wine 要让两套代码在**同一个线程**里来回切，这"反过来"该怎么解决呢？矛盾在这里：同一个 Wine 线程，**跑 PE 代码时 GS 必须指向 TEB**，否则 Windows API 拿不到线程信息；**跑 ELF 代码时 FS 必须指向 glibc 的 TCB**，否则 glibc 一碰 `errno`、一上锁就崩。

Wine 的办法是**分工**：

- **GS 全程留给 TEB。** Linux 上 GS 用户态本来基本闲着，Wine 接管它常驻指向 TEB。所以 PE 代码里 `%gs:0x30` 永远能拿到 TEB，派发器里 `movq %gs:0x378` 取 frame 也是这么来的。
- **FS 在每次进/出门时切换。** 要执行 glibc 代码前把 FS 切回 glibc 的线程基址，回 PE 侧前再切回去。

前面机制一、机制二的汇编里反复出现的"切 FS 基址"，就是答案所在。这段代码在派发器里，见 [signal_x86_64.c][udisp]：

```c
#ifdef __linux__
    "cmpb $0,0x7ffe028a\n\t"        /* ProcessorFeatures[PF_RDWRFSGSBASE_AVAILABLE]？ */
    "jz 1f\n\t"
    "wrfsbase %rsi\n\t"             /* 有 FSGSBASE：一条指令直接写 FS 基址 */
    "jmp 2f\n"
    "1:\tmov $0x1002,%edi\n\t"      /* ARCH_SET_FS */
    "mov $158,%eax\n\t"             /* SYS_arch_prctl */
    "syscall\n\t"                   /* 没有：退化成一次真正的内核 syscall */
    "2:\n\t"
#endif
```

值得注意的是这里有一个明显的跳转指令，其实藏着一个性能差异：

- 如果**CPU 支持 FSGSBASE**（Intel Ivy Bridge / AMD 2012 年起，且内核已开启）指令，用户态十几纳秒就切完；
- **不支持**时只能退回 `arch_prctl(ARCH_SET_FS)`——这是一次**真正的内核系统调用**，每次进出门都多几百纳秒。

能不能用 FSGSBASE，Wine 启动时会查 CPUID 并核对内核是否放行，存进 `user_shared_data`，见 [system.c][sys]：

```c
features[PF_RDWRFSGSBASE_AVAILABLE] = !!(regs[1] & (1 << 0));          // CPUID leaf 7
#if defined(__linux__) && defined(AT_HWCAP2)
features[PF_RDWRFSGSBASE_AVAILABLE] &= !!(getauxval( AT_HWCAP2 ) & 2);  // 内核确实开了吗
#endif
```

汇编里那句 `cmpb $0,0x7ffe028a` 比的就是这个标志。（macOS 上没有 `wrfsbase` 这条路，Wine 改用 `_thread_set_tsd_base` 这个 Mach syscall 切基址，所以那边每次跨界都是 syscall 级开销。）

## 八、收尾

回到开头那句"打开并读一个文件"，现在能完整答出来了：

> Wine 先把 PE 模块和 ELF 模块装进同一个进程——ELF 由宿主 `dlopen` 加载，PE 由 Wine 自己写的加载器 `mmap` 进来。`CreateFile` 落到 `NtCreateFile`，经 **NT 系统调用门**（存上下文、把 **FS 段寄存器切回 glibc**）跨进 ELF 侧的实现；实现再经 **`wine_server_call`** 这道跨进程门，请 `wineserver` 真正 `open()` 文件、登记句柄。`ReadFile` 同样先过系统调用门，第一次经 `SCM_RIGHTS` 把真 fd 取回并**缓存**，之后就在进程内直接 `pread`，不再打扰 server。

三种机制职责分明：

| 机制 | 边界 | 怎么调 | 开销量级 | 典型用途 |
|---|---|---|---|---|
| `__wine_syscall_dispatcher` | 进程内 PE→ELF | 按系统调用号查表 | 几十 ns | `Nt*` 系列 |
| `__wine_unix_call` | 进程内 PE→ELF | 按 `code` 查函数表 | 几十 ns | unixlib（驱动、网络、调试输出…） |
| `wine_server_call` | **跨进程** | 管道 IPC + `SCM_RIGHTS` | 几 µs | 句柄、注册表、同步对象、窗口 |

这两套二进制格式没法在链接层面融合，Wine 就在**运行时**用软件的门把它们接起来；FS/GS 的分工，是为了让同一个线程能在两套世界里来回切身份还不出乱子；而把 fd 缓存进进程、少进 server，则是这套架构里最要紧的性能优化。

所以说，Wine 是在用户态建了一套"**系统调用机制**"：前两道门是用户态的系统调用，这样，Windows 上的 `ReadFile`，就这样在 Linux 上走通了。

想起当年刚学编程时，看过不少台湾的计算机图书作译者侯捷的书，他说过一句话：“源码面前，了无秘密“。这种通过挖掘源码破解疑惑的感觉还是挺美妙的。

[mk]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/Makefile.in#L2-3
[main]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/loader/main.c#L163
[load_ntdll]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/unix/loader.c#L1687
[unixlib]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/include/wine/unixlib.h#L295
[funcs]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/unix/loader.c#L1008
[typedef]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/include/wine/unixlib.h#L35
[udisp]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/unix/signal_x86_64.c#L3309
[dbg]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/unix/debug.c#L273
[sysent]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/signal_x86_64.c#L45
[sysdisp]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/unix/signal_x86_64.c#L2981
[ntcreate]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/unix/file.c#L4648
[openfile]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/unix/file.c#L4627
[getfd]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/unix/server.c#L1156
[readfile]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/unix/file.c#L6082
[sys]: https://gitlab.winehq.org/wine/wine/-/blob/b9f5aa42b1532a80963583cabeaeec2a4b479d9f/dlls/ntdll/unix/system.c#L519
