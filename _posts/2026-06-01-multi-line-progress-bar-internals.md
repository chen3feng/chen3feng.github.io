---
title: "终端多行进度条是怎么实现的"
date: "2026-06-01 09:00:00 +0800"
categories: [System, CLI]
tags: [Terminal, ANSI, Progress Bar, Bazel, Python, Curses]
layout: post
published: true
---

> 错误信息往上滚、多行进度条永远停在最后、还一秒一跳——还不会被错误信息撞乱。
> 这件事在 Bazel、Docker、Cargo 里见过无数次，单行 `\r` 的小把戏明显不够用。中文社
> 区几乎没人讲透，我顺手研究了一下，把"擦帧—写消息—重画"这一套从 VT100 起源讲到
> tqdm 与 Blade 的实际代码（包括 Windows 上必须自己开启 VTP 这件事），最后给一个
> 50 行的 Python 最小可用版。

我这个年代的人，接触计算机时都是从那个黑乎乎的 DOS 字符界面开始的。最近几十年来字符界面
的程序也越来越漂亮了，比如 conda 和 docker 之类的都有多行进度条。

<img width="306" height="165" alt="image" src="https://github.com/user-attachments/assets/e2361114-64ad-44a9-a40c-2d3608bf49c3" />

单行进度条的实现原理人尽皆知，十来年前 [Blade](https://blade-build.github.io/) 就已经支持了单行进度条，
最近看到很多命令行程序支持多行进度条，再加上 AI 编程让实现一个想法的耗时大幅度减少，
就优化一下 `console.py` 模块，希望也支持一下。

要想实现就得先弄明白它的机制，研究了一些源码，终于彻底搞清楚了这套东西背后的机制。

要讲清楚多行进度条的工作原理，不得不先提一下 **单行进度条**，几乎所有讲 Python/Shell
进度条的中文博客都讲过；**多行**进度条——就是 Bazel/Docker/pip/npm/conda/uv 那种
“同时几个 task 各占一行、底部保持鲜活、错误信息往上滚”的——**几乎找不到一篇说清楚的**。
可它的核心套路其实并不神秘，只是涉及好几个独立的细节，每一个都不难，凑齐才能跑得稳。

这篇笔记把这套机制从头到尾拆一遍。所有可运行代码用 Python，因为 ANSI 转义和具体语言
无关，读着也不累。

主要阅读了 [tqdm](https://github.com/tqdm/tqdm) 的源代码，这是是一个功能强大的
Python 进度条库，常用于循环操作、文件处理、网络爬虫或机器学习模型训练中。它能在终端或
命令行中实时显示进度百分比、已处理数量、预估剩余时间及处理速度。

顺便说一下 tqdm 这个拗口又难记的名字。源自阿拉伯语字根 “taqaddum” (تقدّم)，意思是 
“进展”或“进步”（progress）。同时，它也是西班牙语 “Te quiero demasiado” 的首字母缩写，
意思是 “我太爱你（们）了”，这代表了作者对开源社区的喜爱。（是不是还是记不住，我也是😂）

另外也去看了 Bazel 的实现。

---

## 一、单行进度条：`\r` 覆写，5 分钟看完就忘

最简版本：

```python
import sys, time
for i in range(101):
    sys.stdout.write(f"\r[{('=' * i).ljust(100)}] {i}%")
    sys.stdout.flush()
    time.sleep(0.02)
print()
```

核心机制只有个：**`\r`（回车符）把光标移到行首，下一次 `write` 把整行覆盖一遍**。
末尾不要 `\n`，否则会换行、就不再是"原地修改"。

`\r` 是 ASCII 13 号，**比 ANSI 转义码资历还老**——它从打字机时代就存在，"按一下回车
让滚筒把纸退回到行首"，CR (carriage return) 字面意思。终端里所有"原地更新一行"的小
工具（`wget` 下载、`apt-get` 状态、shell `read -p` 提示）全靠它。

这一套有几个不太显眼的坑：

- **行尾残留**：本次输出比上次短，末尾会留下旧字符。修法是末尾补够空格 `ljust`，或
  用 `\033[K`（清光标到行尾）。
- **混入其它输出会破坏**：进度条画到一半被一行错误信息插了进去，覆写就错位了。
- **非 TTY 时一团糟**：`\r` 在管道里只是个字面字符，文件里会留下一堆"乱码"。

这些坑放到多行场景下都会被**放大十倍**——这正是为什么多行不能只是"单行的 N 倍"。

---

## 二、单行 vs 多行：根本上是两种东西

写多行进度条的时候，你不再是在"改一行字"，你是在**维护一帧画面**：

| | 单行 | 多行 |
|---|---|---|
| 覆写方式 | `\r` 拉回行首，重写一行 | `\e[nA` 上移 n 行 + `\e[K` 清行 ×n |
| 必须记什么 | 不用记，反正只有一行 | **必须记"上次画了几行"**——擦多/擦少都崩 |
| 谁负责换行 | 终端自动 | **你自己手动控**，不能信终端的自动折行 |
| 行尾残留 | `\033[K` 一下了事 | 每行都要单独清 |
| 插入新消息 | 顶多 `\n` 把旧的留下 | **擦整块 → 写消息 → 重画整块** 三步 |
| 并发 | 一般不考虑 | 必须加锁，否则刷屏和业务输出撞车 |

心智模型的切换：

> 单行进度条是"在一个固定槽位上改字"；多行进度条是"管理一块画布——每次刷新就是
> 擦掉上一帧，画当前帧"。

这个"双缓冲渲染"的思路一旦接受，剩下的工程细节全是水到渠成。

---

## 三、多行的两根支柱：上移光标 + 清行

底层就这两个 ANSI 转义码（[CSI 子集][ansi]，1979 年标准化，来自 DEC VT100, 1978）：

```
ESC [ n A     # 光标上移 n 行，不滚屏
ESC [ K       # 清除从光标位置到行尾
```

写出来就是字节 `0x1b 0x5b ...`，Python 里通常写 `'\033[1A'`、`'\033[K'`，也可以写
`'\x1b[1A'`、`'\x1b[K'`。

[ansi]: https://en.wikipedia.org/wiki/ANSI_escape_code

一次"擦帧"的最小循环：

```python
ESC = "\x1b["
def clear_block(n_lines):
    for _ in range(n_lines):
        sys.stdout.write("\r" + ESC + "1A" + ESC + "K")
        # \r:    回到列 0
        # ESC[1A: 上移一行
        # ESC[K:  清当前行
```

这里有几个细节决定了整套机制的鲁棒性，新手最容易踩雷：

### 3.1 必须记住"上次画了几行"

你下一次擦的行数 = 上一次写的行数。写少了，旧帧的下半部会留在屏幕上变成"幽灵进度条"；
写多了，会把上面正常滚屏出去的内容（前面的日志、错误）也擦掉。

[tqdm][tqdm] 在 [`tqdm/std.py#L1441-L1444`][tqdm-moveto] 把这件事抽象成一个
`moveto(n)`：

```python
# tqdm v4.67.3, tqdm/std.py:1441
def moveto(self, n):
    self.fp.write('\n' * n + _term_move_up() * -n)
    getattr(self.fp, 'flush', lambda: None)()
```

- `n` 为正：写 n 个 `\n`，光标往下移 n 行（同时滚屏）；
- `n` 为负：写 `-n` 个 `\x1b[A`（[`_term_move_up()`][tqdm-tmu]），光标上移不滚屏。

每一条 bar 记自己的 `pos`（位置），擦自己之前先 `moveto(pos)` 上去、写完再
`moveto(-pos)` 回来。同样，**`pos` 必须严格正确**，差一行整个画面都会崩。

[tqdm]: https://github.com/tqdm/tqdm
[tqdm-moveto]: https://github.com/tqdm/tqdm/blob/v4.67.3/tqdm/std.py#L1441-L1444
[tqdm-tmu]: https://github.com/tqdm/tqdm/blob/v4.67.3/tqdm/utils.py#L370-L371

### 3.2 不能信终端的自动折行

终端有个看似无害的特性：写到第 80 列再写一个字符，它会**自动换行**。但各家终端在这
件事上的行为各不一致（有的"eager"——写完第 80 个字符就换；有的"lazy"——写第 81 个
才换）。你以为画了 3 行，实际上其中一行被折成了 2 行——擦的时候按 3 行擦，留下一
行。屏幕花掉。

正解：**主动控制换行**，永远不让任何一行接近终端宽度。Blade 这次重构里我新加的
[`_compute_progress_bar_width`][blade-width] 就是干这件事的：

```python
# blade/src/blade/console.py:232
def _compute_progress_bar_width(total):
    """Pick a bar width that fits the current terminal, bounded by [MIN, MAX]."""
    cols = shutil.get_terminal_size((80, 24)).columns
    overhead = 9 + 2 * len(str(total))
    return max(_MIN_PROGRESS_BAR_WIDTH,
               min(_MAX_PROGRESS_BAR_WIDTH, cols - overhead - 1))
                                         #  ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
                                         #  关键就这个 -1 留 1 列 margin
```

CodeQL 没抓住这个 off-by-one，**UT 抓住了**——我专门写了一条"整行长度严格小于终
端宽度"的不变式测试，跑出来在 `cols=60` 这一档恰好踩中。我还看了 Bazel 也用的是同一招（套
`LineWrappingAnsiTerminalWriter` 按 `terminalWidth - 1` 强制换行），思路完全一致：
**不让终端替你做这件事**。

[blade-width]: https://github.com/blade-build/blade-build/blob/295a44d/src/blade/console.py#L232-L240

### 3.3 `\033[K` 还是 `\033[2K`？

```
ESC [ K   = ESC [ 0 K   # 清光标到行尾  ← 推荐
ESC [ 1 K              # 清行首到光标
ESC [ 2 K              # 清整行（不影响光标）
```

很多博客抄来抄去都用 `\033[2K`，但配合 `\r`（光标到列 0）用的话，`\033[0K` 和
`\033[2K` 视觉效果一致——而 `\033[0K` 更精确："只清光标后面的内容"。Bazel 用 `K`，
Blade 改完也是 `K`，我也推荐这个。

还有一种不用转义码的做法：**末尾填够空格**。tqdm 的 [`status_printer`][tqdm-sp]
就这么干：

```python
# tqdm v4.67.3, tqdm/std.py:451
def print_status(s):
    len_s = disp_len(s)
    fp_write('\r' + s + (' ' * max(last_len[0] - len_s, 0)))
    last_len[0] = len_s
```

记住"上次写了几个字符"，这次写完后补够空格盖住末尾——和 `\033[K` 殊途同归。优点
是连转义码都不依赖（理论上连最古老的纯 ASCII 终端都能跑）；缺点是要多记一个状态、
要算字符显示宽度（`disp_len`，CJK 全角字符算 2 列）。

[tqdm-sp]: https://github.com/tqdm/tqdm/blob/v4.67.3/tqdm/std.py#L437-L462

---

## 四、错误信息往上滚、进度永远在底部——是怎么做到的？

这是我研究 Bazel 时印象最深的一个细节。**进度条不是"贴在底部"，而是"每次有新东
西要打印，进度条都在新的底部"**——所以进度条永远是最后写出来的那几行，被推上去的旧消息
留在 scrollback buffer 里。

整个机制可以浓缩成这样一段伪代码：

```
on_message(msg):
    1. clear_progress_bar()    # 上移 + 清行 × N
    2. write(msg + "\n")       # 自然换行把光标推到新底部
    3. draw_progress_bar()     # 在新底部重画

on_progress_tick():
    1. clear_progress_bar()
    2. draw_progress_bar()     # 同样的逻辑，只是没消息要写
```

tqdm 把这个模式封成了一个上下文管理器，看下面这段代码会一眼明白
（[`tqdm/std.py#L725-L753`][tqdm-ewm]）：

```python
# tqdm v4.67.3, tqdm/std.py:725
@classmethod
@contextmanager
def external_write_mode(cls, file=None, nolock=False):
    """Disable tqdm within context and refresh tqdm when exits."""
    fp = file if file is not None else sys.stdout
    try:
        if not nolock:
            cls.get_lock().acquire()
        inst_cleared = []
        for inst in getattr(cls, '_instances', []):
            if hasattr(inst, "start_t") and (inst.fp == fp or all(
                    f in (sys.stdout, sys.stderr) for f in (fp, inst.fp))):
                inst.clear(nolock=True)        # ① 擦掉所有进度条
                inst_cleared.append(inst)
        yield                                   # ② 用户在这里写消息
        for inst in inst_cleared:
            inst.refresh(nolock=True)          # ③ 把擦掉的进度条重画回来
    finally:
        if not nolock:
            cls._lock.release()
```

这就是"擦—写—重画"三步原子化的最简形态。所有 bar 在 yield 之前被清空，业务代码
在 yield 内写出去的内容直接落到原来进度条的位置，yield 退出后所有 bar 一次重画回
来。"清+写+重画"被一把**类级锁** `cls._lock` 包成原子。

[tqdm-ewm]: https://github.com/tqdm/tqdm/blob/v4.67.3/tqdm/std.py#L725-L753

`tqdm.write()` 就是这个上下文管理器最薄的一层封装（[`tqdm/std.py#L716-L723`][tqdm-write]）：

```python
# tqdm v4.67.3, tqdm/std.py:716
@classmethod
def write(cls, s, file=None, end="\n", nolock=False):
    """Print a message via tqdm (without overlap with bars)."""
    fp = file if file is not None else sys.stdout
    with cls.external_write_mode(file=file, nolock=nolock):
        fp.write(s)
        fp.write(end)
```

用户写 `tqdm.write("Some message")`，库内部把它翻译成"清→写→重画"。**这就是 tqdm
"打印消息但不破坏进度条"的全部秘密**——你看着觉得多魔法，看完源码发现就是十几行
代码。

Blade 走的是同一条路。[`blade/src/blade/console.py#L331-L336`][blade-doprint] 的
`_do_print` 跟上面那个 yield 块的语义一模一样，只是直接写出来没用 `contextmanager`：

```python
# blade/src/blade/console.py:331
def _do_print(msg, file=sys.stdout):
    with _print_lock:                       # 同一把锁
        _clear_progress_bar_locked()        # ① 擦
        print(msg, file=file)               # ② 写消息（自带 \n）
                                            # ③ 重画由下一次 show_progress_bar 完成
```

注意 Blade 比 tqdm 简单一点：它只有一条 bar，不像 tqdm 要遍历 `cls._instances` 处理
多个。这是单 bar 场景下能走的捷径——但**锁 + 三步擦写画**的骨架是同一个。

[tqdm-write]: https://github.com/tqdm/tqdm/blob/v4.67.3/tqdm/std.py#L716-L723
[blade-doprint]: https://github.com/blade-build/blade-build/blob/295a44d/src/blade/console.py#L331-L336

### 行缓冲：避免逐字节刷屏

子进程的 `stdout` / `stderr` 输出可能一个字节一个字节来，每来一次就触发"擦—写—
重画"会疯狂闪烁。常见做法是**行缓冲**——没看到 `\n` 就只塞进 buffer，看到 `\n` 才
整行一次性吐出去。这样进度条只在"完整一行"边界被打扰，视觉非常稳。Bazel 做了这件
事，tqdm 因为是 Python 库（接收的就是 Python 字符串），调用方天然按行写，所以不
需要库自己缓冲。

### 不是迭代驱动？得加个后台线程

tqdm 假设你在主循环里**不断调用 `update()`**——每次迭代驱动一次重绘。这对"我在跑
1 万次循环"这种场景很自然。

但 Bazel/Blade 这种构建工具的进度更新**不是迭代驱动**的：一个 action 可能跑几十秒，
中间没有任何"tick"事件，你又想让"已用时间"那个数字一秒一跳。这就需要一个独立的后
台线程定期触发重绘。Bazel 的 `cli-update-thread` 每 `minimalUpdateInterval` 毫秒
（最小 200ms）醒一次：

```python
# 思路示意（Python 版改写）
def ticker():
    while not shutdown:
        time.sleep(0.2)
        with lock:
            clear_block(); draw_block()
threading.Thread(target=ticker, daemon=True).start()
```

200ms 是个经验值——肉眼觉得"活着"，又不会刷屏太频繁；`ninja`、`cargo` 基本都在
100~300ms 这个量级。第五节那个最小可用版我也用了 200ms。

---

## 五、动手写一个：50 行 Python 最小可用版

光看 tqdm 和 Bazel 容易"觉得自己懂了"，自己写一遍才是真懂。下面这段代码把前面所有要点都体
现了——锁、行数簿记、隐藏 cursor、消息往上滚、ticker 自动跳秒数——可以直接拷出去
跑（不要在 IDE 输出窗口跑，会失真，必须在真终端）：

```python
import sys, time, threading

ESC = "\x1b["
HIDE, SHOW = ESC + "?25l", ESC + "?25h"

lock = threading.Lock()
last_lines = 0
hidden = False
tasks = {"a": 0, "b": 0, "c": 0}

def _hide():
    global hidden
    if not hidden:
        sys.stderr.write(HIDE); hidden = True

def _show():
    global hidden
    if hidden:
        sys.stderr.write(SHOW); hidden = False

def _clear_block():
    global last_lines
    sys.stderr.write(("\r" + ESC + "1A" + ESC + "K") * last_lines)
    last_lines = 0

def _draw_block():
    global last_lines
    for name, t in tasks.items():
        sys.stderr.write(f"  [RUN] task {name} ... {t}s\n")
        last_lines += 1
    sys.stderr.flush()

def refresh():
    with lock:
        _clear_block()
        _hide()
        _draw_block()

def log(msg):
    with lock:
        _clear_block()
        _show()                      # 进入"业务输出"状态
        sys.stdout.write(msg + "\n") # 留在 scrollback
        sys.stdout.flush()
        _hide()
        _draw_block()                # 进度条立刻重画在底部

def ticker():
    while True:
        time.sleep(0.2)
        for k in tasks: tasks[k] += 1
        refresh()

import atexit
atexit.register(_show)               # Ctrl+C 后 cursor 必须能恢复

_hide(); _draw_block()
threading.Thread(target=ticker, daemon=True).start()

for i in range(20):
    time.sleep(2)
    log(f"INFO: event #{i} happened")
```

跑起来三件事会同时发生：

1. **底部 3 行永远是 task a/b/c 的秒数**，每 200ms 跳一次。
2. **`INFO: event #N happened` 一条条从底部往上滚**，永远留在 scrollback。
3. 进度条不会被消息挤碎、不会闪烁、Ctrl+C 不会让 cursor 消失。

把里面几行注掉做反面测试特别有感觉：

- 把 `log()` 里的 `_clear_block()` 注掉 → 消息直接打在进度条**上面行**，屏幕乱掉。
- 把 `with lock:` 全去掉、`time.sleep(2)` 改成 `0.01` → 屏幕花掉，ticker 和 log 撞。
- 把任务数动态从 3 加到 5 但不更新 `last_lines` → 旧的两行残留，"幽灵进度条"现身。
- 把 `atexit.register(_show)` 注掉、Ctrl+C 退出 → 终端 cursor 永久消失，要敲
  `reset` 才回来。

这三个反例分别对应 tqdm 里**最关键的三块代码**：`clear` / `external_write_mode` 的
擦写 dance、类级 `cls._lock` 互斥、每条 bar 的 `pos` + `moveto(pos)` 行数簿记。再
加一个 cursor 的 atexit 钩子，构成了我在 Blade 里给 `console.py` 补的所有内容。

---

## 六、Windows 的特殊情况：先开 VTP

第五节那个最小可用版我特别注明了"在真终端跑"。Linux / macOS 上这句话默认成立。
**Windows 上不一定**——这是一个独立的、值得拆开讲的话题。

ANSI 转义码在 Windows 上历史上**不被原生支持**。

Windows 起源于 DOS。ANSI.SYS 是 DOS 系列操作系统中的一个设备驱动程序，它允许程序通过
在输出中插入 ANSI 转义序列来控制显示。但它并非默认安装，而且运行速度极慢。

进入 Windows 时代，随着图形用户界面的普及，这套东西就基本丢了。如果程序想实现酷炫的效果，就得调 Console API。

但是 Linux 崛起后，大量优秀的字符界面的应用程序涌入到 Windows 系统。纯命令行界面的程序还好说，
各种用了 ANSI 转义序列的程序如果都改成调 API，移植成本就会很高。只能通过第三方程序提供基本的支持：

- 自动拦截 ANSI 转义符并调用 Win32 API 处理颜色：如 Python 的 Colorama 库通过重定向 sys.stdout，使 Windows 支持彩色输出。
- 注入式全局控制台外挂：如果不想或者不能修改程序代码，可以通过 API Hook 的方式拦截拦截控制台的显示输出流。最著名的是 ANSICON。
- 第三方终端：绕开 Windows 自己的终端，自己实现。最著名的有 ConEmu/Cmder，MinTTY。

但是官方支持却步履蹒跚，直到 Windows 10 build 16257（2016 年的 Anniversary Update）才在抛弃 cmd 重写的
Windows Terminal 引入 "Console Virtual Terminal Sequences" 特性，**而且默认是关的**，必须程序自己显式开启。

### 6.1 检测 + 开启：`GetConsoleMode` / `SetConsoleMode`

Win32 提供 `GetConsoleMode(handle, &mode)` 读当前模式 flag bitmap，关键位是
`ENABLE_VIRTUAL_TERMINAL_PROCESSING = 0x0004`。Blade 的
[`_windows_console_support_ansi_color`][blade-vtp]
用 `ctypes` 直接调原生 API，**检测和开启一气呵成**：

```python
# blade/src/blade/console.py:33
def _windows_console_support_ansi_color():
    from ctypes import byref, windll, wintypes
    ENABLE_VIRTUAL_TERMINAL_PROCESSING = 0x0004
    INVALID_HANDLE_VALUE = -1
    STD_OUTPUT_HANDLE = -11

    handle = windll.kernel32.GetStdHandle(STD_OUTPUT_HANDLE)
    if handle == INVALID_HANDLE_VALUE:
        return False                          # ← 不是真 console（被 IDE 启动等）

    mode = wintypes.DWORD()
    if not windll.kernel32.GetConsoleMode(handle, byref(mode)):
        return False                          # ← 查不到模式，多半也不是 console

    if not (mode.value & ENABLE_VIRTUAL_TERMINAL_PROCESSING):
        if windll.kernel32.SetConsoleMode(
            handle,
            mode.value | ENABLE_VIRTUAL_TERMINAL_PROCESSING) == 0:
            print('kernel32.SetConsoleMode to enable ANSI sequences failed',
                file=sys.stderr)              # ← 老 Windows 没这位，set 失败
    return True
```

注意几个细节：

- **`STD_OUTPUT_HANDLE = -11`** 是 Win32 常量；Python 3 的 `subprocess` 模块上也有
  这个名字，但只在 Windows 上可用——直接写 `-11` 加注释，省一道平台分支。
- **handle == `INVALID_HANDLE_VALUE` (-1)** 表示拿不到 console handle，多半是 stdout
  被重定向到管道/文件，这时根本不该启用任何转义。
- **`SetConsoleMode` 返回 0** 是失败：老 Windows（7/8/8.1）没有 VTP 这一位，设不上。
  失败时函数仍然返回 `True` 但写了一条警告——意思是"已经试过了，不行就让转义码
  按原样落到 stdout 上吧"。
- 调用 `SetConsoleMode` 时**必须保留原 mode 的其他位**（`mode.value | VTP`），不能
  直接 `mode = VTP`——会清掉 `ENABLE_PROCESSED_OUTPUT` 之类的位，控制台反应会很奇怪。

[blade-vtp]: https://github.com/blade-build/blade-build/blob/295a44d/src/blade/console.py#L33-L53

### 6.2 另一条路：colorama（tqdm 的选择）

`tqdm` 不直接调 Win32 API，它把脏活外包给 [`colorama`][colorama]——一个 pure-Python
包，**通过劫持 `sys.stdout` / `sys.stderr` 把流过的 ANSI 序列翻译成 Win32 console
API 调用**。导入时机就在 [`tqdm/utils.py#L14-L31`][tqdm-utils]：

```python
# tqdm v4.67.3, tqdm/utils.py:14
CUR_OS = sys.platform
IS_WIN = any(CUR_OS.startswith(i) for i in ['win32', 'cygwin'])
...
try:
    if IS_WIN:
        import colorama
    else:
        raise ImportError
except ImportError:
    colorama = None
else:
    try:
        colorama.init(strip=False)
    except TypeError:
        colorama.init()
```

没装 colorama 怎么办？tqdm 在 `_term_move_up` 里直接**放弃光标控制**
（[`tqdm/utils.py#L370-L371`][tqdm-tmu]）：

```python
def _term_move_up():
    return '' if (os.name == 'nt') and (colorama is None) else '\x1b[A'
```

返回空字符串——multi-bar 退化成"一条接一条地 print"，至少不会在 cmd.exe 里印出
`^[[A` 之类的乱码。

[colorama]: https://pypi.org/project/colorama/
[tqdm-utils]: https://github.com/tqdm/tqdm/blob/v4.67.3/tqdm/utils.py#L14-L31

### 6.3 两条路线对比

| | 直接走 VTP（Blade） | 经 colorama（tqdm） |
|---|---|---|
| 最低 Windows 版本 | Windows 10 build 16257（2016+） | 任意（含 XP） |
| 额外依赖 | 无（只用 ctypes 调 Win32 API） | colorama |
| 性能 | 终端原生解析，零开销 | 每次写都被 Python 层劫持解析 |
| 跨平台代码 | 加一个 `os.name == 'nt'` 分支 | 几乎无感（colorama 在 Linux 上 no-op） |
| 复杂转义支持 | 100%（终端自己解析全部 CSI） | 限于 colorama 实现的子集 |

2026 年这个时间点，**直接走 VTP 基本零成本**——Windows 7/8 早已停止支持，在用的
Windows 都是 10/11；而新的 [Windows Terminal][win-term]（Microsoft 自家的现代终端
app，预装于 Windows 11）天生就是 ANSI/VTP 友好的。Blade 这次重构走的就是这条路。
如果你的程序追求"装了就能用、不要求 pip install colorama"，ctypes 那条路更省心。

[win-term]: https://learn.microsoft.com/en-us/windows/terminal/

### 6.4 别忘了 `isatty`

VTP 开了不代表 stdout 真的是 TTY——用户可能 `app.exe > out.txt`。所以仍要叠加
`sys.stderr.isatty()` 这一层。Blade 把这两件事在
[`_console_support_cursor_control`][blade-ctrl] 里合并：

```python
# blade/src/blade/console.py:63
def _console_support_cursor_control():
    if os.name == 'nt':
        # VTP gates both color and cursor escapes; if it's off, neither works.
        return _windows_console_support_ansi_color() and sys.stderr.isatty()
    return sys.stderr.isatty() and os.environ.get('TERM') not in ('emacs', 'dumb', '')
```

Linux/Mac 路径就一行（isatty + `TERM` 白名单）；Windows 多一道"先 VTP 检测/开启、
再 isatty"。

[blade-ctrl]: https://github.com/blade-build/blade-build/blob/295a44d/src/blade/console.py#L63-L68

---

## 七、用 curses 行不行？

会有人问：Unix 不是有 `curses` 吗？为什么要自己手撸转义码？

`curses`（[Ken Arnold][arnold] 1980 年左右在 BSD Unix 上写的，建立在 Bill Joy 的
`termcap` 上）确实就是为这种事服务的——管理整块"window"，自动处理终端差异、自动
做最小化重绘。`vi`、`htop`、`tmux`、`mc` 都用它。

[arnold]: https://en.wikipedia.org/wiki/Curses_(programming_library)

但 curses 有一个**致命的不匹配**：它假设你**全屏接管**整个终端。它通过
`initscr()` 进入一个"alternate screen"——你 Ctrl+C 退出后，看到的是你启动它之前的
shell 状态，**所有 curses 期间的输出都消失了**。这对 `vim` 是 feature，对 `cargo
build` / `bazel test` 是灾难——你想要的是"build 跑完之后日志全部留在终端里向上滚
能翻看"，curses 给不了你这个。

所以 Bazel / Docker / Cargo 都选了**手撸转义码**——既要"底部固定进度条"的视觉效
果，又要"输出像普通命令一样进 scrollback"的语义。这条路才是这篇文章讨论的主题。

不过 curses 也能写多行进度条 demo，作为对比：

```python
import curses, time

def main(stdscr):
    curses.curs_set(0)                       # 隐藏 cursor
    tasks = {"a": 0, "b": 0, "c": 0}
    msgs = []                                # 自己维护"已滚出"的消息
    while True:
        stdscr.erase()
        for i, m in enumerate(msgs[-10:]):   # 顶部留 10 行给消息
            stdscr.addstr(i, 0, m)
        for i, (n, t) in enumerate(tasks.items()):
            stdscr.addstr(11 + i, 0, f"  [RUN] task {n} ... {t}s")
        stdscr.refresh()
        time.sleep(0.2)
        for k in tasks: tasks[k] += 1

curses.wrapper(main)
```

可以跑，但你会立刻发现：

- 滚屏行为是 curses 模拟的（你自己维护 `msgs`），**不是真正的 scrollback**。
- 退出后所有内容消失。
- 想接管 stdout/stderr 来截子进程输出？要自己重定向。

curses 的抽象层级**太高**——它对"应用程序"友好，对"日志型工具"不友好。

> 一句话区分：**`vim` 类的全屏幕编辑器用 curses；滚屏方式的命令行程序自己写转义码**。

---

## 八、我在 Blade 里掉过的坑（你自己实现的话大概率也会掉进去😄）

我刚把 Blade 的 `console.py` 加固了一遍，锁、动态终端宽度、cursor 隐藏、各种残留清理。
把踩到的坑全列出来，每个都对应一类 bug：

### 8.1 行尾残留

**症状**：`99/100 99%` 跳到 `1/100 1%`，屏幕看到的是 `1/100 1%9%`。
**原因**：单行 `\r` 覆写只覆盖前 N 个字符，不动后面。
**修法**：写完每帧后追加 `\033[K`。多行版每行都要追加。

### 8.2 数字位数变化导致行长抖动

**症状**：进度数字位数变化时，整行长度抖动，触发自动折行，重绘错位。
**修法**：要么用定宽格式 `{:>3}/{:>3}`，要么用 `shutil.get_terminal_size()` 算可用
宽度，**并主动留出至少 1 列 margin** 避免触发终端的自动折行。我就是被这个 +1 坑了
（CodeQL 没抓住，UT 抓住了）。

### 8.3 "颜色支持"和"光标控制支持"被当成一回事

很多老代码（包括改前的 Blade）都用一个全局 `_color_enabled` 既决定颜色又决定要不要
用 `\r`。但概念上：

- **颜色**：终端能解析 SGR 转义
- **光标控制**：stdout/stderr 是 TTY，不是被管道/文件重定向

非 TTY 但应用强行 `--color=yes` 的场景就会乱写 `\r` 进文件。Bazel 把 `--color` 和
`--curses` 分成两个独立 flag 就是这个原因。

### 8.4 进度条该写到 stderr

`make` / `ninja` / `cargo` / `bazel` 都是这样。如果写 stdout，`blade build >
out.txt` 会把进度条的字节流写进文件——一堆 `\r` 和未清的转义码。

### 8.5 非 TTY 模式该彻底闭嘴，而不是退化成 `\n` 模式

很多实现非 TTY 时改用 `\n` 终结每一帧，结果 CI 日志里堆几百行进度条。**该闭嘴就闭
嘴**。

### 8.6 并发"清—写"必须原子

清完进度条到下次重画之间，业务线程可能往同一个 stream 写消息。两个写序列交叉就花
屏。修法：**一把全局锁包住"清 + 写 + 重画"**。Python 用 `threading.Lock`，Java
用 `synchronized`。

### 8.7 隐藏 cursor 后必须保证恢复

进度条期间隐藏 cursor（`\033[?25l`）很好看（消除行尾那个 blink block），但 Ctrl+C
退出时如果没恢复，**用户终端会永久没有 cursor**——必须 `reset` 或 `stty echo` 才
回来。修法：

```python
import atexit
atexit.register(restore_cursor)
```

atexit 覆盖正常退出、`sys.exit()`、未捕获异常、`KeyboardInterrupt`。不覆盖
`SIGTERM` / `SIGKILL` / 段错误——如果有这些场景需要兜底，得额外装信号处理器。

### 8.8 log 文件 fd 要么 `with`，要么 atexit

`open()` 之后没 `close()` 不一定泄漏（解释器退出会收回），但 buffer 没 flush 出去
就丢日志了。Blade 的 `_log = open(...)` 我加了 `atexit.register(_log.close)` —— 
异常退出时尾部日志能落盘。这条 CodeQL 还会嘟囔 `py/file-not-closed`（它不识别
atexit），用 `# lgtm[py/file-not-closed]` 抑制即可。

---

## 九、谁想出来的（简短历史）

这套多行进度条实现机制，我没有找到一个具体的发明人，虽然它自己出现不过十来年，但是背后是几十年终端的发展：

| 层级 | 谁贡献的 |
|---|---|
| 底层转义码 `\e[A`, `\e[K` | DEC VT100（1978）→ ANSI X3.64 / ECMA-48 标准化（1979） |
| "保留状态区"的通用抽象 | Ken Arnold，`curses`，~1980 |
| 多行底部进度 + 错误向上滚 | 多个工具共同贡献：`apt`、`dpkg`、`pacman`、`docker pull`（2013 把它推向主流）、`cargo`、`bazel` |
| Bazel 里这套代码 | Ulf Adams 2016-02 起头（commit [`d6347a971e7`][bazel-init]），Klaus Aehlig 长期维护 |

[bazel-init]: https://github.com/bazelbuild/bazel/commit/d6347a971e7

Bazel 最早的类名叫 `ExperimentalEventHandler`（"实验性的 UI"），2019 年 Ulf Adams
自己把它改名成现在的 `UiEventHandler`。当年的 PR 描述里很谦虚地说"加了一个实验性
UI 选项"，到今天这套代码已经是 Bazel 用户每天都在看的 UI 默认实现。

[**Docker pull 2013 年发布**][docker-2013]时的多层并行下载进度，是这套套路真正走向
"日常可见"的转折点——它让大家发现"多个 task、每行一个进度、底部固定"不只是
`apt-get` 那种系统工具的专利，应用程序里也可以这么做。后来 Cargo、npm、Yarn 都跟
进，多行进度条成了"现代 CLI 工具"的视觉标志。

[docker-2013]: https://docs.docker.com/engine/release-notes/

---

## 十、各语言常用库

如果不想自己写，就去用下面这些库，它们把所有累活全给你做了：

- **Python**
  - [`tqdm`](https://github.com/tqdm/tqdm)：最流行的单行进度库，多线程也撑得起。
  - [`rich`](https://github.com/Textualize/rich)：高级 TUI 框架，`rich.progress`
    多行进度条 + 颜色 + spinner 全在内，最像 Bazel 那种工业 UI。
- **Rust**
  - [`indicatif`](https://github.com/console-rs/indicatif)：Rust 生态事实标准，
    `cargo`、`uv`、`rustup` 都用它。`MultiProgress` 的"远程 draw target"机制和
    Bazel `UiEventHandler` 几乎是同构的。
- **Node.js**
  - [`cli-progress`](https://github.com/npkgz/cli-progress)：有 `MultiBar` 模式，
    实现思路和 indicatif 一致。
- **Go**
  - [`mpb`](https://github.com/vbauerster/mpb)：Multi-Progress-Bar，最像 Bazel 的
    Go 版。
- **Java**
  - Bazel 自家的 [`UiEventHandler.java`](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/runtime/UiEventHandler.java)
    本身就是最佳参考，没有独立成库。第三方有 [progressbar](https://github.com/ctongfei/progressbar)
    但相对简陋。

---

## 十一、参考资料

**英文（讲原理）**

- [Build your own Command Line with ANSI escape codes — Li Haoyi](https://www.lihaoyi.com/post/BuildyourownCommandLinewithANSIescapecodes.html) — 最经典入门
- [abusing ANSI escape sequences for minimal progress bars — eskerda](https://eskerda.com/minimal-bash-progress-bar-ansi-escape-sequences/) — bash 最小可运行版
- [Returning the Terminal Cursor to Start of Line with Wrapping Enabled](https://www.funwithlinux.net/blog/returning-the-terminal-cursor-to-start-of-line-with-wrapping-enabled/) — 自动折行的具体坑
- [ANSI Escape Sequences — mbedded.ninja](https://blog.mbedded.ninja/programming/ansi-escape-sequences/) — 系统性 reference

**中文（基础概念）**

- [一日一技：命令行进度条是什么原理？— 腾讯云](https://cloud.tencent.com/developer/article/1903058) — 单行最简明解释
- [ANSI 终端输出瞎搞指北 — learnku（Go 论坛）](https://learnku.com/articles/26231) — 中文里最实用的 ANSI 转义介绍
- [终端的特殊控制符 — JacobZ](https://zyxin.xyz/blog/2020-05/terminal-control-characters/) — ASCII 控制字符到 ANSI 转义的层次梳理
- [启用 Windows 下控制台虚拟终端序列 — CSDN](https://blog.csdn.net/BuluGuy/article/details/88067981) — 跨平台 Windows 端必读

**源码**

- [tqdm `tqdm/std.py` @ v4.67.3](https://github.com/tqdm/tqdm/blob/v4.67.3/tqdm/std.py) / [`tqdm/utils.py` @ v4.67.3](https://github.com/tqdm/tqdm/blob/v4.67.3/tqdm/utils.py) — 本文反复引用的 Python 端实现，`external_write_mode`、`moveto`、`_term_move_up` 都在这里
- [Blade `console.py` @ 295a44d](https://github.com/blade-build/blade-build/blob/295a44d/src/blade/console.py) — 本文写作过程中我顺手加固的版本，文中讨论的所有坑都踩过 + 修过 + 写了单测；Windows VTP 检测 / 开启就在 [`_windows_console_support_ansi_color`](https://github.com/blade-build/blade-build/blob/295a44d/src/blade/console.py#L33-L53) 里
- [Bazel `UiEventHandler.java`](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/runtime/UiEventHandler.java) / [`UiStateTracker.java`](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/runtime/UiStateTracker.java) / [`AnsiTerminal.java`](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/util/io/AnsiTerminal.java) — Java 端的工业级实现，刷新线程 + 锁的细节比 tqdm 多
- [`indicatif/src/multi.rs`](https://github.com/console-rs/indicatif/blob/main/src/multi.rs) — Rust 端的对照实现

---

## 后记

写这篇的直接动机是：我好奇多行进度条怎么实现的，搜中文几乎搜不到，搜英文也
得到处拼凑。回头看其实并不深，每个点都不难——只是涉及好几个独立的小细节，凑齐才
能跑得稳。希望这一篇能省下后来者翻几小时源码的时间。
