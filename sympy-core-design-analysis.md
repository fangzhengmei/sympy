# SymPy Basic 子类核心设计机制分析

## 目录

1. [引言](#引言)
2. [`__new__` 规范化机制](#__new__-规范化机制)
   - [2.1 Basic 基类的 `__new__`](#21-basic-基类的-__new__)
   - [2.2 AssocOp 的规范化流程](#22-assocop-的规范化流程)
   - [2.3 Add.flatten - 加法规范化](#23-addflatten---加法规范化)
   - [2.4 Mul.flatten - 乘法规范化](#24-mulflatten---乘法规范化)
   - [2.5 Pow.__new__ - 幂运算规范化](#25-pow__new__---幂运算规范化)
   - [2.6 排序机制与 canonical order](#26-排序机制与-canonical-order)
3. [`args` 树的不可变性与哈希设计](#args-树的不可变性与哈希设计)
   - [3.1 `args` 的不可变性实现](#31-args-的不可变性实现)
   - [3.2 `__hash__` 方法与 `_mhash` 缓存](#32-__hash__-方法与-_mhash-缓存)
   - [3.3 `_hashable_content` 的设计](#33-_hashable_content-的设计)
   - [3.4 `__eq__` 与结构相等性](#34-__eq__-与结构相等性)
4. [`evaluate=False` 的传递机制与规范化旁路逻辑](#evaluatefalse-的传递机制与规范化旁路逻辑)
   - [4.1 global_parameters 与 evaluate 开关](#41-global_parameters-与-evaluate-开关)
   - [4.2 AssocOp 中的 evaluate 处理](#42-assocop-中的-evaluate-处理)
   - [4.3 上下文管理器 `evaluate(False)`](#43-上下文管理器-evaluatefalse)
   - [4.4 `_from_args` 与 `_new_rawargs` - 旁路构建](#44-_from_args-与-_new_rawargs---旁路构建)
5. [大表达式树的哈希缓存策略](#大表达式树的哈希缓存策略)
   - [5.1 `_mhash` 实例级缓存](#51-_mhash-实例级缓存)
   - [5.2 `cacheit` 函数级缓存](#52-cacheit-函数级缓存)
   - [5.3 缓存一致性与失效机制](#53-缓存一致性与失效机制)
   - [5.4 性能优化考量](#54-性能优化考量)
6. [代码引用索引](#代码引用索引)

---

## 引言

SymPy 作为一个符号计算系统，其核心设计面临一个关键挑战：**如何在保持数学正确性的同时，高效地处理大量符号表达式对象**。

核心问题包括：
1. **表达式规范化**：数学上等价的表达式（如 `x + y` 和 `y + x`）如何被视为相同对象？
2. **对象去重**：如何避免创建大量语义相同的不同对象实例？
3. **哈希与相等性**：如何确保相等的对象具有相同的哈希值？
4. **缓存策略**：如何在大表达式树中高效缓存计算结果？

本报告深入分析 SymPy 核心模块中 `Basic` 子类的设计，揭示其解决上述问题的精妙机制。

---

## `__new__` 规范化机制

### 2.1 Basic 基类的 `__new__`

`Basic` 是所有 SymPy 对象的基类，其 `__new__` 方法实现了最基础的对象创建逻辑：

```python
# sympy/core/basic.py:294-300
def __new__(cls, *args):
    obj = object.__new__(cls)
    obj._assumptions = cls.default_assumptions
    obj._mhash = None  # will be set by __hash__ method.

    obj._args = args  # all items in args must be Basic objects
    return obj
```

**关键设计要点**：

1. **直接使用 `object.__new__`**：不调用父类的 `__init__`，完全控制对象创建流程
2. **`_mhash = None`**：哈希值延迟计算，首次访问 `__hash__` 时才计算
3. **`_args = args`**：参数直接存储为元组，保证不可变性
4. **无规范化逻辑**：`Basic` 本身不做任何规范化，留给子类实现

### 2.2 AssocOp 的规范化流程

`AssocOp`（Associative Operation）是 `Add`、`Mul` 等结合操作的基类，其 `__new__` 方法实现了核心的规范化逻辑：

```python
# sympy/core/operations.py:62-116
@cacheit
def __new__(cls, *args, evaluate=None, _sympify=True):
    # Allow faster processing by passing ``_sympify=False``, if all arguments
    # are already sympified.
    if _sympify:
        args = list(map(_sympify_, args))

    # ... 类型检查和弃用警告 ...

    if evaluate is None:
        evaluate = global_parameters.evaluate
    
    # 规范化旁路：evaluate=False 时直接创建对象
    if not evaluate:
        obj = cls._from_args(args)
        obj = cls._exec_constructor_postprocessors(obj)
        return obj

    # 规范化流程开始
    args = [a for a in args if a is not cls.identity]  # 移除单位元

    if len(args) == 0:
        return cls.identity
    if len(args) == 1:
        return args[0]

    # 核心规范化：flatten 方法
    c_part, nc_part, order_symbols = cls.flatten(args)
    is_commutative = not nc_part
    obj = cls._from_args(c_part + nc_part, is_commutative)
    obj = cls._exec_constructor_postprocessors(obj)

    if order_symbols is not None:
        from sympy.series.order import Order
        return Order(obj, *order_symbols)
    return obj
```

**规范化流程分解**：

| 步骤 | 操作 | 示例 |
|------|------|------|
| 1 | `_sympify_` 参数转换 | `1` → `Integer(1)`, `x` (str) → `Symbol('x')` |
| 2 | `evaluate` 检查 | 若 `False`，跳过规范化直接返回 |
| 3 | 移除单位元 | `Add(x, 0, y)` → `[x, y]` |
| 4 | 空参数处理 | `Add()` → `S.Zero` |
| 5 | 单参数处理 | `Add(x)` → `x` |
| 6 | `flatten()` 深度规范化 | 合并同类项、排序等 |
| 7 | 构造最终对象 | `_from_args()` 快速创建 |

### 2.3 Add.flatten - 加法规范化

`Add.flatten` 实现了加法表达式的深度规范化：

```python
# sympy/core/add.py:198-404
@classmethod
def flatten(cls, seq: list[Expr]) -> tuple[list[Expr], list[Expr], None]:
    """
    Takes the sequence "seq" of nested Adds and returns a flatten list.
    Returns: (commutative_part, noncommutative_part, order_symbols)
    """
    
    # 规范化步骤：
    
    # 1. 扁平化嵌套结构
    # Add(x, Add(y, z)) → seq.extend([y, z])
    for o in seq:
        if o.is_Add:
            seq.extend(o.args)  # 递归展开
            continue
        
        # 2. 分离数字系数
        elif o.is_Number:
            coeff += o
            continue
            
        # 3. 提取项的系数部分
        elif o.is_Mul:
            c, s = o.as_coeff_Mul()  # 2*x → (2, x)
            
        # 4. 合并同类项（按符号部分分组）
        if s in terms:
            terms[s] += c  # x + 2*x → terms[x] = 3
        else:
            terms[s] = c
    
    # 5. 处理合并结果
    newseq = []
    for s, c in terms.items():
        if c.is_zero:
            continue  # 移除系数为 0 的项
        elif c is S.One:
            newseq.append(s)  # 1*x → x
        else:
            newseq.append(Mul(c, s))  # 3*x → Mul(3, x)
    
    # 6. 规范排序
    _addsort(newseq)  # 按 canonical order 排序
    
    # 7. 数字系数前置
    if coeff is not S.Zero:
        newseq.insert(0, coeff)
    
    return newseq, [], None
```

**关键规范化行为**：

| 输入 | 规范化后 | 说明 |
|------|----------|------|
| `Add(x, Add(y, z))` | `Add(x, y, z)` | 扁平化嵌套 |
| `Add(x, 0, y)` | `Add(x, y)` | 移除单位元 |
| `Add(x, x)` | `Mul(2, x)` | 合并同类项 |
| `Add(y, x)` | `Add(x, y)` | 规范排序 |
| `Add(x, -x)` | `S.Zero` | 抵消为 0 |

### 2.4 Mul.flatten - 乘法规范化

`Mul.flatten` 的规范化逻辑与 `Add` 类似，但针对乘法特性进行了优化：

```python
# sympy/core/mul.py:210-739
@classmethod
def flatten(cls, seq):
    """Return commutative, noncommutative and order arguments by
    combining related terms."""
    
    # 1. 扁平化嵌套 Mul
    for o in seq:
        if o.is_Mul:
            if o.is_commutative:
                seq.extend(o.args)
            # ... 非交换部分处理 ...
    
    # 2. 分离数字系数
    elif o.is_Number:
        coeff *= o
        continue
    
    # 3. 按底部分组合并指数
    # x * x**2 → x**3
    c_powers = []  # (base, exp) 列表
    for o in seq:
        b, e = o.as_base_exp()
        c_powers.append((b, e))
    
    # 合并同底数
    def _gather(c_powers):
        common_b = {}  # base: exp
        for b, e in c_powers:
            co = e.as_coeff_Mul()
            common_b.setdefault(b, {}).setdefault(
                co[1], []).append(co[0])
        # ... 合并指数 ...
        return new_c_powers
    
    # 4. 处理特殊幂次
    # x**0 → 1, x**1 → x
    for b, e in c_powers:
        if e.is_zero:
            continue  # 移除 x**0
        if e is S.One:
            if b.is_Number:
                coeff *= b
                continue
            p = b
        else:
            p = Pow(b, e)
        c_part.append(p)
    
    # 5. 处理负号和虚数单位
    # (-2)**(1/2) → I*sqrt(2)
    neg1e = S.Zero  # -1 的指数累计
    for o in seq:
        if o is S.ImaginaryUnit:
            neg1e += S.Half  # I = (-1)**(1/2)
            continue
        # ... 其他处理 ...
    
    # 6. 规范排序
    _mulsort(c_part)
    
    # 7. 系数前置
    if coeff is not S.One:
        c_part.insert(0, coeff)
    
    return c_part, nc_part, order_symbols
```

**关键规范化行为**：

| 输入 | 规范化后 | 说明 |
|------|----------|------|
| `Mul(x, Mul(y, z))` | `Mul(x, y, z)` | 扁平化嵌套 |
| `Mul(x, 1, y)` | `Mul(x, y)` | 移除单位元 |
| `Mul(x, x)` | `Pow(x, 2)` | 合并同底数 |
| `Mul(y, x)` | `Mul(x, y)` | 规范排序 |
| `Mul(x, 1/x)` | `S.One` | 抵消为 1 |
| `Mul(-1, -1)` | `S.One` | 负号抵消 |

### 2.5 Pow.__new__ - 幂运算规范化

`Pow` 的规范化处理特殊的幂运算规则：

```python
# sympy/core/power.py:137-233
@cacheit
def __new__(cls, b: Expr | complex, e: Expr | complex, evaluate=None) -> Expr:
    if evaluate is None:
        evaluate = global_parameters.evaluate

    base = _sympify(b)
    exp = _sympify(e)

    if evaluate:
        # 1. 特殊值处理
        if exp is S.ComplexInfinity:
            return S.NaN
        if exp is S.Zero:
            return S.One    # x**0 = 1
        elif exp is S.One:
            return base    # x**1 = x
        
        # 2. 底数为 1 的情况
        elif base is S.One:
            if abs(exp).is_infinite:
                return S.NaN  # 1**∞ = NaN (不定式)
            return S.One
        
        # 3. 底数为 0 的情况
        elif exp == -1 and not base:
            return S.ComplexInfinity  # 0**(-1) = ∞
        
        # 4. 符号提取
        # (-x)**even → x**even, (-x)**odd → -x**odd
        elif (exp.is_Symbol and exp.is_integer or exp.is_Integer
                ) and (base.is_number and base.is_Mul or base.is_Number
                ) and base.could_extract_minus_sign():
            if exp.is_even:
                base = -base
            elif exp.is_odd:
                return -Pow(-base, exp)
        
        # 5. 委托给 base 的 _eval_power
        obj = base._eval_power(exp)
        if obj is not None:
            return obj
    
    # 6. 创建未规范化的 Pow 对象
    obj = Expr.__new__(cls, base, exp)
    obj = cls._exec_constructor_postprocessors(obj)
    if not isinstance(obj, Pow):
        return obj
    obj.is_commutative = (base.is_commutative and exp.is_commutative)
    return obj
```

**关键规范化行为**：

| 输入 | 规范化后 | 说明 |
|------|----------|------|
| `x**0` | `1` | 任何数的 0 次幂 |
| `x**1` | `x` | 任何数的 1 次幂 |
| `1**∞` | `NaN` | 不定式 |
| `(-2)**2` | `4` | 负数偶次幂 |
| `(-2)**3` | `-8` | 负数奇次幂 |
| `0**-1` | `zoo` | 复无穷 |

### 2.6 排序机制与 canonical order

规范化的关键是将表达式转换为**规范形式**（canonical form），使得数学上等价的表达式具有相同的内部表示。SymPy 使用多层排序策略：

#### 2.6.1 `ordering_of_classes` - 类级别排序

```python
# sympy/core/basic.py:58-94
ordering_of_classes = [
    # singleton numbers
    'Zero', 'One', 'Half', 'Infinity', 'NaN', 'NegativeOne', 'NegativeInfinity',
    # numbers
    'Integer', 'Rational', 'Float',
    # singleton symbols
    'Exp1', 'Pi', 'ImaginaryUnit',
    # symbols
    'Symbol', 'Wild',
    # arithmetic operations
    'Pow', 'Mul', 'Add',
    # ... 更多类 ...
]
```

这个列表定义了不同类的优先顺序，确保 `Add(x, 1)` 规范化为 `Add(1, x)`。

#### 2.6.2 `_cmp_name` - 类名比较函数

```python
# sympy/core/basic.py:96-139
def _cmp_name(x: type, y: type) -> int:
    """Return -1, 0, 1 if the name of x is before that of y."""
    n1 = x.__name__
    n2 = y.__name__
    if n1 == n2:
        return 0
    
    # 优先使用 ordering_of_classes 中的顺序
    UNKNOWN = len(ordering_of_classes) + 1
    try:
        i1 = ordering_of_classes.index(n1)
    except ValueError:
        i1 = UNKNOWN
    try:
        i2 = ordering_of_classes.index(n2)
    except ValueError:
        i2 = UNKNOWN
    
    if i1 == UNKNOWN and i2 == UNKNOWN:
        return (n1 > n2) - (n1 < n2)  # 回退到字符串比较
    return (i1 > i2) - (i1 < i2)
```

#### 2.6.3 `Basic.compare` - 完整比较逻辑

```python
# sympy/core/basic.py:372-428
def compare(self, other):
    """
    Return -1, 0, 1 if the object is less than, equal,
    or greater than other in a canonical sense.
    """
    if self is other:
        return 0
    
    # 1. 比较类名
    n1 = self.__class__
    n2 = other.__class__
    c = _cmp_name(n1, n2)
    if c:
        return c
    
    # 2. 比较 hashable_content
    st = self._hashable_content()
    ot = other._hashable_content()
    
    # 3. 比较长度
    len_st = len(st)
    len_ot = len(ot)
    c = (len_st > len_ot) - (len_st < len_ot)
    if c:
        return c
    
    # 4. 递归比较每个元素
    for l, r in zip(st, ot):
        if isinstance(l, Basic):
            c = l.compare(r)
        elif isinstance(l, frozenset):
            # ... frozenset 比较 ...
        else:
            c = (l > r) - (l < r)
        if c:
            return c
    return 0
```

#### 2.6.4 `default_sort_key` - 排序键生成

```python
# sympy/core/sorting.py:11-167
def default_sort_key(item, order=None):
    """Return a key that can be used for sorting.
    
    The key has the structure:
    (class_key, (len(args), args), exponent.sort_key(), coefficient)
    """
    from .basic import Basic
    from .singleton import S

    if isinstance(item, Basic):
        return item.sort_key(order=order)
    
    # ... 非 Basic 对象处理 ...
    
    return (cls_index, 0, item.__class__.__name__
            ), args, S.One.sort_key(), S.One
```

**排序键结构**：

```python
# 对于表达式 2*x**2 + 3*y
# Add 的 sort_key 结构为：
(
    (3, 1, 'Add'),           # class_key: (优先级, 子类型, 类名)
    (2, (                     # (参数数量, 参数的 sort_key 元组)
        ((1, 0, 'Number'), (0, ()), (), 2),  # 2 的 sort_key
        (
            (3, 0, 'Mul'),                    # 3*y 的 sort_key
            (2, (
                ((1, 0, 'Number'), (0, ()), (), 3),
                ((5, 0, 'Symbol'), (1, ('y',)), ((1, 0, 'Number'), (0, ()), (), 1), 1)
            )),
            (),
            1
        )
    )),
    (),  # exponent.sort_key()
    1    # coefficient
)
```

---

## `args` 树的不可变性与哈希设计

### 3.1 `args` 的不可变性实现

SymPy 表达式树的核心设计原则是**不可变性**（immutability）。这意味着一旦创建，对象的内部状态就不能被修改。

#### 3.1.1 `_args` 使用元组存储

```python
# sympy/core/basic.py:208-214
__slots__ = ('_mhash',              # hash value
             '_args',               # arguments
             '_assumptions'
            )

_args: tuple[Basic, ...]
```

**关键设计**：

1. **`__slots__` 声明**：固定实例属性，防止动态添加
2. **`_args` 类型注解**：明确为 `tuple[Basic, ...]`
3. **`__new__` 直接赋值**：

```python
# sympy/core/basic.py:299
obj._args = args  # all items in args must be Basic objects
```

由于 `args` 是在 `__new__` 中直接赋值的元组，无法在后续修改。

#### 3.1.2 `args` 属性只读

```python
# sympy/core/basic.py:912-942
@property
def args(self) -> tuple[Basic, ...]:
    """Returns a tuple of arguments of 'self'.
    
    Notes
    =====
    Never use self._args, always use self.args.
    Only use _args in __new__ when creating a new function.
    Do not override .args() from Basic (so that it is easy to
    change the interface in the future if needed).
    """
    return self._args
```

**设计要点**：

1. **`@property` 装饰器**：使 `args` 成为只读属性
2. **文档强调**：明确禁止直接使用 `_args`（除了 `__new__`）
3. **禁止重写**：建议子类不要重写 `.args()`

#### 3.1.3 修改必须创建新对象

由于不可变性，任何对表达式的"修改"都必须创建新对象：

```python
# 示例：x + y 的 args 是 (x, y)
# 要"修改"它，必须创建新的 Add

# 错误做法（实际上无法做到）：
# add_expr._args = (new_x, new_y)  # 非法操作

# 正确做法：
from sympy import symbols, Add
x, y, z = symbols('x y z')
expr = Add(x, y)

# 创建新表达式，而不是修改原表达式
new_expr = Add(z, *expr.args)  # z + x + y
```

### 3.2 `__hash__` 方法与 `_mhash` 缓存

哈希值是 SymPy 对象的核心属性，用于：
1. **集合成员检测**：`{x, y, x}` → `{x, y}`
2. **字典键**：`{x: 1, y: 2}`
3. **缓存查找**：`cacheit` 装饰器使用哈希作为键

#### 3.2.1 `Basic.__hash__` 实现

```python
# sympy/core/basic.py:321-328
def __hash__(self) -> int:
    # hash cannot be cached using cache_it because infinite recurrence
    # occurs as hash is needed for setting cache dictionary keys
    h = self._mhash
    if h is None:
        h = hash((type(self).__name__,) + self._hashable_content())
        self._mhash = h
    return h
```

**关键设计**：

1. **延迟计算**：`_mhash` 初始为 `None`，首次调用 `__hash__` 时才计算
2. **缓存机制**：计算后存入 `_mhash`，后续直接返回
3. **哈希组成**：`hash((类名,) + _hashable_content())`

#### 3.2.2 为什么不使用 `cacheit`？

注释中提到：
> "hash cannot be cached using cache_it because infinite recurrence occurs as hash is needed for setting cache dictionary keys"

**原因分析**：

```python
# 假设使用 @cacheit 装饰 __hash__：
@cacheit
def __hash__(self):
    return hash(...)

# cacheit 内部使用 lru_cache，需要将 self 作为字典键
# 但字典键需要计算哈希值 → 调用 __hash__ → 无限递归！
```

因此，SymPy 采用**实例级缓存**（`_mhash`）而不是**函数级缓存**（`cacheit`）。

### 3.3 `_hashable_content` 的设计

`_hashable_content` 返回用于计算哈希的核心内容：

```python
# sympy/core/basic.py:330-338
def _hashable_content(self) -> tuple[Hashable, ...]:
    """Return a tuple of information about self that can be used to
    compute the hash. If a class defines additional attributes,
    like ``name`` in Symbol, then this method should be updated
    accordingly to return such relevant attributes.

    Defining more than _hashable_content is necessary if __eq__ has
    been defined by a class. See note about this in Basic.__eq__."""
    return self._args
```

#### 3.3.1 子类重写示例：`Symbol`

`Symbol` 的 `_hashable_content` 直接使用已预处理好的字段，无需在调用时重新排序：

```python
# sympy/core/symbol.py:386-390
# 在 __xnew__ 中预处理 assumptions0
assumptions_kb, assumptions_orig, assumptions0 = Symbol._canonical_assumptions(**assumptions)
# ...
obj._assumptions0 = tuple(sorted(assumptions0.items()))  # 预处理为已排序的元组

# sympy/core/symbol.py:428-429
def _hashable_content(self):
    # 直接使用已预处理好的 _assumptions0，无需再次排序
    return (self.name,) + self._assumptions0
```

**关键点**：
- `_assumptions0` 在对象创建时就已预处理为 `tuple(sorted(assumptions0.items()))`
- `_hashable_content` 直接拼接 `self._assumptions0`，无需额外排序操作
- 这避免了每次调用 `_hashable_content` 时的重复排序开销

#### 3.3.2 哈希与相等性的一致性

Python 要求：
> **如果两个对象相等（`a == b` 为 `True`），则它们的哈希值必须相等（`hash(a) == hash(b)`）**

SymPy 的设计保证了这一点：

```python
# 假设 a 和 b 相等
# a.__eq__(b) 比较 _hashable_content()
# a.__hash__() 也基于 _hashable_content()
# 因此相等的对象必然有相同的哈希值
```

### 3.4 `__eq__` 与结构相等性

SymPy 的相等性是**结构相等性**（structural equality），而非引用相等性：

```python
# sympy/core/basic.py:503-543
def __eq__(self, other):
    """Return a boolean indicating whether a == b on the basis of
    their symbolic trees.

    This is the same as a.compare(b) == 0 but faster.
    """
    if self is other:
        return True  # 快速路径：同一对象

    if not isinstance(other, Basic):
        return self._do_eq_sympify(other)  # 非 Basic 对象的特殊处理

    # 检查类型
    if not (self.is_Number and other.is_Number) and (
            type(self) != type(other)):
        return False  # 类型不同，直接不等
    
    # 比较 _hashable_content
    a, b = self._hashable_content(), other._hashable_content()
    if a != b:
        return False
    
    # 额外检查：表达式中的数字类型
    for a, b in zip(a, b):
        if not isinstance(a, Basic):
            continue
        if a.is_Number and type(a) != type(b):
            return False
    return True
```

#### 3.4.1 结构相等性示例

```python
from sympy import symbols, Add, Mul

x, y = symbols('x y')

# 创建两个语义相同的 Add 对象
a1 = Add(x, y)
a2 = Add(y, x)  # 规范化后顺序相同

print(a1 == a2)  # True（结构相等）
print(a1 is a2)  # False（不同对象实例，除非缓存命中）

# 哈希一致性
print(hash(a1) == hash(a2))  # True
```

#### 3.4.2 快速路径优化

`__eq__` 实现了多层快速路径：

1. **引用相等检查**：`if self is other: return True`
2. **类型检查**：`type(self) != type(other)` 时快速返回 `False`
3. **`_hashable_content` 比较**：元组比较效率高

---

## `evaluate=False` 的传递机制与规范化旁路逻辑

### 4.1 global_parameters 与 evaluate 开关

SymPy 使用**线程本地**的全局参数控制系统行为：

```python
# sympy/core/parameters.py:1-162
"""Thread-safe global parameters"""

class _global_parameters(local):
    """Thread-local global parameters."""
    
    def __init__(self, **kwargs):
        self.__dict__.update(kwargs)
    
    def __setattr__(self, name, value):
        if getattr(self, name) != value:
            clear_cache()  # 更改参数时清除缓存
        return super().__setattr__(name, value)

# 默认全局参数
global_parameters = _global_parameters(
    evaluate=True,      # 默认启用规范化
    distribute=True,     # 默认启用分配律
    exp_is_pow=False     # exp(x) 不作为 Pow(E, x)
)
```

#### 4.1.1 线程本地存储

继承自 `threading.local`，确保多线程环境下参数隔离：

```python
# 示例：多线程安全
from sympy.core.parameters import global_parameters
from sympy.abc import x
import threading

log = []

def f():
    # 此线程内的修改不影响其他线程
    global_parameters.evaluate = False
    log.append(x + x)  # x + x（未规范化）

thread = threading.Thread(target=f)
thread.start()
thread.join()

print(log)          # [x + x]
print(x + x)        # 2*x（主线程仍为默认值）
```

### 4.2 AssocOp 中的 evaluate 处理

`AssocOp.__new__` 是 `evaluate` 参数的核心处理位置：

```python
# sympy/core/operations.py:62-116
@cacheit
def __new__(cls, *args, evaluate=None, _sympify=True):
    # ... 参数处理 ...
    
    # 1. evaluate 默认值机制
    if evaluate is None:
        evaluate = global_parameters.evaluate
    
    # 2. 规范化旁路：evaluate=False 时
    if not evaluate:
        obj = cls._from_args(args)  # 直接创建，不调用 flatten
        obj = cls._exec_constructor_postprocessors(obj)
        return obj
    
    # 3. evaluate=True 时：完整规范化流程
    args = [a for a in args if a is not cls.identity]
    
    if len(args) == 0:
        return cls.identity
    if len(args) == 1:
        return args[0]
    
    # 调用 flatten 进行深度规范化
    c_part, nc_part, order_symbols = cls.flatten(args)
    # ...
```

#### 4.2.1 `evaluate` 参数传递链

```python
# 调用链示例
from sympy import Add, symbols
x = symbols('x')

# 1. 显式传递 evaluate=False
Add(x, x, evaluate=False)  # x + x（未规范化）

# 2. 使用 global_parameters
from sympy.core.parameters import global_parameters
global_parameters.evaluate = False
Add(x, x)  # x + x（使用全局默认值）

# 3. 嵌套调用时的传递
# 注意：内部调用可能不传递 evaluate 参数
# 需要使用上下文管理器确保一致行为
```

### 4.3 上下文管理器 `evaluate(False)`

SymPy 提供了上下文管理器来临时控制 `evaluate` 状态：

```python
# sympy/core/parameters.py:71-104
class evaluate:
    """Control automatic evaluation
    
    Examples
    ========
    >>> from sympy import evaluate
    >>> from sympy.abc import x
    >>> print(x + x)
    2*x
    >>> with evaluate(False):
    ...     print(x + x)
    x + x
    """
    
    def __init__(self, x):
        self.x = x
        self.old = []
    
    def __enter__(self):
        self.old.append(global_parameters.evaluate)
        global_parameters.evaluate = self.x
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        global_parameters.evaluate = self.old.pop()
```

#### 4.3.1 上下文管理器使用示例

```python
from sympy import symbols, Add, Mul, evaluate
from sympy.abc import x, y

# 默认行为
print(x + x)        # 2*x
print(Mul(2, 3))    # 6

# 临时禁用规范化
with evaluate(False):
    print(Add(x, x))           # x + x
    print(Mul(2, 3))            # 2*3
    print(Add(x, Add(y, y)))    # x + (y + y)（嵌套也不规范化）

# 恢复默认
print(x + x)        # 2*x
```

### 4.4 `_from_args` 与 `_new_rawargs` - 旁路构建

当需要绕过完整的规范化流程时，SymPy 提供了快速构建方法：

#### 4.4.1 `_from_args` - 类方法

```python
# sympy/core/operations.py:118-133
@classmethod
def _from_args(cls, args, is_commutative=None):
    """Create new instance with already-processed args.
    If the args are not in canonical order, then a non-canonical
    result will be returned, so use with caution."""
    if len(args) == 0:
        return cls.identity
    elif len(args) == 1:
        return args[0]
    
    # 直接调用 Basic.__new__，不经过规范化
    obj = super().__new__(cls, *args)
    
    # 设置交换性
    if is_commutative is None:
        is_commutative = fuzzy_and(a.is_commutative for a in args)
    obj.is_commutative = is_commutative
    
    return obj
```

#### 4.4.2 `_new_rawargs` - 实例方法

```python
# sympy/core/operations.py:135-182
def _new_rawargs(self, *args, reeval=True, **kwargs):
    """Create new instance of own class with args exactly as provided by
    caller but returning the self class identity if args is empty.
    
    Note: use this with caution. There is no checking of arguments at
    all. This is best used when you are rebuilding an Add or Mul after
    simply removing one or more args."""
    
    # 重用 self 的交换性设置
    if reeval and self.is_commutative is False:
        is_commutative = None
    else:
        is_commutative = self.is_commutative
    
    return self._from_args(args, is_commutative)
```

#### 4.4.3 使用场景示例

```python
from sympy import symbols, Add, Mul
from sympy.abc import x, y, z

# 场景1：已有规范参数，快速重建
expr = Add(1, x, y)
# 移除 y 后快速创建新 Add
new_expr = expr._new_rawargs(1, x)  # Add(1, x)
# 等价但更高效：Add(1, x) 会重新规范化

# 场景2：evaluate=False 时的内部构建
# Add._from_args([x, x]) 直接创建，不合并同类项
unevaluated = Add._from_args([x, x])
print(unevaluated)  # x + x（未合并）
print(unevaluated == Add(x, x, evaluate=False))  # True
```

---

## 大表达式树的哈希缓存策略

### 5.1 `_mhash` 实例级缓存

每个 `Basic` 对象都有自己的 `_mhash` 字段存储缓存的哈希值：

```python
# sympy/core/basic.py:208-214
__slots__ = ('_mhash',              # hash value
             '_args',               # arguments
             '_assumptions'
            )

_mhash: int | None
```

#### 5.1.1 缓存生命周期

```python
# 对象创建时
obj = object.__new__(cls)
obj._mhash = None  # 初始为 None，表示未计算

# 首次访问哈希时
def __hash__(self):
    h = self._mhash
    if h is None:
        # 计算哈希
        h = hash((type(self).__name__,) + self._hashable_content())
        self._mhash = h  # 缓存
    return h

# 后续访问：直接返回缓存值
```

#### 5.1.2 递归计算与缓存

对于大表达式树，哈希计算是递归的：

```python
# 表达式：2*x**2 + 3*y + z
# 结构：
# Add
# ├── Integer(2)
# ├── Mul
# │   ├── Integer(2)
# │   └── Pow
# │       ├── Symbol('x')
# │       └── Integer(2)
# ├── Mul
# │   ├── Integer(3)
# │   └── Symbol('y')
# └── Symbol('z')

# 计算 hash(Add) 时：
# 1. 需要 hash(_hashable_content())
# 2. _hashable_content() 是 args 元组
# 3. 元组的哈希需要每个元素的哈希
# 4. 每个元素（如 Mul）也有自己的 _mhash 缓存

# 首次计算：
# hash(Add) → 需要 hash(Mul1) → 需要 hash(Pow) → 需要 hash(Symbol('x'))
#                                      → 需要 hash(Integer(2))
#                    → 需要 hash(Mul2) → 需要 hash(Symbol('y'))
#                    → 需要 hash(Symbol('z'))

# 第二次计算：
# 所有子节点的 _mhash 已缓存，直接读取
```

### 5.2 `cacheit` 函数级缓存

除了实例级的 `_mhash`，SymPy 还提供函数级缓存装饰器：

```python
# sympy/core/cache.py:46-86
def __cacheit(maxsize):
    """caching decorator.
    
    important: the result of cached function must be *immutable*
    """
    def func_wrapper(func):
        cfunc = lru_cache(maxsize, typed=True)(func)
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            try:
                retval = cfunc(*args, **kwargs)
            except TypeError as e:
                if not e.args or not e.args[0].startswith('unhashable type:'):
                    raise
                retval = func(*args, **kwargs)
            return retval
        
        wrapper.cache_info = cfunc.cache_info
        wrapper.cache_clear = cfunc.cache_clear
        
        CACHE.append(wrapper)
        return wrapper
    
    return func_wrapper
```

#### 5.2.1 缓存配置

```python
# sympy/core/cache.py:124-153
# SYMPY_USE_CACHE=yes/no/debug
USE_CACHE = _getenv('SYMPY_USE_CACHE', 'yes').lower()

# SYMPY_CACHE_SIZE=some_integer/None
# 特殊值：
#   SYMPY_CACHE_SIZE=0    -> 无缓存
#   SYMPY_CACHE_SIZE=None -> 无限缓存
scs = _getenv('SYMPY_CACHE_SIZE', '1000')
if scs.lower() == 'none':
    SYMPY_CACHE_SIZE = None
else:
    SYMPY_CACHE_SIZE = int(scs)

# 根据环境变量选择缓存实现
if USE_CACHE == 'no':
    cacheit = __cacheit_nocache  # 无缓存
elif USE_CACHE == 'yes':
    cacheit = __cacheit(SYMPY_CACHE_SIZE)  # LRU 缓存
elif USE_CACHE == 'debug':
    cacheit = __cacheit_debug(SYMPY_CACHE_SIZE)  # 调试模式（额外检查）
```

#### 5.2.2 `cacheit` 使用场景

`cacheit` 主要用于：
1. **构造函数缓存**：避免重复创建相同对象
2. **方法缓存**：如 `sort_key`、`as_coeff_add` 等

```python
# 示例1：AssocOp.__new__ 缓存
# sympy/core/operations.py:62
@cacheit
def __new__(cls, *args, evaluate=None, _sympify=True):
    # ... 相同参数返回缓存对象

# 示例2：Basic.sort_key 缓存
# sympy/core/basic.py:453
@cacheit
def sort_key(self, order=None):
    # ...

# 示例3：Add.as_coeff_add 缓存
# sympy/core/add.py:426
@cacheit
def as_coeff_add(self, *deps):
    # ...
```

#### 5.2.3 构造函数缓存的效果

```python
from sympy import symbols, Add
x = symbols('x')

# 第一次调用：创建新对象
a1 = Add(x, x)  # 2*x（规范化后）

# 第二次调用：相同参数，返回缓存对象
a2 = Add(x, x)
print(a1 is a2)  # True（缓存命中）

# 不同参数：创建新对象
a3 = Add(x, 1)
print(a1 is a3)  # False
```

### 5.3 缓存一致性与失效机制

#### 5.3.1 `_mhash` 的一致性保证

由于 `Basic` 对象的不可变性：
- `_args` 不会改变
- `_hashable_content()` 不会改变
- 因此 `_mhash` 一旦计算就永远有效

**无需失效机制**，因为对象本身不可变。

#### 5.3.2 `global_parameters` 变更时的缓存清除

```python
# sympy/core/parameters.py:64-67
def __setattr__(self, name, value):
    if getattr(self, name) != value:
        clear_cache()  # 关键：参数变更时清除所有缓存
    return super().__setattr__(name, value)
```

**原因**：`global_parameters.evaluate` 的改变会影响构造函数的返回值：

```python
from sympy import symbols, Add
from sympy.core.parameters import global_parameters
from sympy.core.cache import clear_cache

x = symbols('x')

# evaluate=True 时
a1 = Add(x, x)  # 2*x
print(a1)  # 2*x

# 切换 evaluate=False
global_parameters.evaluate = False  # 自动调用 clear_cache()

# 现在 Add(x, x) 返回不同结果
a2 = Add(x, x)
print(a2)  # x + x
```

#### 5.3.3 手动清除缓存

```python
from sympy.core.cache import clear_cache, print_cache

# 查看缓存信息
print_cache()
# 输出类似：
# as_coeff_add CacheInfo(hits=42, misses=15, maxsize=1000, currsize=15)
# sort_key CacheInfo(hits=128, misses=32, maxsize=1000, currsize=32)
# ...

# 清除所有缓存
clear_cache()
```

### 5.4 性能优化考量

#### 5.4.1 两级缓存策略

SymPy 采用**两级缓存**策略：

```
┌─────────────────────────────────────────────────────────────┐
│                    缓存层次结构                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Level 1: 实例级缓存 (_mhash)                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │   Add 对象   │  │   Mul 对象   │  │  Symbol 对象 │       │
│  │ _mhash: 123 │  │ _mhash: 456 │  │ _mhash: 789 │       │
│  └─────────────┘  └─────────────┘  └─────────────┘       │
│  • 随对象生命周期存在                                          │
│  • 首次访问时计算                                              │
│  • 对象不可变，无需失效                                        │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Level 2: 函数级缓存 (@cacheit)                             │
│  ┌─────────────────────────────────────────┐               │
│  │ lru_cache(maxsize=1000, typed=True)     │               │
│  │ 键: (cls, args, kwargs)                  │               │
│  │ 值: 构造结果 / 方法结果                   │               │
│  └─────────────────────────────────────────┘               │
│  • 全局共享（但线程安全配置）                                  │
│  • LRU 淘汰策略                                              │
│  • 可手动清除（clear_cache）                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 5.4.2 大表达式树的优化

对于深度嵌套的大表达式：

1. **`_mhash` 缓存避免重复计算**：
   - 每个节点的哈希只计算一次
   - 父节点计算时直接使用子节点的缓存值

2. **构造函数缓存避免重复创建**：
   - `Add(x, y, z)` 多次调用返回同一对象
   - 结合规范化，语义相同的表达式共享对象

3. **内存占用优化**：
   - `__slots__` 减少每个对象的内存开销
   - 对象共享（通过 `cacheit`）减少总对象数

#### 5.4.3 缓存失效的性能影响

```python
# 场景：频繁切换 global_parameters.evaluate
# 这会导致性能问题！

from sympy.core.parameters import global_parameters

# 不推荐：频繁切换
for _ in range(1000):
    global_parameters.evaluate = True   # 清除缓存
    # ... 一些操作 ...
    global_parameters.evaluate = False  # 再次清除缓存
    # ... 更多操作 ...

# 推荐：使用上下文管理器，减少切换次数
from sympy import evaluate

# 批量处理
with evaluate(False):
    for _ in range(1000):
        # ... 操作 ...
        pass
```

---

## 代码引用索引

| 概念 | 文件路径 | 行号 |
|------|----------|------|
| Basic.__new__ | `sympy/core/basic.py` | 294-300 |
| Basic.__hash__ | `sympy/core/basic.py` | 321-328 |
| Basic._hashable_content | `sympy/core/basic.py` | 330-338 |
| Basic.args | `sympy/core/basic.py` | 912-942 |
| Basic.compare | `sympy/core/basic.py` | 372-428 |
| Basic.__eq__ | `sympy/core/basic.py` | 503-543 |
| ordering_of_classes | `sympy/core/basic.py` | 58-94 |
| _cmp_name | `sympy/core/basic.py` | 96-139 |
| AssocOp.__new__ | `sympy/core/operations.py` | 62-116 |
| AssocOp._from_args | `sympy/core/operations.py` | 118-133 |
| AssocOp._new_rawargs | `sympy/core/operations.py` | 135-182 |
| Add.flatten | `sympy/core/add.py` | 198-404 |
| _addsort | `sympy/core/add.py` | 40-42 |
| Mul.flatten | `sympy/core/mul.py` | 210-739 |
| _mulsort | `sympy/core/mul.py` | 38-40 |
| Pow.__new__ | `sympy/core/power.py` | 137-233 |
| global_parameters | `sympy/core/parameters.py` | 69 |
| evaluate 上下文管理器 | `sympy/core/parameters.py` | 71-104 |
| cacheit | `sympy/core/cache.py` | 46-153 |
| clear_cache | `sympy/core/cache.py` | 26-36 |
| default_sort_key | `sympy/core/sorting.py` | 11-167 |
| ordered | `sympy/core/sorting.py` | 203-313 |

---

## 总结

SymPy `Basic` 子类的核心设计围绕**规范化**和**不可变性**两大原则：

### 规范化机制
1. **`__new__` 而非 `__init__`**：完全控制对象创建流程，可返回不同类型或缓存对象
2. **`flatten` 深度规范化**：`Add`、`Mul` 等类通过 `flatten` 方法实现合并同类项、扁平化、排序等
3. **canonical order**：通过 `ordering_of_classes`、`compare`、`default_sort_key` 确保数学等价表达式具有相同内部表示

### 不可变性设计
1. **`_args` 使用元组**：一旦创建无法修改
2. **`args` 只读属性**：禁止外部修改
3. **修改必须创建新对象**：所有"修改"操作返回新对象，原对象保持不变

### 哈希与缓存
1. **`_mhash` 实例级缓存**：每个对象的哈希延迟计算并缓存，利用不可变性保证一致性
2. **`cacheit` 函数级缓存**：基于 `lru_cache`，避免重复创建相同对象
3. **两级缓存策略**：实例缓存避免重复计算，函数缓存避免重复创建

### `evaluate=False` 旁路
1. **`global_parameters` 全局开关**：线程本地存储，支持多线程隔离
2. **`evaluate(False)` 上下文管理器**：临时禁用规范化，确保嵌套调用一致
3. **`_from_args` / `_new_rawargs`**：快速构建方法，绕过完整规范化流程

这些设计共同实现了：
- **数学正确性**：等价表达式被视为相同
- **内存效率**：对象共享减少内存占用
- **计算效率**：多层缓存避免重复计算
- **API 灵活性**：`evaluate=False` 满足特殊需求（如显示未规范化形式）

---

*报告生成日期：2026-04-27*
