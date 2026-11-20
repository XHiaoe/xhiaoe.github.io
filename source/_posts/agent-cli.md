---
title: agent_cli
date: 2026-11-20 20:47:12
tags: [agent_cli]
categories:
description:
cover:
sticky:
---
# 只改 CLI：不碰环境


---

## 目录

- [0. 开始之前（在哪敲、边界）](#0-开始之前)
- [1. 先摸清你手上这个 kimi](#1-先摸清你手上这个-kimi)
- [2. 最小可用命令](#2-最小可用命令)
- [3. 参数逐个讲](#3-参数逐个讲)
- [4. 随身 agent 文件（核心）](#4-随身-agent-文件核心)
- [5. 五种场景的完整命令](#5-五种场景的完整命令)
- [6. 没有 --agent-file 时的退路](#6-没有---agent-file-时的退路)
- [7. 输出怎么取](#7-输出怎么取)
- [8. 一场比赛的完整命令流](#8-一场比赛的完整命令流)
- [9. 参数层能做到什么、做不到什么](#9-参数层能做到什么做不到什么)
- [10. 常见问题](#10-常见问题)
- [附：速查卡](#附速查卡)

---

## 0. 开始之前

### 0.1 在哪敲、在哪个目录执行

这是全文最该先看的一段。**本文的命令不是随便在哪个目录敲都一样**——`kimi` 会把**当前目录**当成默认工作区，你站在哪，它就只能看见哪。

#### 窗口在哪

Kali 桌面按 `Ctrl` + `Alt` + `T` 打开终端（或应用菜单搜 `Terminal`）。

不是文件管理器，也不是编辑器里的内置终端。编辑器终端有时不继承你 `export` 的环境变量，会出现"明明设了 key 却报未授权"。

#### 先确认你是谁、站在哪

```bash
whoami        # 通常是 kali
pwd           # 通常是 /home/kali
echo $SHELL   # Kali 默认 /usr/bin/zsh
```

`~` 就等于 `pwd` 打印的那个目录。下文所有 `~` 都指它。

#### 三个目录，各管一件事

```
~/ctf/
├── challs/     题目。一题一个子目录，你 cd 进去再跑 kimi
├── agents/     agent 人设文件（.md）。用绝对路径传给 --agent-file
└── out/        运行日志（.jsonl / .err）。任何时候都能用绝对路径访问
```

建一次就够：

```bash
mkdir -p ~/ctf/challs ~/ctf/agents ~/ctf/out
```

#### 「在哪个目录执行」总表

| 你要做的事 | 先 `cd` 到 | 为什么 | 本文位置 |
|---|---|---|---|
| 摸 kimi 有哪些参数 | **任意** | 不读文件 | 第 1 节 |
| 跑单道题（交互或非交互） | **题目目录** | kimi 把 cwd 当工作区 | 2.1、5.1–5.3 |
| 并行跑多道题 | **题目父目录** | `find .` 扫的是 cwd | 5.4、8.3 |
| 写 / 改 agent 人设 | **任意** | 用绝对路径传参 | 第 4 节 |
| 验证人设生效 | **任意** | 不读题目文件 | 4.4 |
| 收 flag、看日志 | **任意** | 全程绝对路径 | 7.x、8.4–8.5 |
| 定义 `ctf()` 函数 | **任意** | 函数内部自己算绝对路径 | 5.5 |

**只有两种命令真的受目录影响**：

1. `kimi` 本身 —— 它把 cwd 当作 agent 的工作区。你在 `~/ctf/challs` 敲，它就以为整个 `challs` 目录都是工作区，可能翻到别的题去。
2. `find .` —— 点号就是 cwd。多题并行那段必须站在题目父目录。

其余命令（`grep ~/ctf/out/*.jsonl`、`kimi --help`、建目录）都用绝对路径或不读文件，在哪敲都一样。

#### 一个防错习惯

每段命令执行前，先看一眼当前位置：

```bash
pwd
```

出现在题目目录（比如 `~/ctf/challs/re-check`）就对了，出现在 `~` 就说明你少 `cd` 了一步。

> 本文代码块里的 `# 位置：xxx` 注释，标的是**这段命令必须在那个目录下执行**。没标的，默认任意目录均可。

### 0.2 这份文档的边界

**只改 CLI 的意思是：**

- 你的所有改动都发生在**命令行**上
- 落盘的东西只有两样：**一个 `.md` 文件**（agent 人设）和**运行日志**，都在你自己的 `~/ctf/` 下
- `~/.kimi-code/` 下的配置、skills、agents 目录，**一个字节都不动**

**换来什么：**

| 好处 | 说明 |
|---|---|
| 环境重置不影响 | 赛方重装镜像，你的东西还在 U 盘/仓库里 |
| 不需要写权限 | 家目录只读也能用 |
| 可复盘 | agent 文件和命令行都进 git，赛后能还原当时的设定 |
| 每题可不同 | RE 用一套人设、PWN 用另一套，切换只是换个参数 |

**代价（必须提前知道）：**

| 代价 | 说明 |
|---|---|
| 轮数上限改不了 | 内核定的，参数层没有开关 |
| 输出截断改不了 | 同上 |
| 上下文压缩时机改不了 | 只能靠 prompt 让 agent 自己少读大文件 |
| 并发要自己搭 | 用 `xargs` 拉多个进程 |

**先确认一件事**：有些比赛规则明文禁止修改 agent 行为或注入自定义指令。花两分钟看赛事公告的「环境说明」，比事后申诉划算。

---

## 1. 先摸清你手上这个 kimi

版本不同，参数差异很大。**别照抄任何文档（包括这份）**，先看它自己怎么说。

### 1.1 看版本和帮助

```bash
kimi --version
kimi --help
```

把 `--help` 的完整输出存一份，出问题时对照：

```bash
kimi --help > ~/kimi-help.txt 2>&1
wc -l ~/kimi-help.txt
```

### 1.2 一次性检查关键参数在不在

```bash
kimi --help 2>&1 | grep -E -- '--(agent-file|add-dir|output-format|model|skills-dir|mcp-config-file|environment)' || echo "一个都没匹配到，看下面的手动核对"
```

**对照这张表决定你走哪条路：**

| 有没有这个参数 | 你能做什么 | 走本文第几节 |
|---|---|---|
| `--agent-file` | 换掉主 agent 的人设和角色 | **第 4 节（最优）** |
| `--add-dir` | 限定工作目录，避免乱翻 | 第 2、5 节 |
| `--output-format` | 机器可读输出，能批处理 | 第 5、7 节 |
| `--model` | 难题切 thinking 模型 | 第 3 节 |
| 一个都没有 | 只能用 `-p` 塞 prompt | 第 6 节 |

### 1.3 如果 `kimi` 命令本身就没有

```bash
command -v kimi || ls ~/.local/bin/kimi /usr/local/bin/kimi 2>/dev/null
```

找不到就说明环境没配好，先解决这个，本文后面全部依赖它。

---

## 2. 最小可用命令

先跑通一条，再谈优化。

### 2.1 进题目目录

```bash
# 位置：任意目录开始，下面这句会把你带进题目目录
cd ~/ctf/challs/web-easy
pwd
ls -la
```

`pwd` 应该打印 `/home/kali/ctf/challs/web-easy`。**打印的不是这个，就先别往下走**——说明题目没解压到那，或者目录名拼错了。

**为什么要先 cd**：`kimi` 默认把**当前目录**当作工作区。你不 cd，它就在你启动的地方乱找。

> **2.1 到 2.4 这四条命令，都在同一个终端窗口、同一个目录下连续执行。** 中间不要开新终端——新终端会回到 `~`，位置就丢了。

### 2.2 敲第一条命令

```bash
# 位置：~/ctf/challs/web-easy
kimi -p "先 ls 看清目录里的文件，用 file 确认类型，然后告诉我这道题大概是什么方向。先别解题。"
```

**看到什么算正常**：agent 会依次调用 `ls`、`file`，然后给你一段判断。

**这一步不需要任何参数**。跑通了说明基本可用，再往下加。

### 2.3 加上工作目录限定

```bash
# 位置：~/ctf/challs/web-easy
kimi -p "先 ls 看清目录里的文件，用 file 确认类型，判断题目方向。" --add-dir .
```

`--add-dir .` 的作用是把当前目录显式纳入工作范围。**大多数情况下 cd 过去就够了**，加它是为了防止 agent 在 `..` 附近乱翻。

> `--add-dir` 能不能写多个（`--add-dir a --add-dir b`）取决于版本。不确定就 `cd` 到两者的共同父目录，只传一个 `.`。

### 2.4 换成非交互、机器可读

```bash
# 位置：~/ctf/challs/web-easy
kimi -p "判断这道题的类型和解题方向" --add-dir . --output-format text
```

`text` 是纯文本，适合你自己看。要批处理就换 `stream-json`（见第 7 节）。

---

## 3. 参数逐个讲

只讲 CTF 里真会用到的。每个都标了「什么时候别用」。

### 3.1 `-p "..."`：非交互执行

```bash
kimi -p "你的任务描述"
```

- **作用**：给一句 prompt，跑完就退出，不进入交互界面
- **用在哪**：批处理、脚本、`xargs` 并行
- **别用在**：需要来回追问的难题（交互模式更好）
- **注意**：prompt 里最好明确写「当前目录是题目目录」，否则模型可能不确定工作范围

### 3.2 `--add-dir PATH`：限定工作区

```bash
kimi -p "..." --add-dir /home/kali/ctf/challs/web-easy
```

- **作用**：把指定目录纳入 agent 可访问范围
- **用在哪**：不想 cd、或要同时给两个目录（题目 + 共享工具）
- **别用在**：想限制它**只能**访问这个目录——这个参数是「加入」，不是「白名单」

### 3.3 `--output-format text|stream-json`

```bash
kimi -p "..." --output-format text         # 人看
kimi -p "..." --output-format stream-json  # 机器解析
```

- **作用**：控制输出形态
- **用在哪**：`stream-json` 配 `> out.jsonl`，事后用 `jq`/`grep` 提取
- **别用在**：交互模式下（那时没有输出格式的概念）
- **注意**：具体支持哪几种值，以 `kimi --help` 为准

### 3.4 `--model NAME`

```bash
kimi -p "..." --model kimi-k2-thinking
```

- **作用**：本次调用换模型
- **用在哪**：简单题用快模型省钱，难题切 thinking
- **别用在**：不确定模型名的时候——名字错了直接报错
- **怎么知道有哪些名**：看 `kimi provider list`，或问赛方；也可以 `kimi -p "..." --model 随便写` 看报错里有没有候选列表

### 3.5 `--agent-file PATH`：换掉主 agent 的人设（重点）

```bash
# 位置：任意目录。两个路径都写绝对路径，不依赖 cwd
kimi -p "..." \
     --agent-file /home/kali/ctf/agents/ctf.md \
     --add-dir /home/kali/ctf/challs/web-easy
```

- **作用**：用一个 Markdown 文件作为主 agent 的定义，**正文就是它的 system prompt**
- **用在哪**：这是「只改 CLI」能拿到的最强杠杆，详见第 4 节
- **别用在**：文件路径写相对路径（切目录就失效），**一律用绝对路径**
- **注意**：文件格式错（比如 frontmatter 缺字段）可能直接加载失败，先小测再上量

### 3.6 `--skills-dir PATH`：临时指定 skill 目录

```bash
kimi -p "/skill:ctf-re" --skills-dir /home/kali/ctf/skills
```

- **作用**：本次会话从指定目录找 skill，绕过默认搜索路径
- **用在哪**：你带了一套自己的 skill，但不想装进 `~/.kimi-code/skills/`
- **别用在**：只想调 prompt 的时候——skill 比 agent 文件重，能用人设解决就别上 skill

### 3.7 `--mcp-config-file PATH`

```bash
kimi -p "..." --mcp-config-file /home/kali/ctf/mcp.json
```

- **作用**：指定 MCP 配置文件
- **用在哪**：需要接外部服务（很少）
- **别用在**：CTF 里 95% 的情况。`bash` 已经能跑一切，接 MCP 是给自己找麻烦

### 3.8 `kimi mcp add`（子命令）

```bash
kimi mcp add --transport stdio 名字 -- 启动命令 参数
```

- **作用**：往配置里加 MCP server
- **注意**：这会**写配置文件**，和本文「不碰环境」的原则冲突。真要用，先确认规则允许

---

## 4. 随身 agent 文件（核心）

这是全文最重要的一节。一个文件，解决「怎么让通用 coding agent 变成 CTF 解题机」。

### 4.1 放哪、怎么组织

```bash
mkdir -p ~/ctf/agents
```

```
~/ctf/agents/
├── ctf.md          # 通用 CTF 人设（默认用这个）
├── re.md           # 逆向专用
├── pwn.md          # PWN 专用
└── crypto.md       # 密码学专用
```

**为什么放 `~/ctf/agents/`**：和题目目录同级，方便一起打包/进 git/U 盘带走。比赛环境重置不影响。

### 4.2 通用模板（可直接抄）

`~/ctf/agents/ctf.md`：

```markdown
---
name: ctf-solver
description: CTF 竞赛解题 agent，先侦察后动手，拿到 flag 立即报告
---

你是 CTF 竞赛的解题 agent，运行在 Kali Linux 上，当前目录就是题目目录。

## 目标
从题目材料中提取 flag，格式通常为 xxx{...}。拿到后单独一行输出：
FLAG: <完整flag>

## 工作纪律
1. 先侦察，后动手。任何题目第一步都是收集信息：文件类型、大小、字符串、保护机制。
   不要在没看清材料前就开始爆破。
2. 一次只验证一个假设。改一个变量，看结果，再改下一个。
3. 同一思路连续失败 3 次，必须换方向。换之前说明：试了什么、为什么失败、还剩什么没试。
4. 不要重复执行完全相同的命令。如果输出被截断，把结果重定向到文件，再用 grep/sed 看片段。
5. 每步问自己：这个输出说明了什么？下一步能排除哪种可能？

## 工具使用
- bash 是主力，Kali 的命令行工具都可以用（file/strings/r2/gdb/binwalk/python3 等）。
- 输出很长时先 `| head -50` 或 `> out.txt` 再 grep，不要一次读进来。
- 需要交互的连接（pwn 远程、本地服务）用 `tmux` 开会话，不要用 nc 硬等。
- 写脚本用文件（`cat > exp.py <<'EOF' ... EOF`），不要用一行 python -c 堆逻辑。

## 禁止
- 不要提交猜测的 flag。只有从题目数据中实际提取到的才算。
- 不要修改题目目录之外的文件。
- 不要执行破坏性命令（rm -rf /、mkfs、dd 写裸盘）。
- 不要 pip install / apt install，环境是固定的。

## 输出
- 调工具前，一句话说明你要验证什么。
- 拿到 flag 后立即输出 FLAG: 行，然后用 3-5 句话总结关键步骤。
- 没解出来就明确说「未解出」和卡在哪一步，不要编。
```

**四段结构的作用**：目标（做什么）→ 纪律（怎么做）→ 工具（用什么）→ 禁令（别做什么）。

其中第 3、4 条是专门对抗模型的两个毛病：**沉没成本**（错了还硬试）和**重复尝试**（输出被截断就重跑）。没有这两条，它会在一个错方向上耗完全部时间。

### 4.3 分题型的差异版

不必每题都写全新的，**只在通用版上改「侦察段」**。

`~/ctf/agents/re.md` 替换掉通用版的「工作纪律」前半段：

```markdown
---
name: ctf-re
description: 逆向工程题专用，先静态侦察再动态调试
---

你是逆向工程题解题 agent。当前目录是题目目录，目标是拿到 flag。

## 侦察顺序（固定）
1. `file ./chall` —— 架构、位数、是否 stripped
2. `checksec --file=./chall` —— 保护机制
3. `strings -n 6 ./chall | head -80` —— 明文线索
4. `rabin2 -zz ./chall | grep -iE 'flag|wrong|correct|input'`
5. 是 UPX 就先 `upx -d`

## 定位逻辑
- 有符号：`r2 -A -q -c 'afl' ./chall`，找 main/check/verify
- 无符号：先从字符串交叉引用入手 `r2 -A -q -c 'izz~flag'`

## 算法识别
| 特征 | 可能 |
|---|---|
| 位运算 + 32/64 轮循环 | TEA / XTEA / XXTEA |
| 256 字节 S 盒 + 两次交换 | RC4 |
| 4x4 矩阵 + 9/11/13 轮 | AES |
| 大整数常量 0x9E3779B9 等 | 搜一下魔数 |

## 纪律
- 不要一上来上 angr，先读懂逻辑，符号执行是最后手段。
- Go/Rust 二进制函数多，先用字符串定位，别从 main 顺读。
- 遇到 ptrace 反调试，先 patch 掉再调试。
- 写 solve.py 跑通验证，不要心算。

拿到 flag 后输出 FLAG: <完整flag>。
```

`~/ctf/agents/pwn.md` 的关键差异段：

```markdown
## 侦察顺序（固定）
1. `checksec --file=./chall` —— 记录所有保护
2. `file ./chall && ldd ./chall` —— 架构和 libc
3. `strings ./chall | grep -iE 'system|/bin/sh|flag'`
4. 看 main 和危险函数（gets/read/printf）

## 利用流程
1. 算偏移：`cyclic 200` 生成，崩溃后 `cyclic -l <崩溃值>`
2. 写 exp.py，先本地 `process('./chall')` 打通
3. 再 `remote(host, port)` 打远程
4. 交互式连接必须用 tmux：
   `tmux new-session -d -s pwn1 'nc host port'`
   `tmux send-keys -t pwn1 -l 'payload'` 然后 `tmux send-keys -t pwn1 Enter`
   `tmux capture-pane -p -t pwn1 -S -100`
```

`~/ctf/agents/crypto.md` 的关键差异段：

```markdown
## 流程
1. 先读源码和题目给的数据，别急着写脚本
2. 识别数学结构：小指数 e=3、共模攻击、低加密指数广播、Coppersmith、格基约减
3. 简单计算用 `python3 -c` 快速试
4. 复杂计算写 solve.py，用 z3 做约束求解、gmpy2 做大整数
5. 解出后 `long_to_bytes` 转回可读文本

## 纪律
- 先手算一遍小规模，确认思路再写代码
- 注意 padding 和字节序，90% 的"解出来是乱码"都是这个
```

### 4.4 怎么验证它真的生效了

```bash
# 位置：任意目录（只读人设文件，不碰题目）
kimi -p "用一句话复述你的角色设定里关于「失败 3 次」那条规则" \
     --agent-file ~/ctf/agents/re.md \
     --output-format text
```

**预期**：它应该能说出「同一思路连续失败 3 次必须换方向」这类内容。

如果它答非所问、或者表现得像个通用 coding assistant，说明没加载成功。排查顺序：

1. 路径是不是绝对路径
2. 文件第一行是不是 `---`（frontmatter 起始）
3. `name` 和 `description` 有没有写
4. 这个版本到底支不支持 `--agent-file`（回第 1.2 节）

### 4.5 关于「声明工具集」

有些版本的 agent 文件支持在 frontmatter 里声明工具白名单，字段名可能是 `tools` 或 `allowedTools`——**各版本不统一**。

**不要猜。** 两种稳妥做法：

- 把工具约束写进正文（上面模板就是这么做的），模型会遵守
- 或者 `kimi --help` / 官方 agents 文档确认字段名后再加

写错了可能导致整个文件加载失败，得不偿失。

---

## 5. 五种场景的完整命令

### 5.1 单题交互式（人在旁边看着）

```bash
# 位置：任意目录执行，下面这句把你带进题目目录
cd ~/ctf/challs/re-check
kimi --agent-file ~/ctf/agents/re.md --add-dir .
```

进交互界面，可以来回追问。适合卡题时用。

### 5.2 单题非交互（跑完看结果）

```bash
# 位置：任意目录执行，下面这句把你带进题目目录
cd ~/ctf/challs/re-check
kimi -p "分析当前目录的题目，找出 flag。拿到后单独一行输出 FLAG: <完整flag>。" \
     --agent-file ~/ctf/agents/re.md \
     --add-dir . \
     --output-format text \
     2>&1 | tee ~/ctf/out/re-check.log
```

`tee` 让你既能看输出，又留一份日志。

### 5.3 单题带超时（防卡死）

```bash
# 位置：任意目录执行，下面这句把你带进题目目录
cd ~/ctf/challs/re-check
timeout 900 kimi -p "找出 flag，输出 FLAG: <完整flag>" \
     --agent-file ~/ctf/agents/re.md \
     --add-dir . \
     --output-format stream-json \
     > ~/ctf/out/re-check.jsonl 2> ~/ctf/out/re-check.err
echo "exit=$?"
```

**exit 124 表示超时被杀**，不是失败也不是成功。看到 124 就去翻 jsonl，可能已经拿到 flag 但没来得及输出。

> 注意 `2>` 单独接错误输出。把 stderr 混进 jsonl（`2>&1`）会破坏 JSON 结构，后面 `jq` 全报错。

### 5.4 多题并行（4 个一起跑）

```bash
# 位置：任意目录执行
mkdir -p ~/ctf/out
cd ~/ctf/challs          # ← 关键：必须在题目父目录，find . 扫的就是这里
pwd                      # 确认输出 /home/kali/ctf/challs

find . -maxdepth 1 -mindepth 1 -type d -print0 \
| xargs -0 -P 4 -I{} bash -c '
  d="$1"; n=$(basename "$d")
  timeout 900 kimi -p "分析当前目录的题目，找出 flag，单独一行输出 FLAG: <完整flag>" \
      --agent-file "$HOME/ctf/agents/ctf.md" \
      --add-dir "$PWD/$d" \
      --output-format stream-json \
      > "$HOME/ctf/out/$n.jsonl" 2> "$HOME/ctf/out/$n.err"
  echo "$n exit=$?"
' _ {}
```

三个关键点：

- `find -print0` + `xargs -0`：目录名带空格也不会拆断
- **每题输出到独立文件**：`xargs -P` 的输出会交错，共用 stdout 就全乱了
- `$PWD/$d` 拼绝对路径：`--add-dir` 传相对路径在子进程里可能解析错

> `pwd` 打印的不是 `/home/kali/ctf/challs`，这段一定要重来。站错目录的后果是：要么 `find` 扫不到题（输出为空，xargs 报错），要么把 `~/ctf` 下的 `agents`、`out` 也当成题目跑一遍——后者更隐蔽，你会看到莫名其妙的 agent 文件被分析。

跑完看结果：

```bash
for f in ~/ctf/out/*.jsonl; do
  echo "=== $(basename "$f" .jsonl) ==="
  grep -oE '[A-Za-z][A-Za-z0-9_]{1,24}\{[^{}]{4,200}\}' "$f" | sort -u
done
```

### 5.5 用一个 shell 函数封装（推荐）

每次都敲长参数很烦。定义个函数，**只在当前终端生效，不写进 rc**：

```bash
# 位置：任意目录，写在当前终端会话里即可（关掉终端就没了）
ctf() {
  local dir="${1:-.}"; shift || true
  local prompt="${1:?用法: ctf <题目目录> <prompt>}"
  kimi -p "$prompt" \
       --agent-file "$HOME/ctf/agents/ctf.md" \
       --model "${CTF_MODEL:-kimi-k2-0905-preview}" \
       --add-dir "$(cd "$dir" && pwd)" \
       --output-format text
}
```

用法：

```bash
ctf ~/ctf/challs/web-easy "找出 flag，输出 FLAG: <完整flag>"
CTF_MODEL=kimi-k2-thinking ctf ~/ctf/challs/re-check "这题难，慢慢来"
```

想换 agent 文件：`CTF_AGENT=~/ctf/agents/re.md`，在函数里加一行 `--agent-file "${CTF_AGENT:-...}"`。

> 为什么用函数不用 alias：函数能处理默认值、能拼绝对路径、能读环境变量。alias 只能做字符串替换。

---

## 6. 没有 `--agent-file` 时的退路

如果你的版本不支持这个参数，只剩 `-p` 一个注入点。做法是把 agent 文件的内容**当成 prompt 前缀**塞进去。

### 6.1 把 prompt 写成文件

`~/ctf/agents/ctf-prompt.txt`（就是 4.2 的正文，去掉 frontmatter）：

```text
你是 CTF 竞赛的解题 agent，运行在 Kali Linux 上，当前目录就是题目目录。

工作纪律：
1. 先侦察后动手，任何题目第一步都是收集信息。
2. 一次只验证一个假设。
3. 同一思路失败 3 次必须换方向。
4. 不要重复执行相同命令，输出长了先重定向到文件再 grep。
5. 不要 pip/apt install，环境固定。

拿到 flag 后单独一行输出：FLAG: <完整flag>
没解出来就说「未解出」和卡在哪一步，不要猜。
```

### 6.2 用命令替换把文件内容喂进去

```bash
# 位置：任意目录。注意 --add-dir 要给绝对路径，
# 写 "." 的话指的是你当前所在目录，不是题目目录
kimi -p "$(cat ~/ctf/agents/ctf-prompt.txt)

任务：分析当前目录，找出 flag。" \
     --add-dir ~/ctf/challs/题目
```

> 也可以先 `cd ~/ctf/challs/题目` 再用 `--add-dir .`。两种都行，**但别一边站在 `~` 一边写 `.`**——那等于告诉 agent「工作区是家目录」，它会去翻你所有的东西。

**为什么用 `"$(cat ...)"` 而不是直接贴**：

- 避免命令行里的引号、反引号、`$` 被 shell 解释
- prompt 改了不用改命令
- 可以配合 `sed` 动态插入题型

### 6.3 长度控制

prompt 太长会稀释注意力。经验值：**控制在 2000 字以内**，超过就砍掉「算法识别」这类知识性内容，只留纪律和流程。

有个硬限制：Linux 单个命令行参数上限约 2MB，一般够用，但别把整本手册塞进去。

---

## 7. 输出怎么取

### 7.1 先看结构，再写 jq

`stream-json` 的字段结构会随版本变。**别照抄别人的 jq 表达式**，先看两行：

```bash
head -2 ~/ctf/out/web-easy.jsonl | jq .
```

看清字段名（可能是 `type` / `role` / `content` / `message` 之类），再决定怎么写。

### 7.2 不依赖结构的万能兜底

不管什么格式，flag 都在文本里。直接正则扫：

```bash
grep -hoE '[A-Za-z][A-Za-z0-9_]{1,24}\{[^{}]{4,200}\}' ~/ctf/out/*.jsonl | sort -u
```

这个正则覆盖 `flag{}`、`moectf{}`、`DASCTF{}`、`NSSCTF{}` 各种前缀。

### 7.3 看它调了哪些工具（判断方向对不对）

如果 jsonl 里能看出工具调用记录：

```bash
grep -oE '"(bash|read_file|Read|Write|Bash|Edit)"' ~/ctf/out/web-easy.jsonl | sort | uniq -c
```

工具分布能反映行为：全是 `Bash` 说明在跑命令，全是 `Read` 说明在硬读文件。

### 7.4 错误日志单独看

```bash
cat ~/ctf/out/web-easy.err
```

`2>` 单独接出来的 stderr 里，通常是网络错误、参数错误、权限问题——**这些不会出现在 jsonl 里**。

---

## 8. 一场比赛的完整命令流

### 8.1 开赛前 10 分钟

```bash
# 1) 确认工具在
command -v kimi && kimi --version

# 2) 确认关键参数在（决定走第 4 节还是第 6 节）
kimi --help 2>&1 | grep -c -- '--agent-file'

# 3) 铺好目录
mkdir -p ~/ctf/agents ~/ctf/challs ~/ctf/out

# 4) 拿一道旧题试跑，验证一条命令能走通
timeout 600 kimi -p "分析当前目录，找出 flag，输出 FLAG: <flag>" \
     --agent-file ~/ctf/agents/ctf.md \
     --add-dir ~/ctf/challs/sample \
     --output-format text
```

第 4 步最重要。**没跑通过的组合，比赛现场一定出问题**，而且是最难查的那种。

### 8.2 开赛后

```bash
# 位置：任意目录执行，下句进入题目父目录
# 解压题目（假设是 zip）
cd ~/ctf/challs
for z in ~/Downloads/*.zip; do
  n=$(basename "$z" .zip)
  mkdir -p "$n" && unzip -o "$z" -d "$n" >/dev/null
done

# 先扫一眼都是什么题
for d in */; do echo "== $d"; ls "$d" | head -5; done
```

### 8.3 分波跑

```bash
# 位置：~/ctf/challs（先 pwd 确认；不对就 cd ~/ctf/challs）

# 第一波：便宜的（misc/web）。这批有几道题就起几个进程，题多（>6）改用 5.4 的 xargs -P 4
for d in misc-* web-*; do
  [ -d "$d" ] || continue
  timeout 900 kimi -p "分析当前目录，找出 flag，输出 FLAG: <完整flag>" \
      --agent-file ~/ctf/agents/ctf.md \
      --add-dir "$PWD/$d" --output-format stream-json \
      > ~/ctf/out/"$d".jsonl 2> ~/ctf/out/"$d".err &
done
wait

# 第二波：crypto
for d in crypto-*; do
  [ -d "$d" ] || continue
  timeout 900 kimi -p "分析当前目录，找出 flag，输出 FLAG: <完整flag>" \
      --agent-file ~/ctf/agents/crypto.md \
      --add-dir "$PWD/$d" --output-format stream-json \
      > ~/ctf/out/"$d".jsonl 2> ~/ctf/out/"$d".err &
done
wait
```

### 8.4 盯盘

```bash
# 位置：任意目录
watch -n 30 'ls -la ~/ctf/out/*.jsonl | tail -20'
```

或者每十分钟手动收一次 flag：

```bash
# 位置：任意目录
grep -hoE '[A-Za-z][A-Za-z0-9_]{1,24}\{[^{}]{4,200}\}' ~/ctf/out/*.jsonl | sort -u
```

### 8.5 收尾

```bash
# 位置：任意目录
grep -h "FLAG:" ~/ctf/out/*.jsonl | sort -u > ~/ctf/flags.txt
cat ~/ctf/flags.txt
```

---

## 9. 参数层能做到什么、做不到什么

这张表是全文的结论。**买之前先看清楚不包含什么。**

| 能力 | 参数层能不能做 | 说明 |
|---|---|---|
| 换模型 | 能 | `--model` |
| 限定工作目录 | 能 | `--add-dir` |
| 换 agent 人设 | 能 | `--agent-file` |
| 非交互批处理 | 能 | `-p` + `--output-format` |
| 多进程并行 | 能（外部） | `xargs -P` 或 `&` + `wait` |
| 超时控制 | 能（外部） | `timeout` |
| 每题独立日志 | 能（外部） | 重定向到独立文件 |
| **改轮数上限** | **不能** | 内核定，无开关 |
| **改输出截断阈值** | **不能** | 只能靠 prompt 让 agent 自己少读 |
| **改上下文压缩时机** | **不能** | 交互模式下可手动 `/compact` |
| **按题记账** | **不能** | 只有总账单 |
| **断点续跑** | **不能** | 只能靠 `.done` 标记文件勉强凑合 |
| **改工具实现** | **不能** | 那是代码层的事 |

**一句话**：参数层能改「它扮演谁、在哪干活、跟谁说话」，改不了「它怎么思考、怎么记账」。

真需要后三行那些能力，只有两条路：改 `~/.kimi-code`（第 6.6 节），或者自己写调度层（手册第 5 章）。

---

## 10. 常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| `kimi: command not found` | 没装或不在 PATH | `command -v kimi`；找 `~/.local/bin`、`/usr/local/bin` |
| `unrecognized arguments: --agent-file` | 版本旧 | 走第 6 节，用 `-p` 塞 prompt |
| `Error: agent file not found` | 用了相对路径 | 换绝对路径 `~/ctf/agents/ctf.md` 展开后的完整路径 |
| 加载 agent 文件后行为没变 | frontmatter 格式错 | 第一行必须是 `---`，且 `name`/`description` 都有 |
| 输出里出现乱码方块 | 终端编码或二进制输出 | `\| iconv -f utf-8 -t utf-8 -c`，或直接看日志文件 |
| `jq: parse error` | stderr 混进了 jsonl | 别用 `2>&1`，改 `2> xxx.err` |
| 跑到一半卡住 | 模型在长推理，或命令在等输入 | 交互命令加 `</dev/null`；整体加 `timeout` |
| 一道题跑很久没动静 | 内核在压缩上下文 | 正常，等；超过 15 分钟考虑杀掉重跑 |
| `exit=124` | `timeout` 杀掉了 | 不是失败，去 jsonl 里翻有没有已经输出的 flag |
| flag 提取出来是假的 | 模型猜的 | agent 文件里明确「不要提交猜测的 flag」，且只信 `FLAG:` 行 |
| 并行时输出互相覆盖 | 共用了同一个输出文件 | 每题一个文件名，用 `$n.jsonl` |

---

## 附：速查卡

```bash
# ========== 位置：任意目录 ==========

# 看家底
kimi --version
kimi --help | grep -E -- '--(agent-file|add-dir|output-format|model)'

# 收 flag（不依赖输出格式）
grep -hoE '[A-Za-z][A-Za-z0-9_]{1,24}\{[^{}]{4,200}\}' ~/ctf/out/*.jsonl | sort -u

# 没有 --agent-file 时的退路（--add-dir 必须给绝对路径，否则 . 指向你当前所在目录）
kimi -p "$(cat ~/ctf/agents/ctf-prompt.txt)

任务：分析当前目录，找出 flag。" --add-dir ~/ctf/challs/题目


# ========== 位置：题目目录 ~/ctf/challs/<题名> ==========

cd ~/ctf/challs/题目
pwd          # 必须打印 /home/kali/ctf/challs/题目

# 单题交互
kimi --agent-file ~/ctf/agents/ctf.md --add-dir .

# 单题非交互 + 留日志
timeout 900 kimi -p "找出 flag，输出 FLAG: <完整flag>" \
  --agent-file ~/ctf/agents/ctf.md --add-dir . \
  --output-format stream-json > ~/ctf/out/题目.jsonl 2> ~/ctf/out/题目.err


# ========== 位置：题目父目录 ~/ctf/challs ==========

cd ~/ctf/challs
pwd          # 必须打印 /home/kali/ctf/challs

# 多题并行
find . -maxdepth 1 -mindepth 1 -type d -print0 | xargs -0 -P 4 -I{} bash -c '
  d="$1"; n=$(basename "$d")
  timeout 900 kimi -p "找出 flag，输出 FLAG: <完整flag>" \
    --agent-file "$HOME/ctf/agents/ctf.md" --add-dir "$PWD/$d" \
    --output-format stream-json > "$HOME/ctf/out/$n.jsonl" 2> "$HOME/ctf/out/$n.err"
' _ {}
```

**三条纪律**：

1. **路径全用绝对路径**——`--agent-file` 和 `--add-dir` 在子进程里对相对路径的解析不一定如你所愿
2. **stderr 单独接**——`2> xxx.err`，别 `2>&1`，否则 jsonl 会坏
3. **每题一个输出文件**——并行的前提，否则日志互相覆盖

**一个边界**：这套方案能改「谁在解题、在哪解题、怎么汇报」，改不了「它跑多少轮、读多少字、花多少钱」。要那三样，得往下走一层。
