# SymPy 线性系统求解器路由与分流机制分析

## 1. 概述

本文档详细分析 SymPy 方程组求解器中**线性系统**与**非线性系统**的准确识别、路由决策、求解路径和结果统一处理机制。这是对上一版报告中线性系统部分的修正和补充。

**核心入口函数**:
- 传统求解器: `_solve_system()` (`sympy/solvers/solvers.py:1763`)
- 现代求解器: `linsolve()` (`sympy/solvers/solveset.py:2897`)

---

## 2. 识别条件

### 2.1 方程组进入系统求解的条件

**位置**: `sympy/solvers/solvers.py:1163-1170`

```python
# solve() 函数中的关键判断
if bare_f:
    # 单方程: 走 _solve_undetermined 或 _solve
    solution = None
    if len(symbols) != 1:
        solution = _solve_undetermined(f[0], symbols, flags)
    if not solution:
        solution = _solve(f[0], *symbols, **flags)
else:
    # 方程组: 走 _solve_system
    linear, solution = _solve_system(f, symbols, **flags)
```

**识别为方程组的条件**:
- `bare_f = False`，即传入的是可迭代对象（列表、元组、矩阵等）
- 例如: `solve([x + y - 3, x - y - 1], x, y)`

### 2.2 线性系统的精确识别条件

**位置**: `sympy/solvers/solvers.py:1807-1833`

在 `_solve_system` 内部，线性系统识别分为两步：

#### 第一步：尝试转换为多项式

```python
polys = []
failed = []

for j, g in enumerate(exprs):
    # ... 反演和规范化处理 ...
    g = d - i
    g = g.as_numer_denom()[0]  # 取分子
    
    # 关键：尝试转换为多项式
    poly = g.as_poly(*symbols, extension=True)
    
    if poly is not None:
        polys.append(poly)
    else:
        failed.append(g)  # 无法转换为多项式的方程
```

**转换成功的条件**:
- 方程必须能表示为 `Poly` 对象
- `extension=True` 允许代数数系数

#### 第二步：检查是否全部为线性

**位置**: `sympy/solvers/solvers.py:1835-1836`

```python
if polys:
    if all(p.is_linear for p in polys):
        # 进入线性求解分支
        ...
    else:
        # 进入非线性求解分支
        linear = False
        ...
```

**最终线性系统识别条件**:
1. **所有方程**都能成功转换为 `Poly` 对象（`failed` 列表可能为空或包含超越方程）
2. **所有转换成功的多项式**都满足 `p.is_linear == True`

### 2.3 关键概念辨析

| 概念 | 说明 |
|------|------|
| `polys` 列表 | 成功转换为 `Poly` 的方程 |
| `failed` 列表 | 无法转换为 `Poly` 的方程（如包含 `exp`, `sin`, `log` 等） |
| `p.is_linear` | 多项式是否为一次（每个变量的次数 ≤ 1，无交叉项） |
| `linear` 标志 | 返回值，表示整个系统是否被当作线性系统处理 |

### 2.4 最小示例对照

让我们通过具体例子理解识别过程：

#### 示例 1：纯线性系统

```python
from sympy import symbols, solve
x, y = symbols('x y')

# 方程：x + y - 3 = 0, x - y - 1 = 0
solve([x + y - 3, x - y - 1], x, y)
```

**识别过程**:
1. 两个方程都是多项式
2. 转换为 `Poly` 成功
3. 检查 `is_linear`:
   - `Poly(x + y - 3).is_linear` → `True`
   - `Poly(x - y - 1).is_linear` → `True`
4. **进入线性求解分支**

#### 示例 2：非线性多项式系统

```python
# 方程：x**2 + y**2 - 5 = 0, x - y - 1 = 0
solve([x**2 + y**2 - 5, x - y - 1], x, y)
```

**识别过程**:
1. 两个方程都能转换为 `Poly`
2. 检查 `is_linear`:
   - `Poly(x**2 + y**2 - 5).is_linear` → `False`（x² 项）
   - `Poly(x - y - 1).is_linear` → `True`
3. **进入非线性求解分支**

#### 示例 3：混合系统（线性 + 超越）

```python
# 方程：x + y - 3 = 0, exp(x) - y = 0
solve([x + y - 3, exp(x) - y], x, y)
```

**识别过程**:
1. 方程 1 (`x + y - 3`) 能转换为 `Poly` 且 `is_linear=True`
2. 方程 2 (`exp(x) - y`) 无法转换为 `Poly`（包含 `exp`）
3. 所以:
   - `polys = [Poly(x + y - 3)]` （全部是线性的）
   - `failed = [exp(x) - y]`
4. **进入线性求解分支**（因为 `polys` 中全部是线性的）
5. 线性求解完成后，再用 `_vsolve` 处理 `failed` 中的超越方程

---

## 3. 路由决策

### 3.1 完整路由流程图

```
输入: 方程组 exprs, 符号 symbols
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  第 1 层：系统分解 (可选)                                      │
│  位置: solvers.py:1771-1805                                   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
    ┌────────────────┐
    │  是否可分解？    │
    │ (共享符号分组)  │
    └────────────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
  可分解    不可分解
    │         │
    ▼         │
┌──────────┐  │
│递归处理  │  │
│各子系统  │  │
│结果组合  │  │
└──────────┘  │
    │         │
    └────┬────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  第 2 层：多项式转换与分类                                     │
│  位置: solvers.py:1807-1833                                   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
    ┌────────────────────────┐
    │  遍历每个方程:           │
    │  1. _invert() 反演      │
    │  2. as_numer_denom()[0] │
    │  3. as_poly() 尝试转换   │
    └────────────────────────┘
         │
         ▼
    ┌────────────────┐
    │  分类结果:       │
    │  polys = [...]  │  ✓ 成功转换为 Poly
    │  failed = [...] │  ✗ 无法转换（超越方程等）
    └────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  第 3 层：核心分流决策点                                      │
│  位置: solvers.py:1835-1898                                   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
    ┌────────────────────────┐
    │  polys 非空?            │
    │  (有成功转换的方程)      │
    └────────────────────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
   是        否
    │         │
    ▼         │
┌──────────┐  │
│ all(p.   │  │
│ is_linear│  │
│ for p in │  │
│ polys)?  │  │
└──────────┘  │
    │         │
┌───┴───┐     │
│       │     │
▼       ▼     │
线性   非线性  │
分支    分支   │
│       │     │
│       │     │
└───┬───┘     │
    │         │
    └────┬────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  第 4 层：failed 方程后处理                                   │
│  位置: solvers.py:1900-2000                                   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
    ┌────────────────┐
    │ failed 非空?   │
    │ (有超越方程)    │
    └────────────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
   是        否
    │         │
    ▼         │
┌──────────┐  │
│ 逐个解决  │  │
│ failed中 │  │
│ 的方程   │  │
│ (回代法) │  │
└──────────┘  │
    │         │
    └────┬────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  第 5 层：统一后处理                                           │
│  位置: solvers.py:2001-2028                                   │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 关键分流点代码解析

**位置**: `sympy/solvers/solvers.py:1835-1898`

```python
if polys:
    if all(p.is_linear for p in polys):
        # =============================================
        # 分支 A：线性系统求解
        # =============================================
        n, m = len(polys), len(symbols)
        matrix = zeros(n, m + 1)  # 增广矩阵
        
        # 构建系数矩阵
        for i, poly in enumerate(polys):
            for monom, coeff in poly.terms():
                try:
                    j = monom.index(1)
                    matrix[i, j] = coeff
                except ValueError:
                    matrix[i, m] = -coeff
        
        # 调用线性求解器
        if flags.pop('particular', False):
            result = minsolve_linear_system(matrix, *symbols, **flags)
        else:
            result = solve_linear_system(matrix, *symbols, **flags)
        
        result = [result] if result else []
        
        # 记录已解符号，供 failed 处理使用
        if failed:
            if result:
                solved_syms = list(result[0].keys())
            else:
                solved_syms = []
        # linear 保持 True
        
    else:
        # =============================================
        # 分支 B：非线性多项式系统求解
        # =============================================
        linear = False  # 标记为非线性
        
        if len(symbols) > len(polys):
            # 欠定系统：变量数 > 方程数
            # 尝试选择子集变量求解
            free = set().union(*[p.free_symbols for p in polys])
            free = list(ordered(free.intersection(symbols)))
            got_s = set()
            result = []
            
            for syms in subsets(free, min(len(free), len(polys))):
                try:
                    # 尝试用 Groebner 基求解
                    res = solve_poly_system(polys, *syms)
                    if res:
                        # 处理结果...
                        result.append(dict(list(zip(syms, r))))
                except NotImplementedError:
                    pass
            
            if not got_s:
                # 无法求解，加入 failed 列表
                failed.extend([g.as_expr() for g in polys])
        else:
            # 适定/超定系统：变量数 ≤ 方程数
            try:
                # 直接调用 solve_poly_system
                result = solve_poly_system(polys, *symbols)
                if result:
                    solved_syms = symbols
                    result = [dict(list(zip(solved_syms, r))) for r in set(result)]
            except NotImplementedError:
                failed.extend([g.as_expr() for g in polys])
                solved_syms = []
```

### 3.3 分流决策表

| 条件 | 进入分支 | `linear` 标志 |
|------|----------|---------------|
| `polys` 中所有方程都是线性的 | 线性分支 | `True` |
| `polys` 中存在非线性方程 | 非线性分支 | `False` |
| 所有方程都无法转换为 `Poly` | 直接进入 failed 处理 | `False` |

**重要**: `linear` 标志只反映 `polys` 列表中方程的性质，不考虑 `failed` 列表。

---

## 4. 求解路径

### 4.1 线性系统求解路径

#### 路径概览

```
线性分支入口
     │
     ▼
┌─────────────────┐
│ 步骤 1:         │
│ 构建增广矩阵     │
│ zeros(n, m+1)   │
└─────────────────┘
     │
     ▼
┌─────────────────┐
│ 步骤 2:         │
│ 填充系数矩阵     │
│ poly.terms()    │
└─────────────────┘
     │
     ▼
┌─────────────────┐
│ 步骤 3:         │
│ particular=True?│
└─────────────────┘
     │
    ┌┴──────────────┐
    │               │
    ▼               ▼
┌─────────┐   ┌───────────────┐
│ 是      │   │ 否 (默认)      │
│         │   │               │
│ minsolve│   │ solve_linear_ │
│ _linear │   │ system        │
│ _system │   │               │
└─────────┘   └───────────────┘
    │               │
    └───────┬───────┘
            │
            ▼
    ┌───────────────┐
    │ 结果:          │
    │ 字典或 None    │
    │ 包装为 [result]│
    └───────────────┘
```

#### 核心求解器：`solve_linear_system`

**位置**: `sympy/solvers/solvers.py:2296`

```python
def solve_linear_system(system, *symbols, **flags):
    """
    求解 N 个线性方程，M 个变量的系统
    
    输入: N×(M+1) 的增广矩阵
    输出: 解字典 或 None（无解）
    """
    assert system.shape[1] == len(symbols) + 1
    
    # 这是 solve_lin_sys 的包装器
    # 步骤 1: 将矩阵转换回方程列表
    eqs = list(system * Matrix(symbols + (-1,)))
    
    # 步骤 2: 转换到多项式环
    eqs, ring = sympy_eqs_to_ring(eqs, symbols)
    
    # 步骤 3: 调用核心线性求解器
    sol = solve_lin_sys(eqs, ring, _raw=False)
    
    if sol is not None:
        # 过滤掉 trivial 解（符号等于自身）
        sol = {sym: val for sym, val in sol.items() if sym != val}
    return sol
```

#### 底层求解器：`solve_lin_sys`

**位置**: `sympy/polys/solvers.py`

这是实际执行高斯消元的核心函数：

```python
def solve_lin_sys(eqs, ring, _raw=True):
    """
    使用分数无关高斯消元法求解线性系统
    
    算法:
    1. 构建系数矩阵
    2. 行约简为上三角形式
    3. 回代求解
    4. 处理自由变量（欠定系统）
    """
```

### 4.2 非线性多项式系统求解路径

#### 路径概览

```
非线性分支入口
     │
     ▼
┌──────────────────────┐
│ 变量数 > 方程数?       │
│ (欠定系统判断)         │
└──────────────────────┘
     │
    ┌┴──────────────────┐
    │                    │
    ▼                    ▼
┌──────────┐      ┌───────────────┐
│ 是        │      │ 否             │
│           │      │                │
│ 尝试子集   │      │ 直接求解        │
│ 变量求解   │      │                │
└──────────┘      └───────────────┘
    │                    │
    └──────────┬─────────┘
               │
               ▼
    ┌──────────────────┐
    │ 调用              │
    │ solve_poly_system │
    │ (Groebner 基)     │
    └──────────────────┘
               │
               ▼
    ┌──────────────────┐
    │ 结果:             │
    │ 元组列表 或       │
    │ NotImplementedError│
    └──────────────────┘
```

#### 核心求解器：`solve_poly_system`

**位置**: `sympy/solvers/polysys.py`

```python
def solve_poly_system(system, *symbols, **flags):
    """
    求解零维多项式系统
    
    算法:
    1. 计算多项式理想的 Groebner 基
    2. 检查是否为零维（有限个解）
    3. 从三角化的 Groebner 基回代求解
    """
```

### 4.3 Failed 方程处理路径（超越方程等）

#### 路径概览

```
failed 列表非空
     │
     ▼
┌────────────────────────┐
│ 标记 linear = False     │
│ (整个系统视为非线性)     │
└────────────────────────┘
     │
     ▼
┌────────────────────────┐
│ 按"潜在符号数"排序方程   │
│ (简单方程先处理)         │
└────────────────────────┘
     │
     ▼
┌────────────────────────┐
│ 遍历每个 failed 方程:    │
│                        │
│ for eq in ordered(failed│
│         , lambda _: len │
│         (_ok_syms(_))): │
└────────────────────────┘
     │
     ▼
┌────────────────────────┐
│ 步骤 1: 代入已有解       │
│ eq2 = eq.subs(result)   │
└────────────────────────┘
     │
     ▼
┌────────────────────────┐
│ 步骤 2: 检查解的有效性   │
│ (check=True 时)          │
└────────────────────────┘
     │
    ┌┴──────────────────┐
    │                    │
    ▼                    ▼
┌──────────┐      ┌───────────────┐
│ 有效      │      │ 无效           │
│          │      │                │
│ 保留     │      │ 丢弃           │
└──────────┘      └───────────────┘
    │                    │
    └──────────┬─────────┘
               │
               ▼
    ┌──────────────────┐
    │ 步骤 3: 选择符号    │
    │ 优先选:            │
    │ - 低次项           │
    │ - 有理系数         │
    └──────────────────┘
               │
               ▼
    ┌──────────────────┐
    │ 步骤 4: 调用       │
    │ _vsolve(eq2, s)   │
    │ (单变量求解器)     │
    └──────────────────┘
               │
               ▼
    ┌──────────────────┐
    │ 步骤 5: 更新解集   │
    │ 每个解:            │
    │ rnew = r.copy()   │
    │ rnew[s] = sol      │
    └──────────────────┘
               │
               ▼
    ┌──────────────────┐
    │ 结果:             │
    │ 扩展的解字典列表   │
    └──────────────────┘
```

#### 关键代码解析

**位置**: `sympy/solvers/solvers.py:1903-2000`

```python
if failed:
    linear = False  # 整个系统标记为非线性
    
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
    for eq in ordered(failed, lambda _: len(_ok_syms(_))):
        newresult = []
        bad_results = []
        hit = False
        
        for r in result:
            # 代入已有解
            eq2 = eq.subs(r)
            
            # 检查已有解是否满足此方程
            if check and r:
                b = checksol(u, u, eq2, minimal=True)
                if b is not None:
                    if b:
                        newresult.append(r)
                    else:
                        bad_results.append(r)
                    continue
            
            # 选择符号求解
            ok_syms = _ok_syms(eq2, sort=True)
            if not ok_syms:
                if r:
                    newresult.append(r)
                break
            
            for s in ok_syms:
                try:
                    # 调用单变量求解器
                    soln = _vsolve(eq2, s, **flags)
                except NotImplementedError:
                    continue
                
                # 更新解集合
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

### 4.4 最小示例追踪

让我们追踪一个**混合系统**（线性 + 超越）的完整求解路径：

#### 示例

```python
from sympy import symbols, solve, exp
x, y = symbols('x y')

# 混合系统: 1个线性方程 + 1个超越方程
solution = solve([x + y - 3, exp(x) - y], x, y)
print(solution)
```

#### 步骤 1：进入 `_solve_system`

```python
# exprs = [x + y - 3, exp(x) - y]
# symbols = [x, y]
```

#### 步骤 2：系统分解检查

- 两个方程共享符号 `x, y`
- 不可分解，继续

#### 步骤 3：多项式转换与分类

遍历第一个方程 `x + y - 3`:
```python
g = x + y - 3
poly = g.as_poly(x, y, extension=True)
# Poly(x + y - 3, x, y, domain='ZZ')
# poly.is_linear = True
polys.append(poly)
```

遍历第二个方程 `exp(x) - y`:
```python
g = exp(x) - y
poly = g.as_poly(x, y, extension=True)
# None (因为包含 exp(x))
failed.append(g)
```

结果:
```python
polys = [Poly(x + y - 3, x, y, domain='ZZ')]
failed = [exp(x) - y]
```

#### 步骤 4：分流决策

```python
if polys:  # True
    if all(p.is_linear for p in polys):  # True
        # 进入线性分支
        linear = True
```

#### 步骤 5：线性系统求解

```python
# 只有 1 个线性方程，2 个变量
n, m = 1, 2
matrix = zeros(1, 3)  # [[0, 0, 0]]

# 填充系数
# Poly(x + y - 3).terms() = [((1, 0), 1), ((0, 1), 1), ((0, 0), -3)]
for i, poly in enumerate(polys):
    for monom, coeff in poly.terms():
        try:
            j = monom.index(1)
            matrix[i, j] = coeff
        except ValueError:
            matrix[i, m] = -coeff

# matrix = [[1, 1, 3]]
```

调用 `solve_linear_system`:
```python
# 欠定系统: 1 方程, 2 变量
# 解: {x: 3 - y}  或  {y: 3 - x}
# 取决于自由变量选择
result = [{y: 3 - x}]  # 示例解
```

记录已解符号:
```python
if failed:  # True
    if result:  # True
        solved_syms = list(result[0].keys())  # [y]
```

#### 步骤 6：Failed 方程处理

```python
if failed:  # True
    linear = False  # 整个系统现在标记为非线性
    
    # 排序方程 (只有一个)
    for eq in ordered(failed, ...):
        # eq = exp(x) - y
        
        for r in result:
            # r = {y: 3 - x}
            
            # 代入
            eq2 = eq.subs(r)
            # eq2 = exp(x) - (3 - x) = exp(x) + x - 3
            
            # 选择符号
            ok_syms = _ok_syms(eq2, sort=True)
            # eq2 = exp(x) + x - 3，只包含 x
            # ok_syms = [x]
            
            for s in ok_syms:  # s = x
                try:
                    # 调用单变量求解器
                    soln = _vsolve(exp(x) + x - 3, x, **flags)
                    # 解: [LambertW(exp(3)) - 3] 或数值近似
                except NotImplementedError:
                    continue
                
                # 更新解
                for sol in soln:
                    rnew = r.copy()  # {y: 3 - x}
                    # 代入 x 的解
                    rnew[y] = (3 - x).subs(x, sol)  # 3 - sol
                    rnew[x] = sol
                    
                    newresult.append(rnew)
        
        result = newresult
```

#### 步骤 7：统一后处理

经过简化和验证，最终结果:
```python
# 解: x = LambertW(exp(3)) - 3, y = 3 - x
# 或数值近似: x ≈ 1.05, y ≈ 1.95
[{x: LambertW(exp(3)) - 3, y: 3 - LambertW(exp(3)) + 3}]
# 简化后
[{x: LambertW(exp(3)) - 3, y: 6 - LambertW(exp(3))}]
```

---

## 5. 结果统一处理

### 5.1 后处理流程图

```
求解完成 (result = 字典列表)
         │
         ▼
┌────────────────────────────────────────┐
│ 步骤 1: 空结果规范化                      │
│ result = result or [{}]                  │
│ (None 或 [] 转为 [{}])                    │
└────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ 步骤 2: 简化 (条件性)                     │
│ 位置: solvers.py:2012-2017               │
└────────────────────────────────────────┘
         │
    ┌────┴──────────────────────────┐
    │                               │
    ▼                               ▼
┌──────────────┐            ┌──────────────┐
│ default_     │            │ flags.get    │
│ simplify =   │            │ ('simplify', │
│ bool(failed) │            │ default_     │
│              │            │ simplify)    │
│ 有 failed    │            │              │
│ 方程时才简化 │            │ 用户指定时简化│
└──────────────┘            └──────────────┘
    │                               │
    └───────────────┬───────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │ for r in result:               │
    │     for k in r:                │
    │         r[k] = simplify(r[k]) │
    └───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ 步骤 3: 分母检查                         │
│ 位置: solvers.py:2019-2021               │
└────────────────────────────────────────┘
         │
         ▼
    ┌───────────────────────────────┐
    │ if checkdens:                  │
    │     result = [r for r in result│
    │         if not any(checksol(d, │
    │         r, **flags) for d in  │
    │         dens)]                 │
    └───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ 步骤 4: 方程验证 (仅非线性系统)          │
│ 位置: solvers.py:2023-2025               │
└────────────────────────────────────────┘
         │
    ┌────┴──────────────────────────┐
    │                               │
    ▼                               ▼
┌──────────────┐            ┌──────────────┐
│ linear=True  │            │ linear=False │
│              │            │              │
│ 跳过验证      │            │ 执行验证      │
│ (线性求解器  │            │              │
│  保证正确性)  │            │              │
└──────────────┘            └──────────────┘
    │                               │
    └───────────────┬───────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │ if check and not linear:      │
    │     result = [r for r in result│
    │         if not any(checksol(e,│
    │         r, **flags) is False  │
    │         for e in exprs)]       │
    └───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ 步骤 5: 过滤空解                         │
│ 位置: solvers.py:2027                    │
└────────────────────────────────────────┘
         │
         ▼
    ┌───────────────────────────────┐
    │ result = [r for r in result if r]│
    │ 移除空字典 {}                    │
    └───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ 步骤 6: 返回上层                         │
│ return linear, result                    │
└────────────────────────────────────────┘
```

### 5.2 关键后处理函数

#### `checksol` - 解验证

**位置**: `sympy/solvers/solvers.py:184`

```python
def checksol(f, symbol, sol=None, **flags):
    """
    检查 sol 是否是方程 f == 0 的解
    
    返回值:
    - True: 确定是解
    - False: 确定不是解
    - None: 无法确定
    
    验证策略:
    1. 数值检查 (快速)
    2. 简化检查
    3. 强制正号检查
    """
```

#### `_simple_dens` - 分母收集

**位置**: `sympy/solvers/solvers.py:118`

```python
def _simple_dens(f, symbols):
    """
    收集表达式中可能为零的分母
    
    用于检查解是否使分母为零
    """
    dens = set()
    for d in denoms(f, symbols):
        if d.is_Pow and d.exp.is_Number:
            if d.exp.is_zero:
                continue
            d = d.base
        dens.add(d)
    return dens
```

### 5.3 回到 `solve()` 主函数的后处理

`_solve_system` 返回后，`solve()` 函数还有额外的后处理：

**位置**: `sympy/solvers/solvers.py:1175-1290`

```python
# _solve_system 返回 (linear, solution)
# solution 是字典列表

# 步骤 1: 确定输出格式
as_dict = flags.get('dict', False)
as_set = flags.get('set', False)

# 步骤 2: 定义 unpack 函数（用于格式转换）
tuple_format = lambda s: [tuple([i.get(x, x) for x in symbols]) for i in s]

if as_dict or as_set:
    unpack = None
elif bare_f:
    # 单方程输入的格式
    if len(symbols) == 1:
        # 单变量: 解列表
        unpack = lambda s: [i[symbols[0]] for i in s]
    elif len(solution) == 1 and len(solution[0]) == len(symbols):
        # 待定系数法: 单个字典
        unpack = lambda s: s[0]
    elif ordered_symbols:
        # 有序符号: 元组列表
        unpack = tuple_format
    else:
        unpack = lambda s: s
else:
    # 方程组输入的格式
    if solution:
        if linear and len(solution) == 1:
            # 线性系统单解: 直接字典
            unpack = lambda s: s[0]
        elif ordered_symbols:
            # 有序符号: 元组列表
            unpack = tuple_format
        else:
            unpack = lambda s: s
    else:
        unpack = None

# 步骤 3: 恢复被屏蔽的对象 (Derivative, Integral 等)
if non_inverts and type(solution) is list:
    solution = [{k: v.subs(non_inverts) for k, v in s.items()}
        for s in solution]

# 步骤 4: 恢复原始符号 (recast_to_symbols 的逆操作)
if swap_sym:
    symbols = [swap_sym.get(k, k) for k in symbols]
    for i, sol in enumerate(solution):
        solution[i] = {swap_sym.get(k, k): v.subs(swap_sym)
                  for k, v in sol.items()}

# 步骤 5: 符号假设检查
check = flags.get('check', True)

# 浮点数恢复
if floats and solution and flags.get('rational', None) is None:
    solution = nfloat(solution, exponent=False)
    solution = _remove_duplicate_solutions(solution)

# 假设验证
if check and solution:
    warn = flags.get('warn', False)
    got_None = []
    no_False = []
    
    for sol in solution:
        v = fuzzy_and(check_assumptions(val, **symb.assumptions0)
                      for symb, val in sol.items())
        if v is False:
            continue
        no_False.append(sol)
        if v is None:
            got_None.append(sol)
    
    solution = no_False
    
    # 警告无法验证的解
    if warn and got_None:
        warnings.warn(...)

# 步骤 6: 解的规范排序
if not as_set:
    solution = [{k: s[k] for k in ordered(s)} for s in solution]
    solution.sort(key=default_sort_key)

# 步骤 7: 格式转换并返回
if not (as_set or as_dict):
    return unpack(solution)

if as_dict:
    return solution

# set=True 格式: (symbols, {tuple_of_solutions})
if ordered_symbols:
    k = symbols
else:
    k = list(ordered(set(flatten(tuple(i.keys()) for i in solution))))

return k, {tuple([s.get(ki, ki) for ki in k]) for s in solution}
```

---

## 6. 现代求解器 `linsolve`

### 6.1 与传统求解器的关系

`linsolve` 是 `solveset` 模块中专门用于线性系统的求解器，与 `_solve_system` 中的线性分支是**独立实现**。

| 特性 | `_solve_system` 线性分支 | `linsolve` |
|------|-------------------------|------------|
| 模块 | `solvers.py` | `solveset.py` |
| 调用方式 | `solve([eqs], syms)` | `linsolve([eqs], syms)` |
| 返回格式 | 字典列表 或 单个字典 | `FiniteSet` 包含元组 |
| 欠定系统 | 自由变量被消除 | 参数化解（保留自由变量） |
| 错误处理 | 返回 `[]` 或 `{}` | 抛出 `NonlinearError` |

### 6.2 `linsolve` 识别策略

**位置**: `sympy/solvers/solveset.py:3052-3068`

```python
# linsolve 强制要求线性，否则抛出异常
>>> linsolve([x**2 - 1], x)
Traceback (most recent call last):
...
NonlinearError: nonlinear term: x**2
```

**识别逻辑** (在 `_linear_eq_to_dict` 中):
- 检查每个方程是否为线性
- 非线性项包括:
  - 高次项 (`x**2`)
  - 交叉项 (`x*y`)
  - 函数项 (`1/x`, `exp(x)` 等)

---

## 7. 完整示例对照

### 7.1 示例 1：纯线性系统

```python
from sympy import symbols, solve, linsolve

x, y = symbols('x y')
eqs = [x + y - 3, x - y - 1]

# 传统求解器
sol1 = solve(eqs, x, y)
print("solve():", sol1)
# 输出: {x: 2, y: 1}

# 现代求解器
sol2 = linsolve(eqs, x, y)
print("linsolve():", sol2)
# 输出: {(2, 1)}
```

**路径追踪**:
1. 两个方程都能转换为 `Poly`
2. `all(p.is_linear for p in polys)` → `True`
3. 进入线性分支
4. 构建增广矩阵 `[[1, 1, 3], [1, -1, 1]]`
5. 调用 `solve_linear_system` → `{x: 2, y: 1}`
6. `failed = []`，无后处理
7. 返回 `linear=True, result=[{x: 2, y: 1}]`
8. 主函数中 `linear and len(solution) == 1` → `True`，解包为单个字典

### 7.2 示例 2：非线性多项式系统

```python
from sympy import symbols, solve

x, y = symbols('x y')
eqs = [x**2 + y**2 - 5, x - y - 1]

sol = solve(eqs, x, y)
print("solve():", sol)
# 输出: [{x: -1, y: -2}, {x: 2, y: 1}]
```

**路径追踪**:
1. 两个方程都能转换为 `Poly`:
   - `Poly(x**2 + y**2 - 5, x, y, domain='ZZ')`
   - `Poly(x - y - 1, x, y, domain='ZZ')`
2. `all(p.is_linear for p in polys)` → `False` (第一个方程有 `x**2`)
3. 进入非线性分支，`linear = False`
4. 变量数 (2) = 方程数 (2)，非欠定
5. 调用 `solve_poly_system(polys, x, y)`
6. Groebner 基求解 → `[(-1, -2), (2, 1)]`
7. 转换为字典列表 → `[{x: -1, y: -2}, {x: 2, y: 1}]`
8. `failed = []`，无后处理
9. 返回 `linear=False, result=[...]`
10. 主函数中 `linear=False`，格式保持字典列表

### 7.3 示例 3：混合系统（线性 + 超越）

```python
from sympy import symbols, solve, exp

x, y = symbols('x y')
eqs = [x + y - 3, exp(x) - y]

sol = solve(eqs, x, y)
print("solve():", sol)
# 输出: [{x: LambertW(exp(3)) - 3, y: 3 - x}] 代入后
# 实际上: [{x: LambertW(exp(3)) - 3, y: 6 - LambertW(exp(3))}]
```

**路径追踪**:
1. 方程 1 能转换为 `Poly` 且 `is_linear=True`
2. 方程 2 无法转换为 `Poly`（包含 `exp`）
3. `polys = [Poly(x + y - 3)]`，`all(p.is_linear)` → `True`
4. 进入线性分支，`linear = True`
5. 求解欠定线性系统 → `result = [{y: 3 - x}]`
6. `failed = [exp(x) - y]`，记录 `solved_syms = [y]`
7. 进入 failed 处理:
   - `linear = False`（现在整个系统是非线性的）
   - 对 `exp(x) - y` 代入 `{y: 3 - x}` → `exp(x) + x - 3`
   - 调用 `_vsolve(exp(x) + x - 3, x)`
   - 得到 `sol = LambertW(exp(3)) - 3`
   - 更新解: `x = sol`, `y = 3 - sol`
8. 后处理:
   - `linear=False`，执行方程验证
   - 检查解是否满足 `exp(x) - y = 0`
9. 返回最终解

### 7.4 示例 4：纯超越系统

```python
from sympy import symbols, solve, exp, sin

x, y = symbols('x y')
eqs = [exp(x) - y, sin(x) + y - 1]

sol = solve(eqs, x, y)
print("solve():", sol)
# 输出取决于能否求解，可能返回 ConditionSet 或 []
```

**路径追踪**:
1. 两个方程都无法转换为 `Poly`
2. `polys = []`, `failed = [exp(x) - y, sin(x) + y - 1]`
3. `if polys:` → `False`，跳过线性/非线性分支
4. 直接进入 failed 处理
5. `linear = False`
6. 按顺序尝试求解每个方程:
   - 先解 `exp(x) - y = 0` 得 `y = exp(x)`
   - 代入 `sin(x) + y - 1 = 0` 得 `sin(x) + exp(x) - 1 = 0`
   - 尝试用 `_vsolve` 解这个单变量方程
7. 取决于求解能力，返回解或空列表

---

## 8. 关键代码位置汇总

| 功能 | 文件 | 行号 | 函数名 |
|------|------|------|--------|
| **方程组入口** | | | |
| 方程组路由判断 | `solvers.py` | 1163-1170 | `solve()` 内 |
| 系统求解主函数 | `solvers.py` | 1763 | `_solve_system()` |
| **系统分解** | | | |
| 连通分量分解 | `solvers.py` | 1771-1805 | `_solve_system()` 内 |
| **多项式转换** | | | |
| 反演处理 | `solvers.py` | 1818 | `_invert()` |
| 多项式转换尝试 | `solvers.py` | 1828 | `g.as_poly()` |
| **分流决策** | | | |
| 线性检查 | `solvers.py` | 1836 | `all(p.is_linear for p in polys)` |
| **线性求解** | | | |
| 增广矩阵构建 | `solvers.py` | 1837-1846 | `_solve_system()` 内 |
| 线性系统求解器 | `solvers.py` | 2296 | `solve_linear_system()` |
| 底层线性求解器 | `polys/solvers.py` | - | `solve_lin_sys()` |
| **非线性求解** | | | |
| 欠定系统处理 | `solvers.py` | 1862-1889 | `_solve_system()` 内 |
| 多项式系统求解器 | `solvers/polysys.py` | - | `solve_poly_system()` |
| **Failed 处理** | | | |
| 符号选择策略 | `solvers.py` | 1910-1928 | `_ok_syms()` |
| 单变量求解调用 | `solvers.py` | 1963 | `_vsolve()` |
| 解更新逻辑 | `solvers.py` | 1969-1990 | `_solve_system()` 内 |
| **后处理** | | | |
| 简化控制 | `solvers.py` | 2012-2017 | `_solve_system()` 内 |
| 分母检查 | `solvers.py` | 2019-2021 | `_solve_system()` 内 |
| 方程验证 | `solvers.py` | 2023-2025 | `_solve_system()` 内 |
| 解验证函数 | `solvers.py` | 184 | `checksol()` |
| **现代求解器** | | | |
| 线性系统求解器 | `solveset.py` | 2897 | `linsolve()` |
| 非线性错误检测 | `solveset.py` | 3052-3068 | `linsolve()` 内 |

---

## 9. 总结

### 9.1 核心修正点

1. **分流点不是 "是否为线性系统"，而是 "polys 列表中是否全部是线性的"**
   - `failed` 列表中的超越方程不影响分流决策
   - 但会在后续处理中将 `linear` 标志设为 `False`

2. **线性分支的求解器是 `solve_linear_system`，不是 `linsolve`**
   - `solve_linear_system` 在 `solvers.py` 中
   - `linsolve` 是 `solveset.py` 中的独立实现

3. **混合系统的处理流程是 "线性先解，超越后处理"**
   - 线性方程先解出部分变量关系
   - 超越方程用回代法逐个求解
   - 这是 `_solve_system` 最巧妙的设计

### 9.2 设计亮点

1. **灵活的 `polys` / `failed` 二分法**
   - 能处理的先处理，不能处理的后处理
   - 实现了 "多项式 + 超越" 混合系统的求解

2. **欠定系统的自由变量处理**
   - 线性求解器自动选择自由变量
   - 解表示为参数化形式

3. **`failed` 处理中的符号选择策略**
   - 优先选择低次、有理系数的符号
   - 最小化求解难度

4. **回代法求解混合系统**
   - 利用已解符号减少未知量
   - 逐个击破复杂方程

---

*报告版本: 2.0（修正版）*
*生成日期: 2026-05-01*
