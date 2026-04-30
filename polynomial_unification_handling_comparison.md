# SymPy 多项式统一失败处理方式严谨对照分析

## 1. 核心概念澄清

### 1.1 三种处理方式的精确定义

| 处理方式 | 精确定义 | Python 语义 | 典型场景 |
|---------|---------|------------|---------|
| **返回布尔结论** | 方法内部捕获异常并返回 `True`/`False` | 直接给出比较结果，不交给对端 | 相等性判断（`__eq__`） |
| **交由对端处理** | 返回 `NotImplemented` 特殊值 | Python 尝试调用反向方法（`other.__eq__(self)`），若都返回 `NotImplemented` 则最终比较结果为 `False` | 类型不支持的运算 |
| **直接失败终止** | 抛出异常（`UnificationFailed`, `CoercionFailed` 等） | 程序中断，需要调用方捕获处理 | 构造流程、域不兼容的算术运算 |

### 1.2 关键属性确认

| 类 | `_op_priority` | `is_Poly` | 说明 |
|------|--------------|-----------|------|
| `Poly` | `10.001` | `True` | 有 `_op_priority`，`is_Poly=True` |
| `Expr` | `10.0` | `False` | 有 `_op_priority`，`is_Poly=False`（继承自 `Basic`） |
| `Basic` | 未定义 | `False` | 基类，无 `_op_priority` |

**关键发现**：
- `Poly` 和 `Expr` 都有 `_op_priority` 属性
- 当 `other` 有 `_op_priority` 时，`_sympifyit` 装饰器**不会**进行 `sympify`，直接调用方法
- 只有当 `other` 没有 `_op_priority` 时，才会尝试 `sympify(other, strict=True)`

---

## 2. 相等性判断（`__eq__`）详细分析

### 2.1 代码位置与结构

**文件位置**：
- 装饰器：`sympy/core/decorators.py:23-80` (`_sympifyit`)
- 方法：`sympy/polys/polytools.py:4696-4718` (`Poly.__eq__`)

**装饰器代码**（`decorators.py:68-80`）：
```python
if retval is not None:
    @wraps(func)
    def __sympifyit_wrapper(a, b):
        try:
            # If an external class has _op_priority, it knows how to deal
            # with SymPy objects. Otherwise, it must be converted.
            if not hasattr(b, '_op_priority'):
                b = sympify(b, strict=True)
            return func(a, b)
        except SympifyError:
            return retval
```

**方法代码**（`polytools.py:4696-4718`）：
```python
@_sympifyit('other', NotImplemented)
def __eq__(self, other):
    f, g = self, other
    
    # 情况1：other 不是 Poly
    if not g.is_Poly:
        try:
            g = f.__class__(g, f.gens, domain=f.get_domain())
        except (PolynomialError, DomainError, CoercionFailed):
            return False  # ← 布尔结论
    
    # 情况2：生成器数量不同
    if len(f.gens) != len(g.gens):
        return False  # ← 布尔结论
    
    # 情况3：域不同，尝试统一
    if f.rep.dom != g.rep.dom:
        try:
            dom = f.rep.dom.unify(g.rep.dom, f.gens)
        except UnificationFailed:
            return False  # ← 布尔结论
        
        f = f.set_domain(dom)
        g = g.set_domain(dom)
    
    # 情况4：比较内部表示
    return f.rep == g.rep  # ← 布尔结论
```

### 2.2 执行流程决策树

```
Poly.__eq__(self, other)
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 第一层：_sympifyit 装饰器                                                  │
│ 装饰器参数：retval = NotImplemented                                        │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 检查 other 是否有 _op_priority                                             │
│    if not hasattr(b, '_op_priority'):                                     │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ├── Yes ──▶ other 是 Poly 或 Expr（有 _op_priority）
    │              │
    │              ▼
    │         直接调用 __eq__(self, other)
    │              │
    │              ▼
    │         ┌─────────────────────────────────────────────────────────────┐
    │         │ 第二层：__eq__ 方法内部                                        │
    │         └─────────────────────────────────────────────────────────────┘
    │              │
    │              ▼
    │         ┌─────────────────────────────────────────────────────────────┐
    │         │ 检查 g.is_Poly（g 是转换后的 other）                         │
    │         │    if not g.is_Poly:                                         │
    │         └─────────────────────────────────────────────────────────────┘
    │              │
    │         ┌────┴────┐
    │         │         │
    │       True       False
    │         │         │
    │         ▼         ▼
    │    g 是 Poly   尝试转换 g = f.__class__(g, f.gens, domain=f.get_domain())
    │         │              │
    │         │         ┌────┴────┐
    │         │         │         │
    │         │       成功      失败
    │         │         │         │
    │         │         ▼         ▼
    │         │    继续比较   返回 False（布尔结论）
    │         │              (PolynomialError, DomainError, CoercionFailed)
    │         │
    │         ▼
    │    ┌─────────────────────────────────────────────────────────────┐
    │    │ 检查生成器数量：if len(f.gens) != len(g.gens)               │
    │    └─────────────────────────────────────────────────────────────┘
    │         │
    │    ┌────┴────┐
    │    │         │
    │   不同      相同
    │    │         │
    │    ▼         ▼
    │ 返回 False  继续
    │ (布尔结论)   │
    │              ▼
    │         ┌─────────────────────────────────────────────────────────────┐
    │         │ 检查域：if f.rep.dom != g.rep.dom                           │
    │         └─────────────────────────────────────────────────────────────┘
    │              │
    │         ┌────┴────┐
    │         │         │
    │       不同      相同
    │         │         │
    │         ▼         ▼
    │    尝试统一域   继续比较
    │         │
    │    ┌────┴────┐
    │    │         │
    │  成功      失败
    │    │         │
    │    ▼         ▼
    │ 转换后比较  返回 False（布尔结论）
    │              (UnificationFailed)
    │
    └── No ──▶ other 没有 _op_priority（不是 Poly 或 Expr）
                   │
                   ▼
              尝试 sympify(other, strict=True)
                   │
              ┌────┴────┐
              │         │
            成功      失败
              │         │
              ▼         ▼
         调用 __eq__   返回 NotImplemented（交由对端处理）
              │         (SympifyError)
              ▼
         进入第二层方法内部
         （同 Yes 分支）
```

### 2.3 场景分类与证据

#### 场景 A：返回布尔结论（`False`）

**触发条件**：
1. `other` 有 `_op_priority`（是 Poly 或 Expr），且：
   - `other.is_Poly` 为 `False` 且无法转换为 Poly（抛出 `PolynomialError`, `DomainError`, `CoercionFailed`）
   - 生成器数量不同
   - 域统一失败（抛出 `UnificationFailed`）

2. `other` 没有 `_op_priority`，但 `sympify` 成功，且方法内部返回 `False`

**代码证据**：
```python
# polytools.py:4700-4704
if not g.is_Poly:
    try:
        g = f.__class__(g, f.gens, domain=f.get_domain())
    except (PolynomialError, DomainError, CoercionFailed):
        return False  # ← 布尔结论

# polytools.py:4706-4707
if len(f.gens) != len(g.gens):
    return False  # ← 布尔结论

# polytools.py:4709-4713
if f.rep.dom != g.rep.dom:
    try:
        dom = f.rep.dom.unify(g.rep.dom, f.gens)
    except UnificationFailed:
        return False  # ← 布尔结论
```

**典型示例**：
```python
from sympy import Poly, GF
from sympy.abc import x, y

# 示例1：域无法统一
p1 = Poly(x**2 + 1, x, domain=GF(3))
p2 = Poly(x**2 + 1, x, domain=ZZ)
result = p1 == p2
# 结果：False（布尔结论）
# 证据：polytools.py:4712-4713 → 捕获 UnificationFailed，返回 False

# 示例2：生成器数量不同
p1 = Poly(x**2 + 1, x)          # gens = (x,)
p2 = Poly(x + y, x, y)          # gens = (x, y)
result = p1 == p2
# 结果：False（布尔结论）
# 证据：polytools.py:4706-4707 → len(f.gens) != len(g.gens)，返回 False

# 示例3：Expr 无法转换为 Poly
from sympy import sin
p = Poly(x**2 + 1, x)
expr = sin(x)  # is_Poly = False（Expr 实例）
result = p == expr
# 结果：False（布尔结论）
# 证据：polytools.py:4700-4704 → g.is_Poly = False，尝试转换失败，返回 False
```

#### 场景 B：交由对端处理（返回 `NotImplemented`）

**触发条件**：
- `other` 没有 `_op_priority`，且 `sympify(other, strict=True)` 抛出 `SympifyError`

**代码证据**：
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

**`SympifyError` 触发条件**（来自 `sympify.py`）：
```python
# sympify.py:403-407
if is_sympy is not None:
    if not strict:
        return a
    else:
        raise SympifyError(a)  # strict=True 时，已有的 SymPy 对象抛出

# sympify.py:409-410
if isinstance(a, CantSympify):
    raise SympifyError(a)  # CantSympify 类型
```

**关键分析**：
- `Poly` 和 `Expr` 都有 `_op_priority`，所以当比较 `Poly` 与 `Poly` 或 `Expr` 时，**不会**进入 `sympify` 分支
- 只有当 `other` 没有 `_op_priority` 时，才会尝试 `sympify`
- `sympify(other, strict=True)` 抛出 `SympifyError` 的场景：
  - `other` 已经是 SymPy 对象且 `strict=True`（但有 `_op_priority` 的 SymPy 对象不会进入此分支）
  - `other` 是 `CantSympify` 类型
  - 没有定义转换器的类型

**典型示例**：
```python
from sympy import Poly
from sympy.core.sympify import CantSympify
from sympy.abc import x

# 示例：CantSympify 类型
class MyClass(CantSympify):
    pass

p = Poly(x**2 + 1, x)
obj = MyClass()

# p == obj 的执行流程：
# 1. _sympifyit 装饰器检查 hasattr(obj, '_op_priority') → False
# 2. 尝试 sympify(obj, strict=True)
# 3. isinstance(obj, CantSympify) → True
# 4. 抛出 SympifyError
# 5. 装饰器捕获 SympifyError，返回 retval = NotImplemented
# 6. Python 尝试 obj.__eq__(p)（如果定义了）
# 7. 如果都返回 NotImplemented，Python 最终比较结果为 False
```

#### 场景 C：直接失败终止（抛出异常）

**关键发现**：`Poly.__eq__` **不会**抛出异常！所有异常都被捕获并返回 `False`。

**代码证据**：
```python
# polytools.py:4696-4718
@_sympifyit('other', NotImplemented)
def __eq__(self, other):
    # ... 所有异常都被捕获 ...
    
    # 捕获 PolynomialError, DomainError, CoercionFailed → 返回 False
    except (PolynomialError, DomainError, CoercionFailed):
        return False
    
    # ...
    
    # 捕获 UnificationFailed → 返回 False
    except UnificationFailed:
        return False
    
    # 装饰器捕获 SympifyError → 返回 NotImplemented
```

**结论**：`Poly.__eq__` 只有两种结果：
1. 返回 `True`/`False`（布尔结论）
2. 返回 `NotImplemented`（交由对端处理）

**不会**抛出异常让程序终止。

---

## 3. 算术运算详细分析

### 3.1 代码位置与结构

**文件位置**：
- 装饰器：`sympy/polys/polytools.py:72-105` (`_polifyit`)
- 核心方法：`sympy/polys/polyclasses.py:549-612` (`add`, `sub`, `mul` 等)
- 统一方法：`sympy/polys/polyclasses.py:328-337` (`unify_DMP`)

**装饰器代码**（`polytools.py:72-105`）：
```python
def _polifyit(func):
    @wraps(func)
    def wrapper(f, g):
        g = _sympify(g)
        
        # 情况1：g 已经是 Poly
        if isinstance(g, Poly):
            return func(f, g)
        
        # 情况2：g 是 Integer
        elif isinstance(g, Integer):
            g = f.from_expr(g, *f.gens, domain=f.domain)
            return func(f, g)
        
        # 情况3：g 是 Expr
        elif isinstance(g, Expr):
            try:
                # 尝试将 g 转换为 Poly
                g = f.from_expr(g, *f.gens)
            except PolynomialError:
                # 转换失败
                if g.is_Matrix:
                    # Matrix：返回 NotImplemented
                    return NotImplemented
                
                # 其他 Expr：降级到 Expr 层面运算（已废弃）
                expr_method = getattr(f.as_expr(), func.__name__)
                result = expr_method(g)
                
                if result is not NotImplemented:
                    sympy_deprecation_warning(
                        """
                        Mixing Poly with non-polynomial expressions in binary
                        operations is deprecated...
                        """,
                        deprecated_since_version="1.6",
                        active_deprecations_target="deprecated-poly-nonpoly-binary-operations",
                    )
                return result
            else:
                # 转换成功：执行运算
                return func(f, g)
        
        # 情况4：其他类型
        else:
            return NotImplemented
    
    return wrapper
```

**核心运算方法**（`polyclasses.py:549-562`）：
```python
def add(f, g: Self, /) -> Self:
    """Add two multivariate polynomials f and g."""
    F, G = f.unify_DMP(g)  # 可能抛出 UnificationFailed
    return F._add(G)

def sub(f, g: Self, /) -> Self:
    """Subtract two multivariate polynomials f and g."""
    F, G = f.unify_DMP(g)  # 可能抛出 UnificationFailed
    return F._sub(G)

def mul(f, g: Self, /) -> Self:
    """Multiply two multivariate polynomials f and g."""
    F, G = f.unify_DMP(g)  # 可能抛出 UnificationFailed
    return F._mul(G)
```

**统一方法**（`polyclasses.py:328-337`）：
```python
def unify_DMP(f, g: DMP[Es]) -> tuple[DMP[Et], DMP[Et]]:
    """Unify and return DMP instances of f and g."""
    
    # 情况1：类型或级别不匹配
    if not isinstance(g, DMP) or f.lev != g.lev:
        raise UnificationFailed("Cannot unify %s with %s" % (f, g))
    
    # 情况2：域相同
    if f.dom == g.dom:
        return f, g
    
    # 情况3：域不同，尝试统一
    else:
        dom: Domain[Et] = f.dom.unify(g.dom)  # 可能抛出 UnificationFailed
        return f.convert(dom), g.convert(dom)
```

### 3.2 执行流程决策树

```
Poly.__add__(self, other) （以加法为例，减法、乘法类似）
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 第一层：_polifyit 装饰器                                                  │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
g = _sympify(other)  # 先 sympify
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 检查 g 的类型                                                              │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ├── isinstance(g, Poly) ──▶ 是 ──▶ 直接调用 func(f, g) → 进入第二层
    │
    ├── isinstance(g, Integer) ──▶ 是 ──▶ g = f.from_expr(g, *f.gens, domain=f.domain)
    │                                              │
    │                                              ▼
    │                                        调用 func(f, g) → 进入第二层
    │
    └── isinstance(g, Expr) ──▶ 是 ──▶ 尝试 g = f.from_expr(g, *f.gens)
                                        │
                                   ┌────┴────┐
                                   │         │
                                 成功      失败
                                   │         │
                                   ▼         ▼
                            调用 func   检查 g.is_Matrix
                            → 进入第二层      │
                                         ┌────┴────┐
                                         │         │
                                        Yes       No
                                         │         │
                                         ▼         ▼
                                  return     降级到 Expr 层面
                                  NotImple-  运算（已废弃）
                                  mented        │
                                               ▼
                                         expr_method = getattr(f.as_expr(), func.__name__)
                                         result = expr_method(g)
                                         返回 result 或 NotImplemented
    │
    └── 其他类型 ──▶ 返回 NotImplemented（交由对端处理）
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 第二层：核心运算方法（如 add）                                              │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
调用 f.unify_DMP(g)
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ unify_DMP 内部                                                             │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 检查 isinstance(g, DMP) 且 f.lev == g.lev                                │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ├── No ──▶ 抛出 UnificationFailed（直接失败终止）
    │
    └── Yes ──▶ 继续
              │
              ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 检查 f.dom == g.dom                                                   │
    └─────────────────────────────────────────────────────────────────────┘
              │
              ├── Yes ──▶ 返回 (f, g) → 执行运算 → 返回结果
              │
              └── No ──▶ 尝试 dom = f.dom.unify(g.dom)
                        │
                   ┌────┴────┐
                   │         │
                 成功      失败
                   │         │
                   ▼         ▼
              转换后执行   抛出 UnificationFailed
              运算 → 返回   （直接失败终止）
              结果
```

### 3.3 场景分类与证据

#### 场景 A：返回布尔结论

**关键发现**：算术运算**不会**返回布尔结论。算术运算的返回值是运算结果或异常或 `NotImplemented`。

**证据**：
- `_polifyit` 装饰器返回 `NotImplemented` 或运算结果
- 核心方法返回运算结果或抛出 `UnificationFailed`

#### 场景 B：交由对端处理（返回 `NotImplemented`）

**触发条件**：
1. `other` 是 `Matrix` 且无法转换为 Poly
2. `other` 不是 `Poly`/`Integer`/`Expr` 类型

**代码证据**：
```python
# polytools.py:85-86
if g.is_Matrix:
    # Matrix：返回 NotImplemented
    return NotImplemented

# polytools.py:103-104
else:
    return NotImplemented
```

**典型示例**：
```python
from sympy import Poly, Matrix
from sympy.abc import x

# 示例1：与 Matrix 运算
p = Poly(x**2 + 1, x)
M = Matrix([[1, 2], [3, 4]])
result = p + M
# 结果：返回 NotImplemented
# Python 尝试 M + p（调用 M.__radd__(p)）
# 如果都返回 NotImplemented，Python 抛出 TypeError

# 示例2：与完全不支持的类型运算
class MyClass:
    pass

p = Poly(x**2 + 1, x)
obj = MyClass()
result = p + obj
# 结果：返回 NotImplemented（经由 _polifyit 的 else 分支）
```

#### 场景 C：直接失败终止（抛出异常）

**触发条件**：
1. `unify_DMP` 中类型或级别不匹配
2. `unify_DMP` 中域统一失败

**代码证据**：
```python
# polyclasses.py:330-331
if not isinstance(g, DMP) or f.lev != g.lev:
    raise UnificationFailed("Cannot unify %s with %s" % (f, g))

# polyclasses.py:336
dom: Domain[Et] = f.dom.unify(g.dom)  # 可能抛出 UnificationFailed
```

**典型示例**：
```python
from sympy import Poly, GF
from sympy.abc import x

# 示例1：域无法统一
p1 = Poly(x**2 + 1, x, domain=GF(3))
p2 = Poly(x + 1, x, domain=ZZ)
result = p1 + p2
# 结果：抛出 UnificationFailed
# 证据：
# 1. _polifyit：isinstance(g, Poly) → True
# 2. 调用 func(f, g) → add(f, g)
# 3. add 调用 f.unify_DMP(g)
# 4. unify_DMP 调用 f.dom.unify(g.dom) = unify(GF(3), ZZ)
# 5. 特征不同（3 vs 0）→ 抛出 UnificationFailed
# 6. 异常传播，未被捕获

# 示例2：级别不匹配（变量数量不同）
p1 = Poly(x**2 + 1, x)           # lev = 0（单变量）
p2 = Poly(x + y, x, y)            # lev = 1（双变量）
try:
    result = p1 + p2
except UnificationFailed as e:
    print(e)  # "Cannot unify ..."
# 证据：
# 1. _polifyit：isinstance(g, Poly) → True
# 2. 调用 func(f, g) → add(f, g)
# 3. add 调用 f.unify_DMP(g)
# 4. unify_DMP 检查 f.lev != g.lev → True
# 5. 抛出 UnificationFailed
# 6. 异常传播，未被捕获
```

### 3.4 特殊场景：降级到 Expr 层面运算（已废弃）

**代码证据**：
```python
# polytools.py:87-100
else:
    # 其他 Expr：降级到 Expr 层面运算（已废弃）
    expr_method = getattr(f.as_expr(), func.__name__)
    result = expr_method(g)
    
    if result is not NotImplemented:
        sympy_deprecation_warning(
            """
            Mixing Poly with non-polynomial expressions in binary
            operations is deprecated...
            """,
            deprecated_since_version="1.6",
            active_deprecations_target="deprecated-poly-nonpoly-binary-operations",
        )
    return result
```

**典型示例**：
```python
from sympy import Poly, sin
from sympy.abc import x

# 示例：Poly 与非多项式 Expr 运算（已废弃）
p = Poly(x**2 + 1, x)
expr = sin(x)  # 是 Expr，但不是 Poly，也不是 Integer

# p + expr 的执行流程：
# 1. _polifyit：isinstance(g, Expr) → True
# 2. 尝试 g = f.from_expr(g, *f.gens) → 尝试 Poly(sin(x), x)
# 3. 抛出 PolynomialError（sin(x) 不是多项式）
# 4. 捕获 PolynomialError
# 5. 检查 g.is_Matrix → False
# 6. 降级到 Expr 层面运算：
#    expr_method = f.as_expr().__add__
#    result = (x**2 + 1) + sin(x)
# 7. 发出 DeprecationWarning
# 8. 返回 result（Expr 类型）

# 结果：x**2 + sin(x) + 1（Expr 类型）
# 注意：这会发出 DeprecationWarning，建议显式转换：
# p.as_expr() + expr 或 p + Poly(expr, x)（如果可能）
```

---

## 4. 构造流程详细分析

### 4.1 代码位置与结构

**文件位置**：
- 单个表达式构造：`sympy/polys/polytools.py:4758-4802` (`poly_from_expr`, `_poly_from_expr`)
- 多个表达式构造：`sympy/polys/polytools.py:4805-4903` (`parallel_poly_from_expr`, `_parallel_poly_from_expr`)

**单个表达式构造代码**（`polytools.py:4765-4802`）：
```python
def _poly_from_expr(expr, opt):
    """Construct a polynomial from an expression."""
    orig, expr = expr, sympify(expr)
    
    # 情况1：不是 Basic 类型
    if not isinstance(expr, Basic):
        raise PolificationFailed(opt, orig, expr)
    
    # 情况2：已经是 Poly
    elif expr.is_Poly:
        poly = expr.__class__._from_poly(expr, opt)
        opt.gens = poly.gens
        opt.domain = poly.domain
        if opt.polys is None:
            opt.polys = True
        return poly, opt
    
    # 情况3：需要展开
    elif opt.expand:
        expr = expr.expand()
    
    # 步骤1：提取单项式和系数
    rep, opt = _dict_from_expr(expr, opt)
    
    # 情况4：没有生成器
    if not opt.gens:
        raise PolificationFailed(opt, orig, expr)
    
    monoms, coeffs = list(zip(*list(rep.items())))
    domain = opt.domain
    
    # 步骤2：选择域
    if domain is None:
        # 自动选择最小覆盖域（construct_domain 内部会降级到 EX）
        opt.domain, coeffs = construct_domain(coeffs, opt=opt)
    else:
        # 使用用户指定的域
        coeffs = list(map(domain.from_sympy, coeffs))  # 可能抛出 CoercionFailed
    
    # 步骤3：构造 Poly
    rep = dict(list(zip(monoms, coeffs)))
    poly = Poly._from_dict(rep, opt)
    
    if opt.polys is None:
        opt.polys = False
    
    return poly, opt
```

**多个表达式构造代码**（`polytools.py:4812-4829`）：
```python
def _parallel_poly_from_expr(exprs, opt):
    """Construct polynomials from expressions."""
    
    # 特殊情况：两个 Poly 对象
    if len(exprs) == 2:
        f, g = exprs
        
        if isinstance(f, Poly) and isinstance(g, Poly):
            # 从 Poly 转换
            f = f.__class__._from_poly(f, opt)
            g = g.__class__._from_poly(g, opt)
            
            # 统一两个 Poly
            f, g = f.unify(g)  # 可能抛出 UnificationFailed
            
            opt.gens = f.gens
            opt.domain = f.domain
            
            if opt.polys is None:
                opt.polys = True
            
            return [f, g], opt
    
    # ... 其他情况 ...
```

### 4.2 执行流程决策树

```
poly_from_expr(expr, *gens, **args)
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ _poly_from_expr(expr, opt)                                                 │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 检查 isinstance(expr, Basic)                                               │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ├── No ──▶ 抛出 PolificationFailed（直接失败终止）
    │
    └── Yes ──▶ 继续
              │
              ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 检查 expr.is_Poly                                                      │
    └─────────────────────────────────────────────────────────────────────┘
              │
              ├── Yes ──▶ 从 Poly 转换，返回结果
              │
              └── No ──▶ 继续
                        │
                        ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 检查 opt.expand                                                        │
    └─────────────────────────────────────────────────────────────────────┘
                        │
                        ├── Yes ──▶ expr = expr.expand()
                        │
                        └── No ──▶ 继续
                                  │
                                  ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 调用 _dict_from_expr(expr, opt)                                        │
    │ （提取单项式和系数，可能抛出 PolynomialError）                           │
    └─────────────────────────────────────────────────────────────────────┘
                                  │
                              可能抛出异常
                                  │
                                  ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 检查 opt.gens 是否为空                                                 │
    └─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ├── Yes ──▶ 抛出 PolificationFailed（直接失败终止）
                                  │
                                  └── No ──▶ 继续
                                            │
                                            ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 检查 domain 是否已指定（opt.domain is None）                           │
    └─────────────────────────────────────────────────────────────────────┘
                                            │
                        ┌───────────────────┴───────────────────┐
                        │                                       │
                       Yes                                     No
                        │                                       │
                        ▼                                       ▼
    ┌──────────────────────────────┐         ┌──────────────────────────────┐
    │ 自动选择域：                  │         │ 使用用户指定的域：            │
    │ construct_domain(coeffs, opt)│         │ list(map(domain.from_sympy,  │
    │ （内部会降级到 EX）           │         │           coeffs))            │
    └──────────────────────────────┘         │ （可能抛出 CoercionFailed）   │
                        │                      └──────────────────────────────┘
                        │                                       │
                        ▼                                       │
              继续构造 Poly                          可能抛出异常
                        │                                       │
                        ▼                                       ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 调用 Poly._from_dict(rep, opt)                                        │
    └─────────────────────────────────────────────────────────────────────┘
                                            │
                                            ▼
                                      返回 (poly, opt)
```

### 4.3 场景分类与证据

#### 场景 A：返回布尔结论

**关键发现**：构造流程**不会**返回布尔结论。构造流程要么成功返回 Poly，要么抛出异常。

#### 场景 B：交由对端处理

**关键发现**：构造流程**不会**返回 `NotImplemented`。这不是一个二元运算，没有"对端"概念。

#### 场景 C：直接失败终止（抛出异常）

**触发条件**：
1. `expr` 不是 `Basic` 类型
2. `_dict_from_expr` 失败（无法提取单项式结构）
3. 没有生成器
4. 用户指定的域无法表示系数（抛出 `CoercionFailed`）
5. 并行构造时两个 Poly 无法统一（抛出 `UnificationFailed`）

**代码证据**：
```python
# polytools.py:4769-4770
if not isinstance(expr, Basic):
    raise PolificationFailed(opt, orig, expr)  # ← 直接失败终止

# polytools.py:4785-4786
if not opt.gens:
    raise PolificationFailed(opt, orig, expr)  # ← 直接失败终止

# polytools.py:4793-4794
else:
    # 使用用户指定的域
    coeffs = list(map(domain.from_sympy, coeffs))  # 可能抛出 CoercionFailed

# polytools.py:4821
f, g = f.unify(g)  # 可能抛出 UnificationFailed
```

**典型示例**：
```python
from sympy import Poly, poly_from_expr, parallel_poly_from_expr, sin, ZZ
from sympy.abc import x

# 示例1：非多项式表达式（无法提取单项式）
try:
    poly, opt = poly_from_expr(sin(x), x)
except PolificationFailed as e:
    print(e)  # 抛出异常
# 证据：
# 1. _poly_from_expr 调用 _dict_from_expr(sin(x), opt)
# 2. sin(x) 无法表示为多项式 → 抛出 PolynomialError
# 3. 异常传播，未被捕获

# 示例2：用户指定域但无法表示系数
try:
    # 1/2 无法在 ZZ 中表示
    poly, opt = poly_from_expr(x + 1/2, x, domain=ZZ)
except CoercionFailed as e:
    print(e)  # 抛出异常
# 证据：
# 1. _poly_from_expr 中 domain = ZZ（用户指定）
# 2. coeffs = list(map(ZZ.from_sympy, [1, 1/2]))
# 3. ZZ.from_sympy(1/2) 失败 → 抛出 CoercionFailed
# 4. 异常传播，未被捕获

# 示例3：并行构造时无法统一
p1 = Poly(x**2 + 1, x, domain=GF(3))
p2 = Poly(x + 1, x, domain=ZZ)
try:
    polys, opt = parallel_poly_from_expr([p1, p2])
except UnificationFailed as e:
    print(e)  # 抛出异常
# 证据：
# 1. _parallel_poly_from_expr 检测到两个都是 Poly
# 2. 调用 f.unify(g)
# 3. unify 尝试统一域 GF(3) 和 ZZ
# 4. 特征不同 → 抛出 UnificationFailed
# 5. 异常传播，未被捕获

# 示例4：自动选择域（会降级到 EX，不抛异常）
from sympy import sqrt, S
expr = x**2 + sqrt(2) + S(3.14)
poly, opt = poly_from_expr(expr, x)
# 结果：成功，domain = EX
# 证据：
# 1. construct_domain([1, sqrt(2)+3.14])
# 2. _construct_simple 检测到 floats 和 algebraics 共存 → 返回 False
# 3. _construct_expression → 返回 EX 域
# 4. 构造成功，使用 EX 域
```

### 4.4 特殊场景：construct_domain 内部降级

**关键发现**：`construct_domain` 会在内部降级到 `EX` 域，**不会**抛异常。

**代码证据**（`constructor.py:364-380`）：
```python
result = _construct_simple(coeffs, opt)

if result is not None:
    if result is not False:
        domain, coeffs = result
    else:
        # 浮点数 + 代数数 → 直接降级到 EX
        domain, coeffs = _construct_expression(coeffs, opt)
else:
    if opt.composite is False:
        result = None
    else:
        result = _construct_composite(coeffs, opt)
    
    if result is not None:
        domain, coeffs = result
    else:
        # 复合域也失败 → 降级到 EX
        domain, coeffs = _construct_expression(coeffs, opt)
```

**行为差异**：
- `construct_domain` **不会**抛异常，总是返回某个域（可能是 `EX`）
- 但如果表达式无法提取单项式结构（如 `sin(x)`），`_dict_from_expr` 会抛异常

---

## 5. 三种调用方式严谨对照表

### 5.1 处理方式总览

| 维度 | 相等性判断（`__eq__`） | 算术运算（`__add__` 等） | 构造流程（`poly_from_expr`） |
|------|----------------------|-------------------------|-----------------------------|
| **返回布尔结论** | ✅ 是（`False`） | ❌ 否 | ❌ 否 |
| **交由对端处理** | ✅ 是（`NotImplemented`） | ✅ 是（`NotImplemented`） | ❌ 否（无对端概念） |
| **直接失败终止** | ❌ 否（所有异常都被捕获） | ✅ 是（`UnificationFailed`） | ✅ 是（多种异常） |

### 5.2 场景详细对照

#### 场景 1：域无法统一

| 调用方式 | 处理方式 | 代码证据 | 结果 |
|---------|---------|---------|------|
| **相等性判断** | 返回布尔结论（`False`） | `polytools.py:4712-4713` → 捕获 `UnificationFailed`，返回 `False` | `p1 == p2` → `False` |
| **算术运算** | 直接失败终止（抛异常） | `polyclasses.py:336` → 调用 `f.dom.unify(g.dom)` 失败，`UnificationFailed` 传播 | `p1 + p2` → 抛出 `UnificationFailed` |
| **构造流程** | 直接失败终止（抛异常） | `polytools.py:4821` → 调用 `f.unify(g)` 失败，`UnificationFailed` 传播 | `parallel_poly_from_expr([p1, p2])` → 抛出 `UnificationFailed` |

#### 场景 2：类型不支持

| 调用方式 | 处理方式 | 代码证据 | 结果 |
|---------|---------|---------|------|
| **相等性判断** | 交由对端处理（`NotImplemented`）或返回布尔结论 | 装饰器：`sympify` 失败 → `NotImplemented`；方法内部：转换失败 → `False` | 取决于 `other` 类型 |
| **算术运算** | 交由对端处理（`NotImplemented`） | `polytools.py:85-86` → `Matrix` 返回 `NotImplemented`；`polytools.py:103-104` → 其他类型返回 `NotImplemented` | `p + Matrix(...)` → `NotImplemented` |
| **构造流程** | 直接失败终止（抛异常） | `polytools.py:4769-4770` → 非 `Basic` 类型抛出 `PolificationFailed` | `poly_from_expr(not_basic)` → 抛出 `PolificationFailed` |

#### 场景 3：生成器/级别不匹配

| 调用方式 | 处理方式 | 代码证据 | 结果 |
|---------|---------|---------|------|
| **相等性判断** | 返回布尔结论（`False`） | `polytools.py:4706-4707` → `len(f.gens) != len(g.gens)` 返回 `False` | `p1 == p2` → `False` |
| **算术运算** | 直接失败终止（抛异常） | `polyclasses.py:330-331` → `f.lev != g.lev` 抛出 `UnificationFailed` | `p1 + p2` → 抛出 `UnificationFailed` |
| **构造流程** | 直接失败终止（抛异常） | `polytools.py:4785-4786` → 无生成器抛出 `PolificationFailed` | `poly_from_expr(constant)` → 抛出 `PolificationFailed` |

#### 场景 4：表达式无法转换为 Poly

| 调用方式 | 处理方式 | 代码证据 | 结果 |
|---------|---------|---------|------|
| **相等性判断** | 返回布尔结论（`False`） | `polytools.py:4700-4704` → 捕获 `PolynomialError` 等，返回 `False` | `p == sin(x)` → `False` |
| **算术运算** | 降级到 Expr 运算（已废弃）或返回 `NotImplemented` | `polytools.py:87-100` → 非多项式 `Expr` 降级；`polytools.py:85-86` → `Matrix` 返回 `NotImplemented` | `p + sin(x)` → Expr 结果（带警告） |
| **构造流程** | 直接失败终止（抛异常） | `_dict_from_expr` 失败 → 异常传播 | `poly_from_expr(sin(x))` → 抛出 `PolificationFailed` |

### 5.3 异常处理策略对照

| 异常类型 | 相等性判断 | 算术运算 | 构造流程 |
|---------|-----------|---------|---------|
| **`UnificationFailed`** | 捕获 → 返回 `False` | 不捕获 → 传播 | 不捕获 → 传播（并行构造） |
| **`CoercionFailed`** | 捕获 → 返回 `False` | 不适用（在更低层） | 不捕获 → 传播（用户指定域时） |
| **`PolynomialError`** | 捕获 → 返回 `False` | 捕获 → 降级或 `NotImplemented` | 不捕获 → 传播 |
| **`DomainError`** | 捕获 → 返回 `False` | 不适用 | 不适用 |
| **`SympifyError`** | 捕获（装饰器）→ 返回 `NotImplemented` | 不适用（`_polifyit` 使用 `_sympify`） | 不适用 |
| **`PolificationFailed`** | 不适用 | 不适用 | 主动抛出 |

### 5.4 设计哲学对照

| 设计维度 | 相等性判断 | 算术运算 | 构造流程 |
|---------|-----------|---------|---------|
| **核心目标** | 给出确定的比较结果 | 执行运算或明确失败 | 精确构造或明确失败 |
| **容错程度** | 最高（从不抛异常） | 中等（尝试兼容） | 最低（严格报错） |
| **用户期望** | 条件判断应该稳定 | 运算应该尽量执行 | 构造失败应该明确告知 |
| **Python 约定** | `__eq__` 应返回布尔值 | 算术运算可以返回 `NotImplemented` | 构造函数可以抛异常 |
| **异常处理** | 所有异常都被捕获 | 选择性捕获（装饰器），核心异常传播 | 几乎不捕获（让用户知道失败原因） |

---

## 6. 关键修正与澄清

### 6.1 之前分析的修正

| 之前的结论 | 修正后的结论 | 证据 |
|-----------|-------------|------|
| "相等性判断：任何失败都返回 `False`" | "相等性判断：有两种结果 - 方法内部返回 `False`，装饰器在 `sympify` 失败时返回 `NotImplemented`" | `decorators.py:70-78` → 装饰器捕获 `SympifyError` 返回 `NotImplemented` |
| "算术运算：统一失败时降级" | "算术运算：统一失败时抛出 `UnificationFailed`，不会降级" | `polyclasses.py:328-337` → `unify_DMP` 抛出 `UnificationFailed`，未被捕获 |
| "构造流程：严格报错" | "构造流程：严格报错，但 `construct_domain` 会在内部降级到 `EX`" | `constructor.py:364-380` → `construct_domain` 会降级到 `EX`，不会抛异常 |

### 6.2 关键澄清

#### 澄清 1：`_op_priority` 的作用

**之前的误解**：认为 `_sympifyit` 总是会 `sympify` `other`

**实际情况**：
```python
# decorators.py:70-78
if not hasattr(b, '_op_priority'):
    b = sympify(b, strict=True)
return func(a, b)
```

只有当 `other` **没有** `_op_priority` 时，才会尝试 `sympify`。

- `Poly` 有 `_op_priority = 10.001`
- `Expr` 有 `_op_priority = 10.0`

所以当比较 `Poly` 与 `Poly` 或 `Expr` 时，**不会**进行 `sympify`，直接调用方法。

#### 澄清 2：`NotImplemented` vs `False` 的语义差异

| 方式 | Python 语义 | 实际效果 |
|------|------------|---------|
| **`return False`** | 直接给出比较结果 | 比较结果为 `False`，不尝试反向运算 |
| **`return NotImplemented`** | 告知 Python "我不知道如何处理" | Python 尝试 `other.__eq__(self)`，若都返回 `NotImplemented` 则最终比较结果为 `False` |

**关键区别**：
- `return False`：直接结论，不交给对端
- `return NotImplemented`：交由对端处理，可能有不同结果

#### 澄清 3：`construct_domain` 的内部降级

**之前的误解**：认为构造流程"严格报错"

**实际情况**：
- `_dict_from_expr` 失败时会抛异常（表达式无法提取单项式结构）
- 但 `construct_domain` 会在内部降级到 `EX`（系数无法在精确域表示时）

**示例**：
```python
# 成功：系数会用 EX 域表示
poly_from_expr(x**2 + sqrt(2) + 3.14, x)  # 成功，domain=EX

# 失败：表达式无法提取单项式结构
poly_from_expr(sin(x), x)  # 抛出 PolificationFailed
```

---

## 7. 代码位置速查

### 7.1 相等性判断

| 功能 | 文件 | 行号 |
|------|------|------|
| `_sympifyit` 装饰器 | `sympy/core/decorators.py` | 23-80 |
| `Poly.__eq__` 方法 | `sympy/polys/polytools.py` | 4696-4718 |
| `_op_priority` (Poly) | `sympy/polys/polytools.py` | 168 |
| `_op_priority` (Expr) | `sympy/core/expr.py` | 224 |
| `is_Poly` (Poly) | `sympy/polys/polytools.py` | 167 |
| `is_Poly` (Basic) | `sympy/core/basic.py` | 248 |

### 7.2 算术运算

| 功能 | 文件 | 行号 |
|------|------|------|
| `_polifyit` 装饰器 | `sympy/polys/polytools.py` | 72-105 |
| `DMP.add` 方法 | `sympy/polys/polyclasses.py` | 549-552 |
| `DMP.sub` 方法 | `sympy/polys/polyclasses.py` | 554-557 |
| `DMP.mul` 方法 | `sympy/polys/polyclasses.py` | 559-562 |
| `DMP.unify_DMP` | `sympy/polys/polyclasses.py` | 328-337 |

### 7.3 构造流程

| 功能 | 文件 | 行号 |
|------|------|------|
| `poly_from_expr` | `sympy/polys/polytools.py` | 4758-4762 |
| `_poly_from_expr` | `sympy/polys/polytools.py` | 4765-4802 |
| `parallel_poly_from_expr` | `sympy/polys/polytools.py` | 4805-4809 |
| `_parallel_poly_from_expr` | `sympy/polys/polytools.py` | 4812-4903 |
| `construct_domain` | `sympy/polys/constructor.py` | 268-388 |

---

## 8. 执行流程图汇总

### 8.1 相等性判断完整流程

```
输入：p == other (p 是 Poly)
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 第一层：_sympifyit 装饰器（retval = NotImplemented）                      │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 检查 hasattr(other, '_op_priority')                                   │
    └─────────────────────────────────────────────────────────────────────┘
    │
    ├── Yes（other 是 Poly 或 Expr）
    │              │
    │              ▼
    │         直接调用 p.__eq__(other)
    │              │
    │              ▼
    │    ┌─────────────────────────────────────────────────────────────────┐
    │    │ 第二层：__eq__ 方法内部                                           │
    │    └─────────────────────────────────────────────────────────────────┘
    │              │
    │              ▼
    │    ┌─────────────────────────────────────────────────────────────────┐
    │    │ 检查 other.is_Poly                                                │
    │    └─────────────────────────────────────────────────────────────────┘
    │              │
    │         ┌────┴────┐
    │         │         │
    │       True       False
    │         │         │
    │         ▼         ▼
    │    继续比较   尝试转换为 Poly
    │         │              │
    │         │         ┌────┴────┐
    │         │         │         │
    │         │       成功      失败
    │         │         │         │
    │         │         ▼         ▼
    │         │    继续比较   返回 False（布尔结论）
    │         │              (PolynomialError, DomainError, CoercionFailed)
    │         │
    │         ▼
    │    ┌─────────────────────────────────────────────────────────────────┐
    │    │ 检查生成器数量、域统一等                                          │
    │    │ - 生成器不同 → 返回 False                                         │
    │    │ - 域统一失败 → 返回 False                                         │
    │    └─────────────────────────────────────────────────────────────────┘
    │
    └── No（other 没有 _op_priority）
                   │
                   ▼
              尝试 sympify(other, strict=True)
                   │
              ┌────┴────┐
              │         │
            成功      失败
              │         │
              ▼         ▼
         调用 __eq__   返回 NotImplemented（交由对端处理）
              │         (SympifyError)
              ▼
         进入第二层方法内部
         （同 Yes 分支）
```

### 8.2 算术运算完整流程

```
输入：p + other (p 是 Poly)
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 第一层：_polifyit 装饰器                                                  │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
other = _sympify(other)
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 检查 other 的类型                                                          │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ├── isinstance(other, Poly) ──▶ 是 ──▶ 调用 add(p, other)
    │                                              │
    │                                              ▼
    │                                    ┌─────────────────────────────────┐
    │                                    │ 第二层：add 方法内部              │
    │                                    └─────────────────────────────────┘
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
    ├── isinstance(other, Integer) ──▶ 是 ──▶ 转换为 Poly，调用 add
    │
    ├── isinstance(other, Expr) ──▶ 是 ──▶ 尝试转换为 Poly
    │                                        │
    │                                   ┌────┴────┐
    │                                   │         │
    │                                 成功      失败
    │                                   │         │
    │                                   ▼         ▼
    │                              调用 add   检查 other.is_Matrix
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

### 8.3 构造流程完整流程

```
输入：poly_from_expr(expr, *gens, **args)
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ _poly_from_expr(expr, opt)                                                 │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 检查 isinstance(expr, Basic)                                               │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ├── No ──▶ 抛出 PolificationFailed（直接失败终止）
    │
    └── Yes ──▶ 继续
              │
              ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 检查 expr.is_Poly                                                      │
    └─────────────────────────────────────────────────────────────────────┘
              │
              ├── Yes ──▶ 从 Poly 转换，返回结果
              │
              └── No ──▶ 继续
                        │
                        ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 调用 _dict_from_expr(expr, opt)                                        │
    │ （可能抛出 PolynomialError）                                             │
    └─────────────────────────────────────────────────────────────────────┘
                        │
                   可能抛出异常
                        │
                        ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 检查 opt.gens 是否为空                                                 │
    └─────────────────────────────────────────────────────────────────────┘
                        │
                        ├── Yes ──▶ 抛出 PolificationFailed（直接失败终止）
                        │
                        └── No ──▶ 继续
                                  │
                                  ▼
    ┌─────────────────────────────────────────────────────────────────────┐
    │ 检查 opt.domain 是否为 None                                            │
    └─────────────────────────────────────────────────────────────────────┘
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

## 9. 总结与建议

### 9.1 核心结论

| 调用方式 | 返回布尔结论 | 交由对端处理 | 直接失败终止 |
|---------|-------------|-------------|-------------|
| **相等性判断** | ✅ 方法内部返回 `False` | ✅ 装饰器在 `sympify` 失败时返回 `NotImplemented` | ❌ 无 |
| **算术运算** | ❌ 无 | ✅ 类型不支持时返回 `NotImplemented` | ✅ 域/级别不匹配时抛出 `UnificationFailed` |
| **构造流程** | ❌ 无 | ❌ 无（无对端概念） | ✅ 多种异常（`PolificationFailed`, `CoercionFailed`, `UnificationFailed`） |

### 9.2 关键设计原则

1. **相等性判断**：
   - 遵循 Python 约定：`__eq__` 应该返回布尔值，不抛异常
   - 但 `_sympifyit` 装饰器在 `sympify` 失败时返回 `NotImplemented`（交由对端处理）
   - 方法内部所有异常都被捕获并返回 `False`

2. **算术运算**：
   - 类型不支持时返回 `NotImplemented`（交由对端处理）
   - 域/级别不匹配时抛出 `UnificationFailed`（直接失败终止）
   - 非多项式 `Expr` 会降级到 `Expr` 层面运算（已废弃，带警告）

3. **构造流程**：
   - 严格策略：无法构造时明确抛异常
   - 但 `construct_domain` 会在内部降级到 `EX`（系数问题）
   - 表达式无法提取单项式结构时抛异常

### 9.3 用户指南

**相等性判断**：
```python
# 安全：__eq__ 从不抛异常
if p1 == p2:
    print("相等")
else:
    print("不相等")  # 可能是域不同、生成器不同、或 other 无法转换
```

**算术运算**：
```python
# 可能抛出 UnificationFailed，建议捕获
try:
    result = p1 + p2
except UnificationFailed:
    # 域不兼容，显式转换到 Expr
    result = p1.as_expr() + p2.as_expr()
```

**构造流程**：
```python
# 可能抛出多种异常，建议捕获
try:
    poly, opt = poly_from_expr(expr, x)
except PolificationFailed:
    print("表达式无法构造为多项式")
except CoercionFailed:
    print("指定的域无法表示系数")
except UnificationFailed:
    print("多个多项式无法统一")
```

### 9.4 代码证据索引

本报告所有结论都有明确的代码证据支持，详见：
- 第 2-4 节：详细的代码引用和行号
- 第 5 节：对照表中的证据列
- 第 7 节：代码位置速查
- 第 8 节：执行流程图汇总
