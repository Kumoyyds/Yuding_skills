---
name: repo-to-api
description: 把一个现有的脚本型 repo 封装成干净、可被外部调用的 API（Python 函数入口 + 可选 CLI / HTTP），围绕一份显式契约展开：输入单位、输出结构、config 与阈值、并发方式与并发度、副作用依赖注入。Use this skill whenever the user wants to wrap, package, expose, productize, or "make callable" an existing repo, script, notebook, or pipeline — including phrasing like 封装成 API、做成一个库、暴露一个接口、让别人能调用、加个 batch 接口、加并行、抽出 config. Also use it when the user is designing the public interface of a new module and needs to pin down input/output/config/concurrency before writing code. Trigger even if they don't say the word "API".
---

# Repo → API

把"能跑的脚本"变成"能被别人调用的 API"。

核心思路：先把**契约**（谁调用它、传什么、拿到什么、可调什么、怎么并发）用一张表写死，再按契约生成结构。跳过契约直接改代码，是这类重构失败的主要原因——写到一半才发现输入单位定错了，整层都得推倒。

## 什么时候用

- 已有 repo 里有能跑的逻辑，但入口是 `if __name__ == "__main__"`、notebook、或一堆散落的脚本
- 想让另一个项目 / 队友 / 服务能调用它
- 需要加批量处理、并发、可调参数，但不想把这些揉进业务逻辑里

## 工作流

五步，**每步结束后向用户确认再继续**。这不是走过场：契约一旦定错，后面全是返工。

### 第 1 步 — 摸底

读 repo，搞清楚三件事，然后向用户汇报：

1. **真正的业务逻辑在哪**（哪几个函数是核心，哪些是胶水）
2. **所有副作用点**：网络请求、文件读写、数据库、时间戳、随机数、环境变量、`print`。逐个列出文件:行号
3. **硬编码的常量**：路径、URL、超时、阈值、批大小。这些是 config 的候选项

汇报格式：

```
核心逻辑：src/scraper.py:45-120 (parse_listing)
副作用：
  - scraper.py:31  requests.get        → 需要注入
  - scraper.py:128 sqlite3.connect     → 需要注入
  - scraper.py:12  os.environ["KEY"]   → 移入 Config
硬编码常量：
  - TIMEOUT=30, MIN_PRICE=0.01, BATCH=50
```

### 第 2 步 — 填契约

读 `references/contract-template.md`，跟用户一起把它填满。**这是整个 skill 的核心产物**，填完写进 `docs/api-contract.md` 并给用户过目。

填的时候注意这几个最容易定错的地方：

- **输入单位（unit）**：单次调用处理的最小语义单元。选错了后面全乱。判断方法：如果 A 和 B 之间没有依赖、可以分别处理，那单位就是单个而不是列表。
- **部分失败策略**：批量处理时第 3 条炸了，剩下的怎么办？必须现在就定，不能等实现时再拍脑袋。
- **config 里放什么**：只放"调用方合理地会想改"的东西。你自己都不会改的常量留在代码里就好，放进 config 只会增加公开面。
- **模型风格（dataclass / pydantic / mixed）**：如果 repo 里已经有 pydantic 模型，别默认沿用就完事——要先确认它是否该出现在**公开签名**里。出现了，pydantic 的大版本就成了契约的一部分，调用方被迫跟你保持兼容。

用户没明说的项不要自己默默填，列出来问。但可以带上你的推荐默认值，别让用户从零回答。

### 第 3 步 — 生成骨架

按契约生成结构。默认布局（细节和完整代码模板见 `references/patterns.md`）：

```
src/<pkg>/
├── __init__.py      # 唯一公开出口，显式 __all__，只做转发
├── api.py           # run() / run_batch()  ← 对外承诺就这两个
├── models.py        # 输入输出的 dataclass
├── config.py        # frozen dataclass，含 validate()
├── errors.py        # 异常基类 + 子类
├── protocols.py     # 副作用的接口定义
├── _core/           # 纯逻辑，下划线开头表示内部
└── adapters/        # 副作用的默认实现，可被替换
entrypoints/         # 可选，按契约决定生成哪些
├── cli.py
└── server.py
```

几条不要偏离的规则：

- `_core/` 里不 import 任何做 IO 的库。纯计算库（re、numpy、lxml、dataclasses）随便用
- `__init__.py` 里除了 import 转发和 `__all__` 什么都不写，尤其不能有建连接、读环境变量这类 import 时副作用
- 并发只存在于 `api.py` 的 `run_batch()` 一层。`_core` 和 `run()` 永远是同步单条的——这样调试、测试、复用都简单
- HTTP server 只是入口层，不允许包含任何业务逻辑，它只做「解析请求 → 调 `run()` → 序列化返回」

### 第 4 步 — 接线

把原有逻辑搬进 `_core/`，同时：

- 每个副作用点改成从参数传入的依赖（对照第 1 步的清单逐条销账）
- 每个硬编码常量改成读 `config.xxx`
- 第三方异常在 `api.py` 边界处捕获并转成自己的异常类型，不允许 `requests.Timeout` 漏给调用方
- 所有 `print` 改成 `logger = logging.getLogger(__name__)`，且**不配置 handler**

搬完跑一次原来的用例，确认行为没变。这一步只做搬迁，不做优化——同时改两件事，出问题时无法定位。

### 第 5 步 — 验收

生成 `docs/api-contract.md` 的最终版 + README 用法示例，然后逐条过这个清单，把结果告诉用户：

- [ ] `run()` 单条能跑，且完全不依赖并发代码
- [ ] `_core/` 的测试不联网、不碰磁盘、不 mock 第三方库就能跑通
- [ ] `max_workers=1` 时 `run_batch()` 退化成串行，结果与逐条调用 `run()` 一致
- [ ] `import <pkg>` 之后，宿主项目的 logging / 全局状态无变化
- [ ] 删掉 `adapters/` 全部内容，`_core/` 仍然能被测试
- [ ] 契约表里的每一项，在代码里都能指出对应位置

有没过的项直接说，不要粉饰。

## 输出格式

每步结束时给用户一个简短汇报：这步做了什么、产生了哪些文件、下一步是什么、有哪些地方需要他拍板。不要一口气把五步全做完再汇报——中间的确认点是这个流程的价值所在。

## 参考文件

- `references/contract-template.md` — 契约表模板。第 2 步必读。
- `references/patterns.md` — config / 并发 / 错误处理 / 依赖注入的代码模板。第 3、4 步按需查阅。
