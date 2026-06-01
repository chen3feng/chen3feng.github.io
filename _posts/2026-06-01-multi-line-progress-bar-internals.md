---
title: "终端多行进度条是怎么实现的"
date: "2026-06-01 20:00:00 +0800"
categories: [System, CLI]
tags: [Terminal, ANSI, Progress Bar, Bazel, Python, Curses]
layout: post
published: true
---

> 错误信息往上滚、多行进度条永远停在最后、还一秒一跳——还不会被错误信息撞乱。
> 这件事在 Bazel、Docker、Cargo 里见过无数次，单行 `\r` 的小把戏明显不够用。中文社
> 区几乎没人讲透，我顺手研究了一下，把"擦帧—写消息—重画"这一套从 VT100 起源讲到
> Bazel 源码，最后给一个 50 行的 Python 最小可用版。

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

[blade]: https://github.com/blade-build/blade-build

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

Bazel 的做法：用一个 `LineCountingAnsiTerminalWriter` 在画的时候**顺手数行数**，
存到一个全局变量 `numLinesProgressBar` 里——擦的时候按这个数擦。

```java
// bazel/.../UiEventHandler.java（节选并简化）
LineCountingAnsiTerminalWriter countingTerminalWriter =
    new LineCountingAnsiTerminalWriter(terminal);
stateTracker.writeProgressBar(countingTerminalWriter, ...);
numLinesProgressBar = countingTerminalWriter.getWrittenLines();
```

简单粗暴，但是这个簿记**必须严格正确**，差一行整个画面都会崩。

### 3.2 不能信终端的自动折行

终端有个看似无害的特性：写到第 80 列再写一个字符，它会**自动换行**。但各家终端在这
件事上的行为各不一致（有的"eager"——写完第 80 个字符就换；有的"lazy"——写第 81 个
才换）。你以为画了 3 行，实际上其中一行被折成了 2 行——擦的时候按 3 行擦，留下一
行。屏幕花掉。

正解：**主动控制换行**，永远不让任何一行接近终端宽度。Bazel 套了一层
`LineWrappingAnsiTerminalWriter`，按 `terminalWidth - 1` 强制换行，**不让终端帮我
干这件事**。

我在 Blade 这次重构里也踩到了一模一样的坑：

```python
# 错的：bar_width = cols - overhead   ← 整行长度恰好 == cols，某些终端会折
# 对的：bar_width = cols - overhead - 1
```

CodeQL 没抓住，UT 抓住了——我专门写了一条"整行长度严格小于终端宽度"的不变式测试，
跑出来在 `cols=60` 这一档恰好踩中。

### 3.3 `\033[K` 还是 `\033[2K`？

```
ESC [ K   = ESC [ 0 K   # 清光标到行尾  ← 推荐
ESC [ 1 K              # 清行首到光标
ESC [ 2 K              # 清整行（不影响光标）
```

很多博客抄来抄去都用 `\033[2K`，但配合 `\r`（光标到列 0）用的话，`\033[0K` 和
`\033[2K` 视觉效果一致——而 `\033[0K` 更精确："只清光标后面的内容"。Bazel 用 `K`，
Blade 改完也是 `K`，我也推荐这个。

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

Bazel 的 [UiEventHandler.handleLocked][handle-locked] 里清清楚楚就是这个模式，
`synchronized (this)` 把"擦—写—重画"三步包成原子操作：

```java
// 简化版
case ERROR: case WARNING: case INFO: ...
    clearProgressBar();           // ① 擦
    terminal.writeString(...);    // ② 写错误信息 + \n
    if (showProgress && cursorControl) {
        addProgressBar();         // ③ 重画进度条
    }
    terminal.flush();
```

[handle-locked]: https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/runtime/UiEventHandler.java

错误信息留在了原来进度条的位置，紧接着 `\n` 把光标推到下一行，进度条画在那一行——
**视觉上就是消息往上滚、进度条留在最后**。

子进程的 `stdout` / `stderr` 路径有个额外细节：进程输出可能一个字节一个字节来，每
来一次就触发"擦—写—重画"会疯狂闪烁。Bazel 的做法是**行缓冲**——没看到 `\n` 就只
塞进 buffer，看到 `\n` 才整行一次性吐出去。这样进度条只在"完整一行"边界被打扰，视
觉非常稳。

### 后台刷新线程

如果只在"有事件发生时"才刷新，进度条上那个秒数计数器就不会跳。Bazel 起了一个专门
的线程 `cli-update-thread`，每 `minimalUpdateInterval` 毫秒（最小 200ms）醒一次，
触发一次重绘：

```java
new Thread(() -> {
    while (!shutdown) {
        Thread.sleep(minimalUpdateInterval);
        doRefresh(/* fromUpdateThread= */ true);
    }
}, "cli-update-thread").start();
```

这个 200ms 是个经验值：肉眼觉得"活着"，又不会刷屏太频繁；`ninja`、`cargo`、`pip`
基本都在 100~300ms 这个量级。

### 并发的两层防护

后台刷新线程和业务线程会撞。Bazel 用两层防护：

1. `handleLocked()` 整个用 `synchronized(this)` 包住——"擦—写—重画"三步对外原子。
2. 后台 `doRefresh()` 用 `updateLock.tryLock()`，**拿不到锁就直接放弃这次刷新**——
   因为反正马上有下一次，丢一帧无所谓。这样业务线程永远不会被刷新阻塞，刷新也不会
   插进业务线程的写序列中间。

---

## 五、动手写一个：50 行 Python 最小可用版

光看 Bazel 容易"觉得自己懂了"，自己写一遍才是真懂。下面这段代码把前面所有要点都体
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

这三个反例分别对应 Bazel `UiEventHandler` 里**最关键的三块代码**：清屏 dance、
`synchronized` 互斥、`numLinesProgressBar` 行数簿记。再加一个 cursor 的 atexit
钩子，构成了我在 Blade 里给 `console.py` 补的所有内容。

---

## 六、用 curses 行不行？

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

## 七、我在 Blade 里掉过的坑（你自己实现的话大概率也会掉进去😄）

我刚把 Blade 的 `console.py` 加固了一遍，锁、动态终端宽度、cursor 隐藏、各种残留清理。
把踩到的坑全列出来，每个都对应一类 bug：

### 7.1 行尾残留

**症状**：`99/100 99%` 跳到 `1/100 1%`，屏幕看到的是 `1/100 1%9%`。
**原因**：单行 `\r` 覆写只覆盖前 N 个字符，不动后面。
**修法**：写完每帧后追加 `\033[K`。多行版每行都要追加。

### 7.2 数字位数变化导致行长抖动

**症状**：进度数字位数变化时，整行长度抖动，触发自动折行，重绘错位。
**修法**：要么用定宽格式 `{:>3}/{:>3}`，要么用 `shutil.get_terminal_size()` 算可用
宽度，**并主动留出至少 1 列 margin** 避免触发终端的自动折行。我就是被这个 +1 坑了
（CodeQL 没抓住，UT 抓住了）。

### 7.3 "颜色支持"和"光标控制支持"被当成一回事

很多老代码（包括改前的 Blade）都用一个全局 `_color_enabled` 既决定颜色又决定要不要
用 `\r`。但概念上：

- **颜色**：终端能解析 SGR 转义
- **光标控制**：stdout/stderr 是 TTY，不是被管道/文件重定向

非 TTY 但应用强行 `--color=yes` 的场景就会乱写 `\r` 进文件。Bazel 把 `--color` 和
`--curses` 分成两个独立 flag 就是这个原因。

### 7.4 进度条该写到 stderr

`make` / `ninja` / `cargo` / `bazel` 都是这样。如果写 stdout，`blade build >
out.txt` 会把进度条的字节流写进文件——一堆 `\r` 和未清的转义码。

### 7.5 非 TTY 模式该彻底闭嘴，而不是退化成 `\n` 模式

很多实现非 TTY 时改用 `\n` 终结每一帧，结果 CI 日志里堆几百行进度条。**该闭嘴就闭
嘴**。

### 7.6 并发"清—写"必须原子

清完进度条到下次重画之间，业务线程可能往同一个 stream 写消息。两个写序列交叉就花
屏。修法：**一把全局锁包住"清 + 写 + 重画"**。Python 用 `threading.Lock`，Java
用 `synchronized`。

### 7.7 隐藏 cursor 后必须保证恢复

进度条期间隐藏 cursor（`\033[?25l`）很好看（消除行尾那个 blink block），但 Ctrl+C
退出时如果没恢复，**用户终端会永久没有 cursor**——必须 `reset` 或 `stty echo` 才
回来。修法：

```python
import atexit
atexit.register(restore_cursor)
```

atexit 覆盖正常退出、`sys.exit()`、未捕获异常、`KeyboardInterrupt`。不覆盖
`SIGTERM` / `SIGKILL` / 段错误——如果有这些场景需要兜底，得额外装信号处理器。

### 7.8 log 文件 fd 要么 `with`，要么 atexit

`open()` 之后没 `close()` 不一定泄漏（解释器退出会收回），但 buffer 没 flush 出去
就丢日志了。Blade 的 `_log = open(...)` 我加了 `atexit.register(_log.close)` —— 
异常退出时尾部日志能落盘。这条 CodeQL 还会嘟囔 `py/file-not-closed`（它不识别
atexit），用 `# lgtm[py/file-not-closed]` 抑制即可。

---

## 八、谁想出来的（简短历史）

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

## 九、各语言常用库

如果不想自己写，下面这些库已经把所有细节做对了：

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
  - Bazel 自家的 [`UiEventHandler.java`][handle-locked] 本身就是最佳参考，没有独立
    成库。第三方有 [progressbar](https://github.com/ctongfei/progressbar) 但相对简
    陋。

---

## 十、参考资料

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

- [Bazel `UiEventHandler.java`](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/runtime/UiEventHandler.java) / [`UiStateTracker.java`](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/runtime/UiStateTracker.java) — 本文反复引用的工业级实现
- [Bazel `AnsiTerminal.java`](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/util/io/AnsiTerminal.java) — 最底层的转义码封装，干净利落
- [`indicatif/src/multi.rs`](https://github.com/console-rs/indicatif/blob/main/src/multi.rs) — Rust 端的对照实现
- [Blade `console.py`](https://github.com/blade-build/blade-build/blob/master/src/blade/console.py) — 本文写作过程中我顺手加固的版本，本文里讨论的所有坑它都踩过 + 修过 + 写了单测

---

## 后记

写这篇的直接动机是：我好奇多行进度条怎么实现的，搜中文几乎搜不到，搜英文也
得到处拼凑。回头看其实并不深，每个点都不难——只是涉及好几个独立的小细节，凑齐才
能跑得稳。希望这一篇能省下后来者翻几小时源码的时间。
