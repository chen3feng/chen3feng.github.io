---
title: "Windows DLL 导出之痛：从 dllimport 到构建系统"
date: "2026-06-06 10:00:00 +0800"
categories: [System, Build]
tags: [Windows, DLL, dllexport, Import Library, CMake, Blade, MinGW, Linker]
layout: post
published: true
---

> 「只是多写个 `dllimport`」的差异，在工程上怎么一步步变成构建系统的麻烦？从头文件三态、`/EXPORT` 烤进 .obj 逼出双编译，到 `.def` / COMDAT 过滤 / export_map，再到导入库与 Borland / MinGW 的不同处理。是上一篇《符号导出三国志》的工程续篇。

[上一篇《符号导出三国志》]({% post_url 2026-06-07-symbol-export-windows-linux-macos %})讲清了底层机制：Windows 把「间接寻址」做成 opt-in，跨 DLL 用变量必须 `__declspec(dllimport)`，而 Linux/macOS 靠 GOT 把这层做成默认，免标注。

这一篇专门讲：这个看似只是「多写个标注」的差异，在工程上是怎么一步步演变成构建系统的麻烦的，以及 CMake、blade 这些工具是怎么对付的。

## 痛点一：头文件的描述无法一致

ELF/Mach-O 的 `visibility("default")` 是**对称**的——构建端和使用端含义完全一样，就是「这符号是公开的」，一个标注通吃。

Windows 的同一个声明却要表达**三种互斥含义**，且取决于**谁在编译**：

| 角色 | 需要的含义 |
|---|---|
| 构建这个 DLL | `dllexport` |
| 把它当 DLL 用 | `dllimport` |
| 静态链接它 | 都不要 |

更糟的是第三行——「当 DLL 用还是静态用」是**每条依赖边**的属性，不是库自己的属性。同一个库，A 静态链、B 动态链，头文件得对两者都对。

于是诞生了经典的宏：

```cpp
#ifdef BUILDING_MYDLL
#  define MYDLL_API __declspec(dllexport)
#else
#  define MYDLL_API __declspec(dllimport)
#endif

MYDLL_API int add(int, int);
MYDLL_API extern int g_value;
```

构建 DLL 时定义 `BUILDING_MYDLL`，消费时不定义。问题来了：

1. **谁来定义 `BUILDING_MYDLL`？** 必须由构建系统**只给「生产该 DLL 的那个 TU」**注入 `-DBUILDING_MYDLL`，不能泄漏给消费者。
2. **静态链接怎么办？** 静态消费时既不该 export 也不该 import，宏却只有两个分支。严格说要三态。
3. **方向搞反会编译失败。** 如果构建 DLL 时忘了定义 `BUILDING_MYDLL`，头文件展开成 `dllimport`，而 `.cpp` 又给出定义——这时函数报 warning C4273（inconsistent dll linkage，还能编过），但**全局变量/静态成员直接报 error C2491**（definition of dllimport data not allowed），编译失败。

单符号、对称的 visibility 模型里，这些问题统统不存在。

## 痛点二：dllexport 烤进 .obj，逼出双编译

假设你绕过了方向问题，老老实实用 `dllexport`。还有一个更隐蔽的代价。

`__declspec(dllexport)` 编译后，会在 **.obj 的 `.drectve` 段**写入一条 `/EXPORT:name` 链接器指令。这条指令是**粘性的**——谁把这个 .obj 链进去，它就让那个最终镜像也导出该符号：

```
foo.cpp --(dllexport)--> foo.obj [.drectve: /EXPORT:foo]
                              │
              ┌───────────────┴───────────────┐
         链进 DLL                          链进 exe（经静态库）
       正确导出 ✓                    exe 也被迫导出 foo ✗（污染）
```

需要说明的是，exe 多出这张导出表**不影响程序运行**——加载器不在乎可执行文件有没有导出表，没人去 import 它（除非你故意设计成 plugin host）。但它很**难看**，而且有实际代价：

- 一个本不该导出任何东西的 exe，平白携带一张导出表，把内部符号名暴露出来（信息泄露面变大），二进制也变大
- 更要命的是 **PE 导出表有 65536 项的上限**（导出用 16 位序号，ordinal 最多 65535）。这个污染不是「免费」的——它实打实占用导出槽位。模板密集的 C++ 项目一旦走「全导出」路线，很容易把这 64K 顶满，导致链接失败。后面讲 blade 的 COMDAT 过滤，正是为了对付这个上限。

于是**同一批 .obj 没法同时服务静态和动态**：
- 静态用：不想要任何 `/EXPORT`
- 动态用：需要 `/EXPORT`

CMake 这类工具对「同时要 `.lib` 和 `.dll`」的库，干脆**编译两遍**（SHARED target 带 `dllexport`，STATIC target 不带），产出两套 .obj，拖慢构建。

值得一提的是，Linux 上也有双编译现象，但**根因完全不同**：那是 PIC vs 非 PIC（`.a` 常非 PIC，`.so` 要 `-fPIC`），和导出无关。而 Windows 没有 PIC/非 PIC 之分（PE 靠 base relocation，代码天然可重定位），所以在 Windows 上双编译的**唯一理由就是那条 `/EXPORT` 指令**。

## 解法：把导出决策后置到链接期

聪明的做法是：**别在源码里写 `dllexport`，让导出信息从「编译期烤进 .obj」改为「链接期单独生成」。**

具体就是不标注源码，而是在链接时用一个 **`.def`（module definition file）** 列出要导出的符号，`link /DEF:foo.def` 生成 DLL。`.def` 由工具自动从（无标注的）.obj 解析符号生成。

这一挪有两个立竿见影的好处：

1. **.obj 里没有任何 `/EXPORT` 指令** → 同一批 .obj 静态动态共用，**编译一次两用**。Windows 上双编译的唯一理由（导出指令）就此消除。
2. **源码零标注** → 没有 `BUILDING_FOO` 宏，也就没有方向搞反的 C2491/C4273。

### CMake 的 WINDOWS_EXPORT_ALL_SYMBOLS

CMake 提供了这条路：[`WINDOWS_EXPORT_ALL_SYMBOLS`](https://cmake.org/cmake/help/latest/prop_tgt/WINDOWS_EXPORT_ALL_SYMBOLS.html)。打开后，CMake 用内置的 bindexplib 解析 .obj 符号、自动生成 `.def`，导出所有符号，省掉 `dllexport` 标注。

但它有个文档明示的**硬限制**：

> "Global *data* symbols must be explicitly marked with `__declspec(dllimport)` in order to link to data in the `.dll`."

也就是说：**函数自动导出、消费端免标注；但全局变量仍需消费端写 `dllimport`**，含虚函数的类的 vtable 引用也要显式 `dllimport`。这正是上一篇讲的「函数有 thunk、变量没有」的必然后果——任何构建系统都绕不过去，CMake 只能把它写进「已知限制」。

## blade 的做法：auto-`.def` + COMDAT 过滤

[blade](https://github.com/chen3feng/blade-build) 走的也是 auto-`.def` 这条路，但在 CMake 的基础上做了一个关键改进。

CMake 的 bindexplib 是真「导出所有」——包括模板实例化、内联函数、vtable、RTTI 这些 **COMDAT 重复符号**。模板密集的 C++ 很容易把导出表撑爆，撞上 PE 的 **64K 导出上限**。

blade 在解析 COFF 符号时，**过滤掉 COMDAT dedup 符号**。判别依据是 COMDAT 的 *selection*，不是「是不是 COMDAT」：

- 在 `/Gy`（`/O2` 默认开）下，连普通函数都会变成 COMDAT，但 selection 是 `NODUPLICATES`（唯一定义）——**保留**
- 模板/内联/vtable/RTTI 这些是 dedup selection（`ANY` / `SAME_SIZE` / `EXACT_MATCH` / `LARGEST`）——它们由每个使用方自己从头文件实例化，DLL 不需要导出——**丢弃**

这样导出表只剩真正需要跨模块的符号，既省心又不易撞上限。

顺便，变量在 `.def` 里会带上 `DATA` 关键字（`EXPORTS` 段里 `name DATA`），这是 PE 要求导出数据必须标的——blade 通过解析 COFF 符号类型（是否 FUNCTION）自动判定并加上。

三家对比：

| | CMake bindexplib | blade auto-`.def` |
|---|---|---|
| 函数自动导出 | ✓ | ✓ |
| 数据需消费侧 `dllimport` | ✓（文档明示） | ✓（同样无法绕过） |
| COMDAT 模板/vtable | **全导出 → 易撞 64K** | **过滤掉 → 导出表更小** |
| 静态/动态共用 .obj | 取决于配置 | ✓（单次编译） |

## 纪律 vs 省心：export_map

auto-`.def` 的代价是语义上「全导出」（减 COMDAT），缺乏导出纪律——这和「默认不导出能减少符号泛滥、降低名字冲突」的初衷相悖。

blade 的解法是 **`export_map`**：用一份 **GNU-ld version script** 声明要导出哪些符号，跨三平台统一一份源，各平台各自翻译落地：

| 平台 | 落地方式 |
|---|---|
| Linux | 直接 `-Wl,--version-script=`（原生） |
| macOS | 转成 ld64 的 `-exported_symbols_list` |
| Windows | **过滤 auto-`.def`**，只保留 `global:` 命中的符号 |

```
# export.map（一份源，三平台通用）
{
  global:
    mylib::Api::*;     # 按命名空间/类导出
  local:
    *;                 # 其余隐藏
}
```

关键在 Windows 侧：export_map 是在**链接期过滤 `.def`**，源码仍零标注、`.obj` 仍不含 `/EXPORT`。所以：

- **默认（无 export_map）**：auto-export-all − COMDAT，省心
- **带 export_map**：按 `global:` 收窄，拿回导出纪律
- **两种模式都保持**：单次编译 + 静态/动态共用对象 + 源码无 `__declspec`

纪律和省心做成了**正交的开关**，而不是二选一，且没有牺牲单次编译这个好处。

（一个精度细节：Windows 侧为了让 MSVC 修饰名匹配 Itanium 风格的 `extern "C++"` 通配，用 `UnDecorateSymbolName(NAME_ONLY)` 解修饰，只取限定名、丢签名。后果是重载会塌缩成一个名字、带签名的模式只按名字匹配。对「按命名空间/类导出」的常见用法没影响，精确到重载签名的场景要注意。）

## 还有一个躲不开的：LNK4197

如果你用 auto-`.def`，又在源码里残留了 `__declspec(dllexport)`，同一符号会被导出两次（一次来自源码 `/EXPORT` 指令，一次来自 `.def`），链接器报 **warning LNK4197**「export specified multiple times」。所以用 auto-`.def` 方案时，应当**彻底不写 `dllexport`**，把导出完全交给 `.def`。

## 番外：导入库，以及 Borland / MinGW 的不同处理

前面都在讲「导出端」。消费端要用 DLL，靠的是**导入库（import library）**——它不是代码，而是「链接期的胶水」：里面是一堆 `__imp_<name>` 记录（IAT 槽 + 跳板），链接器拿它把你对 DLL 符号的引用解析成「走 IAT」。**DLL 自己只有一张导出表；导入库才是消费侧链接时真正吃进去的东西。**

MSVC 的链接器在产出 DLL 的同时自动吐出一个同名 `.lib`（就是导入库），格式是 **COFF** 归档。但「导入库」本身不是统一标准，各家工具链的格式和生成方式都不一样，跨工具链用同一个 DLL 时这就成了坑。

### Borland C++：OMF 格式 + IMPLIB

Borland（以及更早的 Watcom 等）用的目标/库格式是 **OMF**，不是 COFF。所以 **MSVC 的 `.lib` 和 Borland 的 `.lib` 互不兼容**——把一个 MSVC 导入库丢给 Borland 链接器是不认的。

Borland 的处理：
- 自带 **`IMPLIB.EXE`**，直接从 **DLL 本身**（读它的导出表）或从 `.def` 生成 Borland 格式（OMF）的导入库：`implib foo.lib foo.dll`。
- 厂商只给了 MSVC（COFF）导入库时，Borland 用户得用 `IMPLIB` 从 DLL 重新生成，或用 `COFF2OMF` 转换。
- 这正是当年「同一个 DLL，不同编译器各配一份导入库」乱象的根源之一。

关键认识：**DLL 是通用的（就一张导出表），导入库才是工具链私有的。** 所以「从 DLL 反向生成导入库」各家都提供——因为导出表里的信息足够还原出导入库。

### MinGW（GNU 工具链）：dlltool、直接链 DLL、auto-import

MinGW 用 COFF，导入库是 GNU `ar` 归档，惯例叫 `lib*.a` / `*.dll.a`。它有几种比 MSVC 更灵活的玩法：

1. **`dlltool` 从 `.def` 或 DLL 生成导入库**：`dlltool -d foo.def -l libfoo.dll.a`，也可直接喂 DLL。对应 Borland 的 IMPLIB。
2. **`ld --out-implib`**：构建 DLL 时让链接器**顺手吐出导入库**：`gcc -shared -o foo.dll … -Wl,--out-implib,libfoo.dll.a`。对应 MSVC 自动产出 `.lib`。
3. **直接链 DLL，免导入库**：GNU ld 能**直接对着 `.dll` 链接**——`gcc main.c -L. -lfoo` 找到 `foo.dll` 就直接生成导入引用，根本不需要导入库文件。这是 MSVC 没有的便利。
4. **auto-import / 伪重定位（pseudo-relocation）**：这条最有意思——它部分**绕过了上一篇讲的「跨 DLL 用变量必须 `dllimport`」**。GNU ld 的 auto-import 能在你**没写 `dllimport`** 时也链到 DLL 里的**数据**：在加载期由一段 runtime pseudo-relocation 代码把真实地址回填。于是变量不标注也能用（通常带个 warning）。

   但它不是免费的：
   - 变量地址要等加载期才回填，**不能用于常量初始化 / 静态初始化**（那时还没填好，ld 会直接报「can't be auto-imported」）。
   - 比 `dllimport` 直接走 IAT 慢一点，且依赖 MinGW 运行时那段 pseudo-reloc 代码。
   - 本质是 MinGW 用「加载期回填」补上了 PE 缺的那层间接——精神上和 ELF 的 copy relocation 异曲同工（都在替「不标注的数据访问」擦屁股），代价也类似（边角坑）。

所以同一个「数据导入」难题，三家又是三种权衡：**MSVC 摊牌让你写 `dllimport`；MinGW 用 auto-import 伪重定位替你兜底**（有代价）；**ELF 用 GOT / copy-reloc 从根上免标注**。

（还有个历史坑：`stdcall` 的名字修饰——MSVC 是 `_name@N`。MinGW 处理 `@N` 后缀的方式与 `.def` 里写不写 `@N` 有关，导致它生成的导入库有时和 MSVC DLL 的导出名对不上，要靠 `.def` 或 `dlltool --kill-at` 调。64 位 Windows 取消了 stdcall 修饰，这坑才消停。）

## 结语

把这条线理顺，Windows DLL 导出的工程图景就清楚了：

- **dllexport 源码标注**：纪律最好，但方向不对称（三态、每条边）、`/EXPORT` 烤进 .obj 逼出双编译
- **auto-`.def`（CMake / blade）**：源码零标注、单次编译，但语义上全导出
  - blade 额外用 COMDAT 过滤缓解 64K 上限
- **export_map / version script**：跨平台一份源，链接期过滤 `.def`，把「纪律」作为正交开关加回来

能自动化的，工具基本都自动化了。唯一谁都跨不过去的，是**消费侧用 DLL 的数据/vtable 必须 `dllimport`**——这是上一篇讲的 PE「间接寻址 opt-in」的代数后果，是 Windows ABI 自己的债，不是构建系统能还的。

实践建议也很简单：**跨 DLL 边界尽量用函数（getter/工厂）而非直接暴露全局变量和多态类**。函数走 thunk、免标注、共用对象、不撞这些坑——既规避了 `dllimport` 要求，也让头文件回到 ELF/Mach-O 那种「写一次、跨角色统一」的清爽体验。

