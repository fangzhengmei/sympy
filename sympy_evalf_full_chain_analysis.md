# SymPy evalf 完整调用链路深度分析

## 摘要

本报告详细追踪 SymPy `evalf` 从用户调用到复合表达式求值完成的完整链路，重点分析：
1. **精度传播**：每一步如何传递和提升精度
2. **有效位回传**：`re_acc`/`im_acc` 如何计算和传递
3. **重算触发**：何时、为何、如何触发重新计算
4. **性能代价**：各阶段的计算开销与触发条件的对应关系

---

## 一、完整调用链路概览

### 1.1 链路总览图

```
用户调用: expr.evalf(n=15, subs={x: 3.14}, maxn=100)
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 阶段 1: EvalfMixin.evalf() - 入口点                             │
│   - 十进制精度 n → 二进制精度 prec = dps_to_prec(n)             │
│   - 构建 options 字典 (maxprec, chop, strict, subs, verbose)   │
│   - 初始保护位: 调用 evalf(self, prec + 4, options)             │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 阶段 2: evalf() - 核心分发器                                     │
│   - 通过 evalf_table[type(x)] 查找类型特定处理函数              │
│   - 若未注册，回退到 x._eval_evalf(prec)                        │
│   - 应用 chop 选项处理微小项                                      │
│   - 应用 strict 选项检查精度是否耗尽                              │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼ 假设表达式类型为 Add (加法)
┌─────────────────────────────────────────────────────────────────┐
│ 阶段 3: evalf_add() - 类型特定处理 (示例)                       │
│   - 外层自适应循环: while 1                                       │
│   - 递归求值子表达式: evalf(arg, prec + 10, options)             │
│   - 合并计算: add_terms()                                         │
│   - 精度检查: complex_accuracy()                                  │
│   - 精度不足时: prec += max(10 + 2**i, target_prec - acc)       │
│   - 达到 maxprec 时退出                                           │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼ 子表达式可能是 Mul、Pow、Function 等
┌─────────────────────────────────────────────────────────────────┐
│ 阶段 4: 子表达式递归求值                                          │
│   - 每个子表达式回到阶段 2                                        │
│   - 不同类型有不同的精度调整策略                                  │
│   - 返回 (re, im, re_acc, im_acc) 格式的结果                    │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼ 所有子表达式完成
┌─────────────────────────────────────────────────────────────────┐
│ 阶段 5: 结果合并与精度回传                                        │
│   - add_terms() 计算 sum_accuracy                                │
│   - complex_accuracy() 计算最终相对精度                          │
│   - 结果向上层回传 (re, im, re_acc, im_acc)                     │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 阶段 6: 入口点收尾                                                │
│   - 将 mpf 元组转换为 Float 对象                                  │
│   - 使用 re_acc/im_acc 确定最终精度                              │
│   - 返回 Float 或 ComplexInfinity/NaN                           │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 精度转换关系

| 层级 | 精度单位 | 转换关系 | 示例值 |
|-----|---------|---------|-------|
| 用户接口 | 十进制 (dps) | `n=15` | 15 位有效数字 |
| 内部计算 | 二进制 (bits) | `prec = dps_to_prec(n)` | 53 bits (≈15.95 dps) |
| 初始保护位 | 二进制 | `prec + 4` | 57 bits |
| 子表达式保护位 | 二进制 | `prec + 10` (evalf_add) | 67 bits |
| 三角函数保护位 | 二进制 | `prec + 20` (evalf_trig) | 73 bits |

---

## 二、阶段 1：入口点精度初始化

### 2.1 代码位置与关键逻辑

**文件**：`sympy/core/evalf.py:1569-1690`

**核心代码**：

```python
def evalf(self, n=15, subs=None, maxn=100, chop=False, strict=False, quad=None, verbose=False):
    # ... 参数验证 ...
    
    if not evalf_table:
        _create_evalf_table()
    
    # 关键 1: 十进制 → 二进制精度转换
    prec = dps_to_prec(n)  # 15 dps → 53 bits
    
    # 关键 2: 构建 options 字典
    options = {
        'maxprec': max(prec, int(maxn*LG10)),  # maxn=100 → ~332 bits
        'chop': chop,
        'strict': strict,
        'verbose': verbose
    }
    if subs is not None:
        options['subs'] = subs
    if quad is not None:
        options['quad'] = quad
    
    # 关键 3: 初始保护位 +4
    try:
        result = evalf(self, prec + 4, options)  # 53 + 4 = 57 bits
    except NotImplementedError:
        # 回退路径 ...
```

### 2.2 精度转换公式

```python
# dps_to_prec 实现 (来自 mpmath)
# 1 decimal digit ≈ log2(10) ≈ 3.3219 bits
prec = int(n * math.log2(10) + 1)  # 向上取整

# 反向转换
dps = int(prec / math.log2(10))

# maxprec 计算
# LG10 = math.log2(10) ≈ 3.3219
maxprec = max(prec, int(maxn * LG10))
# 例如: maxn=100 → 100 * 3.3219 ≈ 332 bits
```

### 2.3 性能代价

| 操作 | 代价 | 触发条件 |
|-----|------|---------|
| `dps_to_prec()` | O(1) | 总是 |
| `_create_evalf_table()` | O(1) (首次) | 首次调用 |
| 构建 options | O(1) | 总是 |

---

## 三、阶段 2：核心分发器与类型路由

### 3.1 代码位置与关键逻辑

**文件**：`sympy/core/evalf.py:1459-1544`

**核心代码**：

```python
def evalf(x: Expr, prec: int, options: OPT_DICT) -> TMP_RES:
    """
    返回格式: (re, im, re_acc, im_acc)
    - re/im: mpf 元组 (sign, man, exp, bc) 或 None (表示 0)
    - re_acc/im_acc: 有效精度位数 (二进制)
    """
    try:
        # 关键 1: 类型查找表路由
        rf = evalf_table[type(x)]
        r = rf(x, prec, options)
    except KeyError:
        # 关键 2: 回退路径 - 使用 _eval_evalf 方法
        if 'subs' in options:
            x = x.subs(evalf_subs(prec, options['subs']))
        xe = x._eval_evalf(prec)
        if xe is None:
            raise NotImplementedError
        # ... 转换为 (re, im, re_acc, im_acc) 格式
        r = re, im, reprec, imprec
    
    # 关键 3: 后处理
    chop = options.get('chop', False)
    if chop:
        r = chop_parts(r, chop_prec)
    
    if options.get("strict"):
        check_target(x, r, prec)  # 精度检查
    
    return r
```

### 3.2 evalf_table 路由表

**文件**：`sympy/core/evalf.py:1399-1456`

| 表达式类型 | 处理函数 | 精度策略特点 |
|-----------|---------|-------------|
| `Float` | `evalf_float` | 直接返回存储的 mpf |
| `Rational` | `evalf_rational` | 转换为指定精度浮点数 |
| `Integer` | `evalf_integer` | 转换为指定精度浮点数 |
| `Pi` | `lambda x, prec, ...` | 调用 `mpf_pi(prec)` |
| `Exp1` | `lambda x, prec, ...` | 调用 `mpf_e(prec)` |
| `Add` | `evalf_add` | 外层自适应循环 + 保护位 +10 |
| `Mul` | `evalf_mul` | 精度基本保持 + 保护位与参数数量相关 |
| `Pow` | `evalf_pow` | 整数幂增加 `log2(|p|)`，大指数动态调整 |
| `sin/cos/tan` | `evalf_trig` | 保护位 +20，接近根时指数级提升 |
| `log` | `evalf_log` | 接近 1 时特殊处理 |
| `Integral` | `evalf_integral` | 外层自适应循环 |
| `Sum` | `evalf_sum` | 超几何级数或 Euler-Maclaurin |

### 3.3 回退路径：`_eval_evalf`

当类型未在 `evalf_table` 中注册时，使用各类型的 `_eval_evalf` 方法：

**文件**：`sympy/core/function.py:535-603`

```python
def _eval_evalf(self, prec):
    # 1. 查找 mpmath 函数
    func = _get_mpmath_func(self.func.__name__)
    
    # 2. 参数转换为更高精度 (prec + 5)
    args = [arg._to_mpmath(prec + 5) for arg in args]
    
    # 3. 在指定精度下调用
    with workprec(prec):
        v = func(*args)
    
    return Expr._from_mpmath(v, prec)
```

### 3.4 性能代价

| 操作 | 代价 | 触发条件 |
|-----|------|---------|
| 类型查找 `evalf_table[type(x)]` | O(1) | 总是 |
| 回退路径 `x._eval_evalf()` | O(1) + 子表达式 | 类型未注册 |
| `chop_parts()` | O(1) | `chop=True` 或数值 |
| `check_target()` | O(1) | `strict=True` |

---

## 四、阶段 3：以 Add 为例的完整求值流程

### 4.1 外层自适应循环

**文件**：`sympy/core/evalf.py:585-631`

**核心代码**：

```python
def evalf_add(v: 'Add', prec: int, options: OPT_DICT) -> TMP_RES:
    oldmaxprec = options.get('maxprec', DEFAULT_MAXPREC)  # 默认 333 bits
    
    i = 0
    target_prec = prec  # 目标精度
    
    # 关键: 外层自适应循环
    while 1:
        # 关键 1: 临时限制 maxprec，防止子表达式过度提升
        options['maxprec'] = min(oldmaxprec, 2*prec)
        
        # 关键 2: 递归求值所有子表达式，使用 prec + 10 保护位
        terms = [evalf(arg, prec + 10, options) for arg in v.args]
        
        # 关键 3: 实部/虚部分别合并
        re, re_acc = add_terms(
            [a[0::2] for a in terms if isinstance(a, tuple) and a[0]], 
            prec, target_prec)
        im, im_acc = add_terms(
            [a[1::2] for a in terms if isinstance(a, tuple) and a[1]], 
            prec, target_prec)
        
        # 关键 4: 计算实际达到的精度
        acc = complex_accuracy((re, im, re_acc, im_acc))
        
        # 关键 5: 精度检查
        if acc >= target_prec:
            # 达到目标精度，退出循环
            break
        else:
            # 精度不足，检查是否超过最大限制
            if (prec - target_prec) > options['maxprec']:
                break  # 无法再提升
            
            # 关键 6: 精度提升策略
            # 提升量 = max(10 + 2^i, 目标精度 - 当前精度)
            prec = prec + max(10 + 2**i, target_prec - acc)
            i += 1
    
    # 恢复原始 maxprec
    options['maxprec'] = oldmaxprec
    
    return re, im, re_acc, im_acc
```

### 4.2 精度提升序列示例

假设目标精度 `target_prec = 53` bits (15 dps)：

| 迭代次数 i | 当前 prec | 提升量计算 | 新 prec | 说明 |
|-----------|-----------|-----------|---------|------|
| 0 | 53 | `max(10+1, 53-acc)` | ~64+ | 首次提升至少 11 bits |
| 1 | 64+ | `max(10+2, 53-acc)` | ~76+ | 指数级增长 2^i |
| 2 | 76+ | `max(10+4, 53-acc)` | ~90+ | |
| 3 | 90+ | `max(10+8, 53-acc)` | ~108+ | |
| ... | ... | ... | ... | 直到达到 maxprec |

**最坏情况**：需要多次迭代才能达到目标精度，每次迭代都要重新计算所有子表达式。

### 4.3 add_terms 内部精度计算

**文件**：`sympy/core/evalf.py:499-582`

**核心代码**：

```python
def add_terms(terms: list, prec: int, target_prec: int):
    """
    terms: 列表，每个元素是 (mpf_val, accuracy)
    返回: (result_mpf, result_accuracy)
    """
    working_prec = 2*prec  # 内部使用双倍工作精度
    
    sum_man, sum_exp = 0, 0
    absolute_err: list[int] = []
    
    for x, accuracy in terms:
        sign, man, exp, bc = x  # mpf 元组解析
        if sign:
            man = -man
        
        # 关键 1: 计算绝对误差的对数估计
        # absolute_err = log2(|x|) - accuracy
        # 因为: |error| ≈ |x| / 2^accuracy
        absolute_err.append(bc + exp - accuracy)
        
        # 关键 2: 对齐指数并相加
        delta = exp - sum_exp
        if exp >= sum_exp:
            # 新数更大，左移现有的和
            if delta > working_prec and ...:
                # 差距太大，直接替换
                sum_man = man
                sum_exp = exp
            else:
                sum_man += (man << delta)  # 尾数左移
        else:
            # 新数更小，右移现有的和
            delta = -delta
            if delta - bc > working_prec:
                if not sum_man:
                    sum_man, sum_exp = man, exp
            else:
                sum_man = (sum_man << delta) + man
                sum_exp = exp
    
    # 关键 3: 计算最大绝对误差
    absolute_error = max(absolute_err)
    
    # 关键 4: 抵消检测 - 结果为 0 但不是精确 0
    if not sum_man:
        # 返回 scaled_zero 特殊标记
        return scaled_zero(absolute_error)
    
    # 关键 5: 计算结果精度
    sum_bc = sum_man.bit_length()  # 结果尾数的位数
    # sum_accuracy = log2(|result|) - absolute_error
    sum_accuracy = sum_exp + sum_bc - absolute_error
    
    # 关键 6: 归一化到目标精度
    r = normalize(sum_sign, sum_man, sum_exp, sum_bc, target_prec, rnd)
    
    return r, sum_accuracy
```

### 4.4 精度计算数学原理

#### 4.4.1 术语定义

```
对于浮点数 x = (-1)^sign * man * 2^exp
- man: 尾数 (mantissa)
- exp: 指数 (exponent)
- bc = man.bit_length(): 尾数的二进制位数

近似值:
- |x| ≈ 2^(exp + bc)  (因为 man 最高位为 1)
- log2(|x|) ≈ exp + bc  (使用 fastlog 函数)
```

#### 4.4.2 相对误差与有效位

```
设:
- 真值: x_true
- 计算值: x_approx
- 相对误差: |x_approx - x_true| / |x_true| ≈ 2^(-accuracy)

则:
- accuracy: 有效二进制位数
- 例如: accuracy = 53 表示约 15 位十进制有效数字

绝对误差估计:
|error| ≈ |x| / 2^accuracy
log2(|error|) ≈ log2(|x|) - accuracy
             ≈ (exp + bc) - accuracy
```

#### 4.4.3 加法的精度损失

```
加法: s = a + b

情况 1: a 和 b 同号或相差不大
- 精度基本保持

情况 2: a ≈ -b (抵消)
- s = a + b ≈ 0
- 尾数有效位数大幅减少
- 例如: a = 1.0000001, b = -1.0 → s = 0.0000001
  - 原来有 ~24 位有效位
  - 结果只有 ~1 位有效位

情况 3: |a| >> |b|
- 小的数可能被"吃掉"
- s ≈ a
- 精度由 a 决定
```

### 4.5 complex_accuracy 精度合并

**文件**：`sympy/core/evalf.py:234-260`

```python
def complex_accuracy(result: TMP_RES) -> int | Any:
    """
    计算复数结果的相对精度
    
    对于 z = re + im*I:
    - 分别有 re_acc 和 im_acc
    - 计算整体的相对精度
    """
    re, im, re_acc, im_acc = result
    
    if not im:
        if not re:
            return INF  # 精确 0
        return re_acc  # 纯实数
    
    if not re:
        return im_acc  # 纯虚数
    
    # 复数情况: 使用 max 范数近似
    re_size = fastlog(re)  # ≈ log2(|re|)
    im_size = fastlog(im)  # ≈ log2(|im|)
    
    # 绝对误差 = max(|re|/2^re_acc, |im|/2^im_acc)
    # log2(绝对误差) = max(log2(|re|)-re_acc, log2(|im|)-im_acc)
    absolute_error = max(re_size - re_acc, im_size - im_acc)
    
    # 相对误差 = 绝对误差 / max(|re|, |im|)
    # log2(相对误差) = absolute_error - max(re_size, im_size)
    relative_error = absolute_error - max(re_size, im_size)
    
    # 相对精度 = -log2(相对误差)
    return -relative_error
```

### 4.6 性能代价汇总

| 操作 | 代价 | 触发条件 |
|-----|------|---------|
| 递归求值子表达式 | O(n × 子表达式代价) | 每次迭代 |
| `add_terms()` 尾数对齐 | O(working_prec) | 每次迭代 |
| `complex_accuracy()` | O(1) | 每次迭代 |
| **迭代重试** | **指数级增长** | **精度不足时** |

---

## 五、阶段 4：其他运算的精度策略

### 5.1 乘法运算 evalf_mul

**文件**：`sympy/core/evalf.py:634-757`

**核心代码**：

```python
def evalf_mul(v: 'Mul', prec: int, options: OPT_DICT) -> TMP_RES:
    # 乘法的相对精度通常保持不变
    acc = prec
    
    # 保护位 = 参数数量 + 5
    working_prec = prec + len(args) + 5
    
    # ... 计算过程 ...
    
    for wre, wim, wre_acc, wim_acc in complex_factors[i0:]:
        # 精度取最小值
        acc = min(acc, complex_accuracy((wre, wim, wre_acc, wim_acc)))
    
    return re, im, acc, acc
```

**精度策略**：
- 乘法的相对误差传播是线性的
- 保护位数量与因子数量相关
- 没有外层自适应循环（除非发生 0×∞ 等特殊情况）

### 5.2 幂运算 evalf_pow

**文件**：`sympy/core/evalf.py:760-880`

**核心代码**：

```python
def evalf_pow(v: 'Pow', prec: int, options) -> TMP_RES:
    base, exp = v.args
    
    # 情况 1: 整数幂 x^n
    if exp.is_Integer:
        p = exp.p
        
        # 关键: 整数幂的误差放大
        # d(x^p)/x = p * x^(p-1)
        # 相对误差放大 |p| 倍
        # 需要额外 log2(|p|) 位精度
        prec += int(math.log2(abs(p)))
        
        result = evalf(base, prec + 5, options)
        # ...
    
    # 情况 2: 一般幂 x^y
    # 先评估指数大小
    prec += 10  # 初始保护位
    result = evalf(exp, prec, options)
    yre, yim, _, _ = result
    
    ysize = fastlog(yre)  # log2(|y|)
    
    # 关键: 大指数需要更多精度
    if ysize > 5:  # |y| >= 32
        prec += ysize  # 精度增加 log2(|y|)
        yre, yim, _, _ = evalf(exp, prec, options)
    
    # ... 计算 ...
```

**精度策略数学原理**：

```
对于 f(x) = x^p:
- df/dx = p * x^(p-1)
- 相对误差: |df/f| = |p * dx/x|
- 相对误差放大 |p| 倍
- 需要额外 log2(|p|) 位精度

对于 f(x) = x^y (y 不是整数):
- 使用指数-对数重写: x^y = e^(y * ln x)
- 指数运算对参数精度非常敏感
- 大的 |y| 会放大 ln x 的误差
- 需要额外 log2(|y|) 位精度
```

### 5.3 三角函数 evalf_trig

**文件**：`sympy/core/evalf.py:895-954`

**核心代码**：

```python
def evalf_trig(v: Expr, prec: int, options: OPT_DICT) -> TMP_RES:
    arg = v.args[0]
    
    # 初始保护位 +20
    xprec = prec + 20
    re, im, re_acc, im_acc = evalf(arg, xprec, options)
    
    xsize = fastlog(re)  # log2(|arg|)
    
    # 情况 1: 小参数 (|arg| < 2)
    if xsize < 1:
        return func(re, prec, rnd), None, prec, None
    
    # 情况 2: 大参数 (|arg| >= 1024)
    if xsize >= 10:
        # 需要更多精度来计算模 2π
        xprec = prec + xsize
        re, im, re_acc, im_acc = evalf(arg, xprec, options)
    
    # 情况 3: 接近根的情况 - 内层自适应循环
    while 1:
        y = func(re, prec, rnd)  # 计算 sin/cos/tan
        ysize = fastlog(y)      # log2(|result|)
        
        # gap = -log2(|result|) = 接近 0 的程度
        # 例如: result ≈ 1e-10 → gap ≈ 33 bits
        gap = -ysize
        
        # 精度估计
        # 绝对精度 = xprec - xsize (因为三角函数是周期函数)
        # 有效精度 = 绝对精度 - gap
        accuracy = (xprec - xsize) - gap
        
        if accuracy < prec:
            # 精度不足
            if xprec > options.get('maxprec', DEFAULT_MAXPREC):
                return y, None, accuracy, None  # 无法再提升
            
            # 关键: 精度提升量 = gap
            # 越接近根，需要的精度越高
            xprec += gap
            re, im, re_acc, im_acc = evalf(arg, xprec, options)
            continue
        else:
            return y, None, prec, None
```

**精度策略数学原理**：

```
对于 sin(x):
- 当 x ≈ kπ 时，sin(x) ≈ (-1)^k * (x - kπ)
- 若 |x - kπ| ≈ ε，则 |sin(x)| ≈ ε
- 要计算 sin(x) 到 prec 位精度:
  - 需要 x - kπ 到 prec 位精度
  - 但 x 本身可能很大，例如 x = 10^100 * π + 1e-10
  - 需要计算 x mod 2π 到约 43 位精度 (因为 1e-10 ≈ 2^-33)
  - 因此 x 需要 100 * log2(10) + 43 ≈ 332 + 43 = 375 位精度

一般情况:
- 若 |sin(x)| ≈ 2^(-gap)
- 则需要额外 gap 位精度来计算 x mod 2π
- 精度提升量 = gap
```

### 5.4 对数函数 evalf_log

**文件**：`sympy/core/evalf.py:957-1008`

**核心代码**：

```python
def evalf_log(expr: 'log', prec: int, options: OPT_DICT) -> TMP_RES:
    workprec = prec + 10
    result = evalf(arg, workprec, options)
    xre, xim, xacc, _ = result
    
    # 计算 log(x)
    re = mpf_ln(mpf_abs(xre), prec, rnd)
    
    # 检查 x 是否接近 1
    size = fastlog(re)  # log2(|log(x)|)
    
    # 若 prec - size > workprec:
    # 说明 log(x) 很小，即 x 接近 1
    # 直接计算 log(x) 会损失精度
    if prec - size > workprec:
        # 重写为 log(1 + (x-1)) ≈ x-1
        # 需要高精度计算 x-1
        add = Add(S.NegativeOne, arg, evaluate=False)
        xre, xim, xre_acc, _ = evalf_add(add, prec, options)
        
        prec2 = workprec - fastlog(xre)
        # 计算 log(1 + xre) 而不是 log(x)
        re = mpf_ln(mpf_abs(mpf_add(xre, fone, prec2)), prec, rnd)
```

**精度策略数学原理**：

```
对于 log(x):
- 当 x ≈ 1 时，令 x = 1 + ε，其中 |ε| << 1
- log(1 + ε) ≈ ε - ε²/2 + ε³/3 - ...
- 若直接计算 log(x) = log(1.000...001)
  - 尾数大部分是 1，有效信息在最低位
  - 精度损失严重

更好的方法:
- 计算 δ = x - 1 (可能需要高精度)
- 然后用 log1p 或级数计算 log(1 + δ)
- 当 |δ| 很小时，需要更多位数来表示 δ
```

### 5.5 数值积分 evalf_integral

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
        
        # 精度提升策略
        if accuracy == -1:
            workprec *= 2  # 可能为 0，指数级提升
        else:
            workprec += max(prec, 2**i)
        workprec = min(workprec, maxprec)
        i += 1
    
    return result
```

---

## 六、完整链路追踪示例

### 6.1 示例表达式

```python
from sympy import *
x = Symbol('x')
expr = sin(x**2 + pi) - sin(pi)  # 等价于 sin(x² + π) = -sin(x²)

# 代入 x = 1e-5，计算 15 位精度
result = expr.evalf(15, subs={x: 1e-5})
```

### 6.2 完整链路追踪

```
用户调用: expr.evalf(15, subs={x: 1e-5})
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 阶段 1: EvalfMixin.evalf()                                            │
│                                                                        │
│ n = 15 → prec = dps_to_prec(15) = 53 bits                           │
│ options = {                                                            │
│     'maxprec': max(53, int(100*3.3219)) = 332,                     │
│     'subs': {x: 1e-5},                                                 │
│     'chop': False, 'strict': False, 'verbose': False                 │
│ }                                                                      │
│ 调用 evalf(expr, 53 + 4 = 57, options)                                │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 阶段 2: evalf() 分发器                                                 │
│                                                                        │
│ expr 类型: Add → evalf_table[Add] = evalf_add                        │
│ 调用 evalf_add(expr, 57, options)                                     │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 阶段 3: evalf_add() 外层循环 (第 1 次迭代)                            │
│                                                                        │
│ i = 0, target_prec = 57                                                │
│ options['maxprec'] = min(332, 2*57) = 114                           │
│                                                                        │
│ 需要递归求值两个子表达式:                                               │
│   arg1: sin(x**2 + pi)                                                │
│   arg2: -sin(pi) = 0 (精确)                                           │
│                                                                        │
│ 调用 evalf(arg1, 57 + 10 = 67, options)                               │
│ 调用 evalf(arg2, 67, options) → (None, None, 67, None)  (精确 0)    │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼ 处理 arg1 = sin(x**2 + pi)
┌──────────────────────────────────────────────────────────────────────┐
│ 阶段 2: evalf() 分发器                                                 │
│                                                                        │
│ arg1 类型: sin → evalf_table[sin] = evalf_trig                       │
│ 调用 evalf_trig(sin(...), 67, options)                                │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 阶段 4: evalf_trig()                                                   │
│                                                                        │
│ arg = x**2 + pi (类型: Add)                                            │
│ xprec = 67 + 20 = 87  (初始保护位)                                    │
│                                                                        │
│ 首先需要计算 arg 的值:                                                 │
│ 调用 evalf(x**2 + pi, 87, options)                                    │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼ 递归计算 x**2 + pi
┌──────────────────────────────────────────────────────────────────────┐
│ 阶段 2-4: 递归求值子表达式                                             │
│                                                                        │
│ x**2 + pi 是 Add，需要:                                               │
│   arg_a: x**2                                                          │
│   arg_b: pi (常数)                                                     │
│                                                                        │
│ arg_b = pi: 直接返回 (mpf_pi(97), None, 97, None)                   │
│                                                                        │
│ arg_a = x**2: 类型 Pow                                                 │
│   - 先计算 x 的值: x = 1e-5                                            │
│   - 类型 Symbol，在 subs 中: x → 1e-5                                 │
│   - 转换为 Float: 1e-5 ≈ 2^-17                                        │
│                                                                        │
│ evalf_pow(x**2, 97, options):                                         │
│   - exp = 2 是整数                                                     │
│   - prec += log2(2) = 1 → 98 bits                                     │
│   - 结果: (1e-10, None, 97, None)                                     │
│                                                                        │
│ 现在合并: 1e-10 + π ≈ π (因为 1e-10 很小)                            │
│ 但精确来说: arg = π + 1e-10                                           │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼ 回到 evalf_trig
┌──────────────────────────────────────────────────────────────────────┐
│ 阶段 4: evalf_trig() 继续                                             │
│                                                                        │
│ re = π + 1e-10 ≈ 3.141592653599793                                   │
│ xsize = fastlog(re) ≈ 2 (因为 2^1=2, 2^2=4, π 在中间)               │
│                                                                        │
│ xsize = 2 < 10，不需要大参数处理                                       │
│                                                                        │
│ 进入内层循环:                                                           │
│   y = sin(π + 1e-10) = -sin(1e-10) ≈ -1e-10                         │
│   ysize = fastlog(y) ≈ -33 (因为 1e-10 ≈ 2^-33)                     │
│   gap = -ysize = 33                                                   │
│                                                                        │
│   accuracy = (xprec - xsize) - gap                                    │
│           = (87 - 2) - 33 = 52                                       │
│                                                                        │
│   目标 prec = 67                                                       │
│   accuracy(52) < prec(67) → 精度不足！                                │
│                                                                        │
│   需要提升精度:                                                         │
│   xprec += gap → 87 + 33 = 120                                       │
│                                                                        │
│   重新计算 arg = x**2 + pi，使用 xprec = 120                          │
│   这需要重新走一遍递归链路...                                          │
│                                                                        │
│   第 2 次内层循环:                                                      │
│   accuracy = (120 - 2) - 33 = 85 ≥ 67 ✓                              │
│                                                                        │
│   返回: y = -1e-10, None, 67, None                                   │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼ 回到 evalf_add
┌──────────────────────────────────────────────────────────────────────┐
│ 阶段 3: evalf_add() 继续                                              │
│                                                                        │
│ terms = [                                                              │
│   (-1e-10, None, 67, None),  # sin(x²+π)                             │
│   (None, None, 67, None)     # -sin(π) = 0                           │
│ ]                                                                      │
│                                                                        │
│ add_terms 合并:                                                        │
│   - 只有一个非零项                                                      │
│   - 直接返回 (-1e-10, 67)                                              │
│                                                                        │
│ complex_accuracy:                                                      │
│   - 纯实数，返回 67                                                    │
│                                                                        │
│ acc(67) ≥ target_prec(57) ✓                                           │
│ 退出外层循环                                                            │
│                                                                        │
│ 返回: (-1e-10, None, 67, None)                                        │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 阶段 1: EvalfMixin.evalf() 收尾                                       │
│                                                                        │
│ re = -1e-10, re_acc = 67                                              │
│                                                                        │
│ 最终精度: p = max(min(53, 67), 1) = 53 (目标精度)                    │
│                                                                        │
│ 创建 Float: Float._new(re, 53)                                         │
│                                                                        │
│ 返回: -1.00000000000000e-10                                          │
└──────────────────────────────────────────────────────────────────────┘
```

### 6.3 性能代价分析

| 阶段 | 操作 | 代价估计 | 触发原因 |
|-----|------|---------|---------|
| 入口点 | 精度转换 | O(1) | 总是 |
| evalf_add 外层 | 第 1 次迭代 | O(子表达式代价) | 总是 |
| evalf_trig 初始 | 保护位 +20 | O(子表达式代价) | 总是 |
| **evalf_trig 内层** | **精度不足，提升 +33** | **O(重新递归计算)** | **结果接近根 (gap=33)** |
| evalf_add 收尾 | 合并与检查 | O(1) | 总是 |

**关键性能瓶颈**：
- 三角函数接近根时触发内层自适应循环
- 需要重新递归计算参数值
- 提升量 `gap = -log2(|result|)` 可能很大

---

## 七、性能代价与触发条件汇总

### 7.1 各运算的性能风险

| 运算类型 | 触发条件 | 精度提升量 | 性能代价 |
|---------|---------|-----------|---------|
| **Add (加法)** | 子表达式精度不足 | `max(10 + 2^i, target - acc)` | 指数级增长 |
| **Add (抵消)** | `sum_man = 0` (精确抵消) | 返回 `scaled_zero` | 后续运算可能需要更高精度 |
| **Mul (乘法)** | 一般情况 | 无提升 (acc = prec) | O(1) 相对 |
| **Pow (整数幂)** | `\|p\|` 很大 | `log2(\|p\|)` bits | O(prec × log2(\|p\|)) |
| **Pow (大指数)** | `\|y\| ≥ 32` | `log2(\|y\|)` bits | O(prec × log2(\|y\|)) |
| **Trig (大参数)** | `\|arg\| ≥ 1024` | `log2(\|arg\|)` bits | O(prec × log2(\|arg\|)) |
| **Trig (接近根)** | `\|result\|` 很小 | `gap = -log2(\|result\|)` | **指数级，gap 可能很大** |
| **Log (接近 1)** | `\|log(x)\|` 很小 | 重算 `x - 1` | 需要高精度减法 |
| **Integral** | 精度不足 | `max(prec, 2^i)` 或 ×2 | 多次积分计算 |
| **Sum (级数)** | 收敛慢 | 取决于算法 | 可能需要很多项 |

### 7.2 精度提升策略总结

```
策略 1: 保护位 (Guard Digits)
- 入口点: +4 bits
- 加法子表达式: +10 bits
- 三角函数初始: +20 bits
- 一般函数参数: +5 bits
- 目的: 减少需要重算的概率

策略 2: 基于运算特性的预提升
- 整数幂: +log2(|p|) bits
- 大指数: +log2(|y|) bits
- 大参数三角函数: +log2(|arg|) bits
- 目的: 基于误差分析的精确提升

策略 3: 结果依赖的动态提升 (自适应循环)
- 加法外层: prec += max(10 + 2^i, target - acc)
- 三角函数内层: xprec += gap  (gap = -log2(|result|))
- 积分: workprec += max(prec, 2^i) 或 ×2
- 目的: 根据实际计算结果调整
```

### 7.3 最坏情况性能场景

| 场景 | 表达式示例 | 性能特征 |
|-----|-----------|---------|
| **灾难性抵消** | `N(pi + 1e-1000 - pi, 15, maxn=1500)` | 需要 1000+ 位精度，多次迭代 |
| **接近根的三角函数** | `N(sin(pi * 10**100 + 1e-10), 15)` | 需要 332 + 33 = 365 位精度 |
| **大指数幂运算** | `N(1.0000001**(10**20), 10)` | 需要高精度计算指数 |
| **接近 1 的对数** | `N(log(1 + 1e-50), 15)` | 需要高精度计算 `x - 1` |
| **病态多项式** | Rump 多项式: `N(a, 15, subs={x: 77617, y: 33096})` | 需要高精度来分辨 |

### 7.4 性能优化建议

| 场景 | 建议 | 效果 |
|-----|------|------|
| 知道结果不会接近 0 | 无特殊处理 | 保护位通常足够 |
| 可能发生抵消 | 使用 `chop=True` 或 `maxn=更高` | 避免精度耗尽异常 |
| 三角函数大参数 | 手动约简 `arg mod 2π` | 减少精度需求 |
| 对数接近 1 | 重写为 `log1p(x-1)` 形式 | 避免精度损失 |
| 高精度需求 | 安装 gmpy | 大幅加速大整数运算 |
| 迭代计算 | 设置合理 `maxn` | 防止无限循环 |

---

## 八、关键数据结构与函数速查

### 8.1 数据结构

```python
# mpf 元组 (mpmath 内部浮点数表示)
MPF_TUP = tuple[sign, man, exp, bc]
# - sign: 0 (正) 或 1 (负)
# - man: 尾数 (大整数，奇数)
# - exp: 指数
# - bc: bitcount(man) = 尾数的二进制位数
# 值 = (-1)^sign * man * 2^exp

# 临时结果格式 (所有 evalf_* 函数的返回格式)
TMP_RES = tuple[re, im, re_acc, im_acc]
# - re: 实部的 mpf 元组，或 None (表示精确 0)
# - im: 虚部的 mpf 元组，或 None
# - re_acc: 实部的有效精度位数 (二进制)
# - im_acc: 虚部的有效精度位数 (二进制)

# 特殊: scaled_zero (抵消标记)
SCALED_ZERO_TUP = tuple[list[sign], man, exp, bc]
# - sign 被包装在 list 中表示这不是普通 mpf
# 用于表示 "计算结果为 0 但不是精确 0"
# 精度信息在 absolute_error 中
```

### 8.2 关键函数

| 函数 | 文件位置 | 作用 |
|-----|---------|------|
| `dps_to_prec(n)` | mpmath | 十进制 → 二进制精度 |
| `fastlog(x)` | `evalf.py:112` | 快速 `log2(|x|)` 近似 |
| `complex_accuracy(r)` | `evalf.py:234` | 计算复数结果的相对精度 |
| `add_terms(terms, ...)` | `evalf.py:499` | 带精度追踪的加法合并 |
| `scaled_zero(err)` | `evalf.py:183` | 创建抵消标记 |
| `evalf_table` | `evalf.py:1399` | 类型 → 处理函数路由表 |

### 8.3 精度计算公式速查

```
1. 有效精度计算 (加法后):
   sum_accuracy = sum_exp + sum_bc - absolute_error
   其中:
   - sum_exp + sum_bc ≈ log2(|result|)
   - absolute_error = max(bc + exp - accuracy for each term)
                    ≈ max(log2(|term|) - term_accuracy)

2. 复数精度合并:
   re_size = fastlog(re) ≈ log2(|re|)
   im_size = fastlog(im) ≈ log2(|im|)
   absolute_error = max(re_size - re_acc, im_size - im_acc)
   relative_error = absolute_error - max(re_size, im_size)
   return -relative_error  # 相对精度

3. 三角函数精度估计 (接近根时):
   ysize = fastlog(result) ≈ log2(|sin(x)|)
   gap = -ysize  # 接近 0 的程度
   accuracy = (xprec - xsize) - gap
   其中:
   - xprec - xsize = 绝对精度 (计算 x mod 2π 的精度)
   - 需要减去 gap 因为有效信息只有 gap 位
```

---

## 九、总结

### 9.1 完整链路的关键洞察

1. **多层保护位策略**：
   - 入口点 `+4` bits
   - 子表达式 `+10` bits
   - 三角函数 `+20` bits
   - 目的：减少重算概率

2. **两种精度提升模式**：
   - **预提升**：基于运算特性，如整数幂 `+log2(|p|)`
   - **动态提升**：基于实际结果，如 `xprec += gap`

3. **精度回传机制**：
   - 每个函数返回 `(re, im, re_acc, im_acc)`
   - `re_acc/im_acc` 是有效二进制位数
   - 上层使用 `complex_accuracy()` 合并判断

4. **重算触发条件**：
   - 加法外层：`acc < target_prec`
   - 三角函数内层：`(xprec - xsize) - gap < prec`
   - 积分：`accuracy < prec`

5. **最坏性能元凶**：
   - **接近根的三角函数**：`gap = -log2(|result|)` 可能非常大
   - **灾难性抵消**：需要指数级提升精度
   - **大参数周期函数**：模运算需要高精度

### 9.2 设计哲学

SymPy 的 `evalf` 设计体现了以下权衡：

| 设计选择 | 优点 | 缺点 |
|---------|------|------|
| 保护位策略 | 减少重算次数 | 轻微增加单次计算开销 |
| 指数级精度提升 | 快速达到目标 | 可能"跳过"最优精度 |
| `maxprec` 上限 | 防止无限循环 | 可能返回低精度结果 |
| `scaled_zero` 标记 | 正确追踪抵消后的精度 | 增加代码复杂度 |
| 类型分发表 | 高效路由 | 新类型需要注册 |

这种设计使得 `evalf` 能够：
1. 在普通情况下快速返回结果
2. 在困难情况下自动提升精度
3. 提供可配置的性能-精度权衡
