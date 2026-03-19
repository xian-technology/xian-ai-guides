# XIAN NETWORK — SMART CONTRACT DEVELOPMENT GUIDE

This guide is for writing **valid Xian contracts against the current codebase**.
It reflects the actual `xian-contracting` linter/runtime behavior, plus the
current `xian-abci` time semantics.

Use it as a **hard rules + safe defaults** guide for humans and LLMs.

## 1. Execution Model

- Xian contracts use a **restricted Python subset**, not unrestricted Python.
- Contracts are parsed by the node's pinned CPython runtime.
  Current repos target **Python 3.12+** and are being validated on **3.14**.
- Do **not** rely on newest Python features just because the host Python
  supports them. Write only the Xian subset described below.
- Contract execution must stay **deterministic** across validators.

## 2. Core Language Rules

### Allowed syntax

- assignments
- arithmetic: `+`, `-`, `*`, `/`, `//`, `%`, `**`
- comparisons: `==`, `!=`, `<`, `<=`, `>`, `>=`, `in`, `not in`, `is`, `is not`
- boolean logic: `and`, `or`, `not`
- `if` / `elif` / `else`
- `for` and `while`
- `assert`
- top-level `def`
- `return`
- lists, dicts, sets, tuples
- list comprehensions
- subscripts and ordinary sequence slices
- module-level `import` for non-stdlib modules only

### Explicitly forbidden

- `class`
- `async` / `await`
- `lambda`
- `try` / `except` / `finally`
- `with`
- generators: `yield`, `yield from`, generator expressions
- `from x import y`
- nested function definitions
- imports inside functions
- stdlib or builtin-module imports such as `os`, `sys`, `json`, `datetime`, `decimal`
- names that start or end with `_`
- dangerous builtins like `eval`, `exec`, `getattr`, `dir`, `vars`, `id`, `type`

## 3. Allowed Builtins

The current builtin allowlist is:

```text
Exception False None True abs all any ascii bin bool bytearray bytes chr
dict divmod filter float format frozenset hex int isinstance issubclass len
list map max min oct ord pow range reversed round set sorted str sum tuple zip
```

Anything outside that list should be treated as unavailable.

## 4. Runtime-Injected Names

These are available directly in contract code. Do **not** import them from the
Python stdlib.

### Storage / contract runtime

- `Variable`
- `Hash`
- `ForeignVariable`
- `ForeignHash`
- `LogEvent`
- `ctx`
- `Any`

### Injected helper modules / values

- `datetime`
- `decimal`
- `hashlib`
- `crypto`
- `random`
- `importlib`
- `now`
- `block_num`
- `block_hash`

Important nuances:

- `decimal` is **not** the stdlib `decimal` module. It is the Xian decimal constructor.
- `datetime` is a deterministic injected module-like object with:
  - `datetime.datetime(...)`
  - `datetime.timedelta(...)`
  - `datetime.DAYS`, `datetime.HOURS`, `datetime.MINUTES`, `datetime.SECONDS`, `datetime.WEEKS`
- `random` is deterministic and **not secret randomness**. Call `random.seed()` before using it.

## 5. Contract Structure

### Standard template

```python
owner = Variable()
balances = Hash(default_value=0)

@construct
def seed():
    owner.set(ctx.caller)
    balances[ctx.caller] = 1000

@export
def transfer(to: str, amount: float):
    assert amount > 0, "Amount must be positive"
    balances[ctx.caller] -= amount
    balances[to] += amount
```

### Structural rules

- A contract must contain **at least one `@export` function**.
- At most **one** `@construct` function is allowed.
- A function may have **at most one decorator**.
- Valid decorators are only:
  - `@construct`
  - `@export`
- Functions without decorators are compiled as **private helpers**.
- Nested functions are rejected.

## 6. Import Rules

### What is actually allowed

- Module-level `import some_contract` for **non-stdlib** modules
- Dynamic cross-contract imports through injected `importlib`

### Recommended cross-contract pattern

```python
@export
def call_other_contract():
    token = importlib.import_module("currency")
    token.transfer(amount=1.5, to=ctx.caller)
```

### Hard restrictions

- `from ... import ...` is forbidden
- imports inside functions are forbidden
- importing stdlib/builtin modules is forbidden
- runtime helper names such as `datetime`, `decimal`, `random`, `hashlib`, `crypto`, `importlib`
  are already injected, so **do not import them**

### `importlib.import_module(...)` contract-name rules

The target name must:

- be lowercase
- contain only alphanumeric characters or `_`
- not start with `_`
- not be digits-only

Practical naming note:

- system contracts like `currency`, `members`, `dao` do **not** use `con_`
- user-deployed contracts should normally use names like `con_token`, `con_dex`, etc.

## 7. Type System & Annotations

### Export argument annotations are mandatory

Every `@export` parameter must be annotated.

### Allowed annotation types

- `str`
- `int`
- `float`
- `bool`
- `dict`
- `list`
- `Any`
- `datetime.datetime`
- `datetime.timedelta`

### Return annotations

- Return annotations are **optional**
- If present, they must use the **same allowlist** as export arguments

Valid:

```python
@export
def balance_of(address: str) -> float:
    return balances[address]
```

Invalid:

```python
@export
def bad(value: bytes) -> bytes:
    return value
```

### Important distinction

- The **annotation allowlist** is small.
- The **runtime value codec** can encode more than that.

For example, return values and stored values may still contain encodable types like:

- decimal-backed values
- `bytes`
- `datetime.datetime` / `datetime.timedelta`
- very large integers
- nested `dict` / `list` structures using supported values

If you need to return an encodable value that is not in the annotation allowlist,
leave the return annotation off.

## 8. Decimal / Float Semantics

This is one of the most important Xian-specific rules.

- Contract authors use normal Python `float` syntax.
- At runtime, those values are executed as deterministic **decimal-backed** values.
- Float literals are preserved from source and compiled into Xian decimal values.

Examples:

```python
price = 10.5
fee = decimal("0.0025")
total = price + fee
```

### Current numeric policy

- up to **61 whole digits**
- up to **30 fractional digits**
- extra fractional digits are **truncated toward zero**
- values outside the supported range raise **overflow** and abort execution

This means:

- there is **no silent overflow clamping**
- `float` in contracts is the correct user-facing decimal type
- you do **not** need to expose integer base units just to stay safe

## 9. Storage Rules

### Valid declarations

```python
counter = Variable(default_value=0)
balances = Hash(default_value=0)

token_supply = ForeignVariable(
    foreign_contract="currency",
    foreign_name="total_supply",
)

token_balances = ForeignHash(
    foreign_contract="currency",
    foreign_name="balances",
)
```

### Hard rules

- `Variable`, `Hash`, `ForeignVariable`, `ForeignHash`, and `LogEvent` must be declared explicitly
- tuple unpacking for ORM declarations is forbidden
- for `Variable`, `Hash`, and `LogEvent`, do **not** pass `contract=` or `name=`; the compiler injects them
- `ForeignVariable` and `ForeignHash` are **read-only**

### Hash behavior

- max **16 dimensions**
- each key component must not contain `:` or `.`
- final encoded key length must be at most **1024 bytes**
- `Hash` slices are forbidden
- `key in my_hash` is forbidden
- missing keys return `default_value` if provided, otherwise `None`

Recommended existence pattern:

```python
pool = pools[pool_id]
if pool is None:
    ...
```

Not:

```python
if pool_id in pools:
    ...
```

## 10. Context & Time

### Available context

- `ctx.caller`
- `ctx.signer`
- `ctx.this`
- `ctx.owner`
- `ctx.entry`
- `ctx.submission_name`

### Block/environment values

- `now`
- `block_num`
- `block_hash`

### Time semantics

- `now` is **consensus block time**, not validator wall-clock time
- every transaction in the same block sees the same `now`
- `now` is UTC-based and represented through Xian's deterministic datetime bridge
- precision is effectively **microseconds**

Important contract-design consequence:

- On an `on_demand` network, time does **not** advance while the chain is idle.
- Time-based logic must be checked **when a transaction executes**, not assumed to happen automatically in the background.

Good:

```python
assert now < deadline, "Permit expired"
assert now >= unlock_time.get(), "Too early"
```

Bad assumption:

- "this should auto-execute at exactly 12:00 UTC"

## 11. Event Logging

### Definition

```python
TransferEvent = LogEvent(
    event="Transfer",
    params={
        "from": {"type": str, "idx": True},
        "to": {"type": str, "idx": True},
        "amount": {"type": (int, float, decimal)},
    },
)
```

### Emission

```python
@export
def transfer(to: str, amount: float):
    TransferEvent({"from": ctx.caller, "to": to, "amount": amount})
```

### Hard rules

- max **3 indexed parameters**
- every declared parameter must be present
- no unexpected parameters
- each value must match the declared type
- each parameter value must be at most **1024 bytes** once string-encoded

Allowed event value types are:

- `str`
- `int`
- `float`
- `bool`
- decimal-backed values

## 12. Limits & Metering

Current important limits:

- recursion depth: **1024**
- max hash dimensions: **16**
- max key size: **1024 bytes**
- max aggregate write capacity per transaction: **128 KiB**
- max encoded return value size: **128 KiB**

Current byte-based costs:

- reads: **1 stamp / byte**
- writes: **25 stamps / byte**
- return values: **1 stamp / byte**

Practical implication:

- storage keys and values should stay compact
- large return payloads are both **metered** and **capped**

## 13. Recommended Patterns

### Cross-contract interface checks

```python
@export
def call_other_contract():
    token = importlib.import_module("currency")

    required = [
        importlib.Func("transfer", args=("amount", "to")),
        importlib.Var("balances", Hash),
    ]
    assert importlib.enforce_interface(token, required)
    token.transfer(amount=10, to=ctx.caller)
```

### Deterministic random

```python
@export
def roll() -> int:
    random.seed()
    return random.randint(1, 6)
```

Use this only for low-stakes or game-like logic. It is deterministic and predictable from public inputs.

### Efficient state access

Prefer:

```python
sender = ctx.caller
balance = balances[sender]
balance += amount
balances[sender] = balance
```

over repeated reads/writes to the same key.

## 14. LLM Checklist

When generating Xian contract code:

1. Use the injected runtime names directly; do not import stdlib helpers.
2. Ensure the contract has at least one `@export`.
3. Use at most one `@construct`.
4. Use at most one decorator per function.
5. Keep helper functions top-level.
6. Annotate every `@export` argument.
7. If you add a return annotation, keep it within the allowed annotation list.
8. Use `float` for user-facing decimal amounts.
9. Assume float literals become deterministic decimal values.
10. Never rely on wall-clock time; use `now`.
11. Never use `in` on `Hash`.
12. Keep hash keys free of `:` and `.`.
13. Treat `ForeignVariable` and `ForeignHash` as read-only.
14. Keep return payloads and writes small.
15. Call `random.seed()` before using deterministic random helpers.
16. Prefer explicit, flat, assert-driven logic over clever Python tricks.

## 15. Things To Avoid Even If They Look Pythonic

- importing `datetime`, `decimal`, `random`, `hashlib`, `crypto`, or `importlib`
- `try/except` for normal flow control
- nested helper functions
- huge return dicts or giant write-heavy loops
- assuming background timers or autonomous execution
- assuming binary floating-point semantics

If you want "normal Python that just works" on Xian, the safe mental model is:

- write ordinary-looking Python
- stay inside the restricted subset
- use `float` for decimal-backed user values
- use `now` as chain time
- use explicit state and assertions
