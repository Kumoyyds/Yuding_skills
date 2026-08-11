# plan.md — `sync_agent_docs.py`

## 0. 背景与定位

同一个仓库同时被 Claude Code 与 Codex 开发时，两者各自读取 `CLAUDE.md` 和 `AGENTS.md`。
两个文件内容漂移会导致两个 agent 拿到不同的项目约定，产出不一致的代码。

**交付物是一个自安装的单文件 Python 脚本。** 装一次之后由 git pre-commit hook 驱动，
不再需要任何 AI 参与。跨仓库复用 = 把这个 `.py` 复制过去，跑一次 `--init`。

### 设计约束

- **单文件、零第三方依赖**（仅标准库）—— hook 要在任何协作者的环境里跑通
- **Python 版本下限 3.9**
- **脚本必须进版本库** —— pre-commit hook 在同事机器上执行时要能找到它
- **确定性优先** —— 冲突绝不猜测、绝不合并，直接阻断 commit 交还给人

---

## 1. 安装后的仓库变化

```
<repo>/
├── scripts/sync_agent_docs.py        # 主脚本（进版本库）
├── .githooks/pre-commit              # hook（进版本库）
├── .agents-sync.json                 # state（进版本库，见 §4）
├── CLAUDE.md                         # 追加规则段落
└── AGENTS.md                         # 追加规则段落（或新建）
```

外加一条 git 配置：`git config core.hooksPath .githooks`

> 用 `core.hooksPath` 而非直接写 `.git/hooks/`，是为了让 hook 本身能进版本库。
> 代价是新 clone 的协作者仍需手动执行一次这条 config —— 在规则段落里写明。

---

## 2. 核心同步算法

### 2.1 术语

- **pair**：某个目录下的 `CLAUDE.md` + `AGENTS.md` 组合，两者至少存在其一
- **shared 区**：参与同步的内容片段（见 §3）
- **`h(x)`**：对文件 x 的 shared 区做归一化后取 sha256

### 2.2 归一化规则（必须先做，否则 CRLF 会造成永久假冲突）

1. 换行统一为 `\n`
2. 去除每行行尾空白
3. 去除文件首尾的空行
4. 以 UTF-8 解码，遇到 BOM 剥离

归一化只用于计算 hash 与比较，**写盘时保持源文件原始内容**（换行统一为 `\n`）。

### 2.3 决策表

对每个 pair，取 `c = h(CLAUDE.md)`、`a = h(AGENTS.md)`、`s = state[dir]`（上次同步记录的 hash）：

| 条件 | 动作 | 退出码影响 |
|---|---|---|
| 两侧均不存在 | skip | — |
| 仅 `CLAUDE.md` 存在 | 创建 `AGENTS.md`（复制完整文件），写 state | — |
| 仅 `AGENTS.md` 存在 | 创建 `CLAUDE.md`（复制完整文件），写 state | — |
| `c == a` | no-op，刷新 state = c | — |
| `s` 不存在（首次接入） | **冲突**，除非传了 `--prefer` | 1 |
| `c == s` 且 `a != s` | AGENTS 被改过 → 写入 CLAUDE 的 shared 区 | — |
| `a == s` 且 `c != s` | CLAUDE 被改过 → 写入 AGENTS 的 shared 区 | — |
| `c != s` 且 `a != s` | **冲突**，打印两侧 diff，不写任何文件 | 1 |

冲突时提示用户二选一：手工合并后重跑，或 `--apply --prefer claude|agents` 强制单侧覆盖。

### 2.4 为什么 state 是必需的

commit 完成后仓库状态恒为 `c == a == s`，state 看起来冗余。但在"工作区有未提交改动"的窗口期，
只有 state 能区分**哪一侧被改动了**。没有 state 就无法判断方向，只能猜。

---

## 3. shared 区规格

### 3.1 标记语法

```markdown
<!-- sync:shared:start -->
这里的内容双向同步
<!-- sync:shared:end -->

## Claude Code 专属（不同步）
@docs/design/pricing.md
```

标记区的实际用途：Claude Code 支持 `@path/to/file` import 语法而 Codex 不认识，
反过来 Codex 的部分配置约定 Claude Code 也用不上。这类内容必须能各自保留。

### 3.2 提取规则

- 文件含**恰好一对**配对标记 → shared 区 = 标记之间的内容（不含标记行本身）
- 文件**不含**任何标记 → shared 区 = 整个文件（存量项目零改造接入）
- 标记数量不为 0 或 2、或 start/end 顺序颠倒 → 报错退出，退出码 2

### 3.3 写入规则

- 目标文件**存在且有标记** → 只替换标记之间的内容，标记外原样保留
- 目标文件**存在且无标记** → 整个文件被覆盖
- 目标文件**不存在** → 复制源文件的**完整内容**（包含标记，若源有标记），
  使新文件从一开始就具备加专属内容的能力

### 3.4 一侧有标记、一侧没有

合法且需支持。有标记侧走标记路径，无标记侧走全量路径。
这个组合在"存量项目刚开始引入标记"时必然出现。

---

## 4. state 文件

路径：仓库根目录 `.agents-sync.json`，**进版本库**。

```json
{
  "version": 1,
  "pairs": {
    ".":            { "hash": "sha256:ab12…", "synced_at": "2026-08-10T09:30:00Z" },
    "packages/api": { "hash": "sha256:cd34…", "synced_at": "2026-08-10T09:30:00Z" }
  }
}
```

- key 为相对仓库根的 POSIX 风格目录路径，根目录用 `.`
- 目录 key 排序输出、缩进 2 空格，减少 merge 噪音
- 文件损坏或版本号不认识 → 视作 state 全部缺失，走首次接入路径（要求 `--prefer`）

**进版本库的取舍**：不进版本库的话每个协作者首次都要手动做基线。
代价是它偶尔会 merge 冲突 —— 但因为 commit 后两侧内容恒定相同，
删掉此文件重跑一次同步即可重建。这条写进规则段落。

---

## 5. CLI 接口

```bash
python scripts/sync_agent_docs.py                          # dry-run，打印计划，永不写盘
python scripts/sync_agent_docs.py --apply                  # 执行同步并 git add 变更文件
python scripts/sync_agent_docs.py --check                  # 只检测偏差，不写盘（预留 CI 用）
python scripts/sync_agent_docs.py --apply --prefer claude  # 强制以 CLAUDE.md 为准解冲突
python scripts/sync_agent_docs.py --apply --prefer agents
python scripts/sync_agent_docs.py --init                   # 自安装，见 §6
```

**默认 dry-run，`--apply` 才写盘。**

### 退出码

| 码 | 含义 |
|---|---|
| 0 | 成功；或 dry-run 下无需变更 |
| 1 | 存在冲突 / `--check` 检出偏差 → pre-commit 阻断 |
| 2 | 结构错误（标记不配对）、参数错误、非 git 仓库 |

### pair 发现

```bash
git ls-files -c -o --exclude-standard -- '*CLAUDE.md' '*AGENTS.md'
```

- 用 `git ls-files` 而非 `os.walk`：天然继承 `.gitignore`，不会扫进 `node_modules/`、`.venv/`
- `-c -o --exclude-standard` 同时覆盖已跟踪文件和新建未 add 的文件
- 按 glob 结果的父目录聚合成 pair

---

## 6. `--init` 子命令（自安装）

脚本自带安装能力，把自己接入仓库。步骤：

1. 确认在 git 仓库内（`git rev-parse --show-toplevel`），否则退出码 2
2. 若脚本自身不在 `<root>/scripts/sync_agent_docs.py`，复制自己到该路径
   - 目标已存在且内容不同 → 提示后覆盖；内容相同 → 跳过
3. 写入 `.githooks/pre-commit`（内容见 §7），`chmod +x`
4. 执行 `git config core.hooksPath .githooks`
5. 扫描现有 pair，打印报告：发现 N 个 pair，其中 M 个两侧不一致
6. **不自动做基线同步** —— 打印下一步命令让用户自己决定：
   ```
   发现 2 个 pair 两侧内容不一致，请先决定以哪侧为准：
     python scripts/sync_agent_docs.py                          # 查看差异
     python scripts/sync_agent_docs.py --apply --prefer claude   # 以 CLAUDE.md 为准
   ```
7. 向根目录 `CLAUDE.md` 与 `AGENTS.md` 追加规则段落（§8）
   - 段落以固定标题 `## Agent 文档同步` 开头，已存在则跳过 —— 保证 `--init` 可重复执行

`--init` 是唯一会修改 git config 的路径，其余子命令绝不碰配置。

---

## 7. pre-commit hook

`.githooks/pre-commit`：

```sh
#!/bin/sh
set -e
python scripts/sync_agent_docs.py --apply
```

- 脚本在 `--apply` 且发生实际写入后，自行对变更文件与 `.agents-sync.json` 执行 `git add`
- 退出码非 0 时因 `set -e` 中断，commit 被阻断
- Python 解释器解析顺序：`python3` → `python`，都找不到则打印明确提示并退出码 2

---

## 8. 规则段落内容（注入 CLAUDE.md / AGENTS.md）

必须包含以下要点：

1. 这两个文件由 `scripts/sync_agent_docs.py` 在 commit 时自动双向同步
2. 改任意一侧都可以，但**不要在同一次 commit 前同时改两侧** —— 会触发冲突阻断
3. 工具专属内容放在 `<!-- sync:shared:end -->` 之后
4. 新 clone 仓库后需执行一次 `git config core.hooksPath .githooks`
5. `.agents-sync.json` merge 冲突时，删除该文件重跑同步即可

### 已知限制（同样写进段落）

- **部分暂存**：hook 基于工作区文件同步。若只 stage 了 `CLAUDE.md` 的一部分改动，
  同步会把工作区完整内容带入本次 commit。这是设计取舍，不修复。
- **`core.hooksPath` 需手动启用**：未执行的协作者机器上 hook 静默不生效。
- **不做内容合并**：冲突永远交还给人处理。

---

## 9. 边界情况清单（逐条覆盖）

| # | 场景 | 期望行为 |
|---|---|---|
| 1 | 仅换行符不同（CRLF vs LF） | 归一化后 hash 相等 → no-op |
| 2 | 某侧是指向另一侧的 symlink | 检测到解析后同一文件 → skip 该 pair |
| 3 | 某侧为空文件（0 字节） | 视作正常内容参与决策，不特殊处理 |
| 4 | 标记只有 start 没有 end | 退出码 2，指明文件路径与行号 |
| 5 | 文件名大小写（Windows / macOS 不区分大小写） | 匹配到 `claude.md` 且已有 `CLAUDE.md` → 警告并 skip，不尝试改名 |
| 6 | 仓库无任何 pair | 退出码 0，打印"未发现需同步的文件" |
| 7 | 非 UTF-8 编码文件 | 退出码 2，明确报告编码问题 |
| 8 | `.agents-sync.json` 是损坏 JSON | 视作 state 缺失，走首次接入路径 |
| 9 | 子目录 pair 与根目录 pair 内容不同 | 正常 —— 每个 pair 独立判定，互不影响 |
| 10 | dry-run 模式 | 绝不产生任何文件写入或 `git add` |

---

## 10. 里程碑

### M1 — 同步引擎
- 归一化、shared 区解析、决策表、state 读写
- 不含 git 集成，用显式路径参数驱动，可独立测试
- 覆盖 §9 全部 10 条边界情况

### M2 — git 集成
- `git ls-files` 发现 pair
- `--apply` 后自动 `git add`
- `--check`

### M3 — 自安装
- `--init`：复制自身、写 hook、设 config、注入规则段落
- 在一个真实仓库上端到端验证

**M1 完成前不要动 M2。** 引擎的正确性是全部价值所在，git 集成只是触发外壳。

---

## 11. 验收测试用例

在临时 git 仓库中构造，每条独立：

1. 仅有根 `AGENTS.md` → `--apply` 后生成内容一致的 `CLAUDE.md`，state 写入
2. 两侧一致 → dry-run 报告 no-op，退出码 0，文件未被触碰
3. 基线后仅改 `CLAUDE.md` → 同步方向为 CLAUDE → AGENTS
4. 基线后仅改 `AGENTS.md` → 同步方向为 AGENTS → CLAUDE
5. 基线后两侧都改且内容不同 → 退出码 1，两个文件均未被修改
6. 无 state 且两侧内容不同 → 退出码 1；加 `--prefer claude` 后 AGENTS 被覆盖
7. `CLAUDE.md` 含标记区、`AGENTS.md` 不含 → 同步后 CLAUDE 标记外的专属内容完好无损
8. 三级嵌套目录各有一个 pair → 三个 pair 独立同步，state 有三个 key
9. `node_modules/CLAUDE.md`（被 gitignore）→ 不出现在扫描结果中
10. 挂上 hook 后执行 `git commit` → 同步文件被自动带入本次 commit
11. 挂上 hook 且存在冲突 → commit 被阻断，工作区保持原样
12. 连续执行两次 `--init` → 第二次不重复注入规则段落，git config 不出错