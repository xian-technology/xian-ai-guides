# XIAN NETWORK — SMART CONTRACT DEVELOPMENT GUIDE

## 1. Core Language Rules

### Language Version

* Contracts must use **Python 3.11 syntax**
* Features introduced in later versions are **prohibited**

### Allowed Builtins

**Primitive types / constants**

* `bool`, `int`, `str`, `float`, `dict`, `list`, `tuple`, `set`, `frozenset`
* `True`, `False`, `None`, `Exception`

**Type checks / conversions**

* `isinstance`, `issubclass`, `bytes`, `bytearray`
* `chr`, `ord`, `hex`, `bin`, `ascii`, `format`, `oct`

**Math / iterables**

* `abs`, `divmod`, `max`, `min`, `pow`, `round`, `sum`
* `all`, `any`, `filter`, `map`, `range`, `reversed`, `sorted`, `zip`

**Core utilities**

* `len`

### Explicitly Forbidden

* `print()` / file I/O
* Classes (`class`), `async` / `await`, `lambda`
* Exception handling (`try` / `except` / `finally`)
* Metaprogramming (`eval`, `exec`, `getattr`)
* Introspection helpers (`type()`, `dir()`, `vars()`, `id()`)
* Underscore prefixes/suffixes (`__private_var`, `var__`)
* Standard library imports (`os`, `sys`, `json`, etc.)

---

## 2. Contract Structure

### Mandatory Template

```python
# State declarations (top-level only)
owner = Variable()
balances = Hash()

# Constructor (runs once on deployment)
@construct
def init():
    owner.set(ctx.caller)
    balances[ctx.caller] = 1000

# Exported function (public interface)
@export
def transfer(to: str, amount: int):
    assert amount > 0, "Amount must be positive"
    sender = ctx.caller
    balances[sender] -= amount
    balances[to] += amount
```

---

## 3. Import System Rules

### Allowed Module Access

* The runtime preloads the following into the contract scope (use directly):

  * `Variable`, `Hash`, `ForeignVariable`, `ForeignHash`
  * `LogEvent`, `ctx`, `Any`
  * `importlib`, `datetime`, `decimal`, `hashlib`, `crypto`, `random`
* **Do not** add `import` statements for standard-library helpers — the linter rejects them
* Nested imports (inside functions) are blocked
* For cross-contract calls use injected `importlib`:

  ```python
  token = importlib.import_module('currency')
  ```
* Target contract names must:

  * be lowercase
  * be alphanumeric or `_`
  * not start with `_`
* `importlib.import_module` raises if the contract is missing or invalid

### Forbidden Imports

* Any `from ... import ...` statement
* Standard library modules (`os`, `sys`, `json`, `datetime`, `decimal`, etc.)

---

## 4. Type System & Annotations

### Valid Parameter Types (No Return Types)

```python
@export
def valid_annotations(
    name: str,                   # Primitive
    metadata: dict,              # Key-value store
    deadline: datetime.datetime, # datetime.datetime / datetime.timedelta allowed
    flag: bool,                  # Boolean flags are valid
    raw: Any                     # Use Any when dynamic typing is unavoidable
):                               # NO return type annotations allowed
    pass
```

### Forbidden

* Return type annotations (`-> str`, `-> int`, `-> None`, etc.)
* Missing annotations on `@export` parameters
* Contract addresses must be strings prefixed with `"con_"` (e.g. `"con_xsc001"`)

---

## 5. Decorator Constraints

### Strict Rules

* **Maximum ONE decorator per function**
* **Only ONE `@construct` function per contract**
* Functions without decorators become private (auto-prefixed with `__`)

### Valid Decorators

```python
@construct  # Constructor (only one allowed)
@export     # Public function
```

---

## 6. State Management

### Allowed ORM Objects

```python
# Single-value storage
config = Variable(t=int, default_value=0)

# Key-value storage (max 16 dimensions)
ledger = Hash(default_value=0)
```

```python
# Cross-contract access (read-only)
token_supply = ForeignVariable(
    foreign_contract='currency',
    foreign_name='total_supply'
)
```

* `ForeignHash` / `ForeignVariable` raise `ReferenceError` on writes
* Treat them as **read-only views**

### ORM Construction Rules

```python
# CORRECT
balances = Hash()
supply = Variable()
```

```python
# FORBIDDEN
balances = Hash(contract='token', name='bal')  # Auto-added by compiler
x, y = Hash(), Variable()                      # No tuple unpacking
```

### Hash Limitations

```python
# FORBIDDEN
if pool_id in pools:  # ✗ Raises exception
```

```python
# CORRECT
pool = pools[pool_id]
if pool is not None:  # ✓ Proper existence check
```

* `Hash` returns `default_value` (or `None`) for missing keys
* Keys cannot contain `:` or `.`
* Slices (`balances[:]`) are rejected
* Floats stored in `Hash` return `ContractingDecimal` automatically

---

## 7. Context Variables & Block Info

### Available Context

```python
@export
def get_context_info():
    sender = ctx.caller
    signer = ctx.signer
    contract = ctx.this
    owner = ctx.owner
    entry = ctx.entry
    submission = ctx.submission_name
    timestamp = now
    height = block_num
    hash_val = block_hash
    return {
        "caller": sender,
        "signer": signer,
        "contract": contract,
        "entry": entry,
        "submission": submission,
        "timestamp": timestamp,
        "height": height,
        "hash": hash_val
    }
```

* `now`, `block_num`, `block_hash` are injected by the execution environment
* Testing defaults are deterministic

---

## 8. Variable Naming Restrictions

### Forbidden Names

* Function args cannot reuse ORM variable names
* No leading/trailing underscores
* Reserved words: `rt`, `Hash`, `Variable`, `LogEvent`
* No `ctx`, `now`, `block_num`, `block_hash` as variable names

---

## 9. Allowed Control Flow

### Permitted Patterns

```python
if balances[ctx.caller] > 100:
    status = "VIP"
else:
    status = "Standard"
```

```python
for i in range(10):
    process(i)
```

```python
assert ctx.signer == owner.get(), "Unauthorized"
```

* Nested function definitions are rejected — helpers must be top-level

---

## 10. Cross-Contract Calls

```python
@export
def call_other_contract():
    token = importlib.import_module('currency')

    required = [
        importlib.Func('transfer', args=('amount', 'to')),
        importlib.Var('balances', Hash)
    ]
    assert importlib.enforce_interface(token, required)

    token.transfer(amount=100, to=ctx.caller)
```

---

## 11. Automatic Type Conversion

```python
price = 10.5  # ContractingDecimal('10.5')
```

```python
result = decimal('100.0') + decimal('50.25')
```

```python
total = decimal(str(user_input))
```

---

## 12. System Limits

* **Hash dimensions:** max 16
* **Key size:** max 1024 bytes
* **Stamp costs:** Reads = 1/byte, Writes = 25/byte
* **Recursion depth:** max 1024

---

## 13. Event Logging

### Event Definition

```python
TransferEvent = LogEvent(
    event="Transfer",
    params={
        "from": {"type": str, "idx": True},
        "to": {"type": str, "idx": True},
        "amount": {"type": (int, float, decimal)}
    }
)
```

### Emitting Events

```python
@export
def transfer(amount: float, to: str):
    TransferEvent({"from": ctx.caller, "to": to, "amount": amount})
```

### Rules

* Max **3 indexed parameters**
* All params must be provided
* Each value ≤ 1024 bytes
* Convert non-primitives to `str`

---

## Key Validation Checklist for LLMs

1. Use injected deterministic `random` and call `random.seed()`
2. No nested imports
3. At least one `@export`
4. Use `Variable()` / `Hash()` only
5. No return annotations
6. No underscore prefixes/suffixes
7. No ORM name reuse in args
8. One decorator per function
9. One `@construct`
10. No tuple unpacking
11. Floats auto-convert
12. Use `ctx.caller`, `ctx.signer`
13. Use injected globals only
14. Hash max 16 dimensions
15. Event params must match schema
16. Max 3 indexed event params
17. No `in` operator on Hash
18. Helpers must be top-level

---

## Optimization Rules

```python
balances[ctx.caller] = 500  # High cost
```

```python
balance = balances[ctx.caller]  # Low cost
```

```python
user_balance = balances[ctx.caller]
user_balance += amount
balances[ctx.caller] = user_balance
```

---

## Nuances

### Random

```python
random.seed()
```

### Datetime

```python
datetime.DAYS * 5  # works
```

### Decimal

```python
decimal("0.0")  # works
```

### Arithmetic Order

```python
(funder["amount_contributed"] * pool["otc_actual_received_amount"]) / pool["amount_received"]
```

### Keyword Arguments

```python
exploit_contract.setlisting(l=listing_id)  # works
```
