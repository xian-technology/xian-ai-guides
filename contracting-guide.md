# XIAN NETWORK - SMART CONTRACT DEVELOPMENT GUIDE

This guide is for writing **valid Xian contracts against the current codebase**.
It is written for LLMs and humans who need to produce contract source, tests,
or reviews that match the real `xian-contracting` linter/runtime behavior and
current `xian-abci` time semantics.

Use it as a **hard rules + safe defaults** guide. If a request conflicts with
this guide, prefer this guide and explain the constraint.

## 1. AI Contract-Writing Workflow

When asked to write a Xian contract, do this in order:

1. Extract a contract spec:
   - contract name, normally `con_*` for user-deployed contracts
   - actors and permissions
   - exported functions and argument types
   - state variables, hash keys, and default values
   - invariants and failure messages
   - events needed by wallets, indexers, bots, or tests
   - time, randomness, cross-contract, or token-standard requirements
2. Choose the smallest explicit state model that satisfies the spec.
3. Write restricted Python contract source only.
4. Write local `ContractingClient` tests for success and rejection paths.
5. If deployment is requested, submit source through the public source-backed
   path. Do not produce public RPC defaults or public chain IDs.

Good contract output is usually:

- one `@construct` at most
- several explicit `@export` functions
- top-level helper functions for repeated guards
- module-level `Variable`, `Hash`, `ForeignVariable`, `ForeignHash`, and
  `LogEvent` declarations
- `assert` guards with clear messages
- concise events after meaningful state changes
- tests that prove permissions and invariants

## 2. Execution Model

- Xian contracts use a **restricted Python subset**, not unrestricted Python.
- Contract source is Python-like source, but current network execution targets
  **`xian_vm_v2`**, not a general Python VM.
- The Rust compiler admits contract source and derives Xian VM IR. The native
  VM executes it on validators and in V2 local tests.
- Current Python tooling targets **Python 3.14+** and is validated on **3.14**.
- Do not rely on newest host-Python features unless this guide says they are
  accepted by the Xian source subset and VM policy.
- Contract execution must stay deterministic across validators.
- The network deployment payload is cleartext source. Nodes derive canonical
  Xian VM IR from that source. Do not design workflows that trust client-supplied
  IR as the deployment payload.
- `ContractingClient` is the local Python test harness for contract behavior and
  lint/runtime feedback. It is useful for authoring tests, but deployment and
  current node execution are source-backed and VM-policy based.

## 3. Minimal Contract Template

Use this shape as the default starting point:

```python
OwnerChanged = LogEvent(
    "OwnerChanged",
    {
        "old_owner": {"type": str, "idx": True},
        "new_owner": {"type": str, "idx": True},
    },
)

owner = Variable()

@construct
def seed(initial_owner: str = ""):
    if initial_owner == "":
        owner.set(ctx.caller)
    else:
        owner.set(initial_owner)

def require_owner():
    assert ctx.caller == owner.get(), "Only owner"

@export
def transfer_ownership(new_owner: str):
    require_owner()
    assert new_owner != "", "Owner required"
    old_owner = owner.get()
    owner.set(new_owner)
    OwnerChanged({"old_owner": old_owner, "new_owner": new_owner})

@export
def get_owner() -> str:
    return owner.get()
```

Functions without decorators are private helpers. Keep helpers top-level and
plain.

## 4. Core Language Rules

### Allowed Syntax

- assignments
- arithmetic: `+`, `-`, `*`, `/`, `//`, `%`, `**`
- comparisons: `==`, `!=`, `<`, `<=`, `>`, `>=`, `in`, `not in`, `is`,
  `is not`
- boolean logic: `and`, `or`, `not`
- `if` / `elif` / `else`
- `for` and `while`
- `assert`
- top-level `def`
- `return`
- lists, dicts, tuples
- list comprehensions
- subscripts and ordinary sequence slices
- module-level `import some_contract` for non-stdlib contract modules

### Forbidden Syntax And Patterns

- `class`
- `async` / `await`
- `lambda`
- `try` / `except` / `finally`
- `with`
- generators: `yield`, `yield from`, generator expressions
- `from x import y`
- nested function definitions
- imports inside functions
- stdlib or builtin-module imports such as `os`, `sys`, `json`, `datetime`,
  `decimal`
- names that start or end with `_`
- semicolons
- one-line compound statements such as `if ok: return value`
- set literals and set comprehensions. Use `set(...)` if you truly need a set.
- dangerous/general builtins such as `eval`, `exec`, `getattr`, `dir`, `vars`,
  `id`, `type`

### Augmented Assignment Nuance

`+=` and `-=` are normal. Augmented multiplication, exponentiation, and shifts
are rejected:

```python
# bad
value *= rate
value **= 2

# good
value = value * rate
value = value ** 2
```

## 5. Allowed Builtins

The current builtin allowlist is:

```text
Exception False None True abs all any ascii bin bool bytearray bytes chr
dict divmod filter format frozenset hex int isinstance issubclass len
list map max min oct ord pow range reversed set sorted str sum tuple zip
```

Anything outside that list should be treated as unavailable unless it is an
injected runtime name listed below.

## 6. Runtime-Injected Names

These are available directly in contract code. Do **not** import them from the
Python stdlib.

### Storage / Contract Runtime

- `Variable`
- `Hash`
- `ForeignVariable`
- `ForeignHash`
- `LogEvent`
- `ctx`
- `Any`

### Injected Helper Modules / Values

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

- `decimal` is the Xian deterministic decimal constructor, not the stdlib
  `decimal` module.
- `datetime` is a deterministic injected module-like object with
  `datetime.datetime(...)`, `datetime.timedelta(...)`, and constants such as
  `datetime.DAYS`, `datetime.HOURS`, `datetime.MINUTES`,
  `datetime.SECONDS`, and `datetime.WEEKS`.
- `random` is deterministic and not secret randomness. Call `random.seed()`
  before using it.

### Context Fields

Commonly useful context fields:

- `ctx.caller`
- `ctx.signer`
- `ctx.this`
- `ctx.owner`
- `ctx.entry`
- `ctx.submission_name`

Use `ctx.caller` for the immediate caller and `ctx.signer` when the original
transaction signer matters across contract-to-contract calls.

## 7. Decorators And Contract Structure

Valid decorator forms:

- `@construct`
- `@export`
- `@export(typecheck=True)`
- `@export(typecheck=False)`

Hard rules:

- a contract must contain at least one `@export`
- at most one `@construct`
- at most one decorator per function
- `@construct` accepts no decorator arguments
- `@export` accepts keyword arguments only, and the only supported keyword is
  `typecheck`
- `typecheck` must be exactly `True` or `False`
- functions without decorators are private helpers
- nested functions are rejected

Use `@export(typecheck=True)` when runtime argument and return type enforcement
is important. This also checks nested container annotations such as
`list[int]` and `dict[str, list[int]]`.

## 8. Type System And Annotations

Every `@export` argument must be annotated.

Allowed annotation bases:

- `Any`
- `bool`
- `bytearray`
- `bytes`
- `datetime.datetime`
- `datetime.timedelta`
- `dict`
- `decimal`
- `frozenset`
- `int`
- `list`
- `set`
- `str`

Subscripted annotations are allowed when their base type is allowed:

```python
@export(typecheck=True)
def summarize(items: list[int], metadata: dict[str, list[int]]) -> int:
    return len(items) + len(metadata["counts"])
```

Return annotations are optional. If present, they must use an allowed
annotation base. When `@export(typecheck=True)` is used, annotated return values
are checked at runtime.

If a function returns a valid encoded value that is awkward to annotate, omit
the return annotation.

## 9. Numbers

Use `int` for integer counts and `decimal` for fractional amounts. Decimal
literals retain their source digits. Both literal and input decimal values
must be exactly representable with at most 30 fractional digits and magnitude
below 10^61. Integers must have magnitude below 2^512. Division produces a
decimal truncated toward zero at scale 30. Integer `//` rounds toward negative
infinity. Integer `%` follows the divisor's sign. Decimal multiplication
truncates toward zero at scale 30. Use `quantize(value, places, mode)` for
settlement at a smaller scale. Its modes are `toward_zero`, `floor`, `ceil`,
and `half_even`. Use `sqrt(value)` for a square root truncated to the decimal
grid. For integer amounts, `mul_div_floor(a, b, divisor)` and
`mul_div_ceil(a, b, divisor)` allow a wider product before dividing by a
positive integer.

A `decimal` parameter accepts a decimal or an exactly representable integer.
An `int` parameter accepts integers. Booleans and strings stay distinct.
`//` and `%` require integers. `int(-1.9)` truncates to `-1`.
Use `quantize(1.005, 2, "half_even")` for ties-to-even settlement (result
`1`), or `quantize(-1.005, 2, "floor")` to round down (result `-1.01`).
An out-of-range result or division by zero fails the call and rolls back its
writes and events. The [numeric specification](https://github.com/xian-technology/xian-meta/blob/main/docs/NUMERIC_PROFILE_V2.md)
includes exact bounds and signed arithmetic examples.

```python
price = 10.5
fee = decimal("0.0025")
total = price + fee
```

## 10. Storage Rules

### Valid Declarations

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

### Hard Rules

- `Variable`, `Hash`, `ForeignVariable`, `ForeignHash`, and `LogEvent` must be
  declared explicitly.
- Tuple unpacking for ORM declarations is rejected.
- For `Variable`, `Hash`, and `LogEvent`, do **not** pass `contract=` or
  `name=`; the compiler/runtime injects them.
- `ForeignVariable` and `ForeignHash` are read-only.
- Export arguments must not shadow module-level ORM variable names.

### `Variable`

Use `Variable` for one value:

```python
owner = Variable()
settings = Variable(default_value={})
queue = Variable(default_value=[])
```

`Variable` supports `.get()` and `.set(value)`. For stored dict/list values it
also supports convenience operations such as item access, `update`, `append`,
`extend`, `insert`, `remove`, `clear`, and `pop`. These rewrite the stored
object, so keep values compact.

### `Hash`

Use `Hash` for mappings:

```python
balances = Hash(default_value=0)
allowances = Hash(default_value=0)
allowances[owner, spender] = 100
```

Hash behavior:

- max 16 dimensions
- each key component must not contain `:` or `.`
- final encoded key length must be at most 1024 bytes
- hash slices are forbidden
- `key in my_hash` is forbidden
- missing keys return `default_value` if provided, otherwise `None`

Recommended existence pattern:

```python
pool = pools[pool_id]
assert pool is not None, "Missing pool"
```

Do not do this:

```python
if pool_id in pools:
    ...
```

## 11. Event Logging

Prefer the positional `LogEvent("Name", params)` style:

```python
TransferEvent = LogEvent(
    "Transfer",
    {
        "from": {"type": str, "idx": True},
        "to": {"type": str, "idx": True},
        "amount": {"type": decimal},
    },
)
```

`indexed(str)` is also available, but the dict form makes indexing explicit and
is easier for an AI to modify safely:

```python
TransferEvent = LogEvent(
    "Transfer",
    {
        "from": indexed(str),
        "to": indexed(str),
        "amount": (int, decimal),
    },
)
```

Emit events with exactly the declared fields:

```python
@export
def transfer(to: str, amount: decimal):
    TransferEvent({"from": ctx.caller, "to": to, "amount": amount})
```

Hard rules:

- max 3 indexed parameters
- every declared parameter must be present
- no unexpected parameters
- each emitted value must match its declared type
- each parameter value must be at most 1024 bytes once string-encoded

Common event value types are `str`, `int`, `decimal`, `bool`, and
decimal-backed values. The runtime also supports `bytes`, `bytearray`, and
contracting set/frozenset values, but prefer simple scalar event fields for
indexers and wallets.

## 12. Import Rules And Cross-Contract Calls

### What Is Allowed

- module-level `import some_contract` for non-stdlib contract modules
- dynamic cross-contract imports through injected `importlib`

### Hard Restrictions

- `from ... import ...` is forbidden
- imports inside functions are forbidden
- importing stdlib/builtin modules is forbidden
- runtime helper names such as `datetime`, `decimal`, `random`, `hashlib`,
  `crypto`, and `importlib` are injected, so do not import them

### Dynamic Import Pattern

```python
@export
def pay(token_name: str, to: str, amount: decimal):
    token = importlib.import_module(token_name)
    token.transfer(amount=amount, to=to)
```

`importlib.import_module(...)` target names must:

- be lowercase
- contain only alphanumeric characters or `_`
- not start with `_`
- not be digits-only

System contracts like `currency`, `members`, and `dao` do not use `con_`.
User-deployed contracts should normally use names like `con_token`,
`con_escrow`, or `con_staking`.

### Interface Check Pattern

When trusting another contract's shape matters, check it:

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

## 13. Context And Time

Block/environment values:

- `now`
- `block_num`
- `block_hash`

Time semantics:

- `now` is consensus block time, not validator wall-clock time
- every transaction in the same block sees the same `now`
- `now` is UTC-based and represented through Xian's deterministic datetime
  bridge
- precision is effectively microseconds
- on an `on_demand` network, time does not advance while the chain is idle
- time-based logic must be checked when a transaction executes

Good:

```python
assert now < deadline, "Permit expired"
assert now >= unlock_time.get(), "Too early"
```

Bad assumption:

- "this should auto-execute at exactly 12:00 UTC"

Time lock pattern:

```python
unlock_at = Hash(default_value=None)

@export
def lock(days: int):
    assert days > 0, "Invalid duration"
    unlock_at[ctx.caller] = now + datetime.timedelta(days=days)

@export
def unlock():
    target = unlock_at[ctx.caller]
    assert target is not None, "Nothing locked"
    assert now >= target, "Too early"
    unlock_at[ctx.caller] = None
```

## 14. Randomness And Crypto

Deterministic random pattern:

```python
@export
def roll() -> int:
    random.seed()
    return random.randint(1, 6)
```

Use deterministic random only for low-stakes or game-like logic. It is
predictable from public inputs and must not be used for secret generation,
valuable lotteries, validator selection, or security-critical decisions.

Use injected `hashlib` and `crypto` helpers only when the contract design
requires them and current Xian contracts/tests show the intended pattern. Do not
import Python stdlib crypto modules.

## 15. Limits And Metering

Current important limits:

- execution depth (recursion plus cross-contract reentry): 128 frames
- max hash dimensions: 16
- max key size: 1024 bytes
- max contract source submission size: 128 KiB
- max aggregate write capacity per transaction: 128 KiB
- max `range(...)` / sequence repetition length: 131,072 entries
- max binary allocation / string repetition size: 128 KiB
- max integer string chars: 8192
- max modular pow exponent bits: 8192

There is no general return-value size cap; return bytes are metered.

Current byte-based costs, in raw meter units (1,000 raw units = 1 chi;
`chi_used = raw // 1000 + 5`):

- reads: 2 units / byte
- writes: 25 units / byte
- transaction bytes: 1 unit / byte
- return values: 1 unit / byte

Practical implication:

- storage keys and values should stay compact
- large return payloads are metered and capped
- avoid write-heavy loops over unbounded user-controlled data
- cache repeated reads into local variables and write once

## 16. Safe Patterns

### Permission Guard

```python
owner = Variable()

@construct
def seed():
    owner.set(ctx.caller)

def require_owner():
    assert ctx.caller == owner.get(), "Only owner"
```

### Efficient State Update

Prefer:

```python
sender = ctx.caller
balance = balances[sender]
assert balance >= amount, "Insufficient balance"
balances[sender] = balance - amount
```

Over repeated reads/writes to the same key.

### Token-Like Transfer

```python
TransferEvent = LogEvent(
    "Transfer",
    {
        "from": {"type": str, "idx": True},
        "to": {"type": str, "idx": True},
        "amount": {"type": decimal},
    },
)

balances = Hash(default_value=0)

@export
def transfer(amount: decimal, to: str):
    assert amount > 0, "Amount must be positive"
    sender = ctx.caller
    sender_balance = balances[sender]
    assert sender_balance >= amount, "Insufficient balance"
    balances[sender] = sender_balance - amount
    balances[to] += amount
    TransferEvent({"from": sender, "to": to, "amount": amount})
```

### Allowance Pattern

```python
approvals = Hash(default_value=0)

@export
def approve(amount: decimal, to: str):
    assert amount >= 0, "Cannot approve negative amount"
    approvals[ctx.caller, to] = amount

@export
def transfer_from(amount: decimal, to: str, main_account: str):
    assert amount > 0, "Amount must be positive"
    allowed = approvals[main_account, ctx.caller]
    balance = balances[main_account]
    assert allowed >= amount, "Not approved"
    assert balance >= amount, "Insufficient balance"
    approvals[main_account, ctx.caller] = allowed - amount
    balances[main_account] = balance - amount
    balances[to] += amount
```

## 17. XSC001 Token Defaults

For fungible tokens, target the current XSC001 shape unless the user explicitly
asks for a different token interface.

Expected storage:

- `balances`
- `approvals`
- `metadata`

Expected exports:

- `change_metadata(key, value)`
- `transfer(amount, to)`
- `approve(amount, to)`
- `transfer_from(amount, to, main_account)`
- `balance_of(address)`

Expected metadata fields:

- `token_name`
- `token_symbol`
- `token_logo_url`
- `token_logo_svg`
- `token_website`

Notes:

- The current XSC001 checker does not require `metadata["operator"]`.
- User-facing tokens often still add helpers such as `allowance(...)`,
  `get_metadata()`, `total_supply`, or operator rotation.
- Interface compliance is not economic safety. Add supply, mint/burn,
  ownership, and fee behavior deliberately.

## 18. Local Testing

Use the `xian-contracting` source checkout for native V2 tests. From its
parent directory run `uv sync --project xian-contracting --frozen --group dev`.
The standard `ContractingClient` compiles and runs contracts on `xian_vm_v2`.
[Contract source setup](https://github.com/xian-technology/xian-docs-web/blob/main/getting-started/dev-environment.md#contract-only-setup)

```python
from pathlib import Path

import pytest
from contracting.local import ContractingClient

CONTRACT = Path("contracts/con_example.s.py").read_text()

@pytest.fixture
def client(tmp_path):
    with ContractingClient(signer="alice", storage_home=tmp_path) as c:
        c.submit(CONTRACT, name="con_example")
        yield c

def test_owner_can_transfer_ownership(client):
    contract = client.get_contract_proxy("con_example")
    contract.transfer_ownership(new_owner="bob")
    assert contract.get_owner() == "bob"

def test_non_owner_rejected(client):
    contract = client.get_contract_proxy("con_example")
    with pytest.raises(AssertionError):
        contract.transfer_ownership(new_owner="mallory", signer="mallory")
```

Test guidance:

- submit dependency contracts first when cross-contract calls are used
- pass `signer="..."` on proxy calls to test permissions
- test both success and rejection paths
- test boundary amounts, empty strings, missing records, deadline expiry, and
  duplicate actions when relevant
- assert stored state and emitted behavior, not just that a call returns

Use the sibling `xian-linter` checkout's environment for linting. From the
workspace, run `uv sync --project xian-linter --frozen`. Its standard
check validates the same V2 target.

```python
from xian_linter import lint_code_sync

errors = lint_code_sync(CONTRACT)
assert not errors, errors
```

## 19. Deployment And Source Submission

For network deployment, submit source. Do not submit client-generated IR as the
trusted payload.

Use the matching sibling `xian-py` checkout with
`uv sync --project xian-py --frozen`. This SDK defaults to V2 for signing,
simulation, and decoding. [Pinned SDK setup](https://github.com/xian-technology/xian-docs-web/blob/main/tools/xian-py.md#install)

Use Python integers and `decimal.Decimal` constructed from text for generic
arguments. A generic argument `"1.25"` is a contract string. Binary floats
are rejected. To exercise an archived V1 test configuration, pass
`XianClientConfig(vm_profile="xian_vm_v1")` on its client.

```python
from pathlib import Path

from xian_py import Wallet, Xian

wallet = Wallet("your_private_key")
xian = Xian("http://127.0.0.1:26657", wallet=wallet)
source = Path("contracts/con_example.s.py").read_text()

tx = xian.deploy_contract(
    name="con_example",
    source=source,
    args={"initial_owner": wallet.public_key},
    mode="checktx",
    wait_for_tx=True,
)
```

Rules:

- non-`sys` deployments must use names starting with `con_`
- names must be lowercase ASCII with digits/underscores only
- names are capped at 64 characters
- source size is capped by the runtime submission limit
- do not hardcode public RPC endpoints or public chain IDs in generic examples

## 20. LLM Checklist

Before returning generated Xian contract code:

1. Convert the user request into actors, exports, state, invariants, and events.
2. Use injected runtime names directly; do not import stdlib helpers.
3. Ensure the contract has at least one `@export`.
4. Use at most one `@construct`.
5. Use at most one decorator per function.
6. Keep helper functions top-level and undecorated.
7. Annotate every `@export` argument.
8. Use only allowed annotation bases, or allowed subscripted annotations.
9. Use `@export(typecheck=True)` when runtime type enforcement matters.
10. Use `decimal` for user-facing decimal amounts.
11. Never rely on wall-clock time; use `now`.
12. Never use `in` on `Hash`.
13. Keep hash keys free of `:` and `.`.
14. Treat `ForeignVariable` and `ForeignHash` as read-only.
15. Keep return payloads, event values, and writes small.
16. Call `random.seed()` before deterministic random helpers.
17. Prefer explicit, flat, assert-driven logic over clever Python tricks.
18. Include local tests for permissions and invariants when producing a contract.

## 21. Things To Avoid Even If They Look Pythonic

- importing `datetime`, `decimal`, `random`, `hashlib`, `crypto`, or
  `importlib`
- `try/except` for normal flow control
- nested helper functions
- classes or dataclasses
- set literals
- hidden dynamic dispatch through reflection
- huge return dicts or giant write-heavy loops
- assuming background timers or autonomous execution
- assuming binary floating-point semantics
- treating indexer/dashboard reads as consensus facts inside contract logic

Safe mental model:

- write ordinary-looking Python
- stay inside the restricted subset
- use `decimal` for deterministic decimal-backed user values
- use `now` as chain time
- use explicit state and assertions
- test behavior with `ContractingClient`
