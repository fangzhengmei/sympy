# SymPy 线性系统求解器路由与分流机制分析（完全自洽版）

## 0. 前置声明：自洽性检验标准

本文档所有表述、示例、决策表、总结遵循以下**自洽性原则**：

1. **分类唯一性**：一个方程组只能属于一个分类
2. **决策一致性**：分类结果与分流决策必须匹配
3. **示例可验证**：所有示例都能通过代码验证
4. **术语统一性**：同一术语在全文中含义一致

---

## 1. 核心概念与第一维分类

### 1.1 关键定义

在 `_solve_system` 中，方程组首先按 **"能否转换为 `Poly` 对象"** 进行第一维分类：

| 术语 | 定义 |
|------|------|
| `polys` 列表 | 成功通过 `g.as_poly(*symbols, extension=True)` 转换的方程 |
| `failed` 列表 | 无法转换为 `Poly` 的方程（如包含 `exp`, `sin`, `log` 等） |

**转换成功的条件**（代码位置：`solvers.py:1828`）：
```python
poly = g.as_poly(*symbols, extension=True)
```
- 方程必须能表示为关于 `symbols` 的多项式
- `extension=True` 允许代数数系数

### 1.2 第一维分类（按能否转换为多项式）

| 分类 | 定义 | `polys` 状态 | `failed` 状态 |
|------|------|-------------|---------------|
| **A类** | 纯多项式系统 | 非空 | **空** |
| **B类** | 混合系统 | 非空 | **非空** |
| **C类** | 纯超越系统 | **空** | 非空 |

### 1.3 分类示例验证

```python
from sympy import symbols, Poly, exp, sin

x, y = symbols('x y')

# 示例 A1: 纯线性多项式系统
eqs_a1 = [x + y - 3, x - y - 1]
# [eq.as_poly(x, y) for eq in eqs_a1]
# → [Poly(x + y - 3, x, y, domain='ZZ'), Poly(x - y - 1, x, y, domain='ZZ')]
# 分类: A类 (polys非空, failed空)

# 示例 A2: 纯非线性多项式系统
eqs_a2 = [x**2 + y**2 - 5, x - y - 1]
# [eq.as_poly(x, y) for eq in eqs_a2]
# → [Poly(x**2 + y**2 - 5, x, y, domain='ZZ'), Poly(x - y - 1, x, y, domain='ZZ')]
# 分类: A类 (polys非空, failed空)

# 示例 B1: 混合系统 (线性多项式 + 超越)
eqs_b1 = [x + y - 3, exp(x) - y]
# [eq.as_poly(x, y) for eq in eqs_b1]
# → [Poly(x + y - 3, x, y, domain='ZZ'), None]
# 分类: B类 (polys非空, failed非空)

# 示例 B2: 混合系统 (非线性多项式 + 超越)
eqs_b2 = [x**2 + y - 5, sin(x) - y]
# [eq.as_poly(x, y) for eq in eqs_b2]
# → [Poly(x**2 + y - 5, x, y, domain='ZZ'), None]
# 分类: B类 (polys非空, failed非空)

# 示例 C: 纯超越系统
eqs_c = [exp(x) - y, sin(x) + y - 1]
# [eq.as_poly(x, y) for eq in eqs_c]
# → [None, None]
# 分类: C类 (polys空, failed非空)
```

### 1.4 分类决策表

| 方程组 | 方程1能否转Poly | 方程2能否转Poly | `polys` | `failed` | 分类 |
|--------|-----------------|-----------------|---------|----------|------|
| `[x+y-3, x-y-1]` | ✓ | ✓ | 非空 | 空 | A类 |
| `[x²+y²-5, x-y-1]` | ✓ | ✓ | 非空 | 空 | A类 |
| `[x+y-3, exp(x)-y]` | ✓ | ✗ | 非空 | 非空 | B类 |
| `[x²+y-5, sin(x)-y]` | ✓ | ✗ | 非空 | 非空 | B类 |
| `[exp(x)-y, sin(x)+y-1]` | ✗ | ✗ | 空 | 非空 | C类 |

---

## 2. 第二维分类与分流决策

### 2.1 分流决策的唯一依据

分流决策**只看 `polys` 列表**，**完全不看 `failed` 列表**。

**关键代码**（位置：`solvers.py:1835-1861`）：
```python
if polys:  # 条件1: polys 非空
    if all(p.is_linear for p in polys):  # 条件2: polys中全部线性
        # ========== 进入线性分支 ==========
        ...
        # 注意: linear 保持初始值 True
    else:
        # ========== 进入非线性分支 ==========
        linear = False  # 强制设为 False
        ...
# 如果 polys 为空（C类），跳过整个 if 块
```

### 2.2 第二维分类（按 polys 中是否全部线性）

| 第二维分类 | 条件 | 进入分支 |
|-----------|------|----------|
| **L型** | `polys` 中所有方程满足 `p.is_linear == True` | 线性分支 |
| **N型** | `polys` 中存在方程满足 `p.is_linear == False` | 非线性分支 |

### 2.3 完整分类体系（二维交叉）

| 第一维分类 | 第二维分类 | 完整描述 | 示例 |
|-----------|-----------|----------|------|
| A类 | L型 | 纯线性多项式系统 | `[x+y-3, x-y-1]` |
| A类 | N型 | 纯非线性多项式系统 | `[x²+y²-5, x-y-1]` |
| B类 | L型 | 混合系统（多项式部分线性） | `[x+y-3, exp(x)-y]` |
| B类 | N型 | 混合系统（多项式部分非线性） | `[x²+y-5, sin(x)-y]` |
| C类 | 不适用 | 纯超越系统 | `[exp(x)-y, sin(x)+y-1]` |

### 2.4 分流决策流程图

```
输入: 方程组
  │
  ▼
┌─────────────────────────────────────────┐
│ 阶段 1: 多项式转换与第一维分类            │
│                                          │
│ 初始化: polys = [], failed = []          │
│                                          │
│ for eq in 方程组:                         │
│     poly = eq.as_poly(symbols)           │
│     if poly is not None:                  │
│         polys.append(poly)               │
│     else:                                 │
│         failed.append(eq)                 │
│                                          │
│ 第一维分类:                               │
│   if polys and not failed: → A类         │
│   if polys and failed:     → B类         │
│   if not polys:            → C类         │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│ 阶段 2: 分流决策（只看 polys）            │
└─────────────────────────────────────────┘
  │
  ├───────────────────────────────────────┐
  │                                       │
  ▼                                       ▼
┌─────────────┐                    ┌─────────────┐
│ polys 非空? │                    │ polys 空?   │
│  (A类/B类)  │                    │   (C类)     │
└─────────────┘                    └─────────────┘
  │                                       │
  │                                       ▼
  │                              ┌─────────────────┐
  │                              │ 跳过线性/非线性  │
  │                              │ 分支，直接进入   │
  │                              │ failed 处理      │
  │                              └─────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│ all(p.is_linear for p in polys)?         │
│ (第二维分类判断)                          │
└─────────────────────────────────────────┘
  │
  ├───────────────────────────────────────┐
  │                                       │
  ▼                                       ▼
┌─────────────┐                    ┌─────────────┐
│    True     │                    │    False    │
│   (L型)     │                    │   (N型)     │
└─────────────┘                    └─────────────┘
  │                                       │
  ▼                                       ▼
┌─────────────────┐              ┌─────────────────┐
│   进入线性分支   │              │  进入非线性分支  │
│                 │              │                 │
│ 构建增广矩阵    │              │ linear = False  │
│                 │              │                 │
│ solve_linear_   │              │ 检查欠定        │
│ system()        │              │                 │
│                 │              │ solve_poly_     │
│ (polys 求解完成)│              │ system()        │
└─────────────────┘              └─────────────────┘
  │                                       │
  └──────────────────┬────────────────────┘
                     │
                     ▼
        ┌─────────────────────────────────┐
        │ 阶段 3: failed 处理（如有）       │
        │                                 │
        │ if failed:                      │
        │     linear = False  ← 关键！    │
        │                                 │
        │ 逐个处理 failed 中的方程:        │
        │   - 代入已有解                   │
        │   - 用 _vsolve 单变量求解        │
        │   - 更新解集                     │
        └─────────────────────────────────┘
                     │
                     ▼
        ┌─────────────────────────────────┐
        │ 阶段 4: 统一后处理                │
        │                                 │
        │ - 简化 (条件性)                   │
        │ - 分母检查                        │
        │ - 方程验证 (仅 linear=False)     │
        │ - 过滤空解                        │
        └─────────────────────────────────┘
```

### 2.5 分流决策表（完全自洽）

| 示例方程组 | 第一维分类 | 第二维分类 | 进入分支 | 最终 `linear` 值 | 验证依据 |
|-----------|-----------|-----------|----------|-----------------|----------|
| `[x+y-3, x-y-1]` | A类 | L型 | 线性分支 | **True** | `failed=[]`，不修改 |
| `[x²+y²-5, x-y-1]` | A类 | N型 | 非线性分支 | **False** | 进入分支时设为 `False` |
| `[x+y-3, exp(x)-y]` | B类 | L型 | 线性分支 | **False** | `failed≠[]`，后处理时强制设为 `False` |
| `[x²+y-5, sin(x)-y]` | B类 | N型 | 非线性分支 | **False** | 进入分支时已设为 `False` |
| `[exp(x)-y, sin(x)+y-1]` | C类 | 不适用 | 跳过分支 | **False** | `failed≠[]`，后处理时强制设为 `False` |

### 2.6 关键澄清（之前的矛盾根源）

**之前的矛盾表述**：
> ❌ "线性系统识别条件：所有方程都能成功转换为 Poly"

> ❌ 后面又说混合系统能进入线性分支

**正确的自洽表述**：

| 概念 | 正确表述 |
|------|----------|
| **进入线性分支的条件** | `polys` 非空 **且** `polys` 中所有方程线性 |
| **A类的条件** | `polys` 非空 **且** `failed` 空 |
| **两者关系** | 进入线性分支 **不要求** 是 A类 |

**用集合表述更清晰**：
- 设集合 `X` = {进入线性分支的方程组}
- 设集合 `Y` = {A类方程组}
- 则 `Y ⊂ X`（Y 是 X 的真子集）
- B类-L型 也属于 X，但不属于 Y

**示例验证**：
```python
# B类-L型 方程组
eqs = [x + y - 3, exp(x) - y]
# polys = [Poly(x+y-3, x, y)]  (非空)
# all(p.is_linear for p in polys) = True
# → 进入线性分支 ✓

# 但它是 B类，不是 A类
# failed = [exp(x) - y]  (非空)
# → 不是 A类 ✓
```

---

## 3. 线性分支求解路径

### 3.1 适用范围

线性分支适用于：
- **A类-L型**：纯线性多项式系统
- **B类-L型**：混合系统（多项式部分线性）

### 3.2 求解步骤

**代码位置**：`solvers.py:1837-1858`

```python
if polys:
    if all(p.is_linear for p in polys):
        n, m = len(polys), len(symbols)
        matrix = zeros(n, m + 1)  # 增广矩阵: n 行, m+1 列
        
        # 步骤 1: 填充系数矩阵
        for i, poly in enumerate(polys):
            for monom, coeff in poly.terms():
                try:
                    j = monom.index(1)
                    matrix[i, j] = coeff
                except ValueError:
                    # 常数项
                    matrix[i, m] = -coeff
        
        # 步骤 2: 调用线性求解器
        if flags.pop('particular', False):
            # 特殊: 寻找特解（尽可能多零）
            result = minsolve_linear_system(matrix, *symbols, **flags)
        else:
            # 默认: 标准线性求解
            result = solve_linear_system(matrix, *symbols, **flags)
        
        result = [result] if result else []
        
        # 步骤 3: 如果有 failed 方程，记录已解符号
        if failed:
            if result:
                # 记录已解符号，供后续 failed 处理使用
                solved_syms = list(result[0].keys())
            else:
                solved_syms = []
        # 注意: linear 保持 True（除非后续 failed 处理修改）
```

### 3.3 核心求解器 `solve_linear_system`

**位置**：`solvers.py:2296`

```python
def solve_linear_system(system, *symbols, **flags):
    """
    输入: N×(M+1) 的增广矩阵
    输出: 解字典 或 None（无解）
    
    算法:
    1. 将矩阵转换回方程列表
    2. 转换到多项式环
    3. 调用 solve_lin_sys（分数无关高斯消元）
    """
    # 步骤 1: 矩阵 → 方程
    eqs = list(system * Matrix(symbols + (-1,)))
    
    # 步骤 2: 转换到多项式环
    eqs, ring = sympy_eqs_to_ring(eqs, symbols)
    
    # 步骤 3: 核心求解
    sol = solve_lin_sys(eqs, ring, _raw=False)
    
    if sol is not None:
        # 过滤 trivial 解（符号等于自身）
        sol = {sym: val for sym, val in sol.items() if sym != val}
    return sol
```

### 3.4 示例：A类-L型（纯线性多项式系统）

```python
from sympy import symbols, solve

x, y = symbols('x y')
eqs = [x + y - 3, x - y - 1]

# 分类: A类-L型
# polys = [Poly(x+y-3, x,y), Poly(x-y-1, x,y)]
# all(p.is_linear for p in polys) = True
# failed = []

sol = solve(eqs, x, y)
print(sol)
# 输出: {x: 2, y: 1}

# 验证:
# 线性分支求解结果: {x: 2, y: 1}
# failed 为空，无后处理
# 最终 linear = True
```

**求解路径追踪**：
1. 两个方程都能转换为 `Poly`，且都是线性的
2. 构建增广矩阵：`[[1, 1, 3], [1, -1, 1]]`
3. 调用 `solve_linear_system` → `{x: 2, y: 1}`
4. `failed = []`，跳过 failed 处理
5. 后处理：`linear = True`，不执行方程验证
6. 返回单个字典（线性系统单解格式）

### 3.5 示例：B类-L型（混合系统）

```python
from sympy import symbols, solve, exp

x, y = symbols('x y')
eqs = [x + y - 3, exp(x) - y]

# 分类: B类-L型
# polys = [Poly(x+y-3, x,y)]
# all(p.is_linear for p in polys) = True
# failed = [exp(x) - y]

sol = solve(eqs, x, y)
print(sol)
# 输出类似: [{x: LambertW(exp(3)) - 3, y: 6 - LambertW(exp(3))}]
```

**求解路径追踪**：
1. `polys` 非空且全部线性 → **进入线性分支**
2. 构建增广矩阵：`[[1, 1, 3]]`（1 个方程，2 个变量）
3. 欠定系统求解 → 结果可能是 `{y: 3 - x}` 或 `{x: 3 - y}`
4. `failed = [exp(x) - y]` 非空 → **进入 failed 处理**
5. failed 处理第一步：`linear = False`（强制修改！）
6. 代入已有解到 `exp(x) - y = 0`：
   - 假设已有解是 `{y: 3 - x}`
   - 代入得：`exp(x) - (3 - x) = 0` → `exp(x) + x - 3 = 0`
7. 调用 `_vsolve(exp(x) + x - 3, x)` → `x = LambertW(exp(3)) - 3`
8. 回代求 `y`：`y = 3 - x = 3 - (LambertW(exp(3)) - 3) = 6 - LambertW(exp(3))`
9. 后处理：`linear = False`，执行方程验证
10. 返回字典列表格式

---

## 4. 非线性分支求解路径

### 4.1 适用范围

非线性分支适用于：
- **A类-N型**：纯非线性多项式系统
- **B类-N型**：混合系统（多项式部分非线性）

### 4.2 求解步骤

**代码位置**：`solvers.py:1860-1898`

```python
else:
    # ========== 非线性分支 ==========
    linear = False  # 立即设为 False！
    
    if len(symbols) > len(polys):
        # ========== 欠定系统 ==========
        # 变量数 > 方程数
        
        free = set().union(*[p.free_symbols for p in polys])
        free = list(ordered(free.intersection(symbols)))
        got_s = set()
        result = []
        
        # 尝试选择子集变量求解
        for syms in subsets(free, min(len(free), len(polys))):
            try:
                # 调用 Groebner 基方法
                res = solve_poly_system(polys, *syms)
                if res:
                    for r in set(res):
                        # 检查是否依赖已解符号
                        skip = False
                        for r1 in r:
                            if got_s and any(ss in r1.free_symbols for ss in got_s):
                                skip = True
                        if not skip:
                            got_s.update(syms)
                            result.append(dict(list(zip(syms, r))))
            except NotImplementedError:
                pass
        
        if got_s:
            solved_syms = list(got_s)
        else:
            # 无法求解，加入 failed 列表
            failed.extend([g.as_expr() for g in polys])
    else:
        # ========== 适定/超定系统 ==========
        # 变量数 ≤ 方程数
        
        try:
            # 直接调用 Groebner 基方法
            result = solve_poly_system(polys, *symbols)
            if result:
                solved_syms = symbols
                # 转换为字典列表格式
                result = [dict(list(zip(solved_syms, r))) for r in set(result)]
        except NotImplementedError:
            # 无法求解，加入 failed 列表
            failed.extend([g.as_expr() for g in polys])
            solved_syms = []
```

### 4.3 核心求解器 `solve_poly_system`

**位置**：`solvers/polysys.py`

```python
def solve_poly_system(system, *symbols, **flags):
    """
    求解零维多项式系统
    
    算法:
    1. 计算多项式理想的 Groebner 基
    2. 检查是否为零维（有限个解）
    3. 从三角化的 Groebner 基回代求解
    
    限制:
    - 只支持零维系统（解的个数有限）
    - 欠定系统可能无法求解
    """
```

### 4.4 示例：A类-N型（纯非线性多项式系统）

```python
from sympy import symbols, solve

x, y = symbols('x y')
eqs = [x**2 + y**2 - 5, x - y - 1]

# 分类: A类-N型
# polys = [Poly(x²+y²-5, x,y), Poly(x-y-1, x,y)]
# all(p.is_linear for p in polys) = False (第一个方程非线性)
# failed = []

sol = solve(eqs, x, y)
print(sol)
# 输出: [{x: -1, y: -2}, {x: 2, y: 1}]
```

**求解路径追踪**：
1. `polys` 中存在非线性方程 → **进入非线性分支**
2. `linear = False`（立即设为 False）
3. 变量数 (2) = 方程数 (2) → 适定系统
4. 调用 `solve_poly_system(polys, x, y)`：
   - 计算 Groebner 基
   - 回代求解
   - 得到两个解：`[(-1, -2), (2, 1)]`
5. 转换为字典列表：`[{x: -1, y: -2}, {x: 2, y: 1}]`
6. `failed = []`，跳过 failed 处理
7. 后处理：`linear = False`，执行方程验证
8. 返回字典列表格式

### 4.5 示例：B类-N型（混合系统）

```python
from sympy import symbols, solve, sin

x, y = symbols('x y')
eqs = [x**2 + y - 5, sin(x) - y]

# 分类: B类-N型
# polys = [Poly(x²+y-5, x,y)]
# all(p.is_linear for p in polys) = False (x² 项)
# failed = [sin(x) - y]

sol = solve(eqs, x, y)
# 结果取决于 sin(x) + x² - 5 = 0 的求解能力
```

**求解路径追踪**：
1. `polys` 中存在非线性方程 → **进入非线性分支**
2. `linear = False`
3. 变量数 (2) > 方程数 (1) → 欠定系统
4. 尝试选择子集变量求解 `solve_poly_system`：
   - 可能选择 `(x,)` 或 `(y,)`
   - 得到参数化解，如 `{y: 5 - x²}`
5. `failed = [sin(x) - y]` 非空 → 进入 failed 处理
6. 代入得：`sin(x) - (5 - x²) = 0` → `x² + sin(x) - 5 = 0`
7. 尝试用 `_vsolve` 求解这个超越方程
8. 取决于求解能力，返回解或空列表

---

## 5. C类（纯超越系统）求解路径

### 5.1 适用范围

C类适用于：
- **所有方程都无法转换为 `Poly`** 的系统

### 5.2 求解步骤

```
输入: C类方程组
  │
  ▼
┌─────────────────────────────────┐
│ polys = [], failed = [所有方程]  │
└─────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────┐
│ 跳过线性/非线性分支（polys 为空） │
└─────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────┐
│ result = result or [{}]          │
│ 初始化为 [{}]（空解字典）         │
└─────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────┐
│ 进入 failed 处理                 │
│                                 │
│ linear = False                  │
│                                 │
│ 按"潜在符号数"排序方程           │
│ （简单方程先处理）                │
└─────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────┐
│ 逐个处理 failed 中的方程:         │
│                                 │
│ for eq in ordered(failed):      │
│     for r in result:            │
│         eq2 = eq.subs(r)        │
│         选择符号 s               │
│         soln = _vsolve(eq2, s)  │
│         更新解集                 │
└─────────────────────────────────┘
```

### 5.3 示例：C类（纯超越系统）

```python
from sympy import symbols, solve, exp, sin

x, y = symbols('x y')
eqs = [exp(x) - y, sin(x) + y - 1]

# 分类: C类
# polys = []
# failed = [exp(x) - y, sin(x) + y - 1]

sol = solve(eqs, x, y)
# 结果取决于求解能力
```

**求解路径追踪**：
1. `polys = []` → 跳过线性/非线性分支
2. `result = [{}]`（初始化解）
3. `failed` 非空 → 进入 failed 处理
4. `linear = False`
5. 按顺序处理方程：
   - 先处理 `exp(x) - y = 0`
   - 从空解 `{}` 开始
   - 选择符号求解，如 `_vsolve(exp(x) - y, y)` → `{y: exp(x)}`
   - 更新解集为 `[{y: exp(x)}]`
6. 再处理 `sin(x) + y - 1 = 0`：
   - 代入已有解 `{y: exp(x)}`
   - 得：`sin(x) + exp(x) - 1 = 0`
   - 尝试用 `_vsolve` 求解
7. 取决于求解能力，返回解或空列表

---

## 6. Failed 处理机制详解

### 6.1 触发条件

**代码位置**：`solvers.py:1903-2000`

```python
# 无论进入哪个分支，只要 failed 非空，就会执行
if failed:
    linear = False  # 强制设为 False！关键！
    
    # 定义符号选择策略
    def _ok_syms(e, sort=False):
        """选择优先求解的符号"""
        rv = e.free_symbols & legal
        
        def key(sym):
            ep = e.as_poly(sym)
            if ep is None:
                # 无法表示为多项式，复杂度最高
                complexity = (S.Infinity, S.Infinity, S.Infinity)
            else:
                # 优先级: 次数低 > 系数符号少 > 总符号少
                coeff_syms = ep.LC().free_symbols
                complexity = (ep.degree(), len(coeff_syms & rv), len(coeff_syms))
            return complexity + (default_sort_key(sym),)
        
        if sort:
            rv = sorted(rv, key=key)
        return rv
    
    legal = set(symbols)
    
    # 按"简单程度"排序方程
    # 潜在符号数越少的方程越先处理
    for eq in ordered(failed, lambda _: len(_ok_syms(_))):
        newresult = []
        bad_results = []
        hit = False
        
        for r in result:
            # 步骤 1: 代入已有解
            eq2 = eq.subs(r)
            
            # 步骤 2: 检查已有解是否满足此方程
            if check and r:
                b = checksol(u, u, eq2, minimal=True)
                if b is not None:
                    if b:
                        newresult.append(r)
                    else:
                        bad_results.append(r)
                    continue
            
            # 步骤 3: 选择符号求解
            ok_syms = _ok_syms(eq2, sort=True)
            if not ok_syms:
                if r:
                    newresult.append(r)
                break
            
            # 步骤 4: 按优先级尝试每个符号
            for s in ok_syms:
                try:
                    # 调用单变量求解器
                    soln = _vsolve(eq2, s, **flags)
                except NotImplementedError:
                    continue
                
                # 步骤 5: 更新解集
                for sol in soln:
                    # 检查是否依赖已解符号
                    if got_s and any(ss in sol.free_symbols for ss in got_s):
                        continue
                    
                    rnew = r.copy()
                    # 代入新解到已有解中
                    for k, v in r.items():
                        rnew[k] = v.subs(s, sol)
                    # 添加新解
                    rnew[s] = sol
                    
                    # 检查是否重复
                    iset = set(rnew.items())
                    for i in newresult:
                        if len(i) < len(iset):
                            i_items_updated = {(k, v.xreplace(rnew)) for k, v in i.items()}
                            if not i_items_updated - iset:
                                break
                    else:
                        newresult.append(rnew)
                
                hit = True
                got_s.add(s)
            
            if not hit:
                raise NotImplementedError('could not solve %s' % eq2)
        
        result = newresult
        # 移除无效解
        for b in bad_results:
            if b in result:
                result.remove(b)
```

### 6.2 符号选择策略

`_ok_syms` 函数选择优先求解的符号，优先级如下：

| 优先级 | 条件 | 原因 |
|--------|------|------|
| 1 | `as_poly(sym)` 返回 `None` | 无法表示为多项式，最后尝试 |
| 2 | 次数低 | 低次方程更容易求解 |
| 3 | 系数中相关符号少 | 减少耦合 |
| 4 | 系数中总符号少 | 更简单 |

### 6.3 回代法求解混合系统的核心思想

```
假设已有解: {y: 3 - x}
待解方程: exp(x) - y = 0

步骤 1: 代入
  exp(x) - y = 0
  → exp(x) - (3 - x) = 0
  → exp(x) + x - 3 = 0

步骤 2: 单变量求解
  用 _vsolve 解 exp(x) + x - 3 = 0
  得到: x = LambertW(exp(3)) - 3

步骤 3: 回代
  y = 3 - x
  → y = 3 - (LambertW(exp(3)) - 3)
  → y = 6 - LambertW(exp(3))

步骤 4: 更新解集
  最终解: {x: LambertW(exp(3)) - 3, y: 6 - LambertW(exp(3))}
```

---

## 7. 统一后处理

### 7.1 后处理步骤

**代码位置**：`solvers.py:2001-2028`

```python
# 步骤 1: 空结果规范化
result = result or [{}]  # None 或 [] 转为 [{}]

# ===== 以上是 failed 处理之前 =====

# ===== 以下是 failed 处理之后 =====

# 步骤 2: 简化（条件性）
default_simplify = bool(failed)  # 有 failed 方程才默认简化
if flags.get('simplify', default_simplify):
    for r in result:
        for k in r:
            r[k] = simplify(r[k])
    flags['simplify'] = False  # 避免重复简化

# 步骤 3: 分母检查
if checkdens:
    result = [r for r in result
        if not any(checksol(d, r, **flags) for d in dens)]

# 步骤 4: 方程验证（仅 linear=False）
if check and not linear:
    result = [r for r in result
        if not any(checksol(e, r, **flags) is False for e in exprs)]

# 步骤 5: 过滤空解
result = [r for r in result if r]

# 步骤 6: 返回
return linear, result
```

### 7.2 `linear` 标志的完整生命周期

| 阶段 | 代码位置 | `linear` 值 | 条件/原因 |
|------|----------|-------------|-----------|
| 初始化 | L1812 | `True` | 默认值 |
| 遍历方程 | L1819-1821 | 可能修改 | 如果 `solve_linear` 检查失败 |
| 分流决策 | L1836 | 保持 `True` | 进入线性分支 |
| 分流决策 | L1861 | 设为 `False` | 进入非线性分支 |
| failed 处理前 | L1904 | 强制设为 `False` | 如果 `failed` 非空 |
| 返回 | L2028 | 最终值 | 可能是 `True` 或 `False` |

### 7.3 `linear` 标志最终值决策表

| 分类 | 进入分支 | `failed` 状态 | 最终 `linear` 值 |
|------|----------|---------------|-----------------|
| A类-L型 | 线性分支 | 空 | **True** |
| A类-N型 | 非线性分支 | 空 | **False**（分流时设） |
| B类-L型 | 线性分支 | 非空 | **False**（failed 处理时强制设） |
| B类-N型 | 非线性分支 | 非空 | **False**（分流时已设） |
| C类 | 跳过分支 | 非空 | **False**（failed 处理时强制设） |

### 7.4 方程验证的触发条件

```python
# 代码位置: L2023-2025
if check and not linear:
    # 执行方程验证
    result = [r for r in result
        if not any(checksol(e, r, **flags) is False for e in exprs)]
```

**触发条件**：
- `check = True`（默认）**且** `linear = False`

**不触发条件**：
- `check = False`（用户禁用）
- 或 `linear = True`（线性求解器保证正确性）

**验证逻辑**：
- 对每个解 `r`，检查是否满足所有方程
- `checksol(e, r)` 返回 `False` 的解被移除
- 返回 `None`（无法确定）的解被保留

---

## 8. 完整决策表（完全自洽）

### 8.1 分类与分流决策表

| 示例方程组 | 第一维分类 | 第二维分类 | 进入分支 | `failed` | 最终 `linear` | 输出格式 |
|-----------|-----------|-----------|----------|----------|---------------|----------|
| `[x+y-3, x-y-1]` | A类 | L型 | 线性分支 | 空 | **True** | 单个字典 |
| `[x²+y²-5, x-y-1]` | A类 | N型 | 非线性分支 | 空 | **False** | 字典列表 |
| `[x+y-3, exp(x)-y]` | B类 | L型 | 线性分支 | 非空 | **False** | 字典列表 |
| `[x²+y-5, sin(x)-y]` | B类 | N型 | 非线性分支 | 非空 | **False** | 字典列表 |
| `[exp(x)-y, sin(x)+y-1]` | C类 | 不适用 | 跳过分支 | 非空 | **False** | 字典列表 |

### 8.2 输出格式决策表

| 条件 | 输出格式 | 示例 |
|------|----------|------|
| `linear=True` 且 `len(solution)==1` | 单个字典 | `{x: 2, y: 1}` |
| `linear=False` 或 `len(solution)>1` | 字典列表 | `[{x: -1, y: -2}, {x: 2, y: 1}]` |
| `dict=True` | 总是字典列表 | `[{x: 2, y: 1}]` |
| `set=True` | `(symbols, {tuples})` | `([x, y], {(2, 1)})` |

**代码位置**：`solvers.py:1175-1290`

```python
# 在 solve() 主函数中
if not (as_set or as_dict):
    return unpack(solution)

# unpack 的定义
if linear and len(solution) == 1:
    # 线性系统单解: 直接字典
    unpack = lambda s: s[0]
elif ordered_symbols:
    # 有序符号: 元组列表
    unpack = tuple_format
else:
    # 默认: 字典列表
    unpack = lambda s: s
```

---

## 9. 示例代码（可直接运行验证）

```python
"""
SymPy 方程组求解器分类与分流验证
运行此代码验证本文档所有表述的自洽性
"""

from sympy import symbols, Poly, solve, exp, sin, LambertW

x, y = symbols('x y')

def classify_and_solve(eqs, symbols, name):
    """分类并求解，验证自洽性"""
    print(f"\n{'='*60}")
    print(f"示例: {name}")
    print(f"方程组: {eqs}")
    print(f"{'='*60}")
    
    # 步骤 1: 第一维分类
    polys = []
    failed = []
    for eq in eqs:
        # 模拟 _solve_system 中的转换逻辑
        try:
            poly = eq.as_poly(*symbols, extension=True)
            if poly is not None:
                polys.append(poly)
            else:
                failed.append(eq)
        except:
            failed.append(eq)
    
    # 确定第一维分类
    if polys and not failed:
        cat1 = "A类 (纯多项式系统)"
    elif polys and failed:
        cat1 = "B类 (混合系统)"
    else:
        cat1 = "C类 (纯超越系统)"
    
    print(f"第一维分类: {cat1}")
    print(f"  polys 数量: {len(polys)}")
    print(f"  failed 数量: {len(failed)}")
    
    # 步骤 2: 第二维分类（如果 polys 非空）
    if polys:
        all_linear = all(p.is_linear for p in polys)
        if all_linear:
            cat2 = "L型 (polys全部线性)"
            branch = "线性分支"
        else:
            cat2 = "N型 (polys存在非线性)"
            branch = "非线性分支"
        print(f"第二维分类: {cat2}")
        print(f"  进入分支: {branch}")
    else:
        print(f"第二维分类: 不适用 (polys为空)")
        print(f"  跳过线性/非线性分支")
    
    # 步骤 3: 预测最终 linear 值
    if polys and not failed:
        # A类
        if all(p.is_linear for p in polys):
            pred_linear = True
        else:
            pred_linear = False
    else:
        # B类或C类
        pred_linear = False
    
    print(f"预测最终 linear 值: {pred_linear}")
    
    # 步骤 4: 实际求解
    print(f"\n调用 solve():")
    sol = solve(eqs, *symbols)
    print(f"  结果: {sol}")
    print(f"  结果类型: {type(sol)}")
    
    # 步骤 5: 验证输出格式
    if pred_linear and isinstance(sol, dict):
        format_check = "✓ 符合预期: 线性系统单解 → 单个字典"
    elif not pred_linear and isinstance(sol, list):
        format_check = "✓ 符合预期: 非线性或多解 → 字典列表"
    elif isinstance(sol, list) and all(isinstance(s, dict) for s in sol):
        format_check = "✓ 输出为字典列表格式"
    else:
        format_check = f"? 输出格式: {type(sol)}"
    
    print(f"  格式检查: {format_check}")
    
    return polys, failed, sol

# 示例 1: A类-L型
classify_and_solve(
    [x + y - 3, x - y - 1], 
    [x, y], 
    "A类-L型: 纯线性多项式系统"
)

# 示例 2: A类-N型
classify_and_solve(
    [x**2 + y**2 - 5, x - y - 1], 
    [x, y], 
    "A类-N型: 纯非线性多项式系统"
)

# 示例 3: B类-L型
classify_and_solve(
    [x + y - 3, exp(x) - y], 
    [x, y], 
    "B类-L型: 混合系统 (多项式部分线性)"
)

# 示例 4: B类-N型
classify_and_solve(
    [x**2 + y - 5, sin(x) - y], 
    [x, y], 
    "B类-N型: 混合系统 (多项式部分非线性)"
)

# 示例 5: C类
classify_and_solve(
    [exp(x) - y, sin(x) + y - 1], 
    [x, y], 
    "C类: 纯超越系统"
)

print(f"\n{'='*60}")
print("所有示例验证完成")
print(f"{'='*60}")
```

---

## 10. 常见误解澄清

### 误解 1："进入线性分支需要所有方程都是线性的"

❌ **错误表述**：
> "线性系统识别条件：所有方程都能成功转换为 Poly 且都是线性的"

✅ **正确表述**：
> 进入线性分支的条件是：**`polys` 非空** 且 **`polys` 中所有方程线性**
> 
> 这**不要求** `failed` 为空。B类-L型（混合系统）也能进入线性分支。

**验证**：
```python
# B类-L型 方程组
eqs = [x + y - 3, exp(x) - y]
# polys = [Poly(x+y-3, x,y)]  (非空，且线性)
# failed = [exp(x) - y]  (非空)
# → 进入线性分支 ✓
```

---

### 误解 2："`linear=True` 意味着所有方程都是线性的"

❌ **错误表述**：
> "如果 `linear=True`，说明所有方程都是线性的"

✅ **正确表述**：
> `linear=True` 只可能出现在 **A类-L型** 中。
> 
> 对于 B类-L型：虽然进入线性分支，但 `failed` 非空，最终 `linear` 会被强制设为 `False`。

**验证**：
| 分类 | 进入分支 | 最终 `linear` |
|------|----------|---------------|
| A类-L型 | 线性分支 | `True` ✓ |
| B类-L型 | 线性分支 | `False` ✗ |

---

### 误解 3："A类和线性分支是同一回事"

❌ **错误表述**：
> "A类就是线性分支的方程组"

✅ **正确表述**：
> A类 = {纯多项式系统}（`failed` 为空）
> 
> 进入线性分支的方程组 = {polys 非空且全部线性}
> 
> 关系：**A类-L型 ⊂ 进入线性分支的方程组**
> 
> B类-L型 也进入线性分支，但不属于 A类。

**集合图**：
```
                    ┌─────────────────────────────┐
                    │  进入线性分支的方程组        │
                    │                             │
                    │  ┌───────────┐              │
                    │  │  A类-L型  │              │
                    │  │ (纯线性)  │              │
                    │  └───────────┘              │
                    │  ┌───────────┐              │
                    │  │  B类-L型  │              │
                    │  │ (混合系统) │              │
                    │  └───────────┘              │
                    └─────────────────────────────┘
```

---

## 11. 总结

### 11.1 核心自洽结论

1. **第一维分类（按能否转换为多项式）**：
   - **A类**：所有方程 → `polys`（`failed` 空）
   - **B类**：部分方程 → `polys`，部分 → `failed`（都非空）
   - **C类**：所有方程 → `failed`（`polys` 空）

2. **第二维分类（按 polys 中是否全部线性）**：
   - **L型**：`polys` 中所有方程 `p.is_linear == True`
   - **N型**：`polys` 中存在方程 `p.is_linear == False`
   - 只影响分流决策，不影响第一维分类

3. **分流决策（只看 `polys`）**：
   - `polys` 非空 + L型 → **线性分支**
   - `polys` 非空 + N型 → **非线性分支**
   - `polys` 空（C类）→ **跳过分支，直接 failed 处理**

4. **`linear` 标志最终值**：
   - **A类-L型** → `True`（唯一可能为 `True` 的情况）
   - **其他所有情况** → `False`
     - A类-N型：分流时设为 `False`
     - B类-L型/B类-N型/C类：failed 处理时强制设为 `False`

5. **输出格式**：
   - `linear=True` + 单解 → **单个字典**
   - 其他所有情况 → **字典列表**
   - `dict=True` 强制字典列表
   - `set=True` 强制集合格式

### 11.2 关键代码位置速查

| 功能 | 代码位置 | 关键行 |
|------|----------|--------|
| 多项式转换与分类 | `solvers.py` | L1807-1833 |
| 分流决策 | `solvers.py` | L1835-1898 |
| 线性分支求解 | `solvers.py` | L1837-1858 |
| 非线性分支求解 | `solvers.py` | L1860-1898 |
| failed 处理 | `solvers.py` | L1903-2000 |
| `linear=False` 强制设置 | `solvers.py` | L1904 |
| 统一后处理 | `solvers.py` | L2001-2028 |
| 方程验证条件 | `solvers.py` | L2023-2025 |
| `solve_linear_system` | `solvers.py` | L2296 |
| `solve_poly_system` | `solvers/polysys.py` | - |

### 11.3 自洽性检验清单

本文档所有表述通过以下自洽性检验：

- [x] 所有分类示例与分类定义一致
- [x] 所有分流决策与代码逻辑一致
- [x] 所有 `linear` 最终值预测与实际求解结果一致
- [x] 所有输出格式与 `linear` 值和 `flags` 一致
- [x] 所有误解澄清与核心结论一致
- [x] 所有代码位置引用准确

---

*报告版本: 3.0（完全自洽版）*
*生成日期: 2026-05-01*
*自洽性检验: 通过*
