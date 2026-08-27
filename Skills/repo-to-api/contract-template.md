# API 契约表

跟用户一起填满这张表，然后写进 `docs/api-contract.md`。

未确定的项标 `TODO`，不要自己猜完就往下走。可以带推荐值去问，但要让用户明确点头。

```yaml
# ── 身份 ────────────────────────────────
package: my_module          # import 时的名字
purpose: 一句话说清它干什么   # 说不清 = 职责不单一，先拆

# ── 输入 ────────────────────────────────
input:
  unit: 一个商品 URL          # 单次调用的最小语义单元
  type: str                  # 或 dataclass 名
  batch: true                # 是否需要 run_batch()
  ordering: preserve         # preserve | any  批量返回是否保序

# ── 模型风格 ────────────────────────────
models:
  style: dataclass           # dataclass | pydantic | mixed
  # dataclass = 公开类型零依赖，适合发布给不特定用户
  # pydantic  = 自带校验与序列化，适合已用 pydantic 或要做 HTTP 入口
  # mixed     = 输入 pydantic（校验不可信数据），输出 dataclass
  pydantic_pin: ">=2,<3"     # 仅 pydantic/mixed。公开签名里出现 pydantic
                             # 就等于把版本也写进了契约，必须显式钉住大版本

# ── 输出 ────────────────────────────────
output:
  type: PriceRecord          # 别返回裸 dict——字段名是隐性契约，改了没人知道
  batch_type: list[Result]   # Result 恒为 dataclass，即使 T 是 pydantic
  on_item_failure: collect   # raise | skip | collect
  # raise   = 一条炸全批炸（适合事务性场景）
  # skip    = 静默丢弃（几乎不该选，会掩盖问题）
  # collect = 收进结果里，调用方自己决定（默认推荐）

# ── 配置与阈值 ──────────────────────────
config:
  - name: timeout
    type: float
    default: 30.0
    note: 单次请求超时秒数
  - name: min_price
    type: float
    default: 0.01
    note: 低于此值视为解析失败
  from_env: false            # 提供 Config.from_env() 但绝不自动调用

# ── 并发 ────────────────────────────────
parallel:
  enabled: true
  kind: thread               # thread | process | async
  # thread  = IO 密集（网络、磁盘）——绝大多数情况选这个
  # process = CPU 密集（大量计算、图像处理）
  # async   = 已有 async 生态，或并发量 > 数百
  max_workers: 8             # 默认值，调用方可覆盖
  max_workers_ceiling: 32    # 上限，防止调用方传个 1000 打爆下游
  rate_limit: null           # 每秒上限，null 表示不限
  can_disable: true          # max_workers=1 必须退化成串行，用于调试

# ── 副作用（每一项都要抽成注入依赖）────────
side_effects:
  - kind: http
    protocol: Fetcher        # protocols.py 里的接口名
    default_adapter: HttpxFetcher
    optional_dep: httpx      # 进 optional-dependencies
  - kind: storage
    protocol: Store
    default_adapter: null    # null = 不提供默认实现，必须调用方传
  - kind: clock              # 用到 now() 就列在这，否则测试无法固定时间
    protocol: Clock
    default_adapter: SystemClock

# ── 入口 ────────────────────────────────
entrypoints:
  python: true               # 必选。from my_module import run, run_batch
  cli: false                 # entrypoints/cli.py
  http: false                # entrypoints/server.py (FastAPI)
  http_detail:               # 仅当 http: true
    framework: fastapi
    routes:
      - POST /v1/scrape      # 单条
      - POST /v1/scrape/batch
    auth: none               # none | api_key | bearer
    async_job: false         # 单次耗时 > 30s 时改 true，走任务队列而非同步返回

# ── 公开面 ──────────────────────────────
public_api:                  # __all__ 的内容，越短越好
  - run
  - run_batch
  - Config
  - PriceRecord
  - MyModuleError
```

## 填表时的判断依据

**unit 怎么定**：问"两条输入之间有依赖吗？"没有 → 单位是单条，批量交给 `run_batch()`。有依赖（比如必须按顺序累积状态）→ 单位就是整个序列，此时通常也不该并发。

**on_item_failure 选 collect 的理由**：`raise` 会让 999 条成功的结果因为第 1000 条失败而全部丢掉；`skip` 让调用方永远不知道丢了什么。`collect` 把决定权交回给调用方，代价只是返回类型稍复杂。

**config 该不该收某一项**：问"调用方会想改它吗"。会 → 收进 config。不会 → 留在代码里当常量。config 里的每一项都是一份长期承诺，宁少勿多。

**style 选 dataclass 还是 pydantic**：真正的问题不是"哪个好用"，而是"pydantic 要不要出现在公开签名里"。出现了，调用方就必须装 pydantic 且大版本兼容——v1/v2 的不兼容至今仍在坑人。已经要做 FastAPI 入口的，pydantic 本来就是依赖，直接全用；打算发布给不特定用户的库，公开边界用 dataclass，pydantic 关在 `_core` 内部。判断不了就选 mixed。

**kind 选 thread 还是 process**：看逻辑在等什么。等网络/磁盘 → thread（GIL 在等待时会释放）。等 CPU 算完 → process。判断不了就跑一次 profile，别猜。

**要不要 http**：只有当调用方跨语言、跨机器、或需要独立扩缩容时才需要。同一个 Python 项目内部调用，直接 import 更快更简单。HTTP 加进来就要处理序列化、超时、重试、部署，成本不低。
