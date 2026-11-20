---
title: AI  Agent 
date: 2026-11-05 11:51:40
tags: [agent]
categories: [笔记]
description: 赛制约束下的环境适配、降级方案与提交清单
cover:
sticky:
---


---

# 赛制合规与容器适配

> 适用赛制：赛方统一提供 **Docker 环境** + **授权容器网络** + **仅指定工作目录可读写** + **受控互联网出口** + **获准的模型接口**
> 本文定位：**落地层**。它不替代前三份技术文档，而是在它们之上加一层约束——
> **凡是本文与前三份冲突的地方，以本文为准。**
> 依据：赛事《AI 靶场赛制》五条。

***

## 目录

* [0. 为什么需要这一层](#0-为什么需要这一层)
* [1. 开赛前必须问清的 7 件事](#1-开赛前必须问清的-7-件事)
* [2. 容器内 5 分钟自检](#2-容器内-5-分钟自检)
* [3. 工作目录布局](#3-工作目录布局)
* [4. 模型接口适配](#4-模型接口适配)
* [5. 依赖装不上时的降级方案](#5-依赖装不上时的降级方案)
* [6. 按难度递进的调度](#6-按难度递进的调度)
* [7. 提交物清单与打包](#7-提交物清单与打包)
* [8. 红线对照表](#8-红线对照表)
* [9. 赛前 60 分钟清单](#9-赛前-60-分钟清单)
* [10. 出事怎么办](#10-出事怎么办)

***

## 0. 为什么需要这一层

前三份文档有一个共同假设：**你在自己的 Kali 上，有 sudo、能装包、能改家目录**。

这个赛制把三条全否掉了：

| 前三份文档的假设                  | 赛制的实际约束        |
| ------------------------- | -------------- |
| Kali 本机，`apt install` 随便装 | 受控出口，**装不了**   |
| `~/.kimi-code/` 随便改       | **仅指定工作目录可读写** |
| `api.moonshot.cn` 直连      | **只能用获准的模型接口** |
| 日志随便留，跑完就丢                | **必须提交，禁止清除**  |

所以本文干四件事：**问清楚、摸清楚、摆正确、交整齐**。

先把三份文档的路径映射到容器里，后面都按这个来：

| 原方案（本机）                       | 容器内改成                         |
| ----------------------------- | ----------------------------- |
| `~/ctf/challs`                | `$WORK/challs`                |
| `~/ctf/agents`                | `$WORK/agents`                |
| `~/ctf/out`                   | `$WORK/out`                   |
| `~/.venvs/ctfagent`           | `$WORK/.venv`                 |
| `~/.kimi-code/skills/`（不能改）   | `--skills-dir $WORK/skills`   |
| `https://api.moonshot.cn/v1`  | `$MODEL_BASE_URL`             |
| `~/ctf/tools/ctfagent_min.py` | `$WORK/tools/ctfagent_min.py` |

`$WORK` 就是赛方说的「指定工作目录」。**下面所有命令都假设你先** **`cd $WORK`。**

***

## 1. 开赛前必须问清的 7 件事

这七个问题的答案，决定了你能不能跑起来。**别猜，直接问赛方**（群里问或邮件问都行，比赛通常有答疑通道）。

| # | 问什么                                                   | 为什么必须问                 | 猜错的后果       |
| - | ----------------------------------------------------- | ---------------------- | ----------- |
| 1 | 模型接口的 **base_url / key / 可用模型名**                     | 代码默认打公网，容器里大概率不通       | 一道题都跑不了     |
| 2 | 接口是否 **OpenAI 兼容**（`/v1/chat/completions`、`tools` 字段） | `AsyncOpenAI` 只能对接兼容接口 | 客户端要重写      |
| 3 | 镜像里**预装了什么**（python / pwntools / tmux / r2 / gdb）     | 装不了包，只能用手上有的           | pwn 题直接废    |
| 4 | **指定工作目录的绝对路径**                                       | 所有产物、提交物都得在这           | 写错地方等于没交    |
| 5 | 容器**有没有 root / sudo / 出网**                            | 决定能不能装依赖               | 方案要降级       |
| 6 | **提交格式、路径、截止时间**                                      | 交错等于白做                 | 直接零分        |
| 7 | **自建 Agent 的提交要求**（源码怎么审、要不要说明）                       | 赛制允许自建，但要提交源码          | 说不清就可能被判定违规 |

可以直接复制这段去问：

```text
老师您好，关于 AI 靶场赛制有几个环境问题想确认：

1. 模型接口的 base_url、API Key 和可用模型名分别是什么？接口是否兼容
   OpenAI 的 /v1/chat/completions 格式（支持 tools 字段）？
2. 提供的 Docker 镜像里，是否预装了 python3、pwntools、tmux、radare2、
   gdb、binwalk 等工具？能否提供工具清单？
3. 指定工作目录的绝对路径是什么？该目录是否可写、有没有容量限制？
4. 容器内是否有 root 或 sudo 权限？能否访问外网或内网 pip 源？
5. 提交物的格式、路径和截止时间？源码、Agent 配置、提示词、依赖清单、
   日志分别怎么组织？
6. 使用自建 Agent 时，除了源码还需要提交什么（比如架构说明、运行方式）？
7. 比赛过程中是否有网络或 API 调用审计？哪些行为会被判定为违规？

谢谢！
```

> 第 7 问看着多余，其实最该问。**知道会被审计，你的 agent 设计就会不一样**（比如 prompt 里不写任何可能被误解成"外传"的指令）。

***

## 2. 容器内 5 分钟自检

进容器第一件事不是写代码，是**摸清底细**。把下面脚本存成 `$WORK/check_env.sh`，跑一遍。

```bash
#!/usr/bin/env bash
# 容器环境自检 —— 输出一份可直接贴给队友的报告
set -uo pipefail
WORK="${WORK:-$(cd "$(dirname "$0")" && pwd)}"   # 没传 WORK 就用脚本所在目录
cd "$WORK" 2>/dev/null || { echo "无法进入工作目录: $WORK"; exit 1; }

echo "===== 1. 身份与权限 ====="
echo "user  : $(id -un 2>/dev/null || whoami)"
echo "uid   : $(id -u 2>/dev/null)"
echo "home  : ${HOME:-未设置}"
if command -v sudo >/dev/null 2>&1; then
  sudo -n true 2>/dev/null && echo "sudo  : 免密可用" || echo "sudo  : 需要密码或不可用"
else
  echo "sudo  : 未安装"
fi

echo
echo "===== 2. 工作目录与磁盘 ====="
echo "pwd   : $(pwd)"
df -h . 2>/dev/null | awk 'NR==2{print "disk  : "$4" 可用 / "$2" 总量"}'
if touch .__wtest 2>/dev/null; then rm -f .__wtest; echo "可写  : 是"; else echo "可写  : 否（严重问题，立即联系赛方）"; fi

echo
echo "===== 3. 网络 ====="
if [ -n "${MODEL_BASE_URL:-}" ]; then
  code=$(curl -s -m 8 -o /dev/null -w '%{http_code}' "$MODEL_BASE_URL/models" \
         -H "Authorization: Bearer ${MODEL_API_KEY:-none}" 2>/dev/null)
  code=${code:-000}
  echo "模型接口: HTTP $code  ($MODEL_BASE_URL)"
else
  echo "模型接口: MODEL_BASE_URL 未设置 —— 先向赛方要地址"
fi
code=$(curl -s -m 6 -o /dev/null -w '%{http_code}' https://pypi.org/simple/ 2>/dev/null)
code=${code:-000}
if [ "$code" = "000" ]; then echo "pypi    : 不通（装不了 pip 包，走降级方案）"; else echo "pypi    : HTTP $code"; fi

echo
echo "===== 4. 命令行工具 ====="
miss=""
for t in python3 pip curl jq git file strings xxd readelf objdump \
         nc tmux r2 gdb binwalk foremost exiftool unzip timeout; do
  if command -v "$t" >/dev/null 2>&1; then
    printf '  %-10s OK\n' "$t"
  else
    printf '  %-10s MISSING\n' "$t"
    miss="$miss $t"
  fi
done
[ -n "$miss" ] && echo "  缺失:$miss"

echo
echo "===== 5. Python 库 ====="
python3 - <<'PY' 2>/dev/null || echo "  python3 不可用"
mods = [("pwn","pwntools"),("z3","z3-solver"),("Crypto","pycryptodome"),
        ("capstone","capstone"),("gmpy2","gmpy2"),("elftools","pyelftools"),
        ("requests","requests"),("openai","openai")]
for mod, pkg in mods:
    try:
        __import__(mod)
        print(f"  {pkg:<14} OK")
    except Exception:
        print(f"  {pkg:<14} MISSING")
PY

echo
echo "===== 6. 模型接口功能验证 ====="
if [ -n "${MODEL_BASE_URL:-}" ] && [ -n "${MODEL_API_KEY:-}" ]; then
  curl -s -m 25 "$MODEL_BASE_URL/chat/completions" \
    -H "Authorization: Bearer $MODEL_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"${MODEL_NAME:-kimi-k2-0905-preview}\",\"messages\":[{\"role\":\"user\",\"content\":\"回复两个字：通了\"}],\"max_tokens\":20}" \
    2>/dev/null | head -c 400
  echo
else
  echo "  跳过（缺 MODEL_BASE_URL 或 MODEL_API_KEY）"
fi
```

### 怎么读这份报告

| 看到什么               | 意味着                      | 动作                  |
| ------------------ | ------------------------ | ------------------- |
| `可写: 否`            | 工作目录不可写                  | **立刻找赛方**，这是致命问题    |
| 模型接口 `HTTP 200`    | 通了                       | 继续                  |
| 模型接口 `HTTP 401`    | key 有问题                  | 核对 key，注意别带空格       |
| 模型接口 `HTTP 404`    | base_url 少了 `/v1` 或路径不对 | 试试加/去 `/v1`         |
| 模型接口 `000`         | 网络不通                     | 确认是否需要代理或内网 DNS     |
| `pypi: 不通`         | 装不了包                     | 走第 5 节降级方案          |
| `tmux MISSING`     | pwn 交互方案失效               | 走 pty 替代（5.2）       |
| `pwntools MISSING` | 没有 `checksec`            | 走 readelf 手判（5.2）   |
| `r2 MISSING`       | 没有反汇编器                   | 用 `objdump -d`（5.2） |

**这份报告要存档**，贴进提交材料里能证明环境限制，出问题时也是证据。

***

## 3. 工作目录布局

赛制说「仅可在指定工作目录读写和配置 Agent」，所以**所有东西都放这里，一处都别外溢**。

### 3.1 建骨架

```bash
# 位置：任意目录开始，第一句把你带进工作目录
export WORK="/workspace"        # ← 改成赛方给的实际绝对路径
cd "$WORK"
pwd                             # 必须打印赛方给的那个路径

mkdir -p agents prompts challs out tools skills submit .cache
ls -F
```

```
$WORK/
├── agents/         agent 人设（.md）        ← 要提交
├── prompts/        独立 prompt 文本          ← 要提交
├── challs/         题目（一题一个子目录）
├── out/            运行日志（.jsonl/.err）   ← 要提交，禁止删
├── tools/          自建 agent 源码           ← 要提交
├── skills/         自定义 skill（--skills-dir 指向这）
├── submit/         打包产物
└── .cache/         临时文件（pip 下载、解包中间物）
```

### 3.2 哪段代码存成哪个文件

这是最容易搞混的地方：**文档里的代码块有两种，一种是"敲一遍就完事的命令"，一种是"要存成文件、以后反复用的脚本"**。分不清就会把 `curl` 存成 `.sh`，或者把该留的脚本敲完就丢了。

#### 先记三条判断规则

| 代码块长什么样                                 | 属于           | 怎么处理                         |
| --------------------------------------- | ------------ | ---------------------------- |
| 以 `#!/usr/bin/env bash` 开头              | **脚本**       | 存成 `$WORK/xxx.sh`，`chmod +x` |
| 以 `---` 开头的 Markdown（frontmatter）       | **agent 人设** | 存成 `$WORK/agents/xxx.md`     |
| 纯命令序列（无 shebang，如 `curl`/`mkdir`/`for`） | **一次性命令**    | 直接敲，不用存                      |

#### 文件落位表

| 内容          | 出自          | 存成                | 放哪               | 怎么用                 | 要不要提交   |
| ----------- | ----------- | ----------------- | ---------------- | ------------------- | ------- |
| 环境自检脚本      | 本文 §2       | `check_env.sh`    | `$WORK/`         | `bash check_env.sh` | 建议（作证据） |
| 提交打包脚本      | 本文 §7.4     | `pack.sh`         | `$WORK/`         | `bash pack.sh`      | 是       |
| 通用 agent 人设 | 《只改CLI》§4.2 | `ctf.md`          | `$WORK/agents/`  | `--agent-file`      | **是**   |
| 逆向人设        | 《只改CLI》§4.3 | `re.md`           | `$WORK/agents/`  | `--agent-file`      | **是**   |
| PWN 人设      | 《只改CLI》§4.3 | `pwn.md`          | `$WORK/agents/`  | `--agent-file`      | **是**   |
| 密码人设        | 《只改CLI》§4.3 | `crypto.md`       | `$WORK/agents/`  | `--agent-file`      | **是**   |
| 退路 prompt   | 《只改CLI》§6.1 | `ctf-prompt.txt`  | `$WORK/prompts/` | `-p "$(cat ...)"`   | **是**   |
| 难度标注        | 本文 §6.2     | **存数据库，不是文件**     | 自动               | `ctfagent tag` 交互输入 | 否       |
| 自建 agent 源码 | 项目根目录       | `ctfagent_min.py` | `$WORK/tools/`   | `python3 ...`       | **是**   |
| 环境报告        | 自检输出        | `env-report.txt`  | `$WORK/`         | 打开看                 | 建议      |

#### 第一步：建空文件占位

```bash
# 位置：$WORK
touch "$WORK"/agents/ctf.md "$WORK"/agents/re.md \
      "$WORK"/agents/pwn.md "$WORK"/agents/crypto.md \
      "$WORK"/prompts/ctf-prompt.txt \
      "$WORK"/check_env.sh "$WORK"/pack.sh
```

然后**打开文件把内容粘进去**（`nano "$WORK/agents/ctf.md"`，`Ctrl+O` 保存、`Ctrl+X` 退出）。

内容从哪来，看上面那张表的「出自」列——去对应文档的对应小节复制。**别自己改写**，那些模板是调过的。

#### 第二步：验证文件都对

```bash
# 1) 两个脚本语法对不对
for f in "$WORK"/check_env.sh "$WORK"/pack.sh; do
  bash -n "$f" && echo "OK   $f" || echo "FAIL $f"
done

# 2) 加执行权限
chmod +x "$WORK"/*.sh

# 3) agent 文件 frontmatter 对不对（第一行必须是 ---）
for f in "$WORK"/agents/*.md; do
  [ -s "$f" ] || { echo "EMPTY $f"; continue; }
  head -1 "$f" | grep -qx -- '---' && echo "OK   $(basename "$f")" || echo "BAD  $(basename "$f") (第一行不是 ---)"
done

# 4) 人设内容真的能被 agent 读到（需要 kimi 命令和模型接口就绪）
kimi -p "用一句话复述你的角色设定" --agent-file "$WORK/agents/ctf.md" --output-format text
```

第 3 步那个检查很关键：**frontmatter 第一行不是** **`---`，整个 agent 文件会被当成普通文本**，你以为换了人设，其实没生效。

#### 哪些代码块**不用**存

明确一下，省得你多建一堆没用的文件：

| 代码块                      | 为什么不用存                           |
| ------------------------ | -------------------------------- |
| §1 的问询话术                 | 那是发给赛方的文字，复制到聊天框/邮件              |
| §2 自检脚本里的 Python heredoc | 它是 `check_env.sh` 的一部分，内嵌的，不单独存  |
| §4.1 的 `curl` 验证命令       | 敲一次确认接口通就行                       |
| §5.2 的 exp 骨架 / pty 骨架   | 那是**解题现场**写给具体题目的，不是赛前准备的固定文件    |
| §6.2 的 while 循环          | 敲一次即可；想反复用就另存 `run_basic.sh`，非必需 |
| §9 的清单                   | 打印出来打勾，不是文件                      |

#### 关于 `$WORK`：两个脚本都自带兜底

`check_env.sh` 和 `pack.sh` 开头都有这一行：

```bash
WORK="${WORK:-$(cd "$(dirname "$0")" && pwd)}"
```

意思是：**没传环境变量，就取脚本自己所在的目录**。

所以最省事的用法是——把脚本放在 `$WORK` 根下，直接跑：

```bash
bash "$WORK/check_env.sh"
```

如果你把脚本挪到了别处（比如 `/opt/tools/`），推导出来的就是 `/opt/tools/`，那就必须显式传：

```bash
WORK="/workspace" bash /opt/tools/check_env.sh
```

**别在新终端里不传** **`WORK`** **就直接** **`bash check_env.sh`**——脚本在哪个目录，产物就写哪个目录，很容易写到你没注意的地方。

### 3.3 三条硬规矩

**① 不用** **`~/.kimi-code/`**

赛制说「仅可在指定工作目录写入」，家目录不在授权范围。所以：

* 不写 `~/.kimi-code/config.toml`
* 不往 `~/.kimi-code/skills/` 放东西
* 不往 `~/.kimi-code/agents/` 放东西

官方 CLI 需要配置时，用**参数**指向工作目录内：

```bash
kimi --agent-file "$WORK/agents/ctf.md" \
     --skills-dir "$WORK/skills" \
     --add-dir "$WORK/challs/web-01"
```

> 如果官方 CLI 首次启动强制往 `~/.kimi-code` 写配置，那就是"程序自身行为"，不是你主动配置。这种情况在提交说明里写一句「使用官方 CLI 默认配置，未修改」，留痕即可。

**② 不用** **`/tmp`** **存重要东西**

`/tmp` 可能在容器重启时被清，也可能不在提交范围内。中间产物放 `$WORK/.cache/`。

**③ 所有脚本用** **`$WORK`** **变量，不写死路径**

```bash
export WORK="/workspace"        # 改成实际路径
export WORK="$(cd "$(dirname "$0")" && pwd)"   # 或者脚本自己推导
```

写死了路径，换台机器就全废。

***

## 4. 模型接口适配

### 4.1 情况 A：OpenAI 兼容（最可能）

拿到 `base_url` 和 `key` 后，**先 curl 验证，再改代码**：

```bash
export MODEL_BASE_URL="http://赛方给的地址/v1"
export MODEL_API_KEY="赛方给的key"
export MODEL_NAME="赛方给的模型名"

# 1) 列模型
curl -s "$MODEL_BASE_URL/models" -H "Authorization: Bearer $MODEL_API_KEY" | head -c 500

# 2) 试一次对话
curl -s "$MODEL_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $MODEL_API_KEY" -H "Content-Type: application/json" \
  -d "{\"model\":\"$MODEL_NAME\",\"messages\":[{\"role\":\"user\",\"content\":\"回复两个字：通了\"}],\"max_tokens\":20}"
```

两条都通，改 `ctfagent_min.py` 一行：

```python
# 原来
base_url: str = "https://api.moonshot.cn/v1"
# 改成（让它读环境变量）
base_url: str = os.environ.get("MODEL_BASE_URL", "https://api.moonshot.cn/v1")
```

或者直接 `export MOONSHOT_API_KEY=$MODEL_API_KEY`，再给代码加个 `CTFAGENT_BASE_URL` 环境变量支持。

> 改代码这件事本身没问题——赛制明确允许「自建 Agent」，只要提交源码并说明。

### 4.2 情况 B：不兼容（少见但致命）

如果接口不是 OpenAI 格式（比如只提供自定义 HTTP 或 gRPC），`AsyncOpenAI` 用不了。三条路，按代价排：

| 方案       | 做法                                      | 代价         |
| -------- | --------------------------------------- | ---------- |
| 找适配层     | 问赛方有没有 OpenAI 兼容网关                      | 0（首选）      |
| 自己写薄客户端  | 只实现 `chat/completions` 的 POST，返回结构自己包一层 | 半天         |
| 改用官方 CLI | 让 CLI 走它自己的 provider 配置                 | 受 CLI 能力限制 |

第二条的最小实现思路：写一个 `CompatClient`，暴露 `.chat.completions.create()` 同名方法，内部换成目标协议的请求。**只包装 agent 用到的三个字段**（messages / tools / max_tokens），不要全量实现。

### 4.3 情况 C：接口有调用限制

有些赛制会限 RPM 或总 token。开赛前问清楚，然后：

```bash
# 并发降到 2，给每道题留余量
python3 tools/ctfagent_min.py solve --concurrency 2 --budget 0.5
```

***

## 5. 依赖装不上时的降级方案

「受控互联网出口」基本意味着 `pip install` 会被拦。分三种情况处理。

### 5.1 三层判断

```bash
# 第一层：镜像里有没有
python3 -c "import pwn" && echo "有，直接用"

# 第二层：能不能连内网源
pip install --no-input pwntools 2>&1 | tail -3

# 第三层：完全不能装，走降级
```

如果容器里给了内网 pip 源，装到工作目录而不是系统：

```bash
pip install --target "$WORK/vendor" pwntools
export PYTHONPATH="$WORK/vendor:$PYTHONPATH"
```

`--target` 的好处：所有东西落在 `$WORK` 内，符合「仅指定目录可写」，而且提交时能一起打包。

### 5.2 缺什么、怎么顶

这张表是保命的。**赛前就该背下来**，别等现场查。

| 缺的工具         | 影响                   | 降级方案                                                        |
| ------------ | -------------------- | ----------------------------------------------------------- |
| `pwntools`   | 写不了 exp，没 `checksec` | 手写 `socket` + `struct`（模板见下）                                |
| `checksec`   | 看不了保护机制              | `readelf -hW -lW -d` 手判（见下）                                 |
| `tmux`       | pwn 交互会话失效           | Python `pty` 或 `script -qc`                                 |
| `r2`         | 没有反汇编器               | `objdump -d -M intel`；有 gdb 就 `gdb -batch -ex 'disas main'` |
| `gdb`        | 没有动态调试               | `ltrace`/`strace`；或纯静态推                                     |
| `binwalk`    | 拆不了嵌套文件              | `file` + `xxd` 手工看文件头，`dd` 切                                |
| `z3`         | 解不了约束                | 手推；或暴力（范围小时）                                                |
| `gmpy2`      | 大整数慢                 | 纯 Python `int` 够用，只是慢                                       |
| `openai` SDK | agent 跑不了            | `requests` 手写 POST（20 行）                                    |

**没有 pwntools 时的最小 exp 骨架**：

```python
import socket, struct, time

def p64(x): return struct.pack("<Q", x)
def p32(x): return struct.pack("<I", x)

HOST, PORT = "127.0.0.1", 1337          # 靶场地址，别连公网
s = socket.create_connection((HOST, PORT), timeout=10)
time.sleep(0.3)
print(s.recv(4096, socket.MSG_DONTWAIT))  # 收 banner，可能没有
payload = b"A" * 40 + p64(0x4011a9)
s.send(payload + b"\n")
time.sleep(0.5)
print(s.recv(4096))
s.close()
```

**没有 checksec 时的手判**：

```bash
# PIE / NX / Canary 三件套
readelf -hW ./chall | grep Type          # DYN = PIE，EXEC = 无 PIE
readelf -lW ./chall | grep GNU_STACK     # RWE = 栈可执行（NX 关闭）
readelf -sW ./chall | grep stack_chk     # 有 __stack_chk_fail = 有 Canary
readelf -dW ./chall | grep -E 'BIND_NOW|FLAGS'  # BIND_NOW = Full RELRO
```

**没有 tmux 时的交互**：

```python
import pty, os, time, select

pid, fd = pty.fork()
if pid == 0:
    os.execvp("nc", ["nc", "127.0.0.1", "1337"])   # 子进程跑目标
time.sleep(1)
os.write(fd, b"payload\n")
time.sleep(0.5)
while select.select([fd], [], [], 0.5)[0]:
    try: print(os.read(fd, 4096).decode("utf-8", "replace"))
    except OSError: break
```

### 5.3 退化之后要记账

降级会损失能力，**要在提交说明里写清楚**：

```text
环境中未预装 pwntools，pwn 题使用原生 socket + struct 实现，
未使用 checksec，保护机制通过 readelf 手工判定。
```

这不是找借口，是让评委知道你的结果是在什么限制下拿到的。

***

## 6. 按难度递进的调度

赛制是 **7 题，初级 → 中级 → 高级递进**。这跟按题型分波（misc/web 先跑）是两套逻辑，**这里以难度为准**。

### 6.1 时间与预算分配

假设总时长 T（按 4 小时估，按实际调）：

| 阶段        | 题目       | 时间  | 目标       | 模型          |
| --------- | -------- | --- | -------- | ----------- |
| 初级（1–2 题） | 简单、路径明确  | 25% | **必须全清** | 快模型、低温      |
| 中级（3–4 题） | 要几步推理    | 45% | 争取全清     | 标准模型        |
| 高级（5–7 题） | 难、可能要长时间 | 30% | 挑有思路的搏   | thinking 模型 |

**关键判断**：初级题不许失败。它们是「保底分」，一旦卡住说明环境有问题（不是题难），立刻停下来查环境。

### 6.2 标注难度：运行时输入，不写文件

难度由你判断，工具负责问。一条命令，逐题输入：

```bash
# 位置：任意目录（数据库位置由 CTFAGENT_HOME 决定）
ctfagent tag
```

屏幕会逐题列出题目名和文件，你输入数字：

```
共 7 道题待标注。
  1 = 初级   2 = 中级   3 = 高级   s = 跳过   q = 退出

[1/7] web-01  [web]
        文件：index.php, flag.php
        难度 > 1
        → 初级

[2/7] misc-02  [misc]
        文件：hint.txt, secret.png
        难度 > 1
        → 初级
```

标完看一眼：

```bash
ctfagent status
```

```
ID                           CAT        LV   STATUS    TURNS    COST  FLAG
--------------------------------------------------------------------------------------------
web-01                       web        初    pending       0   0.000
misc-02                      misc       初    pending       0   0.000
crypto-03                    crypto     中    pending       0   0.000
re-04                        re         中    pending       0   0.000
re-06                        re         高    pending       0   0.000

共 7 题，已标难度 7 题。
```

#### 四种用法

| 命令                        | 作用          | 什么时候用     |
| ------------------------- | ----------- | --------- |
| `ctfagent tag`            | 只标**未标注**的题 | 开赛时第一次标   |
| `ctfagent tag --all`      | 全部**重标**    | 判断错了要改    |
| `ctfagent tag web-01 adv` | 直接指定单题      | 改一道，不用走全程 |
| `ctfagent tag web-01`     | 只交互标这一道     | 单题拿不准时    |

输入 `s` 跳过（之后仍会排在最后跑），`q` 退出（已标的都保存了）。

#### 难度影响两件事

**① 执行顺序**：初级 → 中级 → 高级 → 未标注。不用你排序，`solve` 自动按这个来。

**② 单题轮数上限**：

| 难度  | 轮数上限 | 理由            |
| --- | ---- | ------------- |
| 初级  | 25   | 简单题磨太多轮就是方向错了 |
| 中级  | 40   | 默认值           |
| 高级  | 60   | 难题值得多试        |
| 未标注 | 40   | 按默认           |

#### 按阶段跑

```bash
# 第一波：初级，并发 2，快出分
ctfagent solve --level basic --concurrency 2

# 第二波：中级
ctfagent solve --level mid --concurrency 2

# 第三波：高级，单并发，给足时间
ctfagent solve --level adv --concurrency 1
```

`--level` 可以逗号分隔：`--level basic,mid`。也可以和 `--only` 叠加：

```bash
ctfagent solve --level basic,mid --only crypto,web
```

#### 如果用的是官方 CLI（没有 tag 命令）

自建版才有 `tag`。官方 CLI 场景退回手写标签文件：

```bash
cd "$WORK"
cat > tasks.tsv <<'EOF'
01	basic	web-01
02	basic	misc-02
03	mid	crypto-03
EOF

# 按难度跑
while IFS=$'\t' read -r id level dir; do
  [ "$level" = "basic" ] || continue
  timeout 900 kimi -p "分析当前目录，找出 flag，输出 FLAG: <完整flag>" \
      --agent-file "$WORK/agents/ctf.md" \
      --add-dir "$WORK/challs/$dir" \
      --output-format stream-json \
      > "$WORK/out/$dir.jsonl" 2> "$WORK/out/$dir.err"
done < "$WORK/tasks.tsv"
```

> 官方 CLI 没有 `tag`，也没法按难度排序——**顺序只能靠你在脚本里控制**。这也是自建版值钱的地方之一。

#### 标错了怎么办

难度判断失误很正常，尤其是中级题做着做着发现是高级。两种处理：

```bash
# 改单题
ctfagent tag re-04 adv

# 或者不管它，直接加轮数重跑
ctfagent solve --only re --max-turns 70 --resume
```

### 6.3 什么时候放弃

三条硬止损，写在纸上贴屏幕前：

1. **单题超时**（比如 20 分钟）→ 杀掉，标记 `timeout`，挪到最后
2. **同一思路失败 3 次** → 换方向，换不了就放弃
3. **预算超单题上限** → 立刻停，钱要留给后面的题

「坚持」在 CTF 里是美德，但在**限时 + 限预算 + 7 题**的赛制里，**把时间投给下一题**往往更划算。

***

## 7. 提交物清单与打包

赛制原文要求四类：**源码、Agent 配置和提示词、依赖清单、必要日志**。

### 7.1 逐项对应

| 要求           | 从哪来                                | 怎么生成                     |
| ------------ | ---------------------------------- | ------------------------ |
| 自编/修改的**源码** | `$WORK/tools/*.py`                 | 直接复制                     |
| **Agent 配置** | `$WORK/agents/*.md`                | 直接复制                     |
| **提示词**      | agent 文件正文 + `$WORK/prompts/*.txt` | 直接复制                     |
| **依赖清单**     | pip / apt                          | `pip freeze` + `dpkg -l` |
| **必要日志**     | `$WORK/out/*.jsonl`、`*.err`        | 打包，**不删不改**              |

### 7.2 依赖清单（前三份文档完全没提，这里补上）

```bash
mkdir -p "$WORK/submit"

# Python 依赖
python3 -m pip freeze > "$WORK/submit/requirements.txt" 2>/dev/null || echo "(pip 不可用)"

# 系统包
dpkg -l 2>/dev/null | awk '/^ii/{print $2"\t"$3}' > "$WORK/submit/apt-packages.txt" || echo "(非 Debian 系)"

# Python 版本
python3 -V > "$WORK/submit/python-version.txt" 2>&1

# 手工补齐：镜像里没法用包管理器查的
cat > "$WORK/submit/env-note.md" <<'EOF'
# 环境说明
- 容器镜像：赛方提供
- 未安装任何额外系统包（受控出口）
- Python 依赖见 requirements.txt
- 缺失工具及降级方案见降级说明
EOF
```

### 7.3 日志：只归档，不清理

**赛制明文「禁止清除或篡改运行记录」。** 所以：

```bash
# ✅ 正确：只追加、只归档
mkdir -p "$WORK/submit/logs"
cp "$WORK"/out/*.jsonl "$WORK/submit/logs/" 2>/dev/null
cp "$WORK"/out/*.err   "$WORK/submit/logs/" 2>/dev/null

# ❌ 绝对不要
# rm -rf "$WORK"/out/*
# > "$WORK"/out/xxx.jsonl          (清空内容)
# sed -i 's/.../.../' "$WORK"/out/*.jsonl   (改内容)
```

> 前三份文档里出现过「重跑前先删 `.done`」。**那只删标记文件，不动日志**，且要在文件名上和日志区分开：
>
> * `.done` —— 任务标记，可删
>
> * `.jsonl` / `.err` —— 运行记录，**禁删**
>
> 如果分不清，就一律不删。日志占的空间远小于被判违规的代价。

### 7.4 一键打包

存成 `$WORK/pack.sh`，跑之前先 `chmod +x`。

```bash
#!/usr/bin/env bash
# pack.sh —— 生成提交包（不删除任何原始文件）
set -uo pipefail
WORK="${WORK:-$(cd "$(dirname "$0")" && pwd)}"   # 没传 WORK 就用脚本所在目录
STAMP=$(date +%m%d-%H%M)
PKG="$WORK/submit/ctf-submit-$STAMP"
mkdir -p "$PKG"/{src,agents,prompts,logs,meta}

# 源码（含自建 agent）
cp -r "$WORK"/tools/*.py        "$PKG/src/"    2>/dev/null || true
cp -r "$WORK"/skills            "$PKG/src/"    2>/dev/null || true

# Agent 配置与提示词
cp -r "$WORK"/agents/*.md       "$PKG/agents/" 2>/dev/null || true
cp -r "$WORK"/prompts/*         "$PKG/prompts/" 2>/dev/null || true

# 依赖与环境
python3 -m pip freeze > "$PKG/meta/requirements.txt" 2>/dev/null || true
dpkg -l 2>/dev/null | awk '/^ii/{print $2"\t"$3}' > "$PKG/meta/apt-packages.txt" || true
python3 -V > "$PKG/meta/python-version.txt" 2>&1

# 日志（复制，不移动）
cp "$WORK"/out/*.jsonl "$PKG/logs/" 2>/dev/null || true
cp "$WORK"/out/*.err   "$PKG/logs/" 2>/dev/null || true

# 环境自检报告
[ -f "$WORK/env-report.txt" ] && cp "$WORK/env-report.txt" "$PKG/meta/"

# 清单
find "$PKG" -type f | sed "s|$PKG/||" | sort > "$PKG/MANIFEST.txt"
tar czf "$PKG.tar.gz" -C "$WORK/submit" "$(basename "$PKG")"

echo "打包完成: $PKG.tar.gz"
echo "文件数: $(wc -l < "$PKG/MANIFEST.txt")"
```

**打包是复制，不是移动**。原始文件留在 `$WORK` 里，等核验完再说。

### 7.5 提交说明模板

评委看的第一份东西。写清楚，省得被追问。

```markdown
# 提交说明

## 运行方式
- 环境：赛方提供的 Docker 镜像
- 入口：`python3 tools/ctfagent_min.py solve --concurrency 2 --budget 0.5`
- 模型：`$MODEL_BASE_URL` / `$MODEL_NAME`（赛方提供）

## 自建 Agent 说明
- 类型：自建（基于 Kimi 模型接口）
- 核心逻辑：单进程 asyncio 调度 + 每题独立工作目录 + SQLite 状态
- 工具集：bash / read_file / write_file / submit_flag
- 与官方 Agent 的关系：仅调用模型接口，未使用官方 CLI 内核

## Agent 配置与提示词
- `agents/ctf.md`：系统提示词（解题纪律）
- `prompts/`：分题型补充提示

## 依赖
- 见 `meta/requirements.txt`、`meta/apt-packages.txt`
- 未额外安装系统包（受控出口）

## 环境限制与降级
- 镜像未预装 pwntools，pwn 题使用原生 socket + struct
- 无 tmux，交互使用 Python pty
- 详见 `meta/env-report.txt`

## 日志
- `logs/*.jsonl`：每题完整对话记录（未删改）
- `logs/*.err`：标准错误输出
```

***

## 8. 红线对照表

赛制第 3 条最重：「严禁通过网页、模型 API 等途径外传赛题、附件、截图、Flag、靶标地址、任务日志及可还原题意的信息，禁止将靶场暴露到公网。」

拆成人话：

| 行为                                    | 允许？    | 依据 / 说明            |
| ------------------------------------- | ------ | ------------------ |
| 把题目内容发给**获准的模型接口**                    | ✅ 允许   | 这就是解题本身，接口是白名单的    |
| 把题目内容发给**自己的 API key / 公网服务**         | ❌ 禁止   | 属于「外传」             |
| 用网页搜 `pwntools cyclic 用法`             | ✅ 允许   | 「不含赛题信息的通用技术资料」    |
| 用网页搜题目里的**特征字符串**                     | ❌ 禁止   | 可还原题意              |
| 把报错信息贴到公网 pastebin                    | ❌ 禁止   | 可能含附件信息            |
| 反弹 shell 连**自己容器内**地址                 | ✅ 允许   | 靶场内部               |
| 反弹 shell 连**自己的 VPS**                 | ❌ 禁止   | 把靶场暴露到公网           |
| 在容器里 `python3 -m http.server 0.0.0.0` | ❌ 禁止   | 同上，除非确认只在容器内       |
| 删 `out/*.jsonl`                       | ❌ 禁止   | 清除运行记录             |
| 删 `out/*.done`                        | ✅ 允许   | 那是任务标记，不是记录        |
| 改 `~/.kimi-code/config.toml`          | ⚠️ 看情况 | 不在指定工作目录，可能违规      |
| 提交源码                                  | ✅ 要求   | 且要能解释清楚            |
| 用赛方没提的第三方工具                           | ⚠️ 看情况 | 「获准的…安全测试工具」，不确定就问 |

### 三个容易踩的灰区

**① agent 自带的 web 搜索**

如果官方 CLI 有 `WebSearch` 工具，agent 可能自己去搜题目特征。这**风险很高**——你没法控制它搜什么。

在 agent 文件里明确禁掉：

```markdown
## 禁止
- 不要使用任何网页搜索工具查询题目内容、文件名、字符串、报错信息
- 只允许查询工具用法和语法，且不得包含题目中的任何字符串
```

如果 CLI 支持权限规则，直接 `deny` 掉 `WebSearch`。

**② 模型的"记忆"**

你没法控制服务端是否记录 prompt。**假定会记录**，所以：

* prompt 里不要写「把 flag 发到 xxx」
* 不要试图让模型「记住后告诉别人」
* agent 文件里不要有任何看起来像外传的指令

**③ 日志里的敏感信息**

日志必然包含题目内容（这是正常的，因为要提交）。但**别把日志同步到网盘/GitHub**。只在 `$WORK` 里，最后打包提交。

***

## 9. 赛前 60 分钟清单

打印出来，逐项打勾。

```text
□ 1. 拿到 $WORK 路径，cd 进去，pwd 确认
□ 2. 跑 check_env.sh，存档 env-report.txt
□ 3. 三个环境变量设好：MODEL_BASE_URL / MODEL_API_KEY / MODEL_NAME
□ 4. curl /models 返回 200
□ 5. curl /chat/completions 返回「通了」
□ 6. 确认工具缺失清单，决定降级方案
□ 7. 建目录：agents prompts challs out tools skills submit .cache
□ 8. 写 agents/ctf.md（人设），验证生效
□ 9. 拿一道样例题跑通端到端（这一步不能省）
□ 10. 确认日志真的写进 out/ 了
□ 11. 确认提交格式和截止时间
□ 12. 确认自建 Agent 的提交要求
□ 13. 开赛拿到题目后，立刻 ctfagent tag 标难度（逐题输入，约 1 分钟）
□ 14. 用 ctfagent status 确认难度和分类都对
□ 15. 把「三条止损」写在纸上贴屏幕前
```

第 9 步是**决定性的**。没跑通端到端就开赛，一定在比赛最紧张的时候出问题。

第 13 步别拖——**难度标错会让「初级先跑」的策略失效**，等于把高级题放到最前面烧预算。标完用 14 步扫一眼。

***

## 10. 出事怎么办

| 症状                           | 最可能的原因             | 处理                                           |
| ---------------------------- | ------------------ | -------------------------------------------- |
| 模型接口 401                     | key 错/过期/带空格       | `echo -n "$MODEL_API_KEY" \| wc -c` 看长度；重新复制 |
| 模型接口 404                     | base_url 少了 `/v1` | 加或去 `/v1` 各试一次                               |
| 模型接口 000                     | 内网 DNS 或代理         | 问赛方要代理配置；试 `curl -v` 看卡在哪                    |
| `pip install` 超时             | 出网被拦               | 放弃装包，走第 5 节降级                                |
| `python3: command not found` | 镜像极简               | 问赛方；或找 `python`、`python3.x`                  |
| 工作目录不可写                      | 挂载权限问题             | **立刻联系赛方**，这是他们的问题                           |
| 日志没写出来                       | 路径不对 / 权限          | 检查 `out/` 存在且可写；重跑一次确认                       |
| agent 反复调同一工具                | 输出被截断              | 提示它把输出重定向到文件再 grep                           |
| 跑到一半容器重启                     | 资源限制               | 看 `out/` 有没有断点；有就续跑，没有就重来                    |
| 被判"外传"风险                     | agent 用了 web 搜索    | 立刻停，检查日志，主动向赛方说明                             |

### 关于「主动说明」

如果你怀疑某次操作可能踩线（比如 agent 自己调了搜索工具），**主动向赛方说明比等被发现好**。带着日志去说明，比被审计出来强得多。

***

## 附：与前三份文档的冲突点汇总

| 前三份文档写的                      | 本赛制下的正确处理                   |
| ---------------------------- | --------------------------- |
| `~/ctf/...`                  | 改成 `$WORK/...`              |
| `pip install pwntools`       | 装不了，走 5.2 降级                |
| 改 `~/.kimi-code/config.toml` | 不改，改用参数指向 `$WORK`           |
| `rm .done` 重跑                | 只删 `.done`，**日志一个字节不动**     |
| 按题型分波（misc/web 先跑）           | 改成按难度递进（初→中→高）              |
| 手写 `tasks.tsv` 标难度           | 改用 `ctfagent tag` 交互输入，存数据库 |
| 默认 `api.moonshot.cn`         | 改 `$MODEL_BASE_URL`         |
| 日志可留可弃                       | **必须提交，禁止删除**               |
| 未提依赖清单                       | 必须 `pip freeze` + `dpkg -l` |

**一句话**：技术方案不用推翻，把路径换到 `$WORK`、把接口换成赛方的、把日志当证据留着，就能跑。
