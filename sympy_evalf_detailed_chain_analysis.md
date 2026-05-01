# SymPy evalf 完整链路深度分析（修正版）

## 摘要

本报告基于代码精确分析，修正了之前版本的错误结论，并深入分析：
1. **subs 数值替换后的精度重标定链路**
2. **重复求值缓存机制**
3. **默认参数与调高上限的行为差异**
4. **修正的完整调用链路与代码对齐**

---

## 一、关键代码修正

### 1.1 之前报告的错误结论

#### 错误 1：加法退出条件的理解

**之前的理解**：
> 退出条件检查的是 `(prec - target_prec) > options['maxprec']`，即"已经提升的精度量"是否超过 maxprec。

**实际代码**（`sympy/core/evalf.py:618`）：
```python
def evalf_add(v: 'Add', prec: int, options: OPT_DICT) -> TMP_RES:
    oldmaxprec = options.get('maxprec', DEFAULT_MAXPREC)  # 默认 333
    target_prec = prec
    
    while 1:
        options['maxprec'] = min(oldmaxprec, 2*prec)  # 临时限制
        # ...
        
        if acc >= target_prec:
            break
        else:
            if (prec - target_prec) > options['maxprec']:  # 退出条件
                break
            
            prec = prec + max(10 + 2**i, target_prec - acc)
            i += 1
```

**正确分析**：

每次迭代中 `options['maxprec'] = min(oldmaxprec, 2*prec)`，退出条件是 `(prec - target_prec) > options['maxprec']`。

让我们分阶段分析：

**阶段 1：当 2*prec < oldmaxprec（prec 较小时）**

假设 `target_prec = 53`，`oldmaxprec = 333`：

| 迭代 | prec | options['maxprec'] = min(333, 2*prec) | 退出条件检查 |
|-----|------|----------------------------------------|-----------|
| 0 | 53 | min(333, 106) = 106 | `(53-53)=0 > 106`? **False** |
| 1 | ~64+ | min(333, ~128+) = ~128+ | `(64+-53)=11+ > 128+`? **False** |
| ... | ... | ... | **永远 False** |

**关键发现**：当 `2*prec < oldmaxprec` 时，退出条件 `(prec - target_prec) > 2*prec` 等价于 `-target_prec > prec`，由于 `prec` 和 `target_prec` 都是正数，这**永远不会成立**！

**阶段 2：当 2*prec >= oldmaxprec（prec 足够大时）**

当 `prec >= 167` 时（因为 `2*167 = 334 > 333）：

| 迭代 | prec | options['maxprec'] | 退出条件检查 |
|-----|------|-------------------|-----------|
| N | 167 | min(333, 334) = 333 | `(167-53)=114 > 333`? **False** |
| N+1 | 200 | 333 | `(200-53)=147 > 333`? **False** |
| N+2 | 300 | 333 | `(300-53)=247 > 333`? **False** |
| N+3 | 400 | 333 | `(400-53)=347 > 333`? **True!** |

**结论**：
- 实际最大 `prec ≈ target_prec + oldmaxprec`
- 默认值：`53 + 333 = 386` bits（不是 333）

#### 错误 2：evalf_float 的精度返回

**之前的理解**：
> evalf_float 直接返回 Float 内部存储的精度。

**实际代码**（`sympy/core/evalf.py:481-482`）：
```python
def evalf_float(expr: 'Float', prec: int, options: OPT_DICT) -> TMP_RES:
    return expr._mpf_, None, prec, None
```

**正确分析**：

`evalf_float` 返回的 `re_acc = prec`（请求的精度），而不是 Float 实际的精度！

**Float 文档中的说明**（`sympy/core/numbers.py:699-716`）：
```python
>>> approx, exact = Float(.1, 1), Float(.125, 1)

>>> approx.evalf(5)  # 请求 5 位精度
0.099609              # 显示 5 位，但实际只有 1 位的准确性！

>>> exact.evalf(20)     # 0.125 是二进制精确值
0.12500000000000000000  # 可以任意精度
```

**关键发现**：
- `evalf_float` 返回的 `prec` 是**请求的精度**，不是**实际的精度**
- 如果用户传入低精度 Float（如 `Float(.1, 1)`，即使请求高精度，结果也只有 1 位的**实际准确性**
- 这是设计决定：不改变 Float 的底层值，只改变显示精度

#### 错误 3：Float 精度的"提升"

**之前的理解**：
> Float 的精度会根据请求提升。

**实际代码**（`sympy/core/numbers.py:436-438`）：
```python
def _as_mpf_op(self, prec):
    prec = max(prec, self._prec)  # 取较大值！
    return self._as_mpf_val(prec), prec
```

**正确分析**：
- `_as_mpf_op` 使用 `max(prec, self._prec)`
- 如果请求精度 < Float 实际精度：使用 Float 实际精度
- 如果请求精度 > Float 实际精度：**仍然使用 Float 实际精度**
- **Float 的精度永远不会被"提升"，只能保持或降低

---

## 二、subs 数值替换后的精度重标定链路

### 2.1 subs 处理的两个路径

**关键代码**（`sympy/core/evalf.py:1379-1394`）：

```python
def evalf_symbol(x: Expr, prec: int, options: OPT_DICT) -> TMP_RES:
    val = options['subs'][x]
    
    # 路径 1: val 是 mpf 类型
    if isinstance(val, mpf):
        if not val:
            return None, None, None, None
        return val._mpf_, None, prec, None  # 精度设为请求的 prec
    
    # 路径 2: val 是其他类型（需要转换）
    else:
        if '_cache' not in options:
            options['_cache'] = {}
        cache = options['_cache']
        
        cached, cached_prec = cache.get(x, (None, MINUS_INF))
        if cached_prec >= prec:
            return cached  # 使用缓存
        
        # 关键：递归调用 evalf
        v = evalf(sympify(val), prec, options)
        cache[x] = (v, prec)
        return v
```

### 2.2 完整的 subs 精度重标定链路

```
用户调用: expr.evalf(n=15, subs={x: 0.1, y: 1e-5, z: Rational(1, 3)})
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 阶段 1: EvalfMixin.evalf() 入口点                                      │
│                                                                          │
│ options = {                                                               │
│     'maxprec': max(53, int(100*3.3219)) = 332,                      │
│     'subs': {x: 0.1, y: 1e-5, z: Rational(1, 3)}                     │
│ }                                                                        │
│ 调用 evalf(expr, 53 + 4 = 57, options)                                  │
└─────────────────────────────────────────────────────────────────────────┘
         │
         ▼ 遇到 Symbol x
┌─────────────────────────────────────────────────────────────────────────┐
│ 阶段 2: evalf_symbol() 分发到 evalf_table[Symbol] = evalf_symbol          │
│                                                                          │
│ 分三种情况处理 subs 中的值：                                                │
└─────────────────────────────────────────────────────────────────────────┘
         │
         ├──────────────┬──────────────┐
         ▼            ▼              ▼
    Symbol x       Symbol y         Symbol z
    val = 0.1    val = 1e-5    val = Rational(1, 3)
         │            │              │
         ▼            ▼              ▼
┌─────────────┐  ┌─────────────┐  ┌───────────────────┐
│ 情况 1:   │  │ 情况 2:     │  │ 情况 3:       │
│ Python float│  │ SymPy Float │  │ Rational/其他     │
│            │  │            │  │               │
│ val=0.1 是 │  │ val=1e-5 是 │  │ 需要 sympify  │
│ Python float│  │ SymPy Float│  │ 然后递归 evalf│
│            │  │            │  │               │
│ 直接返回:  │  │ 检查 Float  │  │ sympify(val)   │
│ (mpf(0.1),│  │ 实际精度:  │  │ = Rational(1, 3)│
│ None, 57, None│  │ max(57, 15) │  │               │
│            │  │ = 57? 不!   │  │ 递归调用:      │
│ ⚠️ 精度设为 │  │            │  │ evalf(1/3, 57) │
│ 请求的 57,   │  │ Float._as_ │  │               │
│ 但实际只有   │  │ mpf_op 使用 │  │ 计算  │
│ 15 位(双精)│  │ max(prec,  │  │ from_rational(1,3,57)│
│            │  │ self._prec)  │  │               │
│            │  │            │  │ 返回精确转换  │
│            │  │ 如果 Float  │  │ 的 mpf       │
│            │  │ 是 1e-5   │  │               │
│            │  │ (15 位),   │  │               │
│            │  │ max(57, 15)│  │               │
│            │  │ = 57? 不!   │  │               │
│            │  │            │  │               │
│            │  │ ⚠️ 实际只  │  │               │
│            │  │ 使用 15 位  │  │               │
│            │  │ 精度!        │  │               │
└─────────────┘  └─────────────┘  └───────────────────┘
```

### 2.3 三种 subs 值类型的精度处理

| subs 值类型 | 处理方式 | 实际精度 | 精度返回值 |
|-------------|---------|---------|------------|
| **Python float** (`0.1`) | 直接转换为 mpf | 固定 15 位（双精度） | 返回请求的 `prec`（⚠️ 与实际不符） |
| **SymPy Float** (`Float('1e-5', 10) | 检查 `max(prec, self._prec)` | 取两者中**较小值** | 返回 `max(prec, self._prec) |
| **Rational/Integer** | `sympify` 后递归 `evalf` | 精确转换，可任意精度 | 返回请求的 `prec` |
| **表达式** (`1+pi`) | 递归 `evalf` 完整求值 | 取决于子表达式精度 | 返回实际计算精度 |

### 2.4 evalf_subs 的特殊情况

**注意**：`evalf_subs` 函数只在 `evalf_piecewise` 中被调用！

**代码**（`sympy/core/evalf.py:1024-1032`）：
```python
def evalf_subs(prec: int, subs: dict) -> dict:
    """ Change all Float entries in `subs` to have precision prec. """
    newsubs = {}
    for a, b in subs.items():
        b = S(b)
        if b.is_Float:
            b = b._eval_evalf(prec)  # 重新标定 Float 的精度
        newsubs[a] = b
    return newsubs
```

**调用位置**（`sympy/core/evalf.py:1037-1046`）：
```python
def evalf_piecewise(expr: Expr, prec: int, options: OPT_DICT) -> TMP_RES:
    if 'subs' in options:
        expr = expr.subs(evalf_subs(prec, options['subs']))  # 唯一调用处
        newopts = options.copy()
        del newopts['subs']
        return evalf(expr, prec, newopts)
```

**关键发现**：
- `evalf_subs` 只在处理 `Piecewise` 表达式时才会被调用
- 对于普通 Symbol，使用 `evalf_symbol` 中的直接处理
- 这是两种不同的路径！

---

## 三、重复求值缓存机制

### 3.1 缓存的存储结构

**代码**（`sympy/core/evalf.py:1386-1393`）：

```python
def evalf_symbol(x: Expr, prec: int, options: OPT_DICT) -> TMP_RES:
    val = options['subs'][x]
    
    if not isinstance(val, mpf):
        # 缓存初始化
        if '_cache' not in options:
            options['_cache'] = {}
        cache = options['_cache']
        
        # 缓存查找
        cached, cached_prec = cache.get(x, (None, MINUS_INF))
        
        # 缓存命中条件
        if cached_prec >= prec:
            return cached
        
        # 缓存未命中，重新计算
        v = evalf(sympify(val), prec, options)
        
        # 更新缓存
        cache[x] = (v, prec)
        return v
```

### 3.2 缓存机制详解

**缓存结构**：
```python
options['_cache'] = {
    Symbol('x'): (result_tuple, cached_prec),
    Symbol('y'): ((mpf(0.00001), None, 53, None), 53),
    # ...
}
```

**缓存命中条件**：`cached_prec >= prec`

这意味着：

| 场景 | 第一次调用 | 第二次调用 | 缓存行为 |
|-----|-----------|-----------|---------|
| **低精度后高精度 | `evalf(x, 30)` | `evalf(x, 50)` | **缓存未命中**，重新计算 |
| **高精度后低精度** | `evalf(x, 50)` | `evalf(x, 30)` | **缓存命中**，使用缓存 |
| **相同精度** | `evalf(x, 30)` | `evalf(x, 30)` | **缓存命中** |

### 3.3 完整的缓存链路示例

```
表达式: sin(x) + cos(x), subs={x: 1 + pi}

第一次调用: expr.evalf(15)
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 计算 sin(x):                                                           │
│                                                                          │
│ x = 1 + pi (Symbol)                                                    │
│   │                                                                      │
│   ▼                                                                      │
│ evalf_symbol(x, 57 + 10, options)                                   │
│   │                                                                      │
│   ├─ cache = options.get('_cache', {}) = {}                            │
│   ├─ cached, cached_prec = cache.get(x, (None, -INF))                    │
│   └─ cached_prec(-INF) < prec(67)? Yes!                               │
│   │                                                                      │
│   ▼                                                                      │
│ evalf(sympify(1+pi), 67, options)                                        │
│   │                                                                      │
│   ├─ 计算 1 + pi ≈ 4.141592653589793...                                 │
│   └─ 返回 (mpf(4.141...), None, 67, None)                                │
│   │                                                                      │
│   ▼                                                                      │
│ cache[x] = ((mpf(4.141...), None, 67, None), 67)                      │
│                                                                          │
│ sin(4.14159...) ≈ -0.850919...                                         │
└─────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 计算 cos(x):                                                             │
│                                                                          │
│ x = 1 + pi (Symbol)                                                      │
│   │                                                                      │
│   ▼                                                                      │
│ evalf_symbol(x, 67, options)                                            │
│   │                                                                      │
│   ├─ cache = options['_cache']                                            │
│   ├─ 查找 x: (result_tuple, 67)                                            │
│   └─ cached_prec(67) >= prec(67)? Yes!                                 │
│   │                                                                      │
│   └─ 直接返回缓存的 (mpf(4.141...), None, 67, None)                    │
│                                                                          │
│ cos(4.14159...) ≈ -0.525322...                                       │
└─────────────────────────────────────────────────────────────────────────┘
         │
         ▼
         结果: -0.8509... + (-0.5253...) = -1.3762...

─────────────────────────────────────────────────────────────────────────
第二次调用: expr.evalf(30)  # 更低精度
         │
         └─ 此时如果 x 的 cached_prec = 67 >= 30 ✓
         └─ 使用缓存的 x 值
         └─ 但 sin/cos 可能需要重新计算（取决于具体实现）
```

### 3.4 缓存的作用域

**关键发现**：
- 缓存存储在 `options['_cache']` 中
- `options` 字典在**每次** `evalf()` 调用时创建
- 缓存**不会**在多次 `evalf()` 调用之间共享

```python
# 第一次调用
expr.evalf(15, subs={x: complex_expr})  # 创建新的 options，新的 _cache

# 第二次调用
expr.evalf(15, subs={x: complex_expr})  # 另一个 options，另一个 _cache
                                    # 缓存不会被复用！
```

**例外**：如果在同一次 `evalf()` 调用中，同一 Symbol 被多次引用，缓存会被复用。

---

## 四、默认参数与调高上限的行为差异

### 4.1 默认参数分析

**代码**（`sympy/core/evalf.py:1569-1690`）：

```python
def evalf(self, n=15, subs=None, maxn=100, chop=False, strict=False, quad=None, verbose=False):
    prec = dps_to_prec(n)  # 15 -> 53 bits
    options = {
        'maxprec': max(prec, int(maxn*LG10)),  # max(53, 100*3.3219) = 332
        'chop': chop,
        'strict': strict,
        'verbose': verbose
    }
    if subs is not None:
        options['subs'] = subs
    if quad is not None:
        options['quad'] = quad
    
    result = evalf(self, prec + 4, options)  # 初始 +4 保护位
```

**默认值**：

| 参数 | 默认值 | 二进制精度 | 说明 |
|-----|-------|---------|------|
| `n` | 15 | 53 bits | 目标精度 |
| `maxn` | 100 | ~332 bits | 最大临时精度上限 |
| `DEFAULT_MAXPREC` | - | 333 bits | 硬编码默认值 |

### 4.2 maxprec 的两个来源

**注意**：有两个不同的 `maxprec`！

1. **`options['maxprec']`**：来自 `maxn` 参数
   ```python
   options['maxprec'] = max(prec, int(maxn*LG10))
   ```

2. **`DEFAULT_MAXPREC`**：硬编码默认值
   ```python
   DEFAULT_MAXPREC = 333  # 约 100 位十进制
   ```

**它们的交互**（`sympy/core/evalf.py:593`）：
```python
def evalf_add(v: 'Add', prec: int, options: OPT_DICT) -> TMP_RES:
    oldmaxprec = options.get('maxprec', DEFAULT_MAXPREC)
    # ...
```

| 场景 | `options['maxprec'] | `oldmaxprec` |
|-----|------------------|--------------|
| 默认调用 | 332 (来自 maxn=100) | 332 |
| 未传入 maxprec | 未设置 | 333 (DEFAULT_MAXPREC) |
| maxn=200 | max(53, 200*3.3219) = 664 | 664 |

### 4.3 调高上限的影响

让我们对比默认值 vs 调高上限：

**场景 1：默认值 `maxn=100`**

```python
expr.evalf(15)  # 默认
```

| 参数 | 值 |
|-----|-----|
| 目标精度 `prec` | 53 bits |
| `options['maxprec']` | 332 bits |
| 实际最大 `prec`（加法） | ~53 + 332 = 385 bits |
| 保护位初始 | +4 bits |
| 子表达式保护位 | +10 bits |

**场景 2：调高上限 `maxn=1000`**

```python
expr.evalf(15, maxn=1000)
```

| 参数 | 值 |
|-----|-----|
| 目标精度 `prec` | 53 bits |
| `options['maxprec']` | max(53, 1000*3.3219) = 3321 bits |
| 实际最大 `prec`（加法） | ~53 + 3321 = 3374 bits |

### 4.4 精度提升序列对比

**默认 `maxn=100` (maxprec=332)**：

```
迭代 0: prec = 53,  options['maxprec'] = min(332, 106) = 106
迭代 1: prec = ~64, options['maxprec'] = min(332, ~128) = 128
迭代 2: prec = ~80, options['maxprec'] = min(332, 160) = 160
...
迭代 N: prec = 166, options['maxprec'] = min(332, 332) = 332  # 切换到 oldmaxprec
...
迭代 M: prec = 385, 检查 (385-53)=332 > 332? No. Yes! 退出
```

**调高 `maxn=1000` (maxprec=3321)**：

```
迭代 0: prec = 53,   options['maxprec'] = min(3321, 106) = 106
迭代 1: prec = ~64,  options['maxprec'] = min(3321, ~128) = 128
...
迭代 N: prec = 1661, options['maxprec'] = min(3321, 3322) = 3321  # 切换
...
迭代 M: prec = 3374, 检查 (3374-53)=3321 > 3321? No. Yes! 退出
```

### 4.5 调高上限的性能代价

| 对比项 | 默认 `maxn=100` | 调高 `maxn=1000` | 增加倍数 |
|-------|----------------|-------------------|---------|
| 最大 prec | ~385 bits | ~3374 bits | **~9x |
| 单次计算代价 | O(385) | O(3374) | **~9x** |
| 迭代次数 | ~8 次 | ~12 次 | **~1.5x** |
| **总代价估计** | O(385 × 8) | O(3374 × 12) | **~13x** |

**关键发现**：
- 调高 `maxn` 会显著增加最坏情况下的计算时间
- 对于"好"的表达式（精度足够），性能差异不大
- 对于"困难"的表达式（抵消、接近根），性能可能指数级增长

### 4.6 strict=True 的影响

**代码**（`sympy/core/evalf.py:1497-1516`）：

```python
def evalf(x: Expr, prec: int, options: OPT_DICT) -> TMP_RES:
    # ...
    if options.get("strict"):
        check_target(x, r, prec)
    
    return r
```

**`check_target`**（`sympy/core/evalf.py:263-283`）：

```python
def check_target(expr: Expr, result: TMP_RES, prec: int) -> None:
    if result is S.ComplexInfinity:
        return
    re, im, re_acc, im_acc = result
    if re and re_acc < prec:
        raise PrecisionExhausted(
            "Could not get %s accurate bits for real part of %s, "
            "only %s" % (prec, expr, re_acc))
    if im and im_acc < prec:
        raise PrecisionExhausted(
            "Could not get %s accurate bits for imaginary part of %s, "
            "only %s" % (prec, expr, im_acc))
```

**行为对比**：

| 场景 | `strict=False`（默认） | `strict=True` |
|-----|---------------------|----------------|
| 精度不足 | 返回低精度结果 | 抛出 `PrecisionExhausted` 异常 |
| 抵消导致精度损失 | 返回 `scaled_zero` 或低精度 | 抛出异常 |
| 浮点输入低精度 | 显示高精度（实际不准确） | 抛出异常 |

---

## 五、修正后的完整调用链路

### 5.1 完整链路追踪（修正版）

```
用户调用: expr.evalf(n=15, subs={x: 0.1, y: Rational(1, 3)}, maxn=100, strict=False)
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 阶段 1: EvalfMixin.evalf() - 入口点（修正）                             │
│                                                                          │
│ n=15 → prec = dps_to_prec(15) = 53 bits                                   │
│                                                                          │
│ options = {                                                               │
│     'maxprec': max(53, int(100*3.3219)) = 332,                        │
│     'chop': False,                                                        │
│     'strict': False,                                                      │
│     'verbose': False,                                                   │
│     'subs': {x: 0.1, y: Rational(1, 3)}                                 │
│ }                                                                        │
│                                                                          │
│ ⚠️ 关键修正：                                                            │
│ - 初始调用: evalf(self, prec + 4 = 57, options)                           │
│ - 这 4 位是入口点的保护位，不是子表达式的                             │
└─────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 阶段 2: evalf() - 核心分发器                                         │
│                                                                          │
│ 假设 expr 是 Add 类型:                                                │
│ rf = evalf_table[Add] = evalf_add                                        │
│ 调用 evalf_add(expr, 57, options)                                       │
└─────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 阶段 3: evalf_add() - 外层自适应循环（修正）                           │
│                                                                          │
│ oldmaxprec = options.get('maxprec', DEFAULT_MAXPREC) = 332           │
│ target_prec = prec = 57                                                    │
│                                                                          │
│ ════════════════════════════════════════════════════════════════════│
│ 第 1 次迭代:                                                           │
│ ════════════════════════════════════════════════════════════════════│
│                                                                          │
│ options['maxprec'] = min(332, 2*57) = 106  # 临时限制       │
│                                                                          │
│ ⚠️ 关键修正：                                                            │
│ - 此时 maxprec 被限制为 106                                             │
│ - 子表达式不能使用超过 106 位的精度                                    │
│                                                                          │
│ 递归求值子表达式:                                                          │
│ terms = [evalf(arg, 57 + 10 = 67, options) for arg in v.args]       │
│                                                                          │
│ 假设其中一个 arg 是 Symbol x (subs={x: 0.1})                           │
│   │                                                                      │
│   ▼                                                                      │
│   evalf_symbol(x, 67, options)                                           │
│   │                                                                      │
│   ├─ val = 0.1 (Python float)                                             │
│   ├─ isinstance(val, mpf)? Yes!                                           │
│   │                                                                      │
│   ⚠️ 关键修正：                                                            │
│   └─ 返回 (mpf(0.1), None, prec=67, None)                              │
│      │                                                                    │
│      ⚠️ 但实际精度只有 15 位（双精度）！                                     │
│      ⚠️ 返回的 prec 是请求的 67，不是实际精度！                             │
│                                                                          │
│ 合并计算:                                                                 │
│ re, re_acc = add_terms(...)                                             │
│ im, im_acc = add_terms(...)                                               │
│                                                                          │
│ acc = complex_accuracy((re, im, re_acc, im_acc))                          │
│                                                                          │
│ 检查: acc >= target_prec(57)?                                           │
│   ├─ Yes: 退出循环                                                        │
│   └─ No:  继续提升精度                                                     │
│                                                                          │
│ ════════════════════════════════════════════════════════════════════│
│ 如果精度不足（继续迭代）:                                                  │
│ ════════════════════════════════════════════════════════════════════│
│                                                                          │
│ 检查退出条件:                                                             │
│   (prec - target_prec) > options['maxprec']                            │
│   = (57 - 57) = 0 > 106? No!                                        │
│                                                                          │
│ ⚠️ 关键修正：                                                            │
│ - 此时退出条件永远不会满足！                                                │
│ - 因为 options['maxprec'] = 2*prec                                       │
│ - 条件等价于: -target_prec > prec → 永远 False                          │
│                                                                          │
│ 精度提升:                                                                 │
│ prec = prec + max(10 + 2**0 = 11, target_prec - acc)                 │
│      = 57 + 11 = 68                                                       │
│                                                                          │
│ i = 1                                                                    │
│                                                                          │
│ ════════════════════════════════════════════════════════════════════│
│ 第 2 次迭代:                                                           │
│ ════════════════════════════════════════════════════════════════════│
│                                                                          │
│ prec = 68                                                                │
│ options['maxprec'] = min(332, 2*68) = 136                            │
│                                                                          │
│ 重新计算所有子表达式！                                                       │
│ terms = [evalf(arg, 68 + 10 = 78, options) for arg in v.args]            │
│                                                                          │
│ ... 重复合并和精度检查 ...                                                  │
│                                                                          │
│ ════════════════════════════════════════════════════════════════════│
│ 迭代继续，直到:                                                           │
│ ════════════════════════════════════════════════════════════════════│
│                                                                          │
│ 当 prec >= 166 (因为 2*166 = 332 >= oldmaxprec=332)                │
│                                                                          │
│ 此时: options['maxprec'] = min(332, 2*166) = 332                  │
│                                                                          │
│ 退出条件检查: (prec - 57) > 332?                                       │
│   - 当 prec > 57 + 332 = 389 时退出                                   │
│                                                                          │
│ ════════════════════════════════════════════════════════════════════│
│ 循环结束:                                                                 │
│ ════════════════════════════════════════════════════════════════════│
│                                                                          │
│ options['maxprec'] = oldmaxprec = 332  # 恢复原始值                      │
│                                                                          │
│ ⚠️ 关键修正：                                                            │
│ - 检查结果是否为 0 但不是精确 0:                                         │
│   if iszero(re, scaled=True):                                            │
│       re = scaled_zero(re)                                                 │
│   if iszero(im, scaled=True):                                            │
│       im = scaled_zero(im)                                                 │
│                                                                          │
│ 返回: (re, im, re_acc, im_acc)                                          │
└─────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 阶段 4: 入口点收尾                                                    │
│                                                                          │
│ re, im, re_acc, im_acc = result                                        │
│                                                                          │
│ ⚠️ 关键修正：                                                            │
│ - 最终精度取 min(prec, re_acc)                                            │
│ - 但 re_acc 可能被高估（如 Float 的情况）                                   │
│                                                                          │
│ if re:                                                                   │
│     p = max(min(53, re_acc), 1)  # 目标精度是 53，不是 57！         │
│     re = Float._new(re, p)                                                │
│                                                                          │
│ ════════════════════════════════════════════════════════════════════│
│ 关键问题：                                                                 │
│ ════════════════════════════════════════════════════════════════════│
│                                                                          │
│ 1. 如果 subs 中的值是低精度 Float:                                          │
│    - evalf_float 返回 re_acc = prec（请求的精度）                           │
│    - 但实际精度只有 Float._prec                                            │
│    - 最终结果可能显示高精度，但实际不准确                                     │
│                                                                          │
│ 2. 如果发生抵消:                                                            │
│    - re = scaled_zero(re)                                                │
│    - re_acc 可能很低                                                      │
│    - p = max(min(53, low_value), 1) = 1                                  │
│    - 结果只有 1 位有效精度！                                                │
│                                                                          │
│ ═══════════════════════════════════════════════════════════════════════│
│ 返回: Float 或 re + im*I                                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 关键修正点汇总

| 之前的理解（错误） | 实际代码（正确） | 影响 |
|-----------------|-------------------|------|
| 退出条件检查"已经提升的精度量" | 当 `2*prec < oldmaxprec` 时退出条件**永远不满足** | 实际最大 prec 比预期大很多 |
| `evalf_float` 返回实际精度 | 返回**请求的精度**，不是实际精度 | 低精度 Float 可能显示高精度 |
| Float 精度会被提升 | `max(prec, self._prec)` 取**较大值**使用 | 精度永远不会提升 |
| `maxprec 是硬限制 | 分两阶段：先 `2*prec`，后 `oldmaxprec` | 退出条件复杂 |
| 缓存跨调用共享 | 缓存存储在 `options['_cache']`，每次调用新建 | 缓存只在同一次调用内共享 |
| `evalf_subs` 普遍使用 | 只在 `Piecewise` 中调用，Symbol 有单独处理 | 两条不同路径 |

---

## 六、性能代价与触发条件（修正版）

### 6.1 精度提升的完整代价

**加法的精度提升序列**（默认 `maxn=100`）：

```
目标: target_prec = 57 bits

迭代 0:
- prec = 57
- options['maxprec'] = min(332, 114) = 114
- 子表达式精度: 57 + 10 = 67
- 提升量(如果需要): max(11, 57 - acc)

迭代 1:
- prec = 57 + 11 = 68 (假设 acc=0)
- options['maxprec'] = min(332, 136) = 136
- 子表达式精度: 68 + 10 = 78
- 提升量: max(12, 57 - acc)

迭代 2:
- prec = 68 + 12 = 80
- options['maxprec'] = min(332, 160) = 160
- 子表达式精度: 80 + 10 = 90
- 提升量: max(14, 57 - acc)

...

迭代 N (prec >= 166):
- options['maxprec'] = min(332, 332) = 332 (切换到 oldmaxprec)
- 退出条件开始有效

迭代 M:
- prec = 57 + 332 = 389
- (389 - 57) = 332 > 332? No → Yes! 退出
```

### 6.2 各阶段的性能代价

| 阶段 | 操作 | 代价估计 | 触发条件 |
|-----|------|---------|---------|
| **入口点** | 精度转换、options 构建 | O(1) | 总是 |
| **加法迭代** | 每次迭代重新计算所有子表达式 | O(子表达式数 × 单次代价) | 精度不足 |
| **子表达式** | Symbol 查找 subs 值 | O(1) 或 O(递归) | 总是 |
| **Float 处理** | 直接返回（但精度可能被高估） | O(1) | 总是 |
| **Rational 处理** | from_rational 转换 | O(prec) | 总是 |
| **三角函数** | 接近根时内层循环 | O(gap × 子表达式代价) | 结果接近 0 |
| **缓存命中** | 直接返回缓存值 | O(1) | cached_prec >= prec |
| **缓存未命中** | 递归计算 + 存储 | O(递归代价) | cached_prec < prec |

### 6.3 最坏情况性能场景

| 场景 | 表达式示例 | 性能特征 | 精度提升量 |
|-----|-----------|---------|------------|
| **灾难性抵消** | `N(pi + 1e-1000 - pi, 15, maxn=1500)` | 需要 1000+ 位精度，多次迭代 | `gap = 3321 bits |
| **三角函数接近根** | `N(sin(pi * 10**100 + 1e-10), 15)` | 内层循环每次提升 `gap` | `gap = 33 bits |
| **低精度 Float 输入** | `N(x + y, subs={x: Float(0.1, 1), y: 0.0000001})` | 显示高精度但实际不准确 | 无提升（但结果错误） |
| **多次引用同一 Symbol** | `N(sin(x) + cos(x) + tan(x), subs={x: complex_expr})` | 缓存命中，只计算一次 x | 无额外代价 |
| **调高 maxn** | `N(complex_expr, maxn=1000)` | 最坏情况迭代次数增加 | 最大 prec 增加 10x |

### 6.4 调高 maxn 的风险

**默认 `maxn=100` vs 调高 `maxn=1000`**：

| 维度 | 默认 `maxn=100` | 调高 `maxn=1000` |
|-----|-----------------|-------------------|
| `options['maxprec']` | 332 bits | 3321 bits |
| 实际最大 prec | ~53 + 332 = 385 bits | ~53 + 3321 = 3374 bits |
| 切换到 oldmaxprec 的 prec | 166 bits | 1661 bits |
| 迭代次数（估计） | ~8 次 | ~12 次 |
| 单次计算复杂度 | O(385) | O(3374) |
| **最坏情况总代价** | O(385 × 8) = O(3080) | O(3374 × 12) = O(40488) |
| **慢多少** | 1x | **~13x** |

### 6.5 性能优化建议（修正版）

| 场景 | 建议 | 原因 |
|-----|------|------|
| 知道结果不会抵消 | 使用 `chop=True` 或合理的 `maxn` | 防止无限精度提升 |
| 使用低精度 Float | 意识到精度不会被提升 | 不要期望 `evalf_float 返回的 prec 可能被高估 |
| 同一 Symbol 多次引用 | 利用缓存自动生效 | 同一次调用内缓存命中 |
| 多次独立调用 | 缓存不共享 | 考虑手动缓存结果 |
| 精度要求高 | 安装 gmpy | 大幅加速大整数运算 |
| 怀疑精度不足 | 使用 `strict=True` | 及时发现精度问题，而不是返回错误结果 |
| 调高 `maxn` | 谨慎使用 | 最坏情况性能代价很高 |

---

## 七、关键代码位置速查（修正版）

### 7.1 核心函数

| 函数 | 文件位置 | 关键行为 |
|-----|---------|----------|
| `evalf_add` | `evalf.py:585-631` | 外层循环，`options['maxprec'] = min(oldmaxprec, 2*prec)` |
| `evalf_float` | `evalf.py:481-482` | **返回 `prec`（请求精度），不是实际精度 |
| `evalf_symbol` | `evalf.py:1379-1394` | subs 处理，缓存机制 |
| `evalf_subs` | `evalf.py:1024-1032` | 只在 `Piecewise 中调用 |
| `Float._as_mpf_op` | `numbers.py:436-438` | `prec = max(prec, self._prec)` **取较大值** |
| `check_target` | `evalf.py:263-283` | `strict=True` 时抛出异常 |

### 7.2 关键常量

| 常量 | 值 | 含义 |
|-----|---|------|
| `DEFAULT_MAXPREC` | 333 bits | 默认最大精度（约 100 位十进制） |
| 入口保护位 | +4 bits | `evalf(self, prec + 4, options)` |
| 加法子表达式保护位 | +10 bits | `evalf(arg, prec + 10, options)` |
| 三角函数初始保护位 | +20 bits | `xprec = prec + 20` |

### 7.3 关键条件判断

| 条件 | 代码位置 | 含义 |
|-----|---------|------|
| 退出条件（加法） | `evalf.py:618` | `(prec - target_prec) > options['maxprec']` |
| 临时 maxprec | `evalf.py:598` | `options['maxprec'] = min(oldmaxprec, 2*prec)` |
| 缓存命中 | `evalf.py:1390` | `cached_prec >= prec` |
| Float 精度 | `numbers.py:437` | `prec = max(prec, self._prec)` |

---

## 八、总结

### 8.1 最重要的修正结论

#### 结论 1：加法的退出条件有两阶段行为

```
阶段 1（prec 较小时）：
- `options['maxprec'] = min(oldmaxprec, 2*prec) = 2*prec
- 退出条件：`(prec - target_prec) > 2*prec`
- 等价于：`-target_prec > prec`
- **永远不会满足！**

阶段 2（prec 足够大时）：
- `options['maxprec'] = min(oldmaxprec, 2*prec) = oldmaxprec
- 退出条件开始有效
- 当 `prec > target_prec + oldmaxprec` 时退出

**实际最大 prec ≈ target_prec + oldmaxprec

默认值：53 + 333 = 386 bits（不是 333！）

#### 结论 2：evalf_float 返回的精度被高估

```python
def evalf_float(expr: 'Float', prec: int, options: OPT_DICT) -> TMP_RES:
    return expr._mpf_, None, prec, None  # ⚠️ 返回请求的 prec，不是实际精度！
```

- 返回的 `re_acc = prec` 是**请求的精度**，不是 Float 的实际精度
- 如果用户传入 `Float(.1, 1)`，即使请求 50 位精度
- 结果显示 50 位，但实际只有 1 位的准确性！

#### 结论 3：Float 的精度永远不会被提升

```python
def _as_mpf_op(self, prec):
    prec = max(prec, self._prec)  # ⚠️ 取较大值！
    return self._as_mpf_val(prec), prec
```

- 如果请求精度 < Float 实际精度：使用 Float 实际精度
- 如果请求精度 > Float 实际精度：**仍然使用 Float 实际精度**
- 精度不会被"提升"，只能保持或降低

#### 结论 4：缓存只在同一次调用内共享

```python
# 缓存存储在 options['_cache'] 中
# 每次 evalf() 调用创建新的 options

# 第一次调用
expr.evalf(15, subs={x: complex_expr})  # 新的 _cache

# 第二次调用
expr.evalf(15, subs={x: complex_expr})  # 另一个 _cache，不共享！
```

- 缓存只在**同一次** `evalf()` 调用内共享
- 多次独立调用之间缓存不共享

#### 结论 5：evalf_subs 只在 Piecewise 中使用

```python
def evalf_piecewise(expr: Expr, prec: int, options: OPT_DICT) -> TMP_RES:
    if 'subs' in options:
        expr = expr.subs(evalf_subs(prec, options['subs']))  # 唯一调用处
```

- `evalf_subs` 只处理 `Piecewise` 表达式
- 普通 Symbol 使用 `evalf_symbol` 中的直接处理
- 这是两条不同的路径！

### 8.2 调高 maxn 的影响总结

| 对比项 | 默认 `maxn=100` | 调高 `maxn=1000` |
|-------|-----------------|-------------------|
| `options['maxprec']` | 332 bits | 3321 bits |
| 实际最大 prec | ~385 bits | ~3374 bits |
| 切换到 oldmaxprec | prec >= 166 | prec >= 1661 |
| 最坏情况性能 | 基准 | **~13x 更慢 |
| 风险 | 精度可能不足 | 性能可能爆炸 |

### 8.3 最终建议

1. **谨慎调高 `maxn`**：
   - 默认 `maxn=100` 对大多数情况足够
   - 调高前确保确实需要，否则性能代价很高

2. **注意 Float 精度**：
   - 意识到 `evalf_float` 返回的精度可能被高估
   - 低精度 Float 输入不会通过 `evalf()` 提升精度
   - 使用 `strict=True` 可以及时发现精度问题

3. **理解缓存行为**：
   - 同一次调用内同一 Symbol 会被缓存
   - 多次独立调用缓存不共享

4. **subs 处理**：
   - Python float → 固定 15 位精度
   - SymPy Float → 取 `max(prec, self._prec)`
   - Rational/Integer → 精确转换，可任意精度
