# SymPy 多项式统一失败处理方式精炼对照

## 1. 核心属性澄清

| 类 | 继承关系 | `_op_priority` | `is_Poly` |
|------|---------|---------------|-----------|
| `Poly` | `Poly(Basic)` | `10.001` | `True` |
| `Expr` | `Expr(Basic)` | `10.0` | `False`（继承自 `Basic`） |
| `Basic` | 基类 | 未定义 | `False` |

**关键行为**：
- `_sympifyit` 装饰器：只有当 `other` **没有** `_op_priority` 时，才会尝试 `sympify(other, strict=True)`
- `Poly` 和 `Expr` 都有 `_op_priority`，所以当比较 `Poly` 与 `Poly` 或 `Expr` 时，**不会**进入 `sympify` 分支

---

## 2. 三类调用场景对照总表

### 2.1 相等性判断（`__eq__`）

| 场景编号 | 触发条件 | 处理动作 | 对外可见结果 | 设计动机 |
|---------|---------|---------|-------------|---------|
| **A1** | `other` 有 `_op_priority`（是 `Poly` 或 `Expr`），且 `other.is_Poly = True`，且生成器数量相同，且域统一成功 | 比较内部表示 | 返回 `True` 或 `False` | 直接比较两个多项式的内部结构 |
| **A2** | `other` 有 `_op_priority`，且 `other.is_Poly = True`，但生成器数量不同 | 直接返回 `False` | 返回 `False` | 生成器不同的多项式"不相等" |
| **A3** | `other` 有 `_op_priority`，且 `other.is_Poly = True`，但域统一失败（抛出 `UnificationFailed`） | 捕获异常，返回 `False` | 返回 `False` | 域不兼容的多项式"不相等" |
| **A4** | `other` 有 `_op_priority`，且 `other.is_Poly = False`（是 `Expr` 但不是 `Poly`），尝试转换为 `Poly` 成功 | 转换后比较 | 返回 `True` 或 `False` | 尝试将 `Expr` 转换为 `Poly` 再比较 |
| **A5** | `other` 有 `_op_priority`，且 `other.is_Poly = False`，尝试转换为 `Poly` 失败（抛出 `PolynomialError`, `DomainError`, `CoercionFailed`） | 捕获异常，返回 `False` | 返回 `False` | 无法转换为 `Poly` 的表达式"不相等" |
| **A6** | `other` **没有** `_op_priority`，且 `sympify(other, strict=True)` 成功 | 调用 `__eq__` 方法（进入 A1-A5 分支） | 返回 `True` 或 `False` | 将非 SymPy 对象转换为 SymPy 对象再比较 |
| **A7** | `other` **没有** `_op_priority`，且 `sympify(other, strict=True)` 失败（抛出 `SympifyError`） | 返回 `NotImplemented` | Python 尝试 `other.__eq__(self)`，若都返回 `NotImplemented` 则最终结果为 `False` | 交由对端处理，符合 Python 协议 |

### 2.2 算术运算（`__add__`, `__sub__`, `__mul__` 等）

| 场景编号 | 触发条件 | 处理动作 | 对外可见结果 | 设计动机 |
|---------|---------|---------|-------------|---------|
| **B1** | `other` 是 `Poly`（`isinstance(other, Poly)`），且域/级别统一成功 | 执行运算 | 返回运算结果（`Poly` 类型） | 直接执行多项式运算 |
| **B2** | `other` 是 `Poly`，但域/级别不匹配（抛出 `UnificationFailed`） | 不捕获异常，直接传播 | 抛出 `UnificationFailed` | 域不兼容时明确失败，不静默降级 |
| **B3** | `other` 是 `Integer`（`isinstance(other, Integer)`） | 转换为 `Poly` 再运算 | 返回运算结果（`Poly` 类型） | `Integer` 可以安全转换为 `Poly` |
| **B4** | `other` 是 `Expr` 但不是 `Poly`/`Integer`，尝试转换为 `Poly` 成功 | 转换后执行运算 | 返回运算结果（`Poly` 类型） | 尝试将 `Expr` 转换为 `Poly` 再运算 |
| **B5** | `other` 是 `Expr` 但不是 `Poly`/`Integer`，尝试转换为 `Poly` 失败，且 `other.is_Matrix = True` | 返回 `NotImplemented` | Python 尝试 `other.__radd__(self)` 等，若都返回 `NotImplemented` 则抛出 `TypeError` | `Matrix` 可能知道如何处理 `Poly` |
| **B6** | `other` 是 `Expr` 但不是 `Poly`/`Integer`，尝试转换为 `Poly` 失败，且 `other.is_Matrix = False` | 降级到 `Expr` 层面运算（`f.as_expr() + other`），发出 `DeprecationWarning` | 返回 `Expr` 类型结果（带警告） | 历史兼容性（已废弃，建议显式转换） |
| **B7** | `other` 不是 `Poly`/`Integer`/`Expr` | 返回 `NotImplemented` | Python 尝试 `other.__radd__(self)` 等，若都返回 `NotImplemented` 则抛出 `TypeError` | 交由对端处理，符合 Python 协议 |

### 2.3 构造流程（`poly_from_expr`, `parallel_poly_from_expr`）

| 场景编号 | 触发条件 | 处理动作 | 对外可见结果 | 设计动机 |
|---------|---------|---------|-------------|---------|
| **C1** | `expr` 是 `Basic` 类型，且能提取单项式结构，且域选择成功 | 构造 `Poly` | 返回 `(poly, opt)` | 精确构造多项式 |
| **C2** | `expr` 不是 `Basic` 类型 | 抛出 `PolificationFailed` | 抛出 `PolificationFailed` | 非 `Basic` 类型无法构造 |
| **C3** | `expr` 无法提取单项式结构（如 `sin(x)`） | `_dict_from_expr` 抛出异常，不捕获 | 抛出 `PolynomialError` 或其他异常 | 非多项式表达式无法构造 |
| **C4** | 没有生成器（`opt.gens` 为空） | 抛出 `PolificationFailed` | 抛出 `PolificationFailed` | 多项式需要生成器 |
| **C5** | 用户指定域，且系数无法在该域表示（`domain.from_sympy` 失败） | 不捕获异常，直接传播 | 抛出 `CoercionFailed` | 用户指定的域必须能表示所有系数 |
| **C6** | 自动选择域，且系数无法在精确域表示 | `construct_domain` 内部降级到 `EX` 域 | 成功返回（`domain = EX`） | `EX` 域作为最终 fallback |
| **C7** | 并行构造两个 `Poly`，且域统一失败 | 不捕获异常，直接传播 | 抛出 `UnificationFailed` | 并行构造要求域兼容 |

---

## 3. 三类调用方式核心差异对照

### 3.1 处理方式总览

| 维度 | 相等性判断（`__eq__`） | 算术运算（`__add__` 等） | 构造流程（`poly_from_expr`） |
|------|----------------------|-------------------------|-----------------------------|
| **返回布尔结论** | ✅ 有（场景 A2, A3, A5） | ❌ 无 | ❌ 无 |
| **交由对端处理** | ✅ 有（场景 A7） | ✅ 有（场景 B5, B7） | ❌ 无（无对端概念） |
| **直接失败终止** | ❌ 无（所有异常都被捕获或返回 `NotImplemented`） | ✅ 有（场景 B2） | ✅ 有（场景 C2, C3, C4, C5, C7） |
| **内部降级** | ❌ 无 | ⚠️ 有（场景 B6，已废弃） | ✅ 有（场景 C6，`construct_domain` 降级到 `EX`） |

### 3.2 异常处理策略对照

| 异常类型 | 相等性判断 | 算术运算 | 构造流程 |
|---------|-----------|---------|---------|
| **`UnificationFailed`** | 捕获 → 返回 `False`（A3） | 不捕获 → 传播（B2） | 不捕获 → 传播（C7） |
| **`CoercionFailed`** | 捕获 → 返回 `False`（A5） | 不适用 | 不捕获 → 传播（C5） |
| **`PolynomialError`** | 捕获 → 返回 `False`（A5） | 捕获 → 降级或 `NotImplemented`（B5, B6） | 不捕获 → 传播（C3） |
| **`DomainError`** | 捕获 → 返回 `False`（A5） | 不适用 | 不适用 |
| **`SympifyError`** | 捕获（装饰器）→ 返回 `NotImplemented`（A7） | 不适用（`_polifyit` 使用 `_sympify`） | 不适用 |
| **`PolificationFailed`** | 不适用 | 不适用 | 主动抛出（C2, C4） |

### 3.3 设计哲学对照

| 设计维度 | 相等性判断 | 算术运算 | 构造流程 |
|---------|-----------|---------|---------|
| **核心目标** | 给出确定的比较结果，或交由对端处理 | 执行运算、明确失败、或交由对端处理 | 精确构造或明确失败 |
| **容错程度** | 最高（从不抛异常给用户，要么返回 `False`，要么返回 `NotImplemented`） | 中等（类型不支持时返回 `NotImplemented`，域不兼容时抛异常） | 最低（严格报错，仅 `construct_domain` 内部降级） |
| **Python 协议遵循** | 严格遵循（`__eq__` 返回布尔值或 `NotImplemented`） | 严格遵循（返回 `NotImplemented` 或抛异常） | 不适用（不是二元运算） |
| **异常处理** | 所有异常都被捕获（要么返回 `False`，要么返回 `NotImplemented`） | 选择性捕获（类型问题返回 `NotImplemented`，域问题抛异常） | 几乎不捕获（让用户知道失败原因） |

---

## 4. 关键场景代码证据

### 4.1 相等性判断

**场景 A3：域统一失败返回 `False`**
```python
# polytools.py:4709-4713
if f.rep.dom != g.rep.dom:
    try:
        dom = f.rep.dom.unify(g.rep.dom, f.gens)
    except UnificationFailed:
        return False  # ← 捕获异常，返回 False
```

**场景 A7：`sympify` 失败返回 `NotImplemented`**
```python
# decorators.py:70-78
@wraps(func)
def __sympifyit_wrapper(a, b):
    try:
        if not hasattr(b, '_op_priority'):
            b = sympify(b, strict=True)
        return func(a, b)
    except SympifyError:
        return retval  # retval = NotImplemented 对于 __eq__
```

### 4.2 算术运算

**场景 B2：域不匹配抛出 `UnificationFailed`**
```python
# polyclasses.py:328-337
def unify_DMP(f, g: DMP[Es]) -> tuple[DMP[Et], DMP[Et]]:
    if not isinstance(g, DMP) or f.lev != g.lev:
        raise UnificationFailed("Cannot unify %s with %s" % (f, g))  # ← 不捕获
    # ...
    dom: Domain[Et] = f.dom.unify(g.dom)  # ← 可能抛出 UnificationFailed，不捕获
```

**场景 B5, B7：返回 `NotImplemented`**
```python
# polytools.py:85-86
if g.is_Matrix:
    return NotImplemented  # ← 交由对端处理

# polytools.py:103-104
else:
    return NotImplemented  # ← 交由对端处理
```

### 4.3 构造流程

**场景 C2, C4：主动抛出 `PolificationFailed`**
```python
# polytools.py:4769-4770
if not isinstance(expr, Basic):
    raise PolificationFailed(opt, orig, expr)  # ← 主动抛出

# polytools.py:4785-4786
if not opt.gens:
    raise PolificationFailed(opt, orig, expr)  # ← 主动抛出
```

**场景 C6：`construct_domain` 内部降级到 `EX`**
```python
# constructor.py:364-380
result = _construct_simple(coeffs, opt)

if result is not None:
    if result is not False:
        domain, coeffs = result
    else:
        domain, coeffs = _construct_expression(coeffs, opt)  # ← 降级到 EX
else:
    # ...
    if result is not None:
        domain, coeffs = result
    else:
        domain, coeffs = _construct_expression(coeffs, opt)  # ← 降级到 EX
```

---

## 5. 执行流程图精炼

### 5.1 相等性判断（`__eq__`）

```
p == other
    │
    ├─────────────────────────────────────────────────────────────┐
    │ 装饰器检查：hasattr(other, '_op_priority')                    │
    └─────────────────────────────────────────────────────────────┘
    │
    ├── Yes（Poly 或 Expr）──────────────────────────────────────┐
    │              │                                               │
    │              ▼                                               │
    │    ┌─────────────────────────────────────────────────────┐  │
    │    │ 方法内部：检查 other.is_Poly                          │  │
    │    └─────────────────────────────────────────────────────┘  │
    │              │                                               │
    │         ┌────┴────┐                                          │
    │         │         │                                          │
    │       True       False                                        │
    │         │         │                                          │
    │         ▼         ▼                                          │
    │    继续比较   尝试转换为 Poly                                 │
    │         │              │                                     │
    │         │         ┌────┴────┐                               │
    │         │         │         │                               │
    │         │       成功      失败                               │
    │         │         │         │                               │
    │         │         ▼         ▼                               │
    │         │    继续比较   返回 False（布尔结论）               │
    │         │                                                   │
    │         ▼                                                   │
    │    检查生成器、域统一等                                      │
    │         │                                                   │
    │    ┌────┴────┐                                              │
    │    │         │                                              │
    │  成功      失败                                              │
    │    │         │                                              │
    │    ▼         ▼                                              │
    │ True/False  返回 False（布尔结论）                          │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
    │
    └── No（没有 _op_priority）──────────────────────────────────┐
                   │                                               │
                   ▼                                               │
              尝试 sympify(other, strict=True)                    │
                   │                                               │
              ┌────┴────┐                                         │
              │         │                                         │
            成功      失败                                         │
              │         │                                         │
              ▼         ▼                                         │
         进入方法   返回 NotImplemented（交由对端处理）            │
         内部                                                         │
                                                                    │
         （同 Yes 分支）                                            │
```

### 5.2 算术运算（`__add__` 等）

```
p + other
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 装饰器处理：_sympify(other) 并检查类型                           │
└─────────────────────────────────────────────────────────────────┘
    │
    ├── isinstance(other, Poly) ──▶ 是 ──▶ 调用 add(p, other)
    │                                              │
    │                                              ▼
    │                                    调用 p.unify_DMP(other)
    │                                              │
    │                                         ┌────┴────┐
    │                                         │         │
    │                                       成功      失败
    │                                         │         │
    │                                         ▼         ▼
    │                                    执行加法   抛出 UnificationFailed
    │                                    返回结果   （直接失败终止）
    │
    ├── isinstance(other, Integer) ──▶ 是 ──▶ 转换为 Poly，执行加法
    │
    ├── isinstance(other, Expr) ──▶ 是 ──▶ 尝试转换为 Poly
    │                                        │
    │                                   ┌────┴────┐
    │                                   │         │
    │                                 成功      失败
    │                                   │         │
    │                                   ▼         ▼
    │                              执行加法   检查 other.is_Matrix
    │                                         │
    │                                    ┌────┴────┐
    │                                    │         │
    │                                   Yes       No
    │                                    │         │
    │                                    ▼         ▼
    │                              return     降级到 Expr 运算
    │                              NotImple-  （已废弃，带警告）
    │                              mented        │
    │                                            ▼
    │                                      返回 Expr 结果
    │
    └── 其他类型 ──▶ 返回 NotImplemented（交由对端处理）
```

### 5.3 构造流程（`poly_from_expr`）

```
poly_from_expr(expr, *gens, **args)
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 检查 isinstance(expr, Basic)                                      │
└─────────────────────────────────────────────────────────────────┘
    │
    ├── No ──▶ 抛出 PolificationFailed（直接失败终止）
    │
    └── Yes ──▶ 继续
              │
              ▼
    ┌─────────────────────────────────────────────────────────────┐
    │ 检查 expr.is_Poly                                            │
    └─────────────────────────────────────────────────────────────┘
              │
              ├── Yes ──▶ 从 Poly 转换，返回结果
              │
              └── No ──▶ 继续
                        │
                        ▼
    ┌─────────────────────────────────────────────────────────────┐
    │ 调用 _dict_from_expr(expr, opt)                               │
    │ （可能抛出 PolynomialError）                                   │
    └─────────────────────────────────────────────────────────────┘
                        │
                   可能抛出异常
                        │
                        ▼
    ┌─────────────────────────────────────────────────────────────┐
    │ 检查 opt.gens 是否为空                                        │
    └─────────────────────────────────────────────────────────────┘
                        │
                        ├── Yes ──▶ 抛出 PolificationFailed（直接失败终止）
                        │
                        └── No ──▶ 继续
                                  │
                                  ▼
    ┌─────────────────────────────────────────────────────────────┐
    │ 检查 opt.domain 是否为 None                                    │
    └─────────────────────────────────────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                   Yes                         No
                    │                           │
                    ▼                           ▼
    ┌───────────────────────┐      ┌───────────────────────┐
    │ construct_domain()    │      │ 用户指定域             │
    │ （内部会降级到 EX）    │      │ domain.from_sympy()   │
    └───────────────────────┘      │ （可能抛出 CoercionFailed）│
                    │               └───────────────────────┘
                    │                           │
                    ▼                           │
              继续构造 Poly              可能抛出异常
                    │                           │
                    ▼                           ▼
              返回 (poly, opt)           直接失败终止
```

---

## 6. 关键结论与修正

### 6.1 之前分析的修正

| 之前的结论 | 修正后的结论 | 证据 |
|-----------|-------------|------|
| "相等性判断总是返回 `False`" | "相等性判断有两种结果：方法内部返回 `False`，装饰器在 `sympify` 失败时返回 `NotImplemented`" | `decorators.py:70-78` → 装饰器捕获 `SympifyError` 返回 `NotImplemented` |
| "算术运算统一失败时降级" | "算术运算统一失败时抛出 `UnificationFailed`，不会降级；只有类型问题返回 `NotImplemented`" | `polyclasses.py:328-337` → `unify_DMP` 抛出 `UnificationFailed`，未被捕获 |
| "构造流程严格报错" | "构造流程严格报错，但 `construct_domain` 会在内部降级到 `EX`；如果表达式无法提取单项式结构则抛异常" | `constructor.py:364-380` → `construct_domain` 会降级到 `EX`，不会抛异常 |

### 6.2 核心差异总结

| 调用方式 | 对"统一失败"的定义 | 处理方式 | 设计动机 |
|---------|-------------------|---------|---------|
| **相等性判断** | 域不兼容、类型不兼容、生成器不匹配 | 返回 `False` 或 `NotImplemented` | 比较应该稳定，要么给出结论，要么交由对端处理 |
| **算术运算** | 域/级别不匹配 | 抛出 `UnificationFailed` | 运算失败应该明确告知，不静默降级 |
| **构造流程** | 表达式非多项式、用户指定域不兼容 | 抛出异常（但 `construct_domain` 内部降级到 `EX`） | 构造失败应该明确告知，但 `EX` 域作为最终 fallback |

### 6.3 语义一致性保证

四列在所有场景下的语义定义：

| 列名 | 语义定义 |
|------|---------|
| **触发条件** | 什么具体情况下会进入这个场景（代码条件判断） |
| **处理动作** | 代码实际执行了什么操作（捕获异常、返回值、抛出异常等） |
| **对外可见结果** | 用户实际观察到的行为（返回值、异常类型、Python 后续行为） |
| **设计动机** | 为什么这样设计（Python 协议、数学语义、用户体验等） |
