# SymPy 多项式数域降级与统一机制深度分析

## 1. 概述

本文档深入分析 SymPy 中不同数域运算时的**降级触发条件**、**边界分支**和**合并失败时的回退路径**。通过追踪关键代码的执行流程，明确完整的降级链路。

---

## 2. 降级触发条件总览

### 2.1 三大降级场景

| 场景 | 触发阶段 | 主要异常/返回值 |
|------|---------|----------------|
| 域构造失败 | `construct_domain` | 返回 `False` 或 `None` |
| 数域统一失败 | `unify` | 抛出 `UnificationFailed` |
| 元素转换失败 | `convert`/`from_sympy` | 抛出 `CoercionFailed` |

### 2.2 完整降级链路

```
输入表达式
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 第一阶段：construct_domain() 选择最小覆盖域                   │
│ ┌─────────────┐     ┌─────────────────┐     ┌─────────────┐ │
│ │_construct_  │     │_construct_      │     │_construct_  │ │
│ │simple()     │────▶│composite()      │────▶│expression() │ │
│ │ (简单域)    │     │ (复合域)         │     │ (EX域)      │ │
│ └─────────────┘     └─────────────────┘     └─────────────┘ │
│       │                   │                        │         │
│       ▼                   ▼                        ▼         │
│  返回 False/None     返回 None               返回 EX         │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 第二阶段：unify() 统一运算数域                                 │
│ ┌─────────────────────────────────────────────────────────┐  │
│ │ 1. 特征检查：不同特征 → UnificationFailed                │  │
│ │ 2. 符号冲突检查：冲突 → UnificationFailed                │  │
│ │ 3. 按优先级统一：EX > K(x) > K[x] > ALG > CC > RR > QQ │  │
│ │ 4. 最终 fallback：EX                                     │  │
│ └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 第三阶段：convert() 元素转换                                   │
│ ┌─────────────────────────────────────────────────────────┐  │
│ │ 1. 直接转换：成功 → 返回转换后元素                        │  │
│ │ 2. 失败捕获：CoercionFailed                              │  │
│ │ 3. 环升级尝试：环 → 域（如有对应域）                      │  │
│ │ 4. 最终：返回 NotImplemented 或重新抛出异常               │  │
│ └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
    ↓
输出结果（或异常）
```

---

## 3. 第一阶段：construct_domain 详细分析

### 3.1 主流程结构

**代码位置**：`sympy/polys/constructor.py:268-388`

```python
def construct_domain(obj, **args):
    opt = build_options(args)
    
    # 步骤1：尝试简单域
    result = _construct_simple(coeffs, opt)
    
    if result is not None:
        if result is not False:
            domain, coeffs = result  # 成功
        else:
            # 特殊标记：直接降级到 EX
            domain, coeffs = _construct_expression(coeffs, opt)
    else:
        # 步骤2：尝试复合域
        if opt.composite is False:
            result = None
        else:
            result = _construct_composite(coeffs, opt)
        
        if result is not None:
            domain, coeffs = result  # 成功
        else:
            # 步骤3：最终降级到 EX
            domain, coeffs = _construct_expression(coeffs, opt)
    
    return domain, coeffs
```

### 3.2 _construct_simple 降级触发条件

**代码位置**：`sympy/polys/constructor.py:16-79`

#### 3.2.1 触发条件1：浮点数与代数数共存

```python
# constructor.py:30-35
elif coeff.is_Float:
    if algebraics:
        # 同时有浮点数和代数数 → 返回 False（特殊标记）
        return False
    else:
        floats = True
        float_numbers.append(coeff)

# constructor.py:52-56
elif is_algebraic(coeff):
    if floats:
        # 同时有代数数和浮点数 → 返回 False（特殊标记）
        return False
    algebraics = True
```

**判断流程**：
```
遍历所有系数
    ↓
遇到 Float ──▶ 检查 algebraics 标志
    │               │
    │               ├── True ──▶ 返回 False（特殊标记）
    │               │
    │               └── False ──▶ 设置 floats=True
    │
遇到代数数 ──▶ 检查 floats 标志
                    │
                    ├── True ──▶ 返回 False（特殊标记）
                    │
                    └── False ──▶ 设置 algebraics=True
```

**结果**：返回 `False`（特殊标记）→ 直接调用 `_construct_expression` → **EX 域**

#### 3.2.2 触发条件2：复合域类型系数

```python
# constructor.py:57-59
else:
    # 这是复合域类型，如 ZZ[X], EX
    return None
```

**判断流程**：
```
系数类型检查
    ↓
是 Rational? ──No──▶ 是 Float? ──No──▶ 是 Complex? ──No──▶ 是代数数? ──No──▶ 返回 None
    │                    │                    │                    │
    │ Yes                │ Yes                │ Yes                │ Yes
    ▼                    ▼                    ▼                    ▼
处理有理数           处理浮点数            处理复数            处理代数数
```

**结果**：返回 `None` → 进入 `_construct_composite` 尝试

### 3.3 _construct_composite 降级触发条件

**代码位置**：`sympy/polys/constructor.py:132-254`

#### 3.3.1 触发条件1：无生成器

```python
# constructor.py:142-144
polys, gens = parallel_dict_from_basic(numers + denoms)
if not gens:
    return None
```

**判断流程**：
```
调用 parallel_dict_from_basic()
    ↓
gens 为空?
    │
    ├── Yes ──▶ 返回 None
    │
    └── No ──▶ 继续
```

**结果**：返回 `None` → 降级到 EX

#### 3.3.2 触发条件2：生成器是代数数

```python
# constructor.py:146-148
if opt.composite is None:
    if any(gen.is_number and gen.is_algebraic for gen in gens):
        return None  # 生成器是数类型，使用 EX 更合适
```

**判断流程**：
```
遍历所有生成器
    ↓
gen.is_number 且 gen.is_algebraic?
    │
    ├── Yes（任意一个）──▶ 返回 None
    │
    └── No ──▶ 继续
```

**结果**：返回 `None` → 降级到 EX

#### 3.3.3 触发条件3：符号间存在代数关系

```python
# constructor.py:150-158
all_symbols = set()

for gen in gens:
    symbols = gen.free_symbols
    
    if all_symbols & symbols:
        # 生成器之间可能存在代数关系 → 返回 None
        return None
    else:
        all_symbols |= symbols
```

**判断流程**：
```
初始化 all_symbols = 空集
    ↓
遍历每个生成器 gen
    ↓
获取 gen.free_symbols
    ↓
all_symbols & symbols 非空?
    │
    ├── Yes ──▶ 返回 None（存在重叠符号）
    │
    └── No ──▶ 合并 symbols 到 all_symbols，继续
```

**示例**：
- `gens = [x, y]` → 符号无重叠 → 继续
- `gens = [x, x**2]` → 符号 `{x}` 重叠 → 返回 `None`
- `gens = [x+y, x-y]` → 符号 `{x, y}` 重叠 → 返回 `None`

**结果**：返回 `None` → 降级到 EX

### 3.4 construct_domain 降级决策树

```
construct_domain(coeffs, opt)
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 调用 _construct_simple(coeffs, opt)                        │
└────────────────────────────────────────────────────────────┘
    │
    ├────────────────────── result is not None ──────────────▶
    │                           │
    │              ┌────────────┴────────────┐
    │              │                         │
    │         result is False          result is not False
    │              │                         │
    │              ▼                         ▼
    │    ┌─────────────────┐        ┌─────────────────┐
    │    │_construct_      │        │ 返回成功的域     │
    │    │expression()     │        │ (ZZ/QQ/RR/CC/   │
    │    │                 │        │  QQ_I/ZZ_I/ALG) │
    │    │ 结果：EX 域     │        └─────────────────┘
    │    └─────────────────┘
    │
    └────────────────────── result is None ────────────────▶
                                │
                                ▼
                    ┌─────────────────────────────────────┐
                    │ opt.composite is False?              │
                    └─────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                   Yes                      No
                    │                       │
                    ▼                       ▼
              result = None        ┌─────────────────────────┐
                    │              │_construct_composite()   │
                    │              │ 尝试构建复合域           │
                    │              └─────────────────────────┘
                    │                       │
                    │              ┌────────┴────────┐
                    │              │                 │
                    │        result is None    result is not None
                    │              │                 │
                    │              ▼                 ▼
                    │    ┌─────────────────┐  ┌─────────────────┐
                    │    │_construct_      │  │ 返回成功的域     │
                    └───▶│expression()     │  │ (ZZ[x]/QQ(x)/等) │
                         │                 │  └─────────────────┘
                         │ 结果：EX 域     │
                         └─────────────────┘
```

### 3.5 construct_domain 降级触发条件汇总

| 触发条件 | 触发位置 | 返回值 | 回退路径 | 最终域 |
|---------|---------|--------|---------|--------|
| 浮点数 + 代数数共存 | `_construct_simple` | `False` | 直接调用 `_construct_expression` | **EX** |
| 复合域类型系数（非简单数） | `_construct_simple` | `None` | 尝试 `_construct_composite` | 复合域或 **EX** |
| 无生成器（所有系数是常数） | `_construct_composite` | `None` | 调用 `_construct_expression` | **EX** |
| 生成器是代数数 | `_construct_composite` | `None` | 调用 `_construct_expression` | **EX** |
| 生成器符号有重叠（可能有代数关系） | `_construct_composite` | `None` | 调用 `_construct_expression` | **EX** |

---

## 4. 第二阶段：unify 详细分析

### 4.1 主流程结构

**代码位置**：`sympy/polys/domains/domain.py:889-1016`

```python
def unify(K0, K1, symbols=None):
    # 步骤1：带符号约束的统一
    if symbols is not None:
        return K0.unify_with_symbols(K1, symbols)
    
    # 步骤2：相同域直接返回
    if K0 == K1:
        return K0
    
    # 步骤3：特征检查
    if not (K0.has_CharacteristicZero and K1.has_CharacteristicZero):
        if K0.characteristic() != K1.characteristic():
            raise UnificationFailed("Cannot unify %s with %s" % (K0, K1))
        # 特征相同但域不同（如 GF(3) 和 GF(3)[x]）
        return K0.unify_composite(K1)
    
    # 步骤4：EX/EXRAW 优先
    if K0.is_EXRAW:
        return K0
    if K1.is_EXRAW:
        return K1
    if K0.is_EX:
        return K0
    if K1.is_EX:
        return K1
    
    # 步骤5：有限扩域统一
    if K0.is_FiniteExtension or K1.is_FiniteExtension:
        # ... 扩域统一逻辑 ...
    
    # 步骤6：复合域统一
    if K0.is_Composite or K1.is_Composite:
        return K0.unify_composite(K1)
    
    # 步骤7：数值域按优先级统一
    if K1.is_ComplexField:
        K0, K1 = K1, K0
    if K0.is_ComplexField:
        # ... CC 统一逻辑 ...
    
    # ... 依次处理 RR, ALG, QQ_I, ZZ_I, QQ, ZZ ...
    
    # 步骤8：最终 fallback
    from sympy.polys.domains import EX
    return EX
```

### 4.2 符号约束统一（unify_with_symbols）

**代码位置**：`sympy/polys/domains/domain.py:850-854`

```python
def unify_with_symbols(K0, K1, symbols):
    if (K0.is_Composite and (set(K0.symbols) & set(symbols))) or \
       (K1.is_Composite and (set(K1.symbols) & set(symbols))):
        raise UnificationFailed(
            "Cannot unify %s with %s, given %s generators" % (K0, K1, tuple(symbols))
        )
    
    return K0.unify(K1)
```

**判断流程**：
```
unify_with_symbols(K0, K1, symbols)
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 检查 K0 是否是复合域且符号与约束符号重叠                     │
│ K0.is_Composite AND (set(K0.symbols) & set(symbols))      │
└────────────────────────────────────────────────────────────┘
    │
    ├── Yes ──▶ 抛出 UnificationFailed
    │
    └── No ──▶ 继续
              │
              ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 检查 K1 是否是复合域且符号与约束符号重叠                     │
    │ K1.is_Composite AND (set(K1.symbols) & set(symbols))      │
    └────────────────────────────────────────────────────────────┘
              │
              ├── Yes ──▶ 抛出 UnificationFailed
              │
              └── No ──▶ 调用 K0.unify(K1)
```

**示例**：
- `unify(ZZ[x], QQ[y], symbols=(z,))` → 无冲突 → 继续
- `unify(ZZ[x], QQ[y], symbols=(x,))` → K0 符号 `x` 与约束重叠 → **UnificationFailed**
- `unify(ZZ[x], QQ[y], symbols=(y,))` → K1 符号 `y` 与约束重叠 → **UnificationFailed**

### 4.3 特征检查与统一

**代码位置**：`sympy/polys/domains/domain.py:912-920`

```python
if not (K0.has_CharacteristicZero and K1.has_CharacteristicZero):
    # 两个域不都是特征零
    if K0.characteristic() != K1.characteristic():
        # 特征不同 → 无法统一
        raise UnificationFailed("Cannot unify %s with %s" % (K0, K1))
    
    # 特征相同但域不同（如 GF(3) 和 GF(3)[x]）
    return K0.unify_composite(K1)
```

**判断流程**：
```
特征检查
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ K0.has_CharacteristicZero AND K1.has_CharacteristicZero?   │
└────────────────────────────────────────────────────────────┘
    │
    ├── Yes（都是特征零）──▶ 进入后续统一逻辑
    │
    └── No（至少一个不是特征零）
              │
              ▼
    ┌────────────────────────────────────────┐
    │ K0.characteristic() == K1.characteristic()? │
    └────────────────────────────────────────┘
              │
              ├── No（特征不同）──▶ 抛出 UnificationFailed
              │
              └── Yes（特征相同）──▶ 调用 unify_composite()
```

**特征统一规则**：

| 域 A | 域 B | 特征比较 | 结果 |
|------|------|---------|------|
| GF(3) | GF(5) | 3 ≠ 5 | **UnificationFailed** |
| GF(3) | ZZ | 3 ≠ 0 | **UnificationFailed** |
| GF(3) | GF(3)[x] | 3 == 3 | 调用 `unify_composite` → GF(3)[x] |
| ZZ | QQ | 0 == 0 | 继续统一 → QQ |

**测试验证**：
```python
# domains/tests/test_domains.py:50-59
raises(UnificationFailed, lambda: unify(F3, ZZ))
raises(UnificationFailed, lambda: unify(F3, QQ))
raises(UnificationFailed, lambda: unify(F3, ZZ_I))
raises(UnificationFailed, lambda: unify(F3, QQ_I))
raises(UnificationFailed, lambda: unify(F3, ALG))
raises(UnificationFailed, lambda: unify(F3, RR))
raises(UnificationFailed, lambda: unify(F3, CC))
raises(UnificationFailed, lambda: unify(F3, ZZ[x]))
raises(UnificationFailed, lambda: unify(F3, ZZ.frac_field(x)))
raises(UnificationFailed, lambda: unify(F3, EX))
```

### 4.4 复合域统一（unify_composite）

**代码位置**：`sympy/polys/domains/domain.py:856-887`

```python
def unify_composite(K0, K1):
    """Unify two domains where at least one is composite."""
    
    # 提取基域
    K0_ground = K0.dom if K0.is_Composite else K0
    K1_ground = K1.dom if K1.is_Composite else K1
    
    # 提取符号
    K0_symbols = K0.symbols if K0.is_Composite else ()
    K1_symbols = K1.symbols if K1.is_Composite else ()
    
    # 统一基域
    domain = K0_ground.unify(K1_ground)
    
    # 合并符号
    symbols = _unify_gens(K0_symbols, K1_symbols)
    
    # 确定类型（多项式环 vs 分式域）
    # 特殊情况：ZZ[x].unify(QQ.frac_field(x)) -> ZZ.frac_field(x)
    if ((K0.is_FractionField and K1.is_PolynomialRing or
         K1.is_FractionField and K0.is_PolynomialRing) and
         (not K0_ground.is_Field or not K1_ground.is_Field) and domain.is_Field
         and domain.has_assoc_Ring):
        domain = domain.get_ring()
    
    # 确定类
    if K0.is_Composite and (not K1.is_Composite or K0.is_FractionField or K1.is_PolynomialRing):
        cls = K0.__class__
    else:
        cls = K1.__class__
    
    return cls(domain, symbols, order)
```

**判断流程**：
```
unify_composite(K0, K1)
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 1. 提取基域和符号                                           │
│    K0_ground = K0.dom (如果是复合域) 否则 K0              │
│    K1_ground = K1.dom (如果是复合域) 否则 K1              │
│    K0_symbols = K0.symbols (如果是复合域) 否则 ()         │
│    K1_symbols = K1.symbols (如果是复合域) 否则 ()         │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 2. 统一基域                                                  │
│    domain = K0_ground.unify(K1_ground)                     │
│    （可能递归调用 unify）                                    │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 3. 合并符号                                                  │
│    symbols = _unify_gens(K0_symbols, K1_symbols)          │
│    （去重，保持顺序）                                        │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 4. 特殊情况处理：环 vs 域                                    │
│    条件：                                                    │
│    - 一个是 FractionField，另一个是 PolynomialRing         │
│    - 至少一个基域不是 Field                                  │
│    - 统一后的基域是 Field                                    │
│    - 统一后的基域有关联 Ring                                 │
│    满足时：domain = domain.get_ring()                       │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 5. 确定结果类                                                │
│    如果 K0 是复合域且满足以下任一：                          │
│      - K1 不是复合域                                         │
│      - K0 是 FractionField                                   │
│      - K1 是 PolynomialRing                                  │
│    则使用 K0 的类，否则使用 K1 的类                          │
└────────────────────────────────────────────────────────────┘
    │
    ▼
返回 cls(domain, symbols, order)
```

**特殊情况示例**：
```python
# ZZ[x].unify(QQ.frac_field(x))
# 条件检查：
# - ZZ[x] 是 PolynomialRing，QQ.frac_field(x) 是 FractionField ✓
# - ZZ 不是 Field，QQ 是 Field（满足"至少一个不是"）✓
# - 统一后的基域是 QQ，是 Field ✓
# - QQ 有关联 Ring (ZZ) ✓
# 所以：domain = QQ.get_ring() = ZZ
# 结果：ZZ.frac_field(x)
```

### 4.5 数值域优先级统一

**代码位置**：`sympy/polys/domains/domain.py:953-1013`

```python
# 优先级顺序（从高到低）：
# ComplexField (CC) > RealField (RR) > AlgebraicField (ALG) > 
# GaussianField (QQ_I) > GaussianRing (ZZ_I) > 
# RationalField (QQ) > IntegerRing (ZZ)
```

**判断流程**：
```
数值域统一
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤1：ComplexField (CC) 处理                               │
│ - 如果任一域是 CC：                                          │
│   - 另一个是 CC 或 RR：取较高精度                           │
│   - 否则：返回 CC                                            │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤2：RealField (RR) 处理                                  │
│ - 如果任一域是 RR：                                          │
│   - 另一个是 RR：取较高精度                                  │
│   - 另一个是 ZZ_I/QQ_I：升级到 CC                           │
│   - 否则：返回 RR                                            │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤3：AlgebraicField (ALG) 处理                            │
│ - 如果任一域是 ALG：                                         │
│   - 另一个是 ZZ_I：升级到 QQ_I                               │
│   - 另一个是 QQ_I：转换为 ALG                                │
│   - 另一个是 ALG：合并扩域                                   │
│   - 否则：返回 ALG                                           │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤4：GaussianField/Ring (QQ_I/ZZ_I) 处理                 │
│ - QQ_I 优先级高于 ZZ_I                                       │
│ - ZZ_I + QQ → QQ_I                                          │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤5：RationalField (QQ) 处理                              │
│ - QQ 优先级高于 ZZ                                           │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤6：IntegerRing (ZZ) 处理                                │
│ - 最低优先级的精确域                                         │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤7：最终 fallback                                         │
│ - 如果以上都不匹配，返回 EX                                  │
└────────────────────────────────────────────────────────────┘
```

**数值域统一示例**：

| 域 A | 域 B | 统一结果 | 说明 |
|------|------|---------|------|
| ZZ | QQ | QQ | 有理数优先级更高 |
| ZZ | RR | RR | 实数优先级更高 |
| QQ | CC | CC | 复数优先级更高 |
| RR | ALG | RR | 实数在 ALG 前检查 |
| ZZ_I | QQ | QQ_I | 高斯域优先级更高 |
| ZZ_I | RR | CC | 高斯域 + 实数 → 复数 |
| QQ_I | ALG | ALG | 高斯域转换为代数域 |

### 4.6 unify 完整决策树

```
unify(K0, K1, symbols=None)
    │
    ├────────────── symbols is not None ──────────────▶
    │                    │
    │                    ▼
    │          ┌───────────────────────┐
    │          │ unify_with_symbols()  │
    │          │ 检查符号冲突           │
    │          └───────────────────────┘
    │                    │
    │         ┌──────────┴──────────┐
    │         │                     │
    │      冲突                  无冲突
    │         │                     │
    │         ▼                     ▼
    │  UnificationFailed      调用 unify(K0, K1)
    │
    └────────────── symbols is None ──────────────────▶
                         │
                         ▼
               ┌───────────────────────┐
               │ K0 == K1?              │
               └───────────────────────┘
                         │
              ┌──────────┴──────────┐
              │                     │
             Yes                    No
              │                     │
              ▼                     ▼
         返回 K0           ┌───────────────────────┐
                          │ 特征检查               │
                          └───────────────────────┘
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                    特征不同               特征相同
                         │                     │
                         ▼                     ▼
              UnificationFailed      unify_composite()
                                    （如 GF(3).unify(GF(3)[x])）
                                    │
                                    ▼
                         ┌───────────────────────┐
                         │ EX/EXRAW 检查          │
                         │ 任一域是 EXRAW/EX?     │
                         └───────────────────────┘
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                        Yes                    No
                         │                     │
                         ▼                     ▼
                   返回 EXRAW/EX      ┌───────────────────────┐
                                      │ 有限扩域检查            │
                                      │ 任一域是 FiniteExtension?│
                                      └───────────────────────┘
                                                │
                                     ┌──────────┴──────────┐
                                     │                     │
                                    Yes                    No
                                     │                     │
                                     ▼                     ▼
                              扩域统一逻辑        ┌───────────────────────┐
                                                │ 复合域检查              │
                                                │ 任一域是 Composite?     │
                                                └───────────────────────┘
                                                          │
                                               ┌──────────┴──────────┐
                                               │                     │
                                              Yes                    No
                                               │                     │
                                               ▼                     ▼
                                        unify_composite()   ┌────────────────────┐
                                                           │ 数值域优先级统一      │
                                                           │ CC > RR > ALG > ... │
                                                           └────────────────────┘
                                                                     │
                                                                     ▼
                                                            ┌────────────────────┐
                                                            │ 最终 fallback       │
                                                            │ 返回 EX             │
                                                            └────────────────────┘
```

### 4.7 unify 降级触发条件汇总

| 触发条件 | 触发位置 | 行为 | 回退路径 |
|---------|---------|------|---------|
| 符号约束冲突 | `unify_with_symbols` | 抛出 `UnificationFailed` | 调用方捕获处理 |
| 特征不同 | `unify` 主流程 | 抛出 `UnificationFailed` | 调用方捕获处理 |
| 数值域无法匹配 | `unify` 主流程 | 返回 EX | 无（EX 是最终 fallback） |

**UnificationFailed 调用方处理示例**：

```python
# polytools.py:4709-4718
if f.rep.dom != g.rep.dom:
    try:
        dom = f.rep.dom.unify(g.rep.dom, f.gens)
    except UnificationFailed:
        # 统一失败：返回 False（相等性比较）
        return False
    
    f = f.set_domain(dom)
    g = g.set_domain(dom)

return f.rep == g.rep
```

---

## 5. 第三阶段：convert/from_sympy 详细分析

### 5.1 CoercionFailed 触发场景总览

| 场景 | 触发位置 | 行为 | 回退路径 |
|------|---------|------|---------|
| 多项式与标量运算 | `rings.py:898-903` | 返回 `NotImplemented` | Python 尝试反向运算 |
| 有理函数元素构造 | `fields.py:219-233` | 环升级到域 | 升级失败则重新抛出 |
| 表达式重建 | `fields.py:299-305` | 环升级到域 | 升级失败则重新抛出 |

### 5.2 多项式运算中的降级（PolyElement）

**代码位置**：`sympy/polys/rings.py:898-903`（加法示例）

```python
def __add__(self, other: PolyElement[Er] | Er | int) -> PolyElement[Er]:
    # ... 类型检查 ...
    
    if domain.of_type(other):
        return self._add_ground(other)
    
    try:
        cp2 = ring.domain_new(other)
    except CoercionFailed:
        # 转换失败：返回 NotImplemented
        return NotImplemented
    else:
        return self._add_ground(cp2)
```

**判断流程**：
```
PolyElement.__add__(self, other)
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤1：类型检查                                              │
│ - other 是 PolyElement?                                     │
│ - other 是 domain 元素类型?                                 │
└────────────────────────────────────────────────────────────┘
    │
    ├── Yes ──▶ 直接调用相应方法（_add_ground 等）
    │
    └── No ──▶ 继续
              │
              ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤2：尝试转换                                             │
    │    try:                                                     │
    │        cp2 = ring.domain_new(other)                        │
    │    except CoercionFailed:                                  │
    │        return NotImplemented                                │
    └────────────────────────────────────────────────────────────┘
              │
              ├── 成功 ──▶ 调用 _add_ground(cp2)
              │
              └── 失败 ──▶ 返回 NotImplemented
                                 │
                                 ▼
                        Python 尝试反向运算
                        other.__radd__(self)
```

**类似的降级模式**（在 `rings.py` 中多处出现）：

| 运算 | 代码位置 | 降级行为 |
|------|---------|---------|
| `__add__` | `rings.py:898-903` | `CoercionFailed` → `NotImplemented` |
| `__sub__` | `rings.py:958-963` | `CoercionFailed` → `NotImplemented` |
| `__rsub__` | `rings.py:972-977` | `CoercionFailed` → `NotImplemented` |
| `__mul__` | `rings.py:1049-1054` | `CoercionFailed` → `NotImplemented` |
| `__divmod__` | `rings.py:1105-1110` | `CoercionFailed` → `NotImplemented` |
| `__rdivmod__` | `rings.py:1114-1119` | `CoercionFailed` → `NotImplemented` |

### 5.3 有理函数构造中的降级（FracElement）

**代码位置**：`sympy/polys/fields.py:219-233`

```python
def ground_new(self, element) -> FracElement[Er]:
    try:
        return self.new(self.ring.ground_new(element))
    except CoercionFailed:
        domain = self.domain
        
        if not domain.is_Field and domain.has_assoc_Field:
            # 环升级到域
            ring = self.ring
            ground_field: Field = domain.get_field()
            element = ground_field.convert(element)
            numer = ring.ground_new(ground_field.numer(element))
            denom = ring.ground_new(ground_field.denom(element))
            return self.raw_new(numer, denom)
        else:
            # 无法升级：重新抛出
            raise
```

**判断流程**：
```
FracField.ground_new(element)
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤1：尝试直接构造                                          │
│    try:                                                      │
│        return self.new(self.ring.ground_new(element))      │
│    except CoercionFailed:                                   │
│        # 进入降级逻辑                                        │
└────────────────────────────────────────────────────────────┘
    │
    ├── 成功 ──▶ 返回构造的 FracElement
    │
    └── 失败（CoercionFailed）──▶ 继续
                                    │
                                    ▼
                          ┌───────────────────────────────┐
                          │ 步骤2：检查是否可以升级         │
                          │    not domain.is_Field AND    │
                          │    domain.has_assoc_Field     │
                          └───────────────────────────────┘
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                        Yes                    No
                         │                     │
                         ▼                     ▼
                  ┌─────────────┐      ┌─────────────┐
                  │ 环升级到域   │      │ 重新抛出     │
                  │             │      │ CoercionFailed│
                  │ 1. 获取对应域│      └─────────────┘
                  │ 2. 域转换元素│
                  │ 3. 分离分子分母│
                  │ 4. 构造 FracElement│
                  └─────────────┘
```

**示例**：
```python
# ZZ(x) 是 ZZ 上的有理函数域
# ZZ 是 Ring，has_assoc_Field = True（对应 QQ）

# 尝试构造 ZZ(x).ground_new(1/2)
# 步骤1：ring.ground_new(1/2) = ZZ[x].ground_new(1/2)
#        ZZ 无法表示 1/2 → CoercionFailed
# 步骤2：检查 domain（ZZ）：not is_Field（True），has_assoc_Field（True）
# 步骤3：升级：
#         ground_field = QQ
#         element = QQ.convert(1/2) = QQ(1, 2)
#         numer = ZZ[x].ground_new(QQ.numer(QQ(1,2))) = ZZ[x].ground_new(1)
#         denom = ZZ[x].ground_new(QQ.denom(QQ(1,2))) = ZZ[x].ground_new(2)
# 结果：ZZ(x).raw_new(1, 2) = 1/2（在 ZZ(x) 中表示）
```

### 5.4 表达式重建中的降级

**代码位置**：`sympy/polys/fields.py:299-305`

```python
def _rebuild(expr):
    # ... 表达式重建逻辑 ...
    
    try:
        return domain.convert(expr)
    except CoercionFailed:
        if not domain.is_Field and domain.has_assoc_Field:
            # 环升级到域
            return domain.get_field().convert(expr)
        else:
            # 无法升级：重新抛出
            raise
```

**判断流程**：
```
_rebuild(expr) 中的降级
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤1：尝试直接转换                                          │
│    try:                                                      │
│        return domain.convert(expr)                          │
│    except CoercionFailed:                                   │
│        # 进入降级逻辑                                        │
└────────────────────────────────────────────────────────────┘
    │
    ├── 成功 ──▶ 返回转换后元素
    │
    └── 失败（CoercionFailed）──▶ 继续
                                    │
                                    ▼
                          ┌───────────────────────────────┐
                          │ 步骤2：检查是否可以升级         │
                          │    not domain.is_Field AND    │
                          │    domain.has_assoc_Field     │
                          └───────────────────────────────┘
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                        Yes                    No
                         │                     │
                         ▼                     ▼
                  ┌─────────────┐      ┌─────────────┐
                  │ 环升级到域   │      │ 重新抛出     │
                  │             │      │ CoercionFailed│
                  │ domain.     │      └─────────────┘
                  │ get_field() │
                  │ .convert()  │
                  └─────────────┘
```

### 5.5 convert 降级触发条件汇总

| 触发条件 | 触发位置 | 行为 | 回退路径 |
|---------|---------|------|---------|
| 多项式与无法转换的标量运算 | `rings.py` 多处 | 返回 `NotImplemented` | Python 尝试反向运算 |
| 有理函数环无法表示的元素 | `fields.py:219-233` | 环升级到域 | 升级失败则重新抛出 `CoercionFailed` |
| 表达式重建失败 | `fields.py:299-305` | 环升级到域 | 升级失败则重新抛出 `CoercionFailed` |

---

## 6. 多项式统一（_unify）详细分析

### 6.1 Poly._unify 主流程

**代码位置**：`sympy/polys/polytools.py:4723-4755`

```python
def _unify(f, g):
    g = sympify(g)
    
    if not g.is_Poly:
        # 情况1：g 不是多项式
        try:
            return f.rep.dom, f.per, f.rep, f.rep.per(f.rep.dom.from_sympy(g))
        except CoercionFailed:
            # 转换失败 → 抛出 UnificationFailed
            raise UnificationFailed("Cannot unify %s with %s" % (f, g))
    
    if len(f.gens) != len(g.gens):
        # 情况2：生成器数量不同
        raise UnificationFailed("Cannot unify %s with %s" % (f, g))
    
    if not (isinstance(f.rep, DMP) and isinstance(g.rep, DMP)):
        # 情况3：表示类型不兼容
        raise UnificationFailed("Cannot unify %s with %s" % (f, g))
    
    # 情况4：数域统一
    dom = f.rep.dom.unify(g.rep.dom, gens)
    
    # 转换到统一域
    F = f.rep.convert(dom)
    G = g.rep.convert(dom)
    
    return dom, per, F, G
```

### 6.2 Poly._unify 决策树

```
Poly._unify(f, g)
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 步骤1：g 不是多项式?                                         │
│    if not g.is_Poly:                                        │
└────────────────────────────────────────────────────────────┘
    │
    ├── Yes ──▶ 尝试 f.rep.dom.from_sympy(g)
    │              │
    │         ┌────┴────┐
    │         │         │
    │       成功      失败
    │         │         │
    │         ▼         ▼
    │    返回结果   UnificationFailed
    │
    └── No ──▶ 继续
              │
              ▼
    ┌────────────────────────────────────────────────────────────┐
    │ 步骤2：生成器数量不同?                                       │
    │    if len(f.gens) != len(g.gens):                          │
    └────────────────────────────────────────────────────────────┘
              │
              ├── Yes ──▶ UnificationFailed
              │
              └── No ──▶ 继续
                        │
                        ▼
              ┌────────────────────────────────────────────────────────────┐
              │ 步骤3：表示类型不兼容?                                       │
              │    if not (isinstance(f.rep, DMP) and isinstance(g.rep, DMP)):│
              └────────────────────────────────────────────────────────────┘
                        │
                        ├── Yes ──▶ UnificationFailed
                        │
                        └── No ──▶ 继续
                                  │
                                  ▼
                        ┌────────────────────────────────────────────────────────────┐
                        │ 步骤4：数域统一                                              │
                        │    dom = f.rep.dom.unify(g.rep.dom, gens)                  │
                        │    （可能抛出 UnificationFailed）                            │
                        └────────────────────────────────────────────────────────────┘
                                  │
                         ┌────────┴────────┐
                         │                 │
                      成功              失败
                         │                 │
                         ▼                 ▼
                    继续转换         UnificationFailed
                         │
                         ▼
              ┌────────────────────────────────────────────────────────────┐
              │ 步骤5：转换到统一域                                          │
              │    F = f.rep.convert(dom)                                   │
              │    G = g.rep.convert(dom)                                   │
              └────────────────────────────────────────────────────────────┘
                         │
                         ▼
                    返回 (dom, per, F, G)
```

### 6.3 Poly._unify 降级触发条件汇总

| 触发条件 | 触发位置 | 行为 |
|---------|---------|------|
| g 不是多项式且无法转换到 f 的域 | `polytools.py:4726-4730` | 抛出 `UnificationFailed` |
| 生成器数量不同 | `polytools.py:4732-4733` | 抛出 `UnificationFailed` |
| 表示类型不兼容（不是 DMP） | `polytools.py:4735-4736` | 抛出 `UnificationFailed` |
| 数域统一失败 | `polytools.py:4741` | 抛出 `UnificationFailed` |

---

## 7. 完整降级链路总结

### 7.1 三级降级机制

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 第一级：construct_domain() - 域选择阶段                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│ 触发条件：                                                                     │
│   1. 浮点数 + 代数数共存 → 返回 False → 直接降级到 EX                         │
│   2. 复合域类型系数 → 返回 None → 尝试复合域                                  │
│   3. 无生成器 → 返回 None → 降级到 EX                                         │
│   4. 生成器是代数数 → 返回 None → 降级到 EX                                    │
│   5. 符号重叠（可能有代数关系）→ 返回 None → 降级到 EX                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 第二级：unify() - 域统一阶段                                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│ 触发条件：                                                                     │
│   1. 符号约束冲突 → 抛出 UnificationFailed                                    │
│   2. 特征不同 → 抛出 UnificationFailed                                        │
│   3. 数值域无法匹配 → 最终 fallback 到 EX                                      │
│                                                                              │
│ 回退路径：                                                                     │
│   - 调用方捕获 UnificationFailed，自行处理（如返回 False、降级到 EX 等）       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 第三级：convert()/from_sympy() - 元素转换阶段                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│ 触发条件：                                                                     │
│   1. 多项式与无法转换的标量运算 → 返回 NotImplemented → Python 尝试反向运算     │
│   2. 环无法表示的元素 → 环升级到域（如有对应域）→ 升级失败则重新抛出            │
│   3. 表达式重建失败 → 环升级到域（如有对应域）→ 升级失败则重新抛出              │
│                                                                              │
│ 回退路径：                                                                     │
│   - 返回 NotImplemented：让 Python 尝试反向运算                                │
│   - 重新抛出 CoercionFailed：让上层处理                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 异常类型与处理策略

| 异常类型 | 触发场景 | 处理策略 | 典型位置 |
|---------|---------|---------|---------|
| `UnificationFailed` | 数域统一失败 | 调用方捕获，自行决定回退路径 | `domain.py`, `polytools.py` |
| `CoercionFailed` | 元素转换失败 | 1. 返回 `NotImplemented`<br>2. 环升级到域<br>3. 重新抛出 | `rings.py`, `fields.py` |
| `ValueError` | 表达式重建完全失败 | 终止执行，报错 | `fields.py`, `rings.py` |

### 7.3 关键代码位置速查

| 功能 | 文件位置 | 关键代码行 |
|------|---------|-----------|
| `construct_domain` 主流程 | `constructor.py` | 268-388 |
| `_construct_simple` 浮点数+代数数 | `constructor.py` | 30-35, 52-56 |
| `_construct_composite` 符号重叠 | `constructor.py` | 150-158 |
| `unify` 主流程 | `domain.py` | 889-1016 |
| `unify_with_symbols` 符号冲突 | `domain.py` | 850-854 |
| `unify` 特征检查 | `domain.py` | 912-920 |
| `unify_composite` | `domain.py` | 856-887 |
| `PolyElement.__add__` 降级 | `rings.py` | 898-903 |
| `FracField.ground_new` 环升级 | `fields.py` | 219-233 |
| `Poly._unify` | `polytools.py` | 4723-4755 |
| `UnificationFailed` 调用方处理 | `polytools.py` | 4709-4718 |

---

## 8. 典型降级场景示例

### 8.1 场景1：浮点数与代数数共存

```python
from sympy import construct_domain, sqrt, S

# 浮点数 + 代数数
exprs = [sqrt(2), S(3.14)]
domain, elements = construct_domain(exprs)
# 结果：domain = EX
# 触发路径：_construct_simple → 检测到 floats 和 algebraics 同时为 True → 返回 False → _construct_expression
```

### 8.2 场景2：符号重叠（可能有代数关系）

```python
from sympy import construct_domain
from sympy.abc import x, y

# 生成器符号有重叠：x 和 x**2 都包含符号 x
# 这可能意味着存在代数关系（如 x**2 是 x 的平方）
exprs = [x, x**2]
domain, elements = construct_domain(exprs)
# 结果：domain = EX
# 触发路径：_construct_simple → 返回 None → _construct_composite → 
#          检测到 all_symbols & symbols 非空 → 返回 None → _construct_expression
```

### 8.3 场景3：特征不同无法统一

```python
from sympy.polys.domains import ZZ, GF
from sympy.polys.domains.domain import unify

F3 = GF(3)
try:
    result = unify(F3, ZZ)
except UnificationFailed as e:
    print(e)  # "Cannot unify GF(3) with ZZ"
# 触发路径：unify → 特征检查 → 3 ≠ 0 → 抛出 UnificationFailed
```

### 8.4 场景4：多项式与无法转换的标量运算

```python
from sympy import Poly, symbols
from sympy.abc import x

p = Poly(x**2 + 1, x)  # 域：ZZ

# 尝试与字符串相加（无法转换）
try:
    result = p + "abc"
except TypeError as e:
    print(e)  # unsupported operand type(s) for +: 'Poly' and 'str'
# 触发路径：p.__add__("abc") → ring.domain_new("abc") → CoercionFailed → 
#          返回 NotImplemented → Python 尝试 "abc".__radd__(p) → 也返回 NotImplemented →
#          最终抛出 TypeError
```

### 8.5 场景5：环升级到域

```python
from sympy.polys.fields import FracField
from sympy.polys.domains import ZZ
from sympy.abc import x

# 创建 ZZ(x)：ZZ 上的有理函数域
# ZZ 是 Ring，has_assoc_Field = True（对应 QQ）
ZZ_x = FracField((x,), ZZ)

# 尝试构造 1/2（ZZ 无法表示，但 QQ 可以）
frac = ZZ_x.ground_new(ZZ(1) / ZZ(2))
# 结果：1/2（在 ZZ(x) 中成功表示）
# 触发路径：ground_new → ring.ground_new(1/2) → CoercionFailed → 
#          检查 domain.is_Field（False）和 has_assoc_Field（True）→
#          升级到 QQ → QQ.convert(1/2) 成功 → 分离分子分母 → 构造成功
```

### 8.6 场景6：多项式统一失败的调用方处理

```python
from sympy import Poly, symbols
from sympy.abc import x, y

# 两个多项式，生成器数量不同
p1 = Poly(x**2 + 1, x)      # gens = (x,)
p2 = Poly(x + y, x, y)      # gens = (x, y)

# 相等性比较（内部调用 _unify）
result = p1 == p2
# 结果：False（不是抛出异常！）
# 触发路径：__eq__ → _eq_poly_poly → 尝试 unify → UnificationFailed →
#          调用方捕获 → 返回 False
```

---

## 9. 设计原则总结

### 9.1 降级设计原则

1. **最小覆盖优先**：
   - 总是尝试选择能精确表示所有元素的最小域
   - 降级是最后的手段

2. **层次化降级**：
   - 第一级：域选择阶段（`construct_domain`）
   - 第二级：域统一阶段（`unify`）
   - 第三级：元素转换阶段（`convert`/`from_sympy`）

3. **安全降级**：
   - 不丢失信息的前提下降级
   - EX 域作为最终 fallback，可以表示任何表达式

4. **调用方负责**：
   - `UnificationFailed` 和 `CoercionFailed` 由调用方决定如何处理
   - 提供灵活的回退机制

### 9.2 异常设计原则

| 异常类型 | 设计意图 | 典型处理方式 |
|---------|---------|-------------|
| `UnificationFailed` | 表示两个域在语义上无法统一 | 调用方捕获，根据上下文决定：<br>- 返回 `False`（相等性比较）<br>- 降级到 EX<br>- 报错终止 |
| `CoercionFailed` | 表示元素无法转换到目标域 | 1. 返回 `NotImplemented`<br>2. 尝试升级域<br>3. 重新抛出 |
| `ValueError` | 表示完全无法处理 | 终止执行，报错 |

### 9.3 回退策略总结

| 层级 | 回退策略 | 最终目标 |
|------|---------|---------|
| 域选择 | 简单域 → 复合域 → EX | 找到能表示所有元素的域 |
| 域统一 | 按优先级统一 → EX | 找到最小公共域 |
| 元素转换 | 直接转换 → 环升级 → 反向运算 → 报错 | 成功转换或明确失败 |
