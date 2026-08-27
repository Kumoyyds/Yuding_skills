# 实现模板

契约填好后按这里的模式生成代码。全部按需查阅，不必通读。

- [1. 公开出口](#1-公开出口)
- [2. Config](#2-config)
- [3. 输入输出模型](#3-输入输出模型)
- [4. 错误](#4-错误)
- [5. 依赖注入](#5-依赖注入)
- [6. api.py 与并发](#6-apipy-与并发)
- [7. 入口层](#7-入口层)
- [8. pyproject.toml](#8-pyprojecttoml)

---

## 1. 公开出口

`__init__.py` 只做转发，不含逻辑、不含 import 时副作用。

```python
from my_module.api import run, run_batch
from my_module.config import Config
from my_module.models import PriceRecord, Result
from my_module.errors import MyModuleError, ConfigError, FetchError

__all__ = [
    "run", "run_batch",
    "Config", "PriceRecord", "Result",
    "MyModuleError", "ConfigError", "FetchError",
]
__version__ = "0.1.0"
```

内部包用 `_core/` 这样的下划线前缀，明示"进去出事自负"。这样以后重命名内部文件不构成破坏性变更。

---

## 2. Config

```python
from dataclasses import dataclass, replace
import os

@dataclass(frozen=True, slots=True)
class Config:
    timeout: float = 30.0
    min_price: float = 0.01
    max_workers: int = 8

    def __post_init__(self) -> None:
        if self.timeout <= 0:
            raise ConfigError(f"timeout must be positive, got {self.timeout}")
        if not 1 <= self.max_workers <= 32:
            raise ConfigError(f"max_workers must be 1..32, got {self.max_workers}")

    @classmethod
    def from_env(cls, prefix: str = "MYMODULE_") -> "Config":
        """从环境变量构造。库自身绝不自动调用——由调用方显式决定。"""
        kw = {}
        if v := os.getenv(f"{prefix}TIMEOUT"):
            kw["timeout"] = float(v)
        return cls(**kw)

    def with_(self, **changes) -> "Config":
        return replace(self, **changes)
```

`frozen=True` 让配置不可变——避免调用方在并发中途改配置导致的诡异行为。校验放 `__post_init__`，坏配置在构造时就炸，而不是跑到一半才炸。

---

## 3. 输入输出模型

返回 dataclass 而不是 dict。dict 的字段名是隐性契约，改了没人知道；dataclass 改字段 IDE 和类型检查会立刻报警。

```python
from dataclasses import dataclass
from typing import Generic, TypeVar

@dataclass(frozen=True, slots=True)
class PriceRecord:
    url: str
    price: float
    currency: str
    fetched_at: str

T = TypeVar("T")

@dataclass(frozen=True, slots=True)
class Result(Generic[T]):
    """批量处理的单条结果，同时承载成功与失败。"""
    index: int
    value: T | None = None
    error: Exception | None = None

    @property
    def ok(self) -> bool:
        return self.error is None
```

调用方用起来：

```python
results = run_batch(urls, config=cfg)
good = [r.value for r in results if r.ok]
bad  = [(r.index, r.error) for r in results if not r.ok]
```

### 契约里 style 是 pydantic 时

载荷类型换成 `BaseModel`，**但 `Result` 保持 dataclass**：

```python
from pydantic import BaseModel, ConfigDict, Field

class PriceRecord(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    url: str
    price: float = Field(gt=0)
    currency: str
    fetched_at: datetime
```

`Result` 不改成 pydantic 的原因：`Exception` 不是 pydantic 的合法字段类型，塞进去要开 `arbitrary_types_allowed=True`，而且没法序列化。dataclass 外壳 + pydantic 载荷，两边好处都拿到。

要把整批结果转成 JSON 时显式处理，不要指望 `Result` 自己会：

```python
payload = [r.value.model_dump() for r in results if r.ok]
```

三个配套注意事项：

- **版本必须钉死**：`dependencies = ["pydantic>=2,<3"]`。公开签名里出现 pydantic，它就不再是可选依赖，不能放 optional-dependencies
- **批量时的校验开销**：`_core` 内部构造已经可信的数据时用 `PriceRecord.model_construct(...)` 跳过校验。校验只在边界做一次——从外部拿到数据的那一刻
- **进程池**：pydantic v2 模型可以正常 pickle，`ProcessPoolExecutor` 不受影响

### 契约里 style 是 mixed 时

输入用 pydantic（校验外部不可信数据），输出用 dataclass（自己造的数据不需要再校验）。`_core` 只接触 dataclass，pydantic 停在 `api.py` 边界上：

```python
def run(raw: dict, *, config=None) -> PriceRecord:   # PriceRecord 是 dataclass
    req = ScrapeRequest.model_validate(raw)          # pydantic 只在这一行出现
    return _core.process(req.url, min_price=config.min_price)
```

---

## 4. 错误

```python
class MyModuleError(Exception):
    """所有本模块异常的基类。调用方可用它一网打尽。"""

class ConfigError(MyModuleError): ...
class FetchError(MyModuleError): ...
class ParseError(MyModuleError): ...
```

第三方异常在 `api.py` 边界转换，不允许漏出去：

```python
try:
    html = fetcher.get(url, timeout=config.timeout)
except Exception as e:
    raise FetchError(f"failed to fetch {url}") from e
```

`from e` 保留原始堆栈，调试时不丢信息。调用方却只需要认识 `FetchError`——将来你把 httpx 换成 aiohttp，他一行都不用改。

---

## 5. 依赖注入

`protocols.py` 只定义形状，不含实现：

```python
from typing import Protocol

class Fetcher(Protocol):
    def get(self, url: str, *, timeout: float) -> str: ...

class Store(Protocol):
    def save(self, record: PriceRecord) -> None: ...

class Clock(Protocol):
    def now(self) -> str: ...
```

`Protocol` 是结构化类型：调用方的类不需要继承，只要方法签名对得上就算通过。这比强制继承友好得多。

`adapters/` 里放默认实现，第三方 import 放在文件内部而不是包顶层：

```python
# adapters/httpx_fetcher.py
try:
    import httpx
except ImportError as e:
    raise ImportError(
        "HttpxFetcher requires httpx. Install with: pip install my-module[httpx]"
    ) from e

class HttpxFetcher:
    def __init__(self, client: "httpx.Client | None" = None):
        self._client = client or httpx.Client()

    def get(self, url: str, *, timeout: float) -> str:
        return self._client.get(url, timeout=timeout).text
```

测试时用 fake，不 mock：

```python
class FakeFetcher:
    def __init__(self, pages: dict[str, str]):
        self._pages = pages
    def get(self, url: str, *, timeout: float) -> str:
        return self._pages[url]
```

---

## 6. api.py 与并发

`run()` 永远是同步单条，不含任何并发代码。`run_batch()` 是它之上的薄薄一层。

```python
import logging
from concurrent.futures import ThreadPoolExecutor
from collections.abc import Sequence

logger = logging.getLogger(__name__)   # 只取 logger，绝不配置 handler

def run(
    url: str,
    *,
    config: Config | None = None,
    fetcher: Fetcher | None = None,
) -> PriceRecord:
    """处理单个 URL。这是整个模块的语义中心。"""
    config = config or Config()
    fetcher = fetcher or _default_fetcher()

    try:
        html = fetcher.get(url, timeout=config.timeout)
    except Exception as e:
        raise FetchError(f"failed to fetch {url}") from e

    return _core.parse(html, url=url, min_price=config.min_price)


def run_batch(
    urls: Sequence[str],
    *,
    config: Config | None = None,
    fetcher: Fetcher | None = None,
    max_workers: int | None = None,
) -> list[Result[PriceRecord]]:
    """批量处理，保序返回。单条失败不影响其余条目。"""
    config = config or Config()
    workers = max_workers or config.max_workers

    if workers == 1:                       # 串行路径，调试用
        return [_run_one(i, u, config, fetcher) for i, u in enumerate(urls)]

    with ThreadPoolExecutor(max_workers=workers) as ex:
        return list(ex.map(
            lambda p: _run_one(p[0], p[1], config, fetcher),
            enumerate(urls),
        ))


def _run_one(index, url, config, fetcher) -> Result[PriceRecord]:
    try:
        return Result(index=index, value=run(url, config=config, fetcher=fetcher))
    except MyModuleError as e:
        logger.warning("item %d failed: %s", index, e)
        return Result(index=index, error=e)
```

要点：

- `ex.map` 天然保序，不需要额外排序
- `_run_one` 吞掉 `MyModuleError` 转成 `Result`，但**不吞** `KeyboardInterrupt` 之类的 `BaseException`——那些应该让它中断
- `max_workers=1` 走独立的串行路径，堆栈干净，断点好打
- 注入的 `fetcher` 必须线程安全。`httpx.Client` 是安全的；如果适配器不安全，改成每个 worker 各建一个

**换成进程池**（CPU 密集）时：`ThreadPoolExecutor` → `ProcessPoolExecutor`，但注入的依赖和传入数据都必须可 pickle，lambda 要换成模块级函数。

**需要限速**时，在 `_run_one` 里加一个共享的令牌桶，别在外层用 `sleep`——那会把并发白白抵消掉。

---

## 7. 入口层

入口层只做翻译，零业务逻辑。

```python
# entrypoints/server.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from my_module import run, Config, MyModuleError   # 只从公开出口 import

app = FastAPI()
_config = Config.from_env()          # 在这里读 env，而不是在库里

class ScrapeRequest(BaseModel):
    url: str

@app.post("/v1/scrape")
def scrape(req: ScrapeRequest):
    try:
        record = run(req.url, config=_config)
    except MyModuleError as e:
        raise HTTPException(status_code=422, detail=str(e)) from e
    return record
```

注意 server 从 `my_module` 顶层 import，跟外部用户走同一个门。如果 server 需要摸内部才能实现某个功能，说明公开面漏了东西——去补 `api.py`，而不是让 server 抄近路。

---

## 8. pyproject.toml

```toml
[project]
name = "my-module"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = []                      # 核心零依赖是理想状态

[project.optional-dependencies]
httpx = ["httpx>=0.27"]
server = ["fastapi>=0.110", "uvicorn>=0.27"]
dev = ["pytest>=8", "pytest-cov"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_module"]
```

别忘了 `src/my_module/py.typed`（空文件），有它下游才能拿到你的类型提示。

调用方安装方式，按阶段：

```bash
uv pip install -e ../my-module                                   # 本地开发
uv add "my-module @ git+https://github.com/you/my-module@v0.1.0" # 从 Git
uv add my-module                                                 # 已发布
```
