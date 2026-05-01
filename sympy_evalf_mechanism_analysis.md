# SymPy 数值求值（evalf）机制深度分析

## 概述

SymPy 的 `evalf` 机制是其高精度数值计算的核心，它通过自适应精度控制和误差追踪，实现了对任意精度符号表达式的数值求值。本报告深入分析 `evalf` 的三个核心方面：精度参数传播机制、各类数学操作的精度控制模块分布、以及精度升降对性能的影响路径。

---

## 一、精度参数在表达式树中的传播机制

### 1.1 入口点与精度转换

`evalf` 的调用入口是 `EvalfMixin.evalf()` 方法，位于 `sympy/core/evalf.py:1569`。

**精度转换流程：**

```python
# 用户调用（十进制精度）
expr.evalf(n=15)  # 默认 15 位十进制

# 内部转换（二进制精度）
prec = dps_to_prec(n)  # 15 dps ≈ 53 bits
```

**关键代码位置**：`sympy/core/evalf.py:1650`
```python
prec = dps_to_prec(n)
options = {'maxprec': max(prec, int(maxn*LG10)), 'chop': chop,
           'strict': strict, 'verbose': verbose}
```

### 1.2 精度传播的递归机制

`evalf` 函数（`sympy/core/evalf.py:1459`）是核心分发器，通过 `evalf_table` 查找对应类型的处理函数：

```python
def evalf(x: Expr, prec: int, options: OPT_DICT) -> TMP_RES:
    try:
        rf = evalf_table[type(x)]
        r = rf(x, prec, options)
    except KeyError:
        # Fall back to _eval_evalf method
        xe = x._eval_evalf(prec)
        # ...
```

**`evalf_table` 的创建**（`sympy/core/evalf.py:1399`）：

```python
evalf_table = {
    # 基础数值类型
    Float: evalf_float,
    Rational: evalf_rational,
    Integer: evalf_integer,
    
    # 数学常数
    Pi: lambda x, prec, options: (mpf_pi(prec), None, prec, None),
    Exp1: lambda x, prec, options: (mpf_e(prec), None, prec, None),
    
    # 基本运算
    Add: evalf_add,
    Mul: evalf_mul,
    Pow: evalf_pow,
    
    # 初等函数
    exp: evalf_exp,
    cos: evalf_trig,
    sin: evalf_trig,
    tan: evalf_trig,
    log: evalf_log,
    atan: evalf_atan,
    Abs: evalf_abs,
    
    # 高级运算
    Integral: evalf_integral,
    Sum: evalf_sum,
    Product: evalf_prod,
}
```

### 1.3 精度传递的具体示例

以加法运算 `evalf_add`（`sympy/core/evalf.py:585`）为例：

```python
def evalf_add(v: 'Add', prec: int, options: OPT_DICT) -> TMP_RES:
    # 1. 递归求值所有子表达式，使用更高的精度（prec + 10）
    terms = [evalf(arg, prec + 10, options) for arg in v.args]
    
    # 2. 合并实部和虚部分别计算
    re, re_acc = add_terms(
        [a[0::2] for a in terms if isinstance(a, tuple) and a[0]], 
        prec, target_prec)
    im, im_acc = add_terms(
        [a[1::2] for a in terms if isinstance(a, tuple) and a[1]], 
        prec, target_prec)
    
    # 3. 检查精度是否满足，不满足则提升精度重试
    acc = complex_accuracy((re, im, re_acc, im_acc))
    if acc >= target_prec:
        break
    else:
        # 自适应提升精度
        prec = prec + max(10 + 2**i, target_prec - acc)
        i += 1
```

**关键设计要点**：
- **保护位策略**：子表达式求值使用 `prec + 10`，预留额外精度
- **精度追踪**：每个中间结果都附带 `re_acc` 和 `im_acc`（有效精度位数）
- **自适应提升**：当结果精度不足时，动态增加工作精度并重试

### 1.4 精度数据结构

`evalf` 使用特殊的元组格式传递结果：

```python
# 临时结果格式: (re, im, re_acc, im_acc)
TMP_RES = tuple[re, im, re_acc, im_acc]

# 其中：
# - re: 实部的 mpf 元组 (sign, man, exp, bc) 或 None（表示 0）
# - im: 虚部的 mpf 元组或 None
# - re_acc: 实部的有效精度位数（二进制）
# - im_acc: 虚部的有效精度位数（二进制）

# mpf 元组格式: (sign, man, exp, bc)
# 表示: (-1)^sign * man * 2^exp
# bc = bitcount(man) = mantissa 的二进制位数
```

**精度计算示例**（`sympy/core/evalf.py:579`）：

```python
# 加法后的精度计算
sum_accuracy = sum_exp + sum_bc - absolute_error
# sum_exp: 结果的指数
# sum_bc: 尾数的位数
# absolute_error: 绝对误差的对数估计
```

---

## 二、各类数学操作的精度控制实现模块

### 2.1 模块架构概览

| 模块路径 | 职责 | 关键函数/类 |
|---------|------|------------|
| `sympy/core/evalf.py` | 核心调度与基础运算 | `evalf()`, `evalf_add`, `evalf_mul`, `evalf_pow` |
| `sympy/core/function.py` | 通用函数求值 | `Function._eval_evalf()` |
| `sympy/external/mpmath.py` | mpmath 接口层 | `dps_to_prec`, `local_workprec`, `mpf_*` 函数 |
| `sympy/functions/special/*.py` | 特殊函数精度控制 | 各函数的 `_eval_evalf()` 方法 |

### 2.2 基础运算的精度控制

#### 2.2.1 加法运算（`evalf_add`）

**文件**：`sympy/core/evalf.py:585-631`

**精度控制策略**：
- 使用 `working_prec = 2*prec` 进行内部计算
- 跟踪绝对误差 `absolute_error`
- 当发生抵消（结果接近 0）时返回 `scaled_zero` 特殊标记

```python
def add_terms(terms: list, prec: int, target_prec: int):
    working_prec = 2*prec  # 使用双倍工作精度
    # ...
    absolute_error = max(absolute_err)  # 跟踪最大绝对误差
    
    if not sum_man:
        return scaled_zero(absolute_error)  # 抵消检测
    
    sum_accuracy = sum_exp + sum_bc - absolute_error
```

**自适应精度循环**：
```python
while 1:
    options['maxprec'] = min(oldmaxprec, 2*prec)
    terms = [evalf(arg, prec + 10, options) for arg in v.args]
    # ... 计算 ...
    
    acc = complex_accuracy((re, im, re_acc, im_acc))
    if acc >= target_prec:
        break
    else:
        # 指数级提升精度
        prec = prec + max(10 + 2**i, target_prec - acc)
        i += 1
```

#### 2.2.2 乘法运算（`evalf_mul`）

**文件**：`sympy/core/evalf.py:634-757`

**精度控制策略**：
- 乘法相对误差较小，通常 `acc = prec`
- 复杂数乘法需要更多保护位：`working_prec = prec + len(args) + 5`

```python
def evalf_mul(v: 'Mul', prec: int, options: OPT_DICT) -> TMP_RES:
    # 乘法的相对精度通常保持不变
    acc = prec
    
    # 保护位 = 参数数量 + 5
    working_prec = prec + len(args) + 5
    
    # 实部/虚部分开处理
    for wre, wim, wre_acc, wim_acc in complex_factors[i0:]:
        acc = min(acc, complex_accuracy((wre, wim, wre_acc, wim_acc)))
        # ... 复数乘法 ...
```

#### 2.2.3 幂运算（`evalf_pow`）

**文件**：`sympy/core/evalf.py:760-880`

**精度控制策略**：
- **整数幂**：误差放大因子为 `|p|`，需要增加 `log2(|p|)` 位精度
- **大指数**：根据指数大小动态调整工作精度
- **特殊情况**：平方根、指数函数等有专门优化

```python
def evalf_pow(v: 'Pow', prec: int, options) -> TMP_RES:
    base, exp = v.args
    
    if exp.is_Integer:
        p = exp.p
        # 整数幂：相对误差放大 |p| 倍
        prec += int(math.log2(abs(p)))
        result = evalf(base, prec + 5, options)
        # ...
    
    # 评估指数大小以确定工作精度
    ysize = fastlog(yre)
    if ysize > 5:  # 指数很大
        prec += ysize  # 需要额外精度
        yre, yim, _, _ = evalf(exp, prec, options)
```

### 2.3 初等函数的精度控制

#### 2.3.1 三角函数（`evalf_trig`）

**文件**：`sympy/core/evalf.py:895-954`

**精度控制策略**：
- 参数接近 π 倍数时需要额外精度（周期性导致精度损失）
- 大参数需要先计算高精度值

```python
def evalf_trig(v: Expr, prec: int, options: OPT_DICT) -> TMP_RES:
    xprec = prec + 20  # 初始增加 20 位保护位
    
    # 大参数处理：xsize = log2(|x|)
    xsize = fastlog(re)
    if xsize >= 10:  # 绝对值 >= 1024
        xprec = prec + xsize  # 需要更多精度
        re, im, re_acc, im_acc = evalf(arg, xprec, options)
    
    # 接近根时的自适应精度提升
    while 1:
        y = func(re, prec, rnd)
        ysize = fastlog(y)
        gap = -ysize  # 结果接近 0 的程度
        accuracy = (xprec - xsize) - gap
        
        if accuracy < prec:
            # 需要更多精度
            xprec += gap
            re, im, re_acc, im_acc = evalf(arg, xprec, options)
            continue
        else:
            return y, None, prec, None
```

#### 2.3.2 对数函数（`evalf_log`）

**文件**：`sympy/core/evalf.py:957-1008`

**精度控制策略**：
- 参数接近 1 时需要特殊处理（log(1+x) ≈ x，损失精度）
- 负数返回复数结果

```python
def evalf_log(expr: 'log', prec: int, options: OPT_DICT) -> TMP_RES:
    workprec = prec + 10
    
    # x 接近 1 时的特殊处理
    size = fastlog(re)
    if prec - size > workprec:
        # 需要计算 x-1 的高精度值
        add = Add(S.NegativeOne, arg, evaluate=False)
        xre, xim, xre_acc, _ = evalf_add(add, prec, options)
        
        prec2 = workprec - fastlog(xre)
        re = mpf_ln(mpf_abs(mpf_add(xre, fone, prec2)), prec, rnd)
```

### 2.4 通用函数的精度控制（`Function._eval_evalf`）

**文件**：`sympy/core/function.py:535-603`

对于大多数未在 `evalf_table` 中注册的函数，使用通用的 `_eval_evalf` 方法：

```python
def _eval_evalf(self, prec):
    # 1. 查找对应的 mpmath 函数
    func = _get_mpmath_func(self.func.__name__)
    
    # 2. 参数转换为更高精度（prec + 5）
    args = [arg._to_mpmath(prec + 5) for arg in args]
    
    # 3. 在指定精度下调用 mpmath 函数
    with workprec(prec):
        v = func(*args)
    
    return Expr._from_mpmath(v, prec)
```

### 2.5 特殊函数的精度控制

各类特殊函数在各自模块中实现 `_eval_evalf` 方法：

| 函数类别 | 模块文件 | 关键实现 |
|---------|---------|---------|
| Gamma 函数 | `sympy/functions/special/gamma_functions.py` | `gamma._eval_evalf`, `digamma._eval_evalf` |
| Bessel 函数 | `sympy/functions/special/bessel.py` | `besselj._eval_evalf`, `bessely._eval_evalf` |
| 误差函数 | `sympy/functions/special/error_functions.py` | `erf._eval_evalf`, `erfinv._eval_evalf` |
| Zeta 函数 | `sympy/functions/special/zeta_functions.py` | `zeta._eval_evalf` |
| 超几何函数 | `sympy/functions/special/hyper.py` | `hyper._eval_evalf` |
| Beta 函数 | `sympy/functions/special/beta_functions.py` | `beta._eval_evalf` |

**示例：Bessel 函数的精度控制**（`sympy/functions/special/bessel.py:1024-1088`）：

```python
def _eval_evalf(self, prec):
    # 阶数为 0 或正整数时，重写为 besselj
    if self.order.is_zero or (self.order.is_Integer and self.order.is_positive):
        return self.rewrite(besselj)._eval_evalf(prec)
    
    # 否则委托给父类（调用 mpmath）
    return super()._eval_evalf(prec)
```

### 2.6 高级运算的精度控制

#### 2.6.1 数值积分（`evalf_integral`）

**文件**：`sympy/core/evalf.py:1173-1196`

```python
def evalf_integral(expr: 'Integral', prec: int, options: OPT_DICT) -> TMP_RES:
    workprec = prec
    i = 0
    maxprec = options.get('maxprec', INF)
    
    while 1:
        result = do_integral(expr, workprec, options)
        accuracy = complex_accuracy(result)
        
        if accuracy >= prec:
            break
        if workprec >= maxprec:
            break
        
        # 自适应精度提升
        if accuracy == -1:
            workprec *= 2  # 可能为 0，指数级提升
        else:
            workprec += max(prec, 2**i)
        workprec = min(workprec, maxprec)
        i += 1
```

#### 2.6.2 无穷级数（`evalf_sum`）

**文件**：`sympy/core/evalf.py:1331-1370`

```python
def evalf_sum(expr: 'Sum', prec: int, options: OPT_DICT) -> TMP_RES:
    prec2 = prec + 10
    
    try:
        # 快速超几何级数求和
        v = hypsum(func, n, int(a), prec2)
        delta = prec - fastlog(v)
        if fastlog(v) < -10:
            v = hypsum(func, n, int(a), delta)
        return v, None, min(prec, delta), None
    except NotImplementedError:
        # Euler-Maclaurin 求和
        eps = Float(2.0)**(-prec)
        for i in range(1, 5):
            m = n = 2**i * prec
            s, err = expr.euler_maclaurin(m=m, n=n, eps=eps,
                eval_integral=False)
            err = err.evalf()
            if err <= eps:
                break
        # ...
```

---

## 三、精度升降对性能的影响路径

### 3.1 精度-性能权衡的核心机制

`evalf` 的性能代价主要来自以下几个方面：

| 代价来源 | 触发条件 | 性能影响 |
|---------|---------|---------|
| 高精度算术运算 | 工作精度 `prec` 增加 | O(prec²) 或更高复杂度 |
| 自适应迭代重试 | 精度不足时 | 多次重复计算 |
| 大整数操作 | 高精度尾数 | 内存和计算时间随精度线性增长 |
| 特殊函数算法 | 收敛速度依赖精度 | 迭代次数随精度增加 |

### 3.2 精度提升的触发场景

#### 场景 1：加法中的抵消（Cancellation）

**代码位置**：`sympy/core/evalf.py:571-572`

```python
if not sum_man:
    return scaled_zero(absolute_error)
```

**触发条件**：两个相近的数相减导致有效数字丢失。

**性能影响**：
- 需要使用更高的工作精度重新计算
- 最坏情况下可能需要 `maxprec`（默认 ~333 位）

**示例**（`sympy/core/tests/test_evalf.py:67-69`）：
```python
def test_cancellation():
    # π + 1e-1000 - π 需要极高精度才能得到 1e-1000
    assert NS(Add(pi, Rational(1, 10**1000), -pi, evaluate=False), 15,
              maxn=1200) == '1.00000000000000e-1000'
```

#### 场景 2：三角函数接近根

**代码位置**：`sympy/core/evalf.py:939-953`

```python
while 1:
    y = func(re, prec, rnd)
    ysize = fastlog(y)
    gap = -ysize  # 结果接近 0 的程度
    accuracy = (xprec - xsize) - gap
    
    if accuracy < prec:
        xprec += gap  # 精度提升量 = 接近 0 的程度
        re, im, re_acc, im_acc = evalf(arg, xprec, options)
        continue
```

**触发条件**：
- `sin(kπ) ≈ 0`
- `cos(π/2 + kπ) ≈ 0`
- `tan(kπ) ≈ 0`

**性能影响**：
- 精度提升量与 `gap = -log2(|result|)` 成正比
- 非常接近根时可能需要数百位额外精度

**示例**（`sympy/core/tests/test_evalf.py:177-182`）：
```python
# sin(π * 10^100 + 7e-5) ≈ sin(7e-5) ≈ 7e-5
# 需要 100+ 位精度来计算模 π 的余数
assert NS(sin(pi*10**100 + Rational(7, 10**5), evaluate=False), 15, maxn=120) == \
    '6.99999999428333e-5'
```

#### 场景 3：大指数幂运算

**代码位置**：`sympy/core/evalf.py:841-846`

```python
ysize = fastlog(yre)  # 指数的大小
if ysize > 5:  # 指数 >= 32
    prec += ysize  # 需要额外精度
    yre, yim, _, _ = evalf(exp, prec, options)
```

**触发条件**：指数的绝对值较大。

**性能影响**：
- 精度增加量与指数大小成正比
- `10^100` 次幂需要约 332 位额外精度

**示例**（`sympy/core/tests/test_evalf.py:72-77`）：
```python
def test_evalf_powers():
    # π^(10^20) 需要计算 10^20 * log2(π) ≈ 5e19 位？不...
    # 实际上使用对数计算，需要足够精度来得到指数的整数部分
    assert NS('pi**(10**20)', 10) == '1.339148777e+49714987269413385435'
```

#### 场景 4：对数的参数接近 1

**代码位置**：`sympy/core/evalf.py:993-1002`

```python
size = fastlog(re)  # |log(x)| 的大小
if prec - size > workprec:
    # x 非常接近 1，需要高精度计算 x-1
    add = Add(S.NegativeOne, arg, evaluate=False)
    xre, xim, xre_acc, _ = evalf_add(add, prec, options)
```

**触发条件**：`x ≈ 1`，此时 `log(x) ≈ x-1`，精度损失严重。

**性能影响**：
- 需要计算 `x-1` 的高精度值
- 可能触发加法抵消的精度提升

### 3.3 精度降低的场景与性能收益

#### 场景 1：精确数值类型

**代码位置**：`sympy/core/evalf.py:481-490`

```python
def evalf_float(expr: 'Float', prec: int, options: OPT_DICT) -> TMP_RES:
    return expr._mpf_, None, prec, None  # 直接返回存储的 mpf

def evalf_rational(expr: 'Rational', prec: int, options: OPT_DICT) -> TMP_RES:
    return from_rational(expr.p, expr.q, prec), None, prec, None

def evalf_integer(expr: 'Integer', prec: int, options: OPT_DICT) -> TMP_RES:
    return from_int(expr.p, prec), None, prec, None
```

**性能特点**：
- `Float`：直接返回内部存储的 `mpf` 元组，O(1)
- `Rational`/`Integer`：转换为指定精度的浮点数，O(prec) 但非常快

#### 场景 2：简单乘法（无复数因子）

**代码位置**：`sympy/core/evalf.py:717-723`

```python
if not complex_factors:
    v = normalize(sign, man, exp, man.bit_length(), prec, rnd)
    if direction & 1:
        return None, v, None, acc  # 纯虚数
    else:
        return v, None, acc, None  # 纯实数
```

**性能特点**：
- 纯实数/纯虚数乘法跳过复杂的复数运算
- 精度保持 `acc = prec`，无需自适应提升

### 3.4 性能影响的量化分析

#### 3.4.1 算术运算的复杂度

| 运算类型 | mpmath 实现复杂度 | 精度敏感性 |
|---------|------------------|-----------|
| 加法/减法 | O(n) | 低（除非抵消） |
| 乘法 | O(n log n) 或 O(n²) | 中 |
| 除法 | O(n log n) | 中 |
| 幂运算 | O(n² log n) | 高 |
| 平方根 | O(n log n) | 中 |
| 指数/对数 | O(n² log n) | 高 |
| 三角函数 | O(n² log n) | 高（接近根时） |
| Gamma 函数 | O(n² log n) | 高 |

#### 3.4.2 自适应迭代的代价

以 `evalf_add` 为例，最坏情况下的迭代次数：

```python
# 初始精度
prec = 53  # ~15 位十进制

# 迭代精度提升序列
i=0: prec += max(10 + 1, target_prec - acc)  # +11
i=1: prec += max(10 + 2, target_prec - acc)  # +12
i=2: prec += max(10 + 4, target_prec - acc)  # +14
i=3: prec += max(10 + 8, target_prec - acc)  # +18
# ... 指数级增长
```

**最坏情况性能**：
- 每次迭代需要重新计算所有子表达式
- 达到 `maxprec`（默认 333 位）可能需要 5-8 次迭代
- 总代价约为单次计算的 3-5 倍

#### 3.4.3 内存开销

高精度浮点数的内存需求：

```python
# mpf 元组: (sign, man, exp, bc)
# man = 尾数（大整数）
# bc = bitcount(man) = 有效位数

# 内存估计（假设 Python int 开销为 28 字节 + 4 字节/数字）
prec_53 = 53    # ~15 位十进制: man ≈ 2 个 30 位数字 → ~36 字节
prec_333 = 333  # ~100 位十进制: man ≈ 11 个数字 → ~72 字节
prec_3322 = 3322 # ~1000 位十进制: man ≈ 111 个数字 → ~472 字节
```

**关键代码**（`sympy/core/evalf.py:541-569`）显示加法中尾数的增长：

```python
working_prec = 2*prec
sum_man, sum_exp = 0, 0

for x, accuracy in terms:
    # ... 对齐指数并相加
    sum_man += (man << delta)  # 尾数左移，位数增加
    
    # 防止尾数过大，定期截断
    while bc > 3*working_prec:
        man >>= working_prec
        exp += working_prec
        bc -= working_prec
```

### 3.5 性能优化策略

#### 策略 1：设置合理的 `maxn`

```python
# 默认 maxn=100（十进制）≈ 332 位二进制
N(expr, maxn=50)  # 限制最大精度，防止过度计算
```

#### 策略 2：使用 `chop` 处理微小项

```python
# 自动将小于阈值的项设为 0，避免精度耗尽
N(expr, chop=True)           # 使用默认阈值
N(expr, chop=1e-10)          # 自定义阈值
```

#### 策略 3：避免表达式中的抵消

```python
# 差的形式可能导致抵消
N(sin(x + 1e-10) - sin(x), 15)  # 可能需要高精度

# 重写为更好的形式
N(2*cos(x + 5e-11)*sin(5e-11), 15)  # 更稳定
```

#### 策略 4：利用符号简化

```python
# 先简化再求值
simplified = simplify(expr)
N(simplified, 15)  # 简化后的表达式可能数值更稳定
```

---

## 四、总结

### 4.1 精度传播机制核心要点

1. **双层精度表示**：
   - 用户接口：十进制精度 `n`（通过 `dps_to_prec` 转换）
   - 内部实现：二进制精度 `prec`，配合保护位（通常 +5 或 +10）

2. **递归传播模式**：
   - 通过 `evalf_table` 分发到类型特定的处理函数
   - 子表达式求值使用更高精度（`prec + 保护位`）
   - 结果携带精度信息 `(re, im, re_acc, im_acc)`

3. **自适应精度控制**：
   - 加法：抵消检测 + 指数级精度提升
   - 乘法：精度基本保持，复数运算需要更多保护位
   - 幂运算：整数幂增加 `log2(|p|)` 位，大指数动态调整

### 4.2 模块架构

```
sympy/
├── core/
│   ├── evalf.py          # 核心：evalf_table, 基础运算 (Add/Mul/Pow), 
│   │                      # 初等函数 (trig/log/exp), 高级运算 (Integral/Sum)
│   ├── function.py       # 通用 Function._eval_evalf
│   └── numbers.py        # 数值类型的 _eval_evalf
│
├── external/
│   └── mpmath.py         # mpmath 接口层：dps_to_prec, local_workprec, 
│                          # mpf_*/mpc_* 低精度函数
│
└── functions/
    └── special/
        ├── gamma_functions.py   # Gamma, digamma, polygamma 等
        ├── bessel.py            # Bessel 函数族
        ├── error_functions.py   # erf, erfc, erfinv 等
        ├── zeta_functions.py    # Riemann zeta, Dirichlet eta 等
        ├── hyper.py             # 超几何函数
        └── beta_functions.py    # Beta, incomplete beta 等
```

### 4.3 性能关键路径

**高代价操作**：
1. **抵消加法**：可能需要 `maxprec` 精度和多次迭代
2. **接近根的三角函数**：精度需求与 `|1/result|` 成正比
3. **大指数幂运算**：精度需求与指数大小成正比
4. **数值积分/级数求和**：自适应算法可能需要多次精度提升

**低代价操作**：
1. **精确数值转换**：Integer/Rational → Float
2. **简单乘法**：纯实数/纯虚数，无复数因子
3. **小参数初等函数**：远离奇点和根的情况

### 4.4 关键设计权衡

| 设计决策 | 优点 | 缺点 |
|---------|------|------|
| 保护位策略（+5/+10） | 减少迭代重试次数 | 轻微增加单次计算代价 |
| 指数级精度提升 | 快速达到所需精度 | 可能"跳过"最优精度 |
| `maxprec` 上限 | 防止无限循环 | 可能返回低精度结果 |
| `scaled_zero` 标记 | 正确跟踪抵消后的精度 | 增加代码复杂度 |

---

## 附录：关键代码位置索引

### 核心模块

| 功能 | 文件路径 | 行号 |
|-----|---------|------|
| `evalf()` 主函数 | `sympy/core/evalf.py` | 1459 |
| `EvalfMixin.evalf()` 入口 | `sympy/core/evalf.py` | 1569 |
| `evalf_table` 创建 | `sympy/core/evalf.py` | 1399 |
| `evalf_add` 加法 | `sympy/core/evalf.py` | 585 |
| `evalf_mul` 乘法 | `sympy/core/evalf.py` | 634 |
| `evalf_pow` 幂运算 | `sympy/core/evalf.py` | 760 |
| `evalf_trig` 三角函数 | `sympy/core/evalf.py` | 895 |
| `evalf_log` 对数 | `sympy/core/evalf.py` | 957 |
| `evalf_integral` 积分 | `sympy/core/evalf.py` | 1173 |
| `evalf_sum` 级数 | `sympy/core/evalf.py` | 1331 |
| 通用 `Function._eval_evalf` | `sympy/core/function.py` | 535 |
| mpmath 接口层 | `sympy/external/mpmath.py` | 1 |

### 精度相关工具函数

| 功能 | 文件路径 | 行号 |
|-----|---------|------|
| `complex_accuracy()` 精度计算 | `sympy/core/evalf.py` | 234 |
| `fastlog()` 快速对数近似 | `sympy/core/evalf.py` | 112 |
| `add_terms()` 带精度的加法 | `sympy/core/evalf.py` | 499 |
| `scaled_zero()` 抵消标记 | `sympy/core/evalf.py` | 183 |
| `dps_to_prec()` 精度转换 | `sympy/external/mpmath.py` | 导入 |
| `local_workprec()` 上下文管理 | `sympy/external/mpmath.py` | 231 |
