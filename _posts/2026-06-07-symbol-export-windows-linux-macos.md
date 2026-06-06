---
title: "符号导出三国志：Windows、Linux、macOS 动态库的导入导出机制"
date: "2026-06-07 09:00:00 +0800"
categories: [System, Linking]
tags: [Dynamic Library, Linker, ELF, PE, Mach-O, GOT, PLT, dllimport, Visibility]
layout: post
published: true
---

> 写跨平台 C/C++ 库的人，大概都被 Windows 的 `__declspec(dllimport)` / `dllexport` 折磨过，换到 Linux / macOS 却什么都不用写。本文把 PE / ELF / Mach-O 三家「动态库符号到底怎么访问」的底层机制——IAT / GOT / PLT、copy relocation、two-level namespace、可见性控制——讲清楚，看三种不同的设计权衡。

写过跨平台 C/C++ 库的人，大概都被 Windows 的 `__declspec(dllexport)` / `__declspec(dllimport)` 折磨过：一会儿要导出，一会儿要导入，换到 Linux 和 macOS 却什么都不用写。这不是 Windows 故意找麻烦，而是三个平台在「动态库符号如何访问」这件事上做了**不同的设计权衡**。

这篇先把三家的底层机制讲清楚，下一篇再专门讲由此引出的 DLL 工程难题。

## 一切的起点：地址在运行时才确定

动态库（`.dll` / `.so` / `.dylib`）的存在意义就是「加载时才链接」。这带来一个根本问题：库里某个函数或变量的真实地址，要等加载器把库映射进进程地址空间后才知道，而且每次运行、每个进程都可能不同（ASLR、被加载到任意基址）。

可执行文件里的代码却是编译期就生成好的。它怎么去访问一个运行时才定址的符号？

答案三家一致：**加一张运行时填充的指针表，代码通过这张表间接寻址。** 区别在于这张表叫什么、什么时候走它、要不要程序员显式声明。

| | Windows (PE) | Linux (ELF) | macOS (Mach-O) |
|---|---|---|---|
| 间接表 | IAT | GOT | `__got` / `__la_symbol_ptr` |
| 函数跳板 | thunk | PLT | `__stubs` + `dyld_stub_binder` |

- **IAT**（Import Address Table，导入地址表）
- **GOT**（Global Offset Table，全局偏移表）
- **PLT**（Procedure Linkage Table，过程链接表）

名字不同，思路一样：表里每个符号一个槽，加载器把真实地址填进去，代码读槽拿地址再访问。

## 核心分歧：间接是「默认」还是「opt-in」

三家真正的区别，是**间接访问到底是不是默认行为**。

### Windows：默认本地直连，跨模块要显式声明

PE 的设计哲学是「默认最快」。编译器默认假设符号是本模块的，生成**直接寻址**指令。想跨 DLL 访问，必须用 `__declspec(dllimport)` 显式告诉编译器「这玩意儿在别的模块，请走 IAT」。

```cpp
__declspec(dllimport) extern int g_value;   // 显式声明：走 IAT 间接
int x = g_value;
```

不加 `dllimport`，编译器就按本地变量处理，结果出错（后面细说）。

### Linux：PIC 下统一走 GOT，无需声明

ELF 的哲学相反。在位置无关代码（PIC，`.so` 必须 `-fPIC`）下，编译器对**任何可能是外部的全局符号**一律生成「经 GOT 间接」的代码——**不需要预先知道它是本地还是在某个 `.so` 里**：

```c
extern int g_value;     // 零标注
int x = g_value;
// 编译成：addr = *GOT[g_value];  x = *addr;
// 动态链接器加载时把真实地址填进 GOT 槽
```

这段代码无论 `g_value` 最终在本模块还是在共享库里都正确。所以根本不需要 `dllimport`。

### macOS：同样走 GOT，无需声明

Mach-O 和 ELF 一脉相承：PIC 下数据经 `__got`（non-lazy pointer）间接，dyld 加载时填好。所以变量跨 `.dylib` 也不用任何标注。

## 为什么变量比函数更麻烦

这是 `dllimport` 之痛的核心，也是理解三家差异的钥匙。

**函数**有个救济机制：链接器能自动生成一小段跳板（thunk / PLT stub）：

```asm
foo:  jmp [__imp_foo]   ; 跳到 IAT/GOT 槽指向的真实函数
```

所以即使你不写 `dllimport`，调用 `foo()` 会落到这个 thunk，thunk 再跳真实地址。多一次跳转，但**结果正确**。三家的函数都因此可以免标注。

**变量没有 thunk**。在 Windows 上，如果不加 `dllimport`：

```cpp
extern int g_value;   // 缺 dllimport
int x = g_value;
```

链接器在导入库里只有 `__imp_g_value`（IAT 槽地址），于是把 `g_value` 解析到了 **IAT 槽本身**。代码直接读这个地址，读到的是**槽里存的那个指针**（即 `g_value` 的地址，一个看似随机的大数），而不是 `g_value` 的值 `42`：

```
__imp_g_value  ──►  [ IAT 槽 ]  ──►  g_value 真实地址  ──►  实际的值 42
   (槽地址)         (槽内容=指针)        (DLL 里的变量)

加了 dllimport：读槽 → 拿地址 → 解引用 → 42  ✓
没加 dllimport：直接把槽内容当值用 → 拿到指针数值 ✗
```

差了一层解引用，结果错乱。这就是为什么 **Windows 跨 DLL 用变量基本必须 `dllimport`，而函数不用**。

## ELF 为什么连标注都省了

Linux 能做到「函数变量都免标注」，靠三个东西配合：

**1. PIC + GOT 默认间接**：前面说过，编译器对外部符号统一走 GOT，把「间接」做成了结构性默认。

**2. 符号抢占（preemption）**：ELF 默认 visibility 下，任何全局符号都可以被运行时覆盖（`LD_PRELOAD` 就靠这个）。既然任何符号都可能在别处，编译器只能保守地全部走 GOT 间接。于是「间接」成了默认，标注变得多余。

**3. copy relocation**：对「非 PIE 可执行文件引用 `.so` 里的变量」这种情况，ELF 还有一招——链接器在 exe 的 `.bss` 里分配一份同名副本，发 `R_X86_64_COPY`，加载时由动态链接器把 `.so` 里的初始值 memcpy 进副本，于是直接访问也对，无需标注。

但这一招代价不小，是 ELF 一个有名的坑源（下面单独讲）。代价就是 PE 想避免的那点开销：即使符号最后是本地的，也白白走了一次 GOT 间接；符号抢占还增加了优化难度。想要回 Windows 那种「本地直连」的速度，ELF 用 `-fvisibility=hidden` + `-fno-semantic-interposition` 显式收窄——这恰好是和 `dllexport` 对称的反向操作。

### copy relocation 的代价

memcpy「一份字节副本」这件事本身就埋了三类问题：

- **大小冻结 → ABI 脆弱**：副本大小在 exe 链接期就定死。`.so` 升级把变量变大，exe 里的旧尺寸副本就溢出。所以「给导出的全局变量加字段」在 ELF 下是 ABI break，根源就是 copy reloc。
- **C++ 对象：构造被绕过 + 资源双重释放**：copy reloc 纯字节拷贝、不重跑构造函数。`.so` 在原始对象上构造，加载器把字节 memcpy 进 exe 副本——于是有了两个字节相同的对象，指针成员指向同一块堆资源。退出时析构在原始对象上跑一次、`free` 掉资源，而 exe 一直用的副本里那个指针已悬空 → use-after-free；两边都触发释放就是 double free。准确说是：对象作为 C++ 对象从未被正确构造/析构（memcpy 绕过了语义），两个副本共享内部指针导致资源重复释放或悬空。
- **自引用指针失效**：对象里指向自身或自己成员的指针，字节副本在不同地址，指针仍指向原始对象 → 悬空。

好在 copy reloc 只在**非 PIE** 可执行文件里发生。现代发行版多数默认 **PIE**，PIE 的 exe 引用外部数据也走 GOT 间接，根本不产生 copy reloc。所以这坑在 PIE-by-default 之后大幅减少——但 `-no-pie`、老工具链、某些嵌入式场景仍会撞上。

## Mach-O 的两个特色

macOS 大方向跟 ELF 一样（GOT 间接、visibility 控制导出），但有两个鲜明的自己的设计。

### Two-level namespace（双层命名空间）

ELF 默认是 flat namespace：导入符号只记名字，谁提供都行，所以容易被抢占（也容易意外冲突）。Mach-O 默认每个导入符号同时记录**符号名 + 来自哪个 dylib**：

```
ELF 导入:    "g_value"                   （谁提供都行 → 可被抢占）
Mach-O 导入: ("libfoo.dylib", "g_value")  （绑定到具体 dylib）
```

效果：macOS 默认**更少意外抢占**（精神上更接近 Windows「知道符号从哪来」），但仍不需要 `dllimport`。想要 ELF 式行为得显式 `-flat_namespace`；要做 interpose 得用 `DYLD_INTERPOSE` / `DYLD_INSERT_LIBRARIES`。

### 不用 copy relocation

ELF 里主程序引用 `.so` 数据可能走 copy relocation（复制进 exe 的 BSS），有 size 必须固定、与 `-fPIC` 共享数据冲突等坑。Mach-O **从不这么做**——数据一律走 `__got`，主程序引用 dylib 变量也是间接。更统一、更干净，没有 copy reloc 那些边角问题。

## 导出端：也是对称的两种哲学

前面讲的是「导入端」。导出端同样体现了两种哲学：

- **PE**：默认不导出，想导出用 `__declspec(dllexport)` **opt-in 导出**
- **ELF / Mach-O**：默认全导出（历史默认），想隐藏用 `-fvisibility=hidden` **opt-in 隐藏**

有意思的是，业界后来都「抄」了 Windows 的方向：`-fvisibility=hidden` 现在是所有正经库的推荐做法（Chrome、各大 `.so` 都这么干），GNU 官方文档自己都承认默认导出全部是历史遗留问题。所以「默认不导出」这个方向 Windows 一开始就对了，是另外两家在补课。

## 顺带一提：64 位让 PIC 几乎免费

ELF/Mach-O 全程 GOT 间接，听起来很贵，为什么大家不在乎？因为 **x86-64 的 RIP-relative 寻址**。

32 位时代 PIC 很贵：没有 PC 相对寻址，要靠 `call/pop` 取当前地址，还要**烧掉一个寄存器（EBX）**常驻当 GOT 基址，本就紧张的 x86 寄存器雪上加霜。

x86-64 加了 RIP-relative（相对指令指针直接编码偏移）：
- 访问本模块代码/数据 → 一条 RIP 相对指令，和非 PIC 一样快，不烧寄存器
- 只有跨模块符号才走 GOT/PLT，而这层开销你做动态链接本来就要付

所以 64 位下 PIC 的成本几乎只剩「跨模块访问的那一次间接」，本地访问近乎免费。

如果你写非 PIC 代码再想链进 `.so`，会撞上这个经典错误：

```
relocation R_X86_64_32 against `.text' can not be used
when making a shared object; recompile with -fPIC
```

非 PIC 代码在 small code model 下假设一切在低 2GB，引用符号时发**绝对 32 位重定位**（`R_X86_64_32`）。但 `.so` 可被加载到任意高地址，32 位绝对值装不下，链接器只能拒绝。加 `-fPIC` 后改用 `R_X86_64_PC32`（RIP 相对）和 GOT 系重定位，与基址无关，放进 `.so` 才安全。

## 三平台总览

| | PE (Windows) | ELF (Linux) | Mach-O (macOS) |
|---|---|---|---|
| 间接表 | IAT | GOT | `__got` / `__la_symbol_ptr` |
| 跨模块函数 | thunk，免标注 | PLT，免标注 | stub，免标注 |
| **跨模块数据** | **需 `dllimport`** | 免标注（GOT / copy reloc） | 免标注（GOT） |
| 主程序引用库数据 | IAT | GOT 或 copy reloc | **只用 GOT** |
| 命名空间 | 类似双层（按 DLL） | flat（默认可抢占） | **two-level（默认）** |
| 导出控制 | `dllexport` opt-in 导出 | visibility opt-in 隐藏 | visibility opt-in 隐藏 |
| PIC/非 PIC 之分 | 无（base relocation） | 有（`-fPIC`） | 基本都 PIC |
| 默认哲学 | 默认快，标注换跨模块 | 默认通用可抢占 | 默认两层，GOT 统一 |

## 结语

三家在「动态库符号访问」上各做了一套自洽的权衡：

- **Windows**：默认本地直连最快，跨模块数据用 `dllimport` 显式买那一层间接。换来速度，代价是标注负担。
- **Linux**：PIC 下统一走 GOT，符号默认可抢占。换来「写一次、跨角色统一」的对称体验，代价是一点点性能和优化空间。
- **macOS**：同样 GOT 间接免标注，但用 two-level namespace 减少抢占、用纯 GOT（不碰 copy reloc）处理主程序引用库数据，比 ELF 更统一干净。

一句话：**PE 选了纪律（默认不导出），Unix 选了对称（写一次通吃），没有免费的午餐。**

而那个「只有 Windows 才需要 `dllimport`」的设计，会在工程上引出一连串麻烦——头文件方向不一致、静态动态双编译、构建系统得专门处理。这些就留到[下一篇《Windows DLL 导出之痛》]({% post_url 2026-06-07-windows-dll-export-pain %})细说。

## 参考资料

- Ulrich Drepper, [*How To Write Shared Libraries*](https://akkadia.org/drepper/dsohowto.pdf) —— ELF/DSO 机制的权威长文。本文 Linux 部分涉及的 GOT/PLT、符号可见性、符号抢占（preemption）、copy relocation、`-fvisibility=hidden` 等，这里都有更深入的展开，想吃透 ELF 动态库必读。
- Raymond Chen, [*Index to the series on DLL imports and exports*](https://devblogs.microsoft.com/oldnewthing/20060727-04/?p=30333) —— 微软 Raymond Chen 2006 年关于 Windows DLL 导入/导出的系列总目（含「导入库里的名字为何要修饰」「32 位 Windows 如何导出 DLL 函数」等），正对应本文 Windows 部分的 IAT / thunk / 导入库 / `dllimport` 机制。

