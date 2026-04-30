# SymPy 多项式统一失败处理方式对照分析

## 1. 概述

本文档深入分析 SymPy 中三种典型调用方对**统一失败**的不同处理方式：
- **相等性判断** (`__eq__`)
- **算术运算** (`__add__`, `__sub__`, `__mul__` 等)
- **构造流程** (`poly_from_expr`, `_parallel_poly_from_expr`)

通过对比分析，揭示不同场景下的设计决策、行为差异及其背后的设计哲学。

---

## 2. 三种调用方处理方式对照表

| 维度 | 相等性判断 (`__eq__`) | 算术运算 (`__add__` 等) | 构造流程 (`poly_from_expr`) |
|------|----------------------|-------------------------|-----------------------------|
| **核心目标** | 返回布尔值 (True/False) | 执行运算并返回结果 | 创建 Poly 对象 |
| **失败处理** | 捕获异常，返回 `False` | 返回 `NotImplemented` 或抛出 | 直接抛出异常 |
| **异常类型** | `UnificationFailed` | `UnificationFailed`, `CoercionFailed` | `PolificationFailed`, `UnificationFailed` |
| **降级策略** | 无（直接返回 False） | 多级降级：Poly → Expr → NotImplemented | 无（直接报错） |
| **用户体验** | 最友好（从不抛异常） | 中等（尝试兼容） | 最严格（明确报错） |
| **设计哲学** | "安全比较" | "兼容优先" | "精确构造" |
| **代码位置** | `polytools.py:4696-4718` | `polytools.py:72-105`, `polyclasses.py:549-612` | `polytools.py:4758-4903` |

---

## 3. 相等性判断 (`__eq__`) 详细分析

### 3.1 核心代码

**代码位置**：`sympy/polys/polytools.py:4696-4718`

```python
@_sympifyit('other', NotImplemented)
def __eq__(self, other):
    f, g = self, other
    
    # 情况1：other 不是 Poly
    if not g.is_Poly:
        try:
            # 尝试将 other 转换为 Poly
            g = f.__class__(g, f.gens, domain=f.get_domain())
        except (PolynomialError, DomainError, CoercionFailed):
            # 转换失败：返回 False
            return False
    
    # 情况2：生成器数量不同
    if len(f.gens) != len(g.gens):
        return False
    
    # 情况3：域不同，尝试统一
    if f.rep.dom != g.rep.dom:
        try:
            dom = f.rep.dom.unify(g.rep.dom, f.gens)
        except UnificationFailed:
            # 统一失败：返回 False
            return False
        
        # 统一成功：转换到公共域
        f = f.set_domain(dom)
        g = g.set_domain(dom)
    
    # 情况4：比较内部表示
    return f.rep == g.rep
```

### 3.2 处理流程决策树

```
Poly.__eq__(self, other)
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤1：other 是 Poly 类型？                                  │
│    if not g.is_Poly:                                        │
└────────────────────────────────────────────────────────────┘
    │
    ├── No ──▶ 尝试转换：g = f.__class__(g, f.gens, domain=f.get_domain())
    │              │
    │         ┌────┴────┐
    │         │         │
    │       成功      失败
    │         │         │
    │         ▼         ▼
    │      继续比较   返回 False
    │                   (PolynomialError/DomainError/CoercionFailed)
    │
    └── Yes ──▶ 继续
              │
              ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤2：生成器数量相同？                                      │
    │    if len(f.gens) != len(g.gens):                          │
    └────────────────────────────────────────────────────────────┘
              │
              ├── No ──▶ 返回 False
              │
              └── Yes ──▶ 继续
                        │
                        ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤3：域相同？                                              │
    │    if f.rep.dom != g.rep.dom:                              │
    └────────────────────────────────────────────────────────────┘
                        │
                        ├── Yes ──▶ 继续 → 比较 f.rep == g.rep
                        │
                        └── No ──▶ 尝试统一：dom = f.rep.dom.unify(g.rep.dom, f.gens)
                                    │
                             ┌──────┴──────┐
                             │             │
                           成功          失败
                             │             │
                             ▼             ▼
                        转换后比较    返回 False
                                          (UnificationFailed)
```

### 3.3 为什么这样处理？

**设计哲学："安全比较"**

1. **Python 约定**：
   - `__eq__` 应该总是返回布尔值，不应该抛出异常
   - 例如：`5 == "abc"` 返回 `False`，而不是抛出 `TypeError`

2. **数学语义**：
   - 不同域的多项式"看起来"可能相同，但实际上属于不同的数学结构
   - 例如：`Poly(x, x, domain=ZZ)` 和 `Poly(x, x, domain=QQ)` 虽然表达式相同，但严格来说不相等

3. **用户体验**：
   - 用户在做条件判断时（如 `if p == q:`）不希望程序崩溃
   - 返回 `False` 是最安全的选择

### 3.4 行为差异示例

```python
from sympy import Poly, GF
from sympy.abc import x

# 示例1：域不同，无法统一
p1 = Poly(x**2 + 1, x, domain=ZZ)      # 整数域
p2 = Poly(x**2 + 1, x, domain=GF(3))   # 有限域 GF(3)
result = p1 == p2
# 结果：False
# 触发路径：__eq__ → 尝试 unify(ZZ, GF(3)) → UnificationFailed → 返回 False

# 示例2：生成器数量不同
p1 = Poly(x**2 + 1, x)          # gens = (x,)
p2 = Poly(x + y, x, y)          # gens = (x, y)
result = p1 == p2
# 结果：False
# 触发路径：__eq__ → len(f.gens) != len(g.gens) → 返回 False

# 示例3：无法转换为 Poly 的类型
p = Poly(x**2 + 1, x)
result = p == "abc"
# 结果：False
# 触发路径：__eq__ → 尝试 Poly("abc", x, domain=ZZ) → PolynomialError → 返回 False

# 示例4：表达式形式相同但域不同（数学上不相等）
p1 = Poly(x, x, domain=ZZ)      # ZZ[x]
p2 = Poly(x, x, domain=QQ)      # QQ[x]
result = p1 == p2
# 结果：False
# 触发路径：__eq__ → 尝试 unify(ZZ, QQ) → 成功（返回 QQ）→ 转换后比较
# 注意：这里实际上会返回 True！因为 ZZ 和 QQ 可以统一到 QQ
# 让我再找一个真正无法统一的例子：

p1 = Poly(x, x, domain=GF(3))   # GF(3)[x]
p2 = Poly(x, x, domain=ZZ)      # ZZ[x]
result = p1 == p2
# 结果：False
# 触发路径：__eq__ → 尝试 unify(GF(3), ZZ) → UnificationFailed → 返回 False
```

### 3.5 相等性判断的特殊情况

**可以统一的情况**：
```python
from sympy import Poly, ZZ, QQ
from sympy.abc import x

p1 = Poly(x**2 + 1, x, domain=ZZ)
p2 = Poly(x**2 + 1, x, domain=QQ)
result = p1 == p2
# 结果：True（！）
# 原因：ZZ 和 QQ 可以统一到 QQ
# 流程：unify(ZZ, QQ) → 返回 QQ → p1.convert(QQ) → p2.convert(QQ) → 比较相等
```

**设计考量**：
- 这是一个有意的设计决策：如果两个多项式可以通过域转换变得相同，则认为它们相等
- 这与 Python 的数值比较一致：`5 == 5.0` 返回 `True`

---

## 4. 算术运算详细分析

### 4.1 核心代码

算术运算涉及两层处理：
1. `_polifyit` 装饰器：处理非 Poly 类型操作数
2. 核心方法（`add`, `sub`, `mul` 等）：处理 DMP 层面的统一

#### 4.1.1 `_polifyit` 装饰器

**代码位置**：`sympy/polys/polytools.py:72-105`

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
                        operations is deprecated. Either explicitly convert
                        the non-Poly operand to a Poly with as_poly() or
                        convert the Poly to an Expr with as_expr().
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

#### 4.1.2 核心运算方法

**代码位置**：`sympy/polys/polyclasses.py:549-612`

```python
def add(f, g: Self, /) -> Self:
    """Add two multivariate polynomials ``f`` and ``g``. """
    F, G = f.unify_DMP(g)  # 可能抛出 UnificationFailed
    return F._add(G)

def sub(f, g: Self, /) -> Self:
    """Subtract two multivariate polynomials ``f`` and ``g``. """
    F, G = f.unify_DMP(g)  # 可能抛出 UnificationFailed
    return F._sub(G)

def mul(f, g: Self, /) -> Self:
    """Multiply two multivariate polynomials ``f`` and ``g``. """
    F, G = f.unify_DMP(g)  # 可能抛出 UnificationFailed
    return F._mul(G)

# ... div, rem, quo, exquo, pdiv, prem, pquo, pexquo 等方法类似
```

#### 4.1.3 `unify_DMP` 方法

**代码位置**：`sympy/polys/polyclasses.py:328-337`

```python
def unify_DMP(f, g: DMP[Es]) -> tuple[DMP[Et], DMP[Et]]:
    """Unify and return ``DMP`` instances of ``f`` and ``g``. """
    
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

### 4.2 处理流程决策树

```
算术运算（以 __add__ 为例）
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 第一层：_polifyit 装饰器                                     │
│ 目标：将非 Poly 操作数转换为 Poly                            │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤1：g 是 Poly 类型？                                      │
│    if isinstance(g, Poly):                                  │
└────────────────────────────────────────────────────────────┘
    │
    ├── Yes ──▶ 直接调用 func(f, g) → 进入第二层
    │
    └── No ──▶ 继续
              │
              ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤2：g 是 Integer？                                       │
    │    elif isinstance(g, Integer):                            │
    └────────────────────────────────────────────────────────────┘
              │
              ├── Yes ──▶ g = f.from_expr(g, *f.gens, domain=f.domain)
              │              │
              │         总是成功（Integer 可以在任何域表示）
              │              │
              │              ▼
              │         调用 func(f, g) → 进入第二层
              │
              └── No ──▶ 继续
                        │
                        ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤3：g 是 Expr？                                          │
    │    elif isinstance(g, Expr):                               │
    └────────────────────────────────────────────────────────────┘
                        │
                        ├── No ──▶ return NotImplemented
                        │
                        └── Yes ──▶ 尝试转换：g = f.from_expr(g, *f.gens)
                                    │
                             ┌──────┴──────┐
                             │             │
                           成功          失败
                             │             │
                             ▼             ▼
                        func(f, g)   ┌─────────────────────┐
                        → 进入第二层  │ 检查是否是 Matrix    │
                                      └─────────────────────┘
                                                    │
                                             ┌──────┴──────┐
                                             │             │
                                           Yes            No
                                             │             │
                                             ▼             ▼
                                      return    降级到 Expr 运算
                                      NotImplemented   (已废弃)
                                                      │
                                                      ▼
                                                expr_method = getattr(f.as_expr(), func.__name__)
                                                result = expr_method(g)
                                                返回 result 或 NotImplemented
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 第二层：核心运算方法（如 add）                                │
│ 目标：统一域并执行运算                                        │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 调用 unify_DMP(f, g)                                         │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤1：类型或级别匹配？                                       │
│    if not isinstance(g, DMP) or f.lev != g.lev:            │
└────────────────────────────────────────────────────────────┘
    │
    ├── No ──▶ 抛出 UnificationFailed
    │
    └── Yes ──▶ 继续
              │
              ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤2：域相同？                                              │
    │    if f.dom == g.dom:                                       │
    └────────────────────────────────────────────────────────────┘
              │
              ├── Yes ──▶ 返回 (f, g) → 执行运算
              │
              └── No ──▶ 尝试统一：dom = f.dom.unify(g.dom)
                            │
                     ┌──────┴──────┐
                     │             │
                   成功          失败
                     │             │
                     ▼             ▼
                转换后执行     抛出 UnificationFailed
                运算
```

### 4.3 为什么这样处理？

**设计哲学："兼容优先"**

1. **多级降级策略**：
   - **第一级**：尝试保持 Poly 类型（最精确）
   - **第二级**：降级到 Expr 类型（兼容但失去精确性）
   - **第三级**：返回 `NotImplemented`（让 Python 处理）

2. **`NotImplemented` 的意义**：
   - 告诉 Python："我不知道如何处理这个操作数"
   - Python 会尝试反向运算（如 `g.__radd__(f)`）
   - 如果两边都返回 `NotImplemented`，Python 会抛出 `TypeError`

3. **废弃的 Expr 降级**：
   - 历史原因：早期版本允许 `Poly(x) + sin(x)` 这种混合运算
   - 但这会导致语义模糊：结果应该是 Poly 还是 Expr？
   - 现在已废弃，建议用户显式转换：`p.as_expr() + sin(x)`

### 4.4 行为差异示例

```python
from sympy import Poly, GF, sin, Matrix, Integer
from sympy.abc import x, y

# 示例1：与 Integer 运算（总是成功）
p = Poly(x**2 + 1, x, domain=ZZ)
result = p + Integer(5)
# 结果：Poly(x**2 + 6, x, domain='ZZ')
# 触发路径：_polifyit → isinstance(g, Integer) → f.from_expr(g, *f.gens, domain=f.domain)
# 注意：Integer 可以在任何域表示，所以总是成功

# 示例2：与简单 Expr 运算（可以转换为 Poly）
p = Poly(x**2 + 1, x, domain=ZZ)
result = p + 2*x
# 结果：Poly(x**2 + 2*x + 1, x, domain='ZZ')
# 触发路径：_polifyit → isinstance(g, Expr) → 尝试 f.from_expr(2*x, x) → 成功
# → 调用 func(f, g) → 执行运算

# 示例3：与 Matrix 运算（返回 NotImplemented）
p = Poly(x**2 + 1, x)
M = Matrix([[1, 2], [3, 4]])
result = p + M
# 结果：NotImplemented（然后 Python 抛出 TypeError）
# 触发路径：_polifyit → isinstance(g, Expr) → 尝试 f.from_expr(M, x) → PolynomialError
# → 检查 g.is_Matrix → True → 返回 NotImplemented

# 示例4：与超越函数运算（降级到 Expr，已废弃）
p = Poly(x**2 + 1, x)
result = p + sin(x)
# 结果：x**2 + sin(x) + 1（Expr 类型，伴随 DeprecationWarning）
# 触发路径：_polifyit → isinstance(g, Expr) → 尝试 f.from_expr(sin(x), x) → PolynomialError
# → 检查 g.is_Matrix → False → 降级到 Expr 运算
# → expr_method = f.as_expr().__add__ → result = (x**2 + 1) + sin(x)
# → 发出 DeprecationWarning，返回 result

# 示例5：域不同且无法统一（抛出 UnificationFailed）
p1 = Poly(x**2 + 1, x, domain=GF(3))
p2 = Poly(x + 1, x, domain=ZZ)
result = p1 + p2
# 结果：抛出 UnificationFailed
# 触发路径：_polifyit → isinstance(g, Poly) → 调用 func(f, g)
# → add(f, g) → unify_DMP(f, g) → f.dom.unify(g.dom) = unify(GF(3), ZZ)
# → 特征不同（3 vs 0）→ 抛出 UnificationFailed

# 示例6：域不同但可以统一（成功）
p1 = Poly(x**2 + 1, x, domain=ZZ)
p2 = Poly(x + 1, x, domain=QQ)
result = p1 + p2
# 结果：Poly(x**2 + x + 2, x, domain='QQ')
# 触发路径：_polifyit → isinstance(g, Poly) → 调用 func(f, g)
# → add(f, g) → unify_DMP(f, g) → f.dom.unify(g.dom) = unify(ZZ, QQ) → QQ
# → 转换到 QQ → 执行加法
```

### 4.5 `NotImplemented` vs 异常

| 行为 | 触发场景 | Python 后续行为 |
|------|---------|----------------|
| `return NotImplemented` | 操作数类型不支持 | 尝试反向运算（`g.__radd__(f)`），如果都返回 `NotImplemented` 则抛出 `TypeError` |
| `raise UnificationFailed` | 操作数支持但域无法统一 | 直接抛出异常，不尝试反向运算 |
| `raise TypeError` | 完全不支持的操作 | 直接抛出异常 |

**为什么区分 `NotImplemented` 和异常？**

```python
# 场景1：Poly + Matrix（Matrix 可能知道如何处理）
p = Poly(x**2 + 1, x)
M = Matrix([[1, 2], [3, 4]])
result = p + M  # 返回 NotImplemented

# Python 会自动尝试：
result = M + p  # 调用 M.__radd__(p)
# Matrix 可能知道如何将 Poly 元素添加到矩阵中

# 场景2：Poly(GF(3)) + Poly(ZZ)（两边都是 Poly，但无法统一）
p1 = Poly(x**2 + 1, x, domain=GF(3))
p2 = Poly(x + 1, x, domain=ZZ)
result = p1 + p2  # 抛出 UnificationFailed

# 即使尝试反向运算也无济于事：
# p2 + p1 也会抛出 UnificationFailed
# 所以直接抛出异常更高效
```

---

## 5. 构造流程详细分析

### 5.1 核心代码

#### 5.1.1 `_poly_from_expr`（单个表达式构造）

**代码位置**：`sympy/polys/polytools.py:4765-4802`

```python
def _poly_from_expr(expr, opt):
    """Construct a polynomial from an expression. """
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
        # 自动选择最小覆盖域
        opt.domain, coeffs = construct_domain(coeffs, opt=opt)
    else:
        # 使用用户指定的域
        coeffs = list(map(domain.from_sympy, coeffs))
    
    # 步骤3：构造 Poly
    rep = dict(list(zip(monoms, coeffs)))
    poly = Poly._from_dict(rep, opt)
    
    if opt.polys is None:
        opt.polys = False
    
    return poly, opt
```

#### 5.1.2 `_parallel_poly_from_expr`（多个表达式构造）

**代码位置**：`sympy/polys/polytools.py:4812-4903`

```python
def _parallel_poly_from_expr(exprs, opt):
    """Construct polynomials from expressions. """
    
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
    
    # 一般情况：处理多个表达式
    origs, exprs = list(exprs), []
    _exprs, _polys = [], []
    failed = False
    
    # 步骤1：分类处理
    for i, expr in enumerate(origs):
        expr = sympify(expr)
        
        if isinstance(expr, Basic):
            if expr.is_Poly:
                _polys.append(i)
            else:
                _exprs.append(i)
                if opt.expand:
                    expr = expr.expand()
        else:
            failed = True
        
        exprs.append(expr)
    
    # 情况1：有非 Basic 类型
    if failed:
        raise PolificationFailed(opt, origs, exprs, True)
    
    # 情况2：有 Poly 对象（临时解决方案）
    if _polys:
        for i in _polys:
            exprs[i] = exprs[i].as_expr()
    
    # 步骤2：并行提取单项式和系数
    reps, opt = _parallel_dict_from_expr(exprs, opt)
    
    # 情况3：没有生成器
    if not opt.gens:
        raise PolificationFailed(opt, origs, exprs, True)
    
    # 检查生成器类型
    from sympy.functions.elementary.piecewise import Piecewise
    for k in opt.gens:
        if isinstance(k, Piecewise):
            raise PolynomialError("Piecewise generators do not make sense")
    
    # 步骤3：收集所有系数
    coeffs_list, lengths = [], []
    all_monoms = []
    all_coeffs = []
    
    for rep in reps:
        monoms, coeffs = list(zip(*list(rep.items())))
        coeffs_list.extend(coeffs)
        all_monoms.append(monoms)
        lengths.append(len(coeffs))
    
    # 步骤4：选择域
    domain = opt.domain
    
    if domain is None:
        # 自动选择最小覆盖域（所有系数的公共域）
        opt.domain, coeffs_list = construct_domain(coeffs_list, opt=opt)
    else:
        # 使用用户指定的域
        coeffs_list = list(map(domain.from_sympy, coeffs_list))
    
    # 步骤5：分配系数回各个多项式
    for k in lengths:
        all_coeffs.append(coeffs_list[:k])
        coeffs_list = coeffs_list[k:]
    
    # 步骤6：构造各个 Poly
    polys = []
    for monoms, coeffs in zip(all_monoms, all_coeffs):
        rep = dict(list(zip(monoms, coeffs)))
        poly = Poly._from_dict(rep, opt)
        polys.append(poly)
    
    if opt.polys is None:
        opt.polys = bool(_polys)
    
    return polys, opt
```

### 5.2 处理流程决策树

```
构造流程（以 _poly_from_expr 为例）
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤1：类型检查                                              │
│    if not isinstance(expr, Basic):                          │
└────────────────────────────────────────────────────────────┘
    │
    ├── Yes ──▶ 继续
    │
    └── No ──▶ 抛出 PolificationFailed
              │
              ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤2：已经是 Poly？                                        │
    │    elif expr.is_Poly:                                       │
    └────────────────────────────────────────────────────────────┘
              │
              ├── Yes ──▶ 从 Poly 转换，返回
              │
              └── No ──▶ 继续
                        │
                        ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤3：需要展开？                                            │
    │    elif opt.expand:                                          │
    └────────────────────────────────────────────────────────────┘
                        │
                        ├── Yes ──▶ expr = expr.expand()
                        │
                        └── No ──▶ 继续
                                  │
                                  ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤4：提取单项式和系数                                       │
    │    rep, opt = _dict_from_expr(expr, opt)                    │
    └────────────────────────────────────────────────────────────┘
                                  │
                                  ├── 失败 ──▶ 抛出 PolificationFailed
                                  │
                                  └── 成功 ──▶ 继续
                                            │
                                            ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤5：有生成器？                                            │
    │    if not opt.gens:                                          │
    └────────────────────────────────────────────────────────────┘
                                            │
                                            ├── No ──▶ 抛出 PolificationFailed
                                            │
                                            └── Yes ──▶ 继续
                                                      │
                                                      ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤6：选择域                                                │
    │    if domain is None:                                        │
    │        opt.domain, coeffs = construct_domain(coeffs, opt)  │
    │    else:                                                     │
    │        coeffs = list(map(domain.from_sympy, coeffs))        │
    └────────────────────────────────────────────────────────────┘
                                                      │
                                                      ├── construct_domain 失败 ──▶ 降级到 EX
                                                      │         （construct_domain 内部处理）
                                                      │
                                                      └── 成功 ──▶ 继续
                                                                │
                                                                ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤7：构造 Poly                                             │
    │    poly = Poly._from_dict(rep, opt)                         │
    └────────────────────────────────────────────────────────────┘
                                                                │
                                                                ▼
                                                          返回 (poly, opt)
```

### 5.3 为什么这样处理？

**设计哲学："精确构造"**

1. **构造是一个"严格"的操作**：
   - 用户明确要求"将这个表达式转换为多项式"
   - 如果无法精确表示，应该明确报错，而不是静默降级
   - 这与 `int("abc")` 抛出 `ValueError` 而不是返回 `0` 是一致的

2. **`PolificationFailed` vs 其他异常**：
   - `PolificationFailed`：专门用于多项式构造失败
   - 包含详细的上下文信息（原始表达式、选项等）
   - 调用方可以捕获并处理

3. **并行构造的特殊处理**：
   - 两个 Poly 对象：直接调用 `unify`，失败则传播异常
   - 多个表达式：先收集所有系数，再统一选择域
   - 这确保所有多项式使用相同的域

### 5.4 行为差异示例

```python
from sympy import Poly, poly_from_expr, parallel_poly_from_expr, GF, sin
from sympy.abc import x, y

# 示例1：简单表达式（成功）
expr = x**2 + 2*x + 1
poly, opt = poly_from_expr(expr, x)
# 结果：Poly(x**2 + 2*x + 1, x, domain='ZZ')
# 触发路径：_poly_from_expr → _dict_from_expr → construct_domain([1, 2, 1]) → ZZ
# → 构造成功

# 示例2：含超越函数（失败）
expr = x**2 + sin(x)
try:
    poly, opt = poly_from_expr(expr, x)
except PolificationFailed as e:
    print(e)  # 抛出异常
# 触发路径：_poly_from_expr → _dict_from_expr → 无法提取单项式 → PolificationFailed

# 示例3：指定域但无法转换（失败）
expr = x**2 + 1/2
try:
    poly, opt = poly_from_expr(expr, x, domain=ZZ)
except CoercionFailed as e:
    print(e)  # 抛出异常
# 触发路径：_poly_from_expr → domain=ZZ 已指定
# → coeffs = list(map(ZZ.from_sympy, [1/2, 1])) → ZZ.from_sympy(1/2) 失败
# → CoercionFailed

# 示例4：自动选择域（包含分数）
expr = x**2 + 1/2
poly, opt = poly_from_expr(expr, x)
# 结果：Poly(x**2 + 1/2, x, domain='QQ')
# 触发路径：_poly_from_expr → domain=None → construct_domain([1/2, 1]) → QQ
# → 构造成功

# 示例5：并行构造两个 Poly（统一域）
p1 = Poly(x**2 + 1, x, domain=ZZ)
p2 = Poly(x + 1, x, domain=QQ)
polys, opt = parallel_poly_from_expr([p1, p2])
# 结果：[Poly(x**2 + 1, x, domain='QQ'), Poly(x + 1, x, domain='QQ')]
# 触发路径：_parallel_poly_from_expr → len(exprs) == 2 且都是 Poly
# → f = f._from_poly(f, opt), g = g._from_poly(g, opt)
# → f, g = f.unify(g) → unify(ZZ, QQ) → QQ
# → 转换到 QQ，返回

# 示例6：并行构造两个 Poly（无法统一域）
p1 = Poly(x**2 + 1, x, domain=GF(3))
p2 = Poly(x + 1, x, domain=ZZ)
try:
    polys, opt = parallel_poly_from_expr([p1, p2])
except UnificationFailed as e:
    print(e)  # 抛出异常
# 触发路径：_parallel_poly_from_expr → f.unify(g) → unify(GF(3), ZZ)
# → 特征不同 → UnificationFailed

# 示例7：并行构造多个表达式（统一选择域）
exprs = [x**2 + 1, x + 1/2]  # 一个整数系数，一个分数系数
polys, opt = parallel_poly_from_expr(exprs, x)
# 结果：[Poly(x**2 + 1, x, domain='QQ'), Poly(x + 1/2, x, domain='QQ')]
# 触发路径：_parallel_poly_from_expr → 收集所有系数 [1, 1, 1, 1/2]
# → construct_domain([1, 1, 1, 1/2]) → QQ
# → 所有多项式使用 QQ 域
```

### 5.5 `construct_domain` 的自动降级

注意：`_poly_from_expr` 本身不会捕获 `construct_domain` 的异常，而是让 `construct_domain` 内部处理降级。

```python
# 回顾 construct_domain 的流程：
# 1. 尝试 _construct_simple（简单域）
# 2. 失败则尝试 _construct_composite（复合域）
# 3. 都失败则使用 _construct_expression（EX 域）

# 示例：浮点数 + 代数数（construct_domain 内部降级到 EX）
from sympy import sqrt, S

expr = x**2 + sqrt(2) + S(3.14)
poly, opt = poly_from_expr(expr, x)
# 结果：Poly(x**2 + 3.14 + sqrt(2), x, domain='EX')
# 触发路径：
# _poly_from_expr → construct_domain([1, sqrt(2)+3.14])
# → _construct_simple → 检测到 floats 和 algebraics 同时为 True → 返回 False
# → _construct_expression → 返回 EX 域
# → 构造成功（使用 EX 域）
```

**关键区别**：
- `construct_domain` 会**自动降级**到 EX（如果需要）
- 但 `_poly_from_expr` 会在更早的阶段（如 `_dict_from_expr` 失败时）**直接抛出异常**
- 这是设计决策：如果表达式根本无法提取单项式结构，则是"真正的失败"；但如果系数可以用 EX 表示，则可以"降级成功"

---

## 6. 三种调用方深度对比

### 6.1 设计哲学对比

| 维度 | 相等性判断 | 算术运算 | 构造流程 |
|------|-----------|---------|---------|
| **核心问题** | "这两个对象相等吗？" | "这两个对象可以运算吗？" | "这个表达式可以表示为多项式吗？" |
| **容错程度** | 最高（从不抛异常） | 中等（多级降级） | 最低（严格报错） |
| **用户期望** | 条件判断应该稳定 | 运算应该尽量兼容 | 构造失败应该明确告知 |
| **数学语义** | 不同域的多项式"不相等" | 不同域的多项式"可能可以运算" | 构造失败意味着"不是多项式" |
| **Python 约定** | `__eq__` 应返回布尔值 | 算术运算可以返回 `NotImplemented` | 构造函数可以抛异常 |

### 6.2 异常处理策略对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 相等性判断（__eq__）                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   输入操作数                                                                   │
│      │                                                                       │
│      ▼                                                                       │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐              │
│   │ 类型检查    │─────▶│ 转换尝试    │─────▶│ 域统一尝试  │              │
│   └─────────────┘      └─────────────┘      └─────────────┘              │
│         │                     │                     │                       │
│         ▼                     ▼                     ▼                       │
│   ┌─────────────────────────────────────────────────────────┐              │
│   │              任何失败 → 捕获异常 → 返回 False              │              │
│   └─────────────────────────────────────────────────────────┘              │
│                                                                              │
│   结果：总是布尔值（True/False）                                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ 算术运算（__add__ 等）                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   第一层：_polifyit 装饰器                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  输入操作数                                                             │  │
│   │     │                                                                   │  │
│   │     ▼                                                                   │  │
│   │  ┌──────────────┐                                                      │  │
│   │  │ 是 Poly？     │────Yes────▶ 进入第二层                              │  │
│   │  └──────────────┘                                                      │  │
│   │     │ No                                                                │  │
│   │     ▼                                                                   │  │
│   │  ┌──────────────┐                                                      │  │
│   │  │ 是 Integer？  │────Yes────▶ 转换为 Poly → 进入第二层                │  │
│   │  └──────────────┘                                                      │  │
│   │     │ No                                                                │  │
│   │     ▼                                                                   │  │
│   │  ┌──────────────┐                                                      │  │
│   │  │ 是 Expr？     │                                                      │  │
│   │  └──────────────┘                                                      │  │
│   │     │                                                                   │  │
│   │  ┌──┴──┐                                                                │  │
│   │  │     │                                                                │  │
│   │ Yes    No                                                               │  │
│   │  │     │                                                                │  │
│   │  ▼     ▼                                                                │  │
│   │ 转换尝试  return NotImplemented                                         │  │
│   │  │                                                                      │  │
│   │  ├──成功──▶ 进入第二层                                                  │  │
│   │  │                                                                      │  │
│   │  └──失败──▶ 检查是否是 Matrix                                           │  │
│   │              │                                                          │  │
│   │         ┌────┴────┐                                                     │  │
│   │         │         │                                                     │  │
│   │        Yes       No                                                     │  │
│   │         │         │                                                     │  │
│   │         ▼         ▼                                                     │  │
│   │  return   降级到 Expr 运算                                              │  │
│   │  NotImplemented  (已废弃)                                                │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   第二层：核心运算方法                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  调用 unify_DMP(f, g)                                                  │  │
│   │     │                                                                   │  │
│   │     ▼                                                                   │  │
│   │  ┌──────────────┐                                                      │  │
│   │  │ 类型/级别匹配？│────No────▶ 抛出 UnificationFailed                  │  │
│   │  └──────────────┘                                                      │  │
│   │     │ Yes                                                               │  │
│   │     ▼                                                                   │  │
│   │  ┌──────────────┐                                                      │  │
│   │  │ 域相同？      │────Yes────▶ 执行运算                                 │  │
│   │  └──────────────┘                                                      │  │
│   │     │ No                                                                │  │
│   │     ▼                                                                   │  │
│   │  尝试统一域                                                              │  │
│   │     │                                                                   │  │
│   │  ┌──┴──┐                                                                │  │
│   │  │     │                                                                │  │
│   │ 成功   失败                                                             │  │
│   │  │     │                                                                │  │
│   │  ▼     ▼                                                                │  │
│   │ 执行   抛出 UnificationFailed                                           │  │
│   │ 运算                                                                    │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   结果：可能成功、返回 NotImplemented、或抛出异常                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ 构造流程（poly_from_expr）                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   输入表达式                                                                   │
│      │                                                                       │
│      ▼                                                                       │
│   ┌─────────────┐                                                           │
│   │ 类型检查    │────不是 Basic────▶ 抛出 PolificationFailed                │
│   └─────────────┘                                                           │
│      │ Yes                                                                   │
│      ▼                                                                       │
│   ┌─────────────┐                                                           │
│   │ 已经是 Poly │────Yes────▶ 转换后返回                                    │
│   └─────────────┘                                                           │
│      │ No                                                                    │
│      ▼                                                                       │
│   ┌─────────────┐                                                           │
│   │ 提取单项式  │────失败────▶ 抛出 PolificationFailed                      │
│   └─────────────┘                                                           │
│      │ 成功                                                                  │
│      ▼                                                                       │
│   ┌─────────────┐                                                           │
│   │ 有生成器？  │────No────▶ 抛出 PolificationFailed                        │
│   └─────────────┘                                                           │
│      │ Yes                                                                   │
│      ▼                                                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ 选择域                                                                   │  │
│   │  - 用户指定域：尝试转换，失败则抛出 CoercionFailed                      │  │
│   │  - 自动选择：construct_domain 内部降级到 EX（如果需要）                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│      │                                                                       │
│      ▼                                                                       │
│   ┌─────────────┐                                                           │
│   │ 构造 Poly   │                                                           │
│   └─────────────┘                                                           │
│      │                                                                       │
│      ▼                                                                       │
│   返回 (poly, opt)                                                           │
│                                                                              │
│   结果：成功返回，或抛出 PolificationFailed/CoercionFailed                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 典型场景行为对比

| 场景 | 相等性判断 (`__eq__`) | 算术运算 (`__add__`) | 构造流程 (`poly_from_expr`) |
|------|----------------------|---------------------|-----------------------------|
| **Poly(ZZ) + Poly(QQ)** | `True`（统一到 QQ） | 成功（统一到 QQ） | - |
| **Poly(GF(3)) + Poly(ZZ)** | `False` | 抛出 `UnificationFailed` | - |
| **Poly + Integer** | `False`（转换失败） | 成功（转换为 Poly） | - |
| **Poly + sin(x)** | `False`（转换失败） | 返回 Expr（已废弃） | 抛出 `PolificationFailed` |
| **Poly + Matrix** | `False`（转换失败） | 返回 `NotImplemented` | - |
| **x**2 + 1/2（自动选域） | - | - | 成功（QQ 域） |
| **x**2 + 1/2（指定 ZZ） | - | - | 抛出 `CoercionFailed` |
| **x**2 + sqrt(2) + 3.14 | - | - | 成功（EX 域） |
| **x**2 + sin(x) | - | - | 抛出 `PolificationFailed` |

### 6.4 代码位置速查

| 功能 | 相等性判断 | 算术运算 | 构造流程 |
|------|-----------|---------|---------|
| **主方法** | `polytools.py:4696-4718` | `polytools.py:4563-4585` | `polytools.py:4758-4903` |
| **装饰器/辅助** | 无 | `polytools.py:72-105` (`_polifyit`) | 无 |
| **核心统一** | `domain.py:889-1016` (`unify`) | `polyclasses.py:328-337` (`unify_DMP`) | `polytools.py:4821` (`unify`) |
| **异常类型** | `UnificationFailed` | `UnificationFailed`, `CoercionFailed`, `NotImplemented` | `PolificationFailed`, `CoercionFailed` |
| **捕获策略** | 捕获所有异常，返回 `False` | 不捕获（传播），但 `_polifyit` 会降级 | 不捕获（传播），但 `construct_domain` 内部降级 |

---

## 7. 设计原则总结

### 7.1 三大设计原则

#### 1. **一致性原则（Consistency）**
- 与 Python 语言约定保持一致
- `__eq__` 应返回布尔值，不抛异常
- 算术运算可以返回 `NotImplemented`

#### 2. **精确性原则（Precision）**
- 相等性判断：如果无法精确比较，返回 `False`
- 算术运算：尽量保持 Poly 类型，必要时降级
- 构造流程：如果无法精确表示，明确报错

#### 3. **兼容性原则（Compatibility）**
- 算术运算支持多级降级：Poly → Expr → NotImplemented
- 构造流程中 `construct_domain` 会自动降级到 EX
- 相等性判断从不抛异常，确保条件判断稳定

### 7.2 异常处理策略选择

| 策略 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **返回 `False`** | 相等性判断 | 符合 Python 约定，用户友好 | 可能掩盖问题 |
| **返回 `NotImplemented`** | 算术运算（类型不支持） | 让 Python 尝试反向运算 | 需要调用方处理 |
| **降级到 Expr** | 算术运算（已废弃） | 保持兼容性 | 语义模糊，已废弃 |
| **抛出异常** | 构造流程、域不兼容 | 明确告知失败原因 | 可能中断程序 |

### 7.3 用户指南

**如何正确处理不同场景？**

1. **相等性判断**：
   - 直接使用 `==`，它永远不会抛异常
   - 注意：不同域的多项式可能返回 `False`，即使表达式看起来相同

2. **算术运算**：
   - 尽量确保两个操作数都是 Poly 类型，且域兼容
   - 如果不确定，可以显式转换：`p.as_expr() + q.as_expr()`
   - 注意：与非多项式表达式的混合运算已废弃

3. **构造流程**：
   - 捕获 `PolificationFailed` 来处理无法构造的情况
   - 让 `construct_domain` 自动选择域（最安全）
   - 指定域时注意：如果表达式无法在该域表示，会抛出 `CoercionFailed`

**示例代码**：
```python
from sympy import Poly, poly_from_expr, PolificationFailed
from sympy.abc import x

# 1. 相等性判断（安全）
p1 = Poly(x**2 + 1, x)
p2 = Poly(x**2 + 1, x)
if p1 == p2:
    print("相等")
else:
    print("不相等")

# 2. 算术运算（注意兼容性）
try:
    p1 = Poly(x**2 + 1, x, domain=GF(3))
    p2 = Poly(x + 1, x, domain=ZZ)
    result = p1 + p2
except UnificationFailed:
    # 显式转换到 Expr 再运算
    result = p1.as_expr() + p2.as_expr()

# 3. 构造流程（捕获异常）
expr = x**2 + sin(x)
try:
    poly, opt = poly_from_expr(expr, x)
except PolificationFailed:
    print("无法构造为多项式，请使用 Expr 类型")
```

---

## 8. 附录：关键代码索引

### 8.1 相等性判断

| 文件 | 行号 | 功能 |
|------|------|------|
| `polytools.py` | 4696-4718 | `Poly.__eq__` 主方法 |
| `polytools.py` | 4710-4713 | 捕获 `UnificationFailed` 并返回 `False` |
| `polytools.py` | 4702-4704 | 捕获转换失败并返回 `False` |

### 8.2 算术运算

| 文件 | 行号 | 功能 |
|------|------|------|
| `polytools.py` | 72-105 | `_polifyit` 装饰器 |
| `polytools.py` | 4563-4565 | `Poly.__add__` |
| `polytools.py` | 4567-4569 | `Poly.__radd__` |
| `polytools.py` | 4571-4573 | `Poly.__sub__` |
| `polytools.py` | 4579-4581 | `Poly.__mul__` |
| `polyclasses.py` | 328-337 | `DMP.unify_DMP` |
| `polyclasses.py` | 549-552 | `DMP.add` |
| `polyclasses.py` | 554-557 | `DMP.sub` |
| `polyclasses.py` | 559-562 | `DMP.mul` |

### 8.3 构造流程

| 文件 | 行号 | 功能 |
|------|------|------|
| `polytools.py` | 4758-4762 | `poly_from_expr` |
| `polytools.py` | 4765-4802 | `_poly_from_expr` |
| `polytools.py` | 4806-4809 | `parallel_poly_from_expr` |
| `polytools.py` | 4812-4903 | `_parallel_poly_from_expr` |
| `polytools.py` | 4817-4829 | 两个 Poly 的特殊处理（调用 `unify`） |
| `constructor.py` | 268-388 | `construct_domain` 主流程 |
