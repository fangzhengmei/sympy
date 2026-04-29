# SymPy 三种替换机制分析报告

## 目录
1. [概述](#概述)
2. [带数学化简语义的替换：subs](#带数学化简语义的替换subs)
3. [结构精确替换：xreplace](#结构精确替换xreplace)
4. [模式匹配替换：replace + Wild](#模式匹配替换replace--wild)
5. [缓存装饰器 cacheit 的影响](#缓存装饰器-cacheit-的影响)
6. [字典批量替换的执行顺序](#字典批量替换的执行顺序)
7. [总结与对比](#总结与对比)

---

## 概述

SymPy 提供了三种本质不同的表达式替换机制，它们在数学语义、结构行为和实现路径上有显著差异：

| 机制 | 入口方法 | 核心特点 | 是否触发化简 | 匹配精度 |
|------|----------|----------|--------------|----------|
| **数学语义替换** | `subs()` | 带数学理解的替换 | 是 | 数学等价匹配 |
| **结构精确替换** | `xreplace()` | 精确节点替换 | 否 | 严格结构相等 |
| **模式匹配替换** | `replace()` | 通配符模式匹配 | 可选 | 模式匹配 |

---

## 带数学化简语义的替换：subs

### 1. 核心设计理念

`subs` 是 SymPy 最常用的替换方法，其设计目标是**数学语义驱动**的替换，而非单纯的文本或结构替换。它理解表达式的数学含义，能够在替换过程中进行必要的数学化简和调整。

### 2. 实现路径分析

#### 入口方法 (`basic.py:971-1180`)

```python
def subs(self, arg1, arg2=None, **kwargs):
    # 参数处理：支持 (old, new) 或 dict/iterable 形式
    if arg2 is None:
        if isinstance(arg1, (set, Dict, Mapping)):
            # 无序容器：需要排序
            items = arg1.items()
            unordered = True
        else:
            # 有序容器：保持原有顺序
            items = arg1
    else:
        items = [(arg1, arg2)]
    
    # 1. 预处理：sympify 输入参数
    sequence = [(sympify_old(s1), sympify_new(s2)) for s1, s2 in items]
    
    # 2. 过滤掉无意义的替换 (old == new)
    sequence = [(s1, s2) for s1, s2 in sequence if not _aresame(s1, s2)]
    
    # 3. 处理无序容器的排序
    if unordered:
        # 按复杂度排序：更复杂的表达式优先
        sequence = sorted(sequence, key=lambda x: (-_nodes(x[0]), default_sort_key(x[0])))
        # 无穷大值优先处理
        if not simultaneous:
            redo = [i for i, seq in enumerate(sequence) if seq[1] in _illegal]
            for i in reversed(redo):
                sequence.insert(0, sequence.pop(i))
    
    # 4. 执行替换
    if simultaneous:
        # 同时替换：使用 Dummy 避免中间结果影响
        # ... 详见下文
    else:
        # 顺序替换
        rv = self
        for old, new in sequence:
            rv = rv._subs(old, new, **kwargs)
        return rv
```

#### 核心替换方法 `_subs` (`basic.py:1182-1292`)

```python
@cacheit  # 关键：_subs 被缓存装饰
def _subs(self, old, new, **hints):
    """
    替换策略：
    1. 首先检查 self 是否与 old 完全相同 (使用 _aresame)
    2. 如果相同，直接返回 new
    3. 否则调用 _eval_subs 进行类特定的替换逻辑
    4. 如果 _eval_subs 返回 None，使用 fallback 递归处理子表达式
    """
    
    def fallback(self, old, new):
        """
        兜底策略：递归替换所有子表达式
        遍历 self.args，对每个参数调用 _subs
        如果任何参数发生变化，重新构造表达式
        """
        hit = False
        args = list(self.args)
        for i, arg in enumerate(args):
            if not hasattr(arg, '_eval_subs'):
                continue
            arg = arg._subs(old, new, **hints)
            if not _aresame(arg, args[i]):
                hit = True
                args[i] = arg
        if hit:
            rv = self.func(*args)  # 重新构造，可能触发化简
            # ... 特殊处理逻辑
            return rv
        return self
    
    # 主逻辑
    if _aresame(self, old):
        return new
    
    rv = self._eval_subs(old, new)
    if rv is None:
        rv = fallback(self, old, new)
    return rv
```

### 3. 数学化简的触发时机

#### 类特定的 `_eval_subs` 方法

不同的表达式类型（Add, Mul, Pow 等）有自己的 `_eval_subs` 实现，这是数学化简的主要入口。

**Add._eval_subs (`add.py:898-933`)**:
```python
def _eval_subs(self, old, new):
    if not old.is_Add:
        if old is S.Infinity and -old in self.args:
            # foo - oo 内部表示为 foo + (-oo)
            return self.xreplace({-old: -new})
        return None
    
    # 数学语义：理解常数项和符号项的关系
    coeff_self, terms_self = self.as_coeff_Add()
    coeff_old, terms_old = old.as_coeff_Add()
    
    # 案例1: (2 + a).subs(3 + a, y) -> -1 + y
    if coeff_self.is_Rational and coeff_old.is_Rational:
        if terms_self == terms_old:
            return self.func(new, coeff_self, -coeff_old)
        if terms_self == -terms_old:  # (2 + a).subs(-3 - a, y) -> -1 - y
            return self.func(-new, coeff_self, coeff_old)
    
    # 案例2: (a+b+c).subs(b+c, x) -> a+x
    if len(args_old) < len(args_self):
        self_set = set(args_self)
        old_set = set(args_old)
        if old_set < self_set:  # old 是 self 的子集
            ret_set = self_set - old_set
            return self.func(new, coeff_self, -coeff_old,
                       *[s._subs(old, new) for s in ret_set])
```

**Mul._eval_subs (`mul.py:1709-1850`)**:
```python
def _eval_subs(self, old, new):
    if not old.is_Mul:
        return None
    
    # 处理负数系数的特殊情况
    if old.args[0].is_Number and old.args[0] < 0:
        if self.args[0].is_Number:
            if self.args[0] < 0:
                return self._subs(-old, -new)  # 符号匹配
            return None  # 不匹配，保持字面替换
    
    # 幂次分解和匹配
    def breakup(eq):
        """将乘积分解为 {base: exponent} 字典"""
        (c, nc) = (defaultdict(int), [])
        for a in Mul.make_args(eq):
            a = powdenest(a)
            (b, e) = base_exp(a)
            if e is not S.One:
                (co, _) = e.as_coeff_mul()
                b = Pow(b, e/co)
                e = co
            if a.is_commutative:
                c[b] += e
            else:
                nc.append([b, e])
        return (c, nc)
    
    # 支持数学等价的替换：例如 (x**4).subs(x**2, y) == y**2
    # 这种替换需要数学理解，而非简单的结构匹配
```

### 4. 化简触发的关键因素

| 因素 | 说明 | 示例 |
|------|------|------|
| **类特定的 _eval_subs** | Add, Mul, Pow 等有特殊逻辑 | `(a+b+c).subs(b+c, x)` → `a+x` |
| **表达式重构** | `self.func(*args)` 可能触发构造器化简 | `Add(2, 3)` → `5` |
| **幂次匹配** | Mul 支持幂次分解和重组 | `(x**4).subs(x**2, y)` → `y**2` |
| **符号处理** | 负数系数的特殊匹配逻辑 | `(-2*x).subs(2*x, y)` 保持不变 |

### 5. 不会触发化简的情况

1. **完全结构匹配**: `_aresame(self, old)` 返回 True 时直接返回 new
2. **_eval_subs 返回具体值**: 而非 None 时，直接使用返回值
3. **使用 evaluate=False**: 某些场景下可以阻止化简
4. **hack2 模式**: 用于保持未求值状态（2-arg hack）

---

## 结构精确替换：xreplace

### 1. 核心设计理念

`xreplace` 是**纯结构替换**，不理解任何数学语义，只进行**精确的节点匹配**。其设计目标是：

- **精确性**: 只替换完全相等的节点
- **无副作用**: 不引入任何额外的数学化简
- **一致性**: 行为可预测，不受表达式数学含义影响

### 2. 实现路径分析

#### 入口方法 (`basic.py:1305-1390`)

```python
def xreplace(self, rule):
    """
    精确的节点替换：
    - 只替换表达式树中完全匹配的节点
    - 不进行任何数学化简
    - 不理解数学等价性
    """
    value, _ = self._xreplace(rule)
    return value

def _xreplace(self, rule):
    """
    核心实现：返回 (替换后的表达式, 是否发生变化)
    """
    # 1. 首先检查整个节点是否在替换规则中
    if self in rule:
        return rule[self], True  # 精确匹配：直接替换
    
    # 2. 递归处理子表达式
    elif rule:
        args = []
        changed = False
        for a in self.args:
            _xreplace = getattr(a, '_xreplace', None)
            if _xreplace is not None:
                a_xr = _xreplace(rule)
                args.append(a_xr[0])
                changed |= a_xr[1]
            else:
                args.append(a)
        args = tuple(args)
        
        # 3. 只有当子表达式发生变化时才重构
        if changed:
            return self.func(*args), True  # 注意：这里可能触发构造器化简！
    
    return self, False
```

### 3. 与 subs 的关键差异

| 维度 | subs | xreplace |
|------|------|-----------|
| **匹配方式** | 数学等价 + 结构匹配 | 仅结构相等 (`self in rule`) |
| **递归策略** | _subs → _eval_subs → fallback | 简单的 args 递归 |
| **化简行为** | 可能触发多种化简 | 仅重构时可能触发构造器化简 |
| **适用场景** | 数学替换、符号替换 | 精确节点替换、无副作用操作 |
| **绑定变量** | 区分自由变量和绑定变量 | 不区分，全部替换 |

### 4. 行为差异示例

```python
from sympy import symbols, sqrt, exp
x, y = symbols('x y')

# 示例1: 幂次匹配
expr = x**4

# subs: 理解数学等价性
expr.subs(x**2, y)  # y**2 (数学化简)

# xreplace: 只替换精确节点
expr.xreplace({x**2: y})  # x**4 (x**2 不是 x**4 的直接子节点)

# 示例2: 子表达式位置
expr2 = x*y + z

# subs: 可能进行数学语义的匹配
# (取决于具体的 _eval_subs 实现)

# xreplace: 只替换精确的节点
expr2.xreplace({x*y: 1})  # 1 + z (x*y 是精确的子节点)
expr2.xreplace({y*x: 1})  # 可能也是 1 + z (取决于规范化顺序)

# 示例3: 嵌套结构
expr3 = (x*y*z)
expr3.xreplace({x*y: 1})  # x*y*z (x*y 不是直接子节点)

# 示例4: 绑定变量
from sympy import Integral
expr4 = Integral(x, (x, 1, 2*x))

# subs: 不替换绑定变量
expr4.subs(x, y)  # Integral(x, (x, 1, 2*y))

# xreplace: 替换所有出现
expr4.xreplace({x: y})  # Integral(y, (y, 1, 2*y))
```

### 5. xreplace 的"隐性化简"

注意：虽然 xreplace 设计为无化简，但 `self.func(*args)` 调用仍可能触发构造器的化简：

```python
# 例如：
from sympy import Add, S
expr = Add(x, S(1), evaluate=False)  # 未求值的加法
result = expr.xreplace({S(1): S(2)})  # 替换后
# result 可能是 x + 2 (经过 Add 构造器的求值)
```

这是因为即使 xreplace 本身不进行化简，表达式的重构过程会经过类的构造器，而构造器通常会进行规范化。

---

## 模式匹配替换：replace + Wild

### 1. 核心设计理念

`replace` 是**模式驱动**的替换机制，结合了：
- **通配符匹配**: 使用 `Wild` 符号作为占位符
- **底向上遍历**: 从叶子节点开始处理
- **多模式支持**: 支持 type→type, pattern→expr, func→func 等多种模式

### 2. Wild 符号的语义

#### Wild 类定义 (`symbol.py:545-673`)

```python
class Wild(Symbol):
    """
    Wild 符号可以匹配任何表达式（或满足特定条件的表达式）
    
    关键属性：
    - exclude: 不能匹配的表达式列表
    - properties: 匹配所需满足的属性函数列表
    """
    is_Wild = True
    
    __slots__ = ('exclude', 'properties')
    
    def __new__(cls, name, exclude=(), properties=(), **assumptions):
        exclude = tuple([sympify(x) for x in exclude])
        properties = tuple(properties)
        return Wild.__xnew__(cls, name, exclude, properties, **assumptions)
    
    @staticmethod
    @cacheit
    def __xnew__(cls, name, exclude, properties, **assumptions):
        obj = Symbol.__xnew__(cls, name, **assumptions)
        obj.exclude = exclude
        obj.properties = properties
        return obj
    
    def matches(self, expr, repl_dict=None, old=False):
        """
        Wild 的匹配逻辑：
        1. 检查 expr 是否包含 exclude 中的任何表达式
        2. 检查 expr 是否满足所有 properties 函数
        3. 如果都满足，将 {self: expr} 加入替换字典
        """
        if any(expr.has(x) for x in self.exclude):
            return None
        if not all(f(expr) for f in self.properties):
            return None
        if repl_dict is None:
            repl_dict = {}
        else:
            repl_dict = repl_dict.copy()
        repl_dict[self] = expr
        return repl_dict
```

#### Wild 与普通 Symbol 的关键差异

| 特性 | Wild | Symbol |
|------|------|--------|
| **匹配语义** | 匹配任何（符合条件的）表达式 | 只匹配自身 |
| **exclude 列表** | 可指定不能匹配的表达式 | 无此机制 |
| **properties 函数** | 可指定匹配条件函数 | 无此机制 |
| **哈希与相等** | 包含 exclude 和 properties | 只包含 name 和 assumptions |
| **使用场景** | 模式匹配中的占位符 | 数学变量 |

### 3. replace 方法实现

#### 核心实现 (`basic.py:1548-1802`)

```python
def replace(self, query, value, map=False, simultaneous=True, exact=None):
    """
    支持多种查询模式：
    
    1. type -> type: obj.replace(type, newtype)
       - 替换类型：将 type 类型的对象替换为 newtype(*args)
       
    2. type -> func: obj.replace(type, func)
       - 对 type 类型的对象应用 func(*args)
       
    3. pattern -> expr: obj.replace(pattern(wild), expr(wild))
       - 使用 Wild 模式匹配，替换为 expr
       
    4. pattern -> func: obj.replace(pattern(wild), lambda wild: expr)
       - 匹配后调用函数处理
       
    5. func -> func: obj.replace(filter, func)
       - 对 filter(e) == True 的表达式应用 func(e)
    """
    
    # 步骤1: 确定查询和值的类型
    try:
        query = _sympify(query)
    except SympifyError:
        pass
    try:
        value = _sympify(value)
    except SympifyError:
        pass
    
    # 步骤2: 根据类型构建匹配和替换函数
    if isinstance(query, type):
        # 模式1: 类型替换
        _query = lambda expr: isinstance(expr, query)
        if isinstance(value, type) or callable(value):
            _value = lambda expr, result: value(*expr.args)
        else:
            raise TypeError("given a type, replace() expects another type or a callable")
    
    elif isinstance(query, Basic):
        # 模式2: 表达式模式匹配（使用 Wild）
        _query = lambda expr: expr.match(query)  # 关键：调用 match 方法
        
        # 确定 exact 标志
        if exact is None:
            from .symbol import Wild
            exact = (len(query.atoms(Wild)) > 1)  # 多个 Wild 时默认为精确匹配
        
        # 构建值处理函数
        if isinstance(value, Basic):
            if exact:
                # 精确模式：所有 Wild 必须匹配非零值
                _value = lambda expr, result: (value.subs(result)
                    if all(result.values()) else expr)
            else:
                _value = lambda expr, result: value.subs(result)
        elif callable(value):
            # 函数值：将匹配结果作为关键字参数传递
            if exact:
                _value = lambda expr, result: (value(**
                    {str(k)[:-1]: v for k, v in result.items()})
                    if all(val for val in result.values()) else expr)
            else:
                _value = lambda expr, result: value(**
                    {str(k)[:-1]: v for k, v in result.items()})
    
    elif callable(query):
        # 模式3: 函数过滤
        _query = query
        if callable(value):
            _value = lambda expr, result: value(expr)
        else:
            raise TypeError("given a callable, replace() expects another callable")
    
    # 步骤3: 定义遍历函数（底向上）
    def walk(rv, F):
        """
        从底向上遍历表达式树：
        1. 先递归处理所有子节点
        2. 然后对当前节点应用 F
        """
        args = getattr(rv, 'args', None)
        if args is not None:
            if args:
                newargs = tuple([walk(a, F) for a in args])
                if args != newargs:
                    rv = rv.func(*newargs)
                    if simultaneous:
                        # 同时模式：避免重复替换已修改的节点
                        for i, e in enumerate(args):
                            if rv == e and e != newargs[i]:
                                return rv
                rv = F(rv)
        return rv
    
    # 步骤4: 定义替换函数
    mapping = {}
    
    def rec_replace(expr):
        result = _query(expr)
        if result or result == {}:
            v = _value(expr, result)
            if v is not None and v != expr:
                if map:
                    mapping[expr] = v
                expr = v
        return expr
    
    # 步骤5: 执行替换
    rv = walk(self, rec_replace)
    return (rv, mapping) if map else rv
```

### 4. match/matches 方法

#### Basic.match 和 Basic.matches (`basic.py:1818-1924`)

```python
def matches(self, expr, repl_dict=None, old=False):
    """
    模式匹配的核心方法：
    - pattern.matches(expr) 检查 pattern 是否匹配 expr
    - 返回匹配字典 {Wild: 匹配的子表达式} 或 None
    
    匹配规则：
    1. 类型必须相同
    2. 参数数量必须相同
    3. 递归匹配参数
    4. Wild 符号可以匹配任何表达式（受 exclude/properties 限制）
    """
    expr = sympify(expr)
    
    # 类型检查
    if not isinstance(expr, self.__class__):
        return None
    
    if repl_dict is None:
        repl_dict = {}
    else:
        repl_dict = repl_dict.copy()
    
    # 完全相等
    if self == expr:
        return repl_dict
    
    # 参数数量检查
    if len(self.args) != len(expr.args):
        return None
    
    # 递归匹配参数
    d = repl_dict
    for arg, other_arg in zip(self.args, expr.args):
        if arg == other_arg:
            continue
        # 对参数应用已有的替换，然后继续匹配
        d = arg.xreplace(d).matches(other_arg, d, old=old)
        if d is None:
            return None
    return d

def match(self, pattern, old=False):
    """
    expr.match(pattern) 等价于 pattern.matches(expr)
    """
    pattern = sympify(pattern)
    return pattern.matches(self, old=old)
```

### 5. 模式匹配的高级特性

#### exact 标志的作用

当 pattern 包含多个 Wild 符号时，`exact=True`（默认）要求所有 Wild 匹配非零值：

```python
from sympy import Wild, symbols
a, b = symbols('a b', cls=Wild)
x, y = symbols('x y')

# 多个 Wild 的情况
expr = 2*x
pattern = a*x + b

# exact=True (默认): b 必须匹配非零值
result = expr.replace(pattern, b - a)
print(result)  # 2*x (不匹配，因为 b 会匹配 0)

# exact=False: 允许零值匹配
result = expr.replace(pattern, b - a, exact=False)
print(result)  # -2 (b=0, a=2)
```

#### exclude 和 properties 的使用

```python
from sympy import Wild, Symbol, sin, cos
x = Symbol('x')

# 使用 exclude: 不匹配包含 x 的表达式
a = Wild('a', exclude=[x])

expr = 3*x
pattern = a*x
result = expr.match(pattern)
print(result)  # None (因为 3 匹配 a，但 a 不能包含 x？不对...)

# 正确使用：exclude 限制的是匹配的表达式不能包含指定内容
# 例如：我们想匹配 3*x 中的系数，但不希望系数包含 x
# a = Wild('a', exclude=[x]) 意味着匹配 a 的表达式不能包含 x

# 使用 properties: 只匹配整数
from sympy import Integer
is_integer = lambda k: k.is_Integer
a = Wild('a', properties=[is_integer])

expr = 3*x
result = expr.match(a*x)
print(result)  # {a: 3} (3 是整数)

expr = 3.5*x
result = expr.match(a*x)
print(result)  # None (3.5 不是整数)
```

### 6. 底向上遍历与同时模式

`walk` 函数的设计：

```python
def walk(rv, F):
    args = getattr(rv, 'args', None)
    if args is not None:
        if args:
            # 先处理子节点
            newargs = tuple([walk(a, F) for a in args])
            if args != newargs:
                rv = rv.func(*newargs)
                if simultaneous:
                    # 关键：避免对已经修改过的节点再次应用 F
                    for i, e in enumerate(args):
                        if rv == e and e != newargs[i]:
                            return rv
            # 然后处理当前节点
            rv = F(rv)
    return rv
```

**simultaneous=True 的含义**：
- 子节点的修改不会导致父节点被再次处理
- 每个节点最多被匹配和替换一次
- 避免了 `2*x.replace(a*x, a*2)` 这种可能的无限循环

---

## 缓存装饰器 cacheit 的影响

### 1. cacheit 实现分析

#### 核心代码 (`cache.py:1-167`)

```python
from functools import lru_cache, wraps

def __cacheit(maxsize):
    """
    缓存装饰器实现：
    - 使用 functools.lru_cache
    - 支持 typed=True (不同类型的参数分别缓存)
    - 处理不可哈希参数的情况
    """
    def func_wrapper(func):
        cfunc = lru_cache(maxsize, typed=True)(func)
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            try:
                retval = cfunc(*args, **kwargs)
            except TypeError as e:
                # 处理不可哈希类型：绕过缓存
                if not e.args or not e.args[0].startswith('unhashable type:'):
                    raise
                retval = func(*args, **kwargs)
            return retval
        
        # 暴露缓存控制方法
        wrapper.cache_info = cfunc.cache_info
        wrapper.cache_clear = cfunc.cache_clear
        
        # 注册到全局缓存列表
        CACHE.append(wrapper)
        return wrapper
    
    return func_wrapper

# 环境变量控制
USE_CACHE = _getenv('SYMPY_USE_CACHE', 'yes').lower()
# SYMPY_CACHE_SIZE=0 -> 无缓存
# SYMPY_CACHE_SIZE=None -> 无界缓存
# 默认: 1000

if USE_CACHE == 'no':
    cacheit = __cacheit_nocache  # 完全不缓存
elif USE_CACHE == 'yes':
    cacheit = __cacheit(SYMPY_CACHE_SIZE)
elif USE_CACHE == 'debug':
    cacheit = __cacheit_debug(SYMPY_CACHE_SIZE)  # 调试模式
```

### 2. 被缓存的替换相关方法

#### _subs 方法 (`basic.py:1182`)

```python
@cacheit
def _subs(self, old, new, **hints):
    """
    关键：_subs 被 cacheit 装饰！
    
    缓存键包括：
    - self: 表达式本身（通过 __hash__）
    - old: 要替换的表达式
    - new: 替换后的表达式
    - **hints: 其他关键字参数
    
    这意味着相同的 (self, old, new, hints) 组合会返回缓存结果
    """
    # ... 实现
```

#### 其他被缓存的方法

| 方法 | 位置 | 缓存影响 |
|------|------|----------|
| `_subs` | basic.py:1182 | 替换结果缓存 |
| `has` | basic.py:1392 | 包含检查缓存 |
| `has_free` | basic.py:1468 | 自由符号包含检查缓存 |
| `sort_key` | basic.py:453 | 排序键缓存 |
| `Symbol.__xnew_cached_` | symbol.py:412 | 符号创建缓存 |
| `Wild.__xnew__` | symbol.py:652 | Wild 创建缓存 |

### 3. 缓存的失效边界

#### 缓存键的计算

缓存基于 `lru_cache(typed=True)`，键由参数的 `__hash__` 和 `__eq__` 决定。

对于 `Basic` 对象：
```python
# basic.py:321-328
def __hash__(self) -> int:
    h = self._mhash
    if h is None:
        h = hash((type(self).__name__,) + self._hashable_content())
        self._mhash = h
    return h

# 例如 Symbol 的 _hashable_content:
# symbol.py:428-429
def _hashable_content(self):
    return (self.name,) + self._assumptions0
```

#### 缓存失效的场景

| 场景 | 是否失效 | 说明 |
|------|----------|------|
| **相同表达式，相同替换** | 不失效 | 直接使用缓存 |
| **修改表达式结构** | 失效 | 新表达式有不同的 hash |
| **使用不同的 old/new** | 失效 | 缓存键不同 |
| **清除全局缓存** | 失效 | `clear_cache()` |
| **不可哈希参数** | 不缓存 | 绕过 lru_cache |

#### 重要：缓存与可变对象

`cacheit` 的文档明确指出：
```python
"""
caching decorator.

important: the result of cached function must be *immutable*

Examples
========

>>> from sympy import cacheit
>>> @cacheit
... def f(a, b):
...    return a+b

>>> @cacheit
... def f(a, b): # noqa: F811
...    return [a, b] # <-- WRONG, returns mutable object
"""
```

### 4. 缓存对替换路径的影响

#### subs 路径

```
subs(old, new)
    │
    ├──► 预处理参数（无缓存）
    │
    ├──► 循环处理每个 (old, new) 对
    │       │
    │       └──► _subs(old, new, **hints)
    │               │
    │               ├──► 检查缓存：(self, old, new, hints)
    │               │       │
    │               │       ├──► 命中：直接返回缓存结果
    │               │       │
    │               │       └──► 未命中：执行实际替换
    │               │
    │               └──► 执行替换逻辑
    │                       │
    │                       ├──► _aresame(self, old) 检查
    │                       ├──► _eval_subs (类特定)
    │                       └──► fallback (递归)
```

#### 潜在的缓存问题

**问题1: 相同结构，不同含义**

```python
# 理论上的风险：
from sympy import symbols, cacheit, clear_cache

x = symbols('x')
expr1 = x + 1
expr2 = x + 1  # 相同结构，相同 hash

# 由于 SymPy 的符号规范化，这实际上不是问题
# Symbol('x') == Symbol('x') 在相同假设下
```

**问题2: 副作用函数**

如果被缓存的函数有副作用，缓存会导致问题：
```python
# 反例模式（不要这样做）
side_effect_list = []

@cacheit
def bad_func(expr):
    side_effect_list.append(expr)
    return expr + 1

# 多次调用只执行一次副作用
```

### 5. 缓存控制

#### 清除缓存

```python
from sympy.core.cache import clear_cache, print_cache

# 打印缓存信息
print_cache()
# _subs CacheInfo(hits=42, misses=100, maxsize=1000, currsize=100)
# ...

# 清除所有缓存
clear_cache()
```

#### 环境变量控制

```bash
# 禁用缓存
export SYMPY_USE_CACHE=no

# 设置缓存大小
export SYMPY_CACHE_SIZE=5000  # 更大的缓存
export SYMPY_CACHE_SIZE=None  # 无界缓存
export SYMPY_CACHE_SIZE=0     # 等同于禁用

# 调试模式：检查缓存一致性
export SYMPY_USE_CACHE=debug
```

---

## 字典批量替换的执行顺序

### 1. subs 方法的参数处理

#### 代码分析 (`basic.py:971-1180`)

```python
def subs(self, arg1, arg2=None, **kwargs):
    """
    参数形式支持：
    
    1. 两个参数: expr.subs(old, new)
       - 单次替换
       
    2. 字典: expr.subs({old1: new1, old2: new2})
       - 无序，会被排序
       
    3. 集合: expr.subs({(old1, new1), (old2, new2)})
       - 无序，会被排序
       
    4. 列表/元组: expr.subs([(old1, new1), (old2, new2)])
       - 有序，保持原有顺序
    """
    
    # 区分有序和无序容器
    unordered = False
    if arg2 is None:
        if isinstance(arg1, set):
            items = arg1
            unordered = True
        elif isinstance(arg1, (Dict, Mapping)):
            unordered = True
            items = arg1.items()
        elif not iterable(arg1):
            raise ValueError("...")
        else:
            items = arg1  # 列表/元组：保持顺序
    else:
        items = [(arg1, arg2)]
    
    # 处理无序容器的排序
    if unordered:
        from .sorting import _nodes, default_sort_key
        sequence_dict = dict(sequence)
        
        # 排序策略：
        # 1. 按复杂度降序（更复杂的表达式优先）
        # 2. 按 default_sort_key 排序（确保确定性）
        k = list(ordered(sequence_dict, default=False, keys=(
            lambda x: -_nodes(x),  # 负号表示降序
            default_sort_key,
        )))
        sequence = [(k, sequence_dict[k]) for k in k]
        
        # 额外：无穷大值优先处理
        if not simultaneous:
            redo = [i for i, seq in enumerate(sequence) if seq[1] in _illegal]
            for i in reversed(redo):
                sequence.insert(0, sequence.pop(i))
```

### 2. 排序策略详解

#### _nodes 函数

`_nodes` 计算表达式的"节点数"，用于衡量复杂度：

```python
# 示例复杂度排序：
# f(g(x))  >  x + y  >  x  >  1
#   4节点     3节点    1节点   1节点

# 更复杂的表达式先被替换的原因：
# 避免先替换简单表达式破坏复杂表达式的结构
```

#### 排序示例

```python
from sympy import symbols, sin, exp
x = symbols('x')

# 字典方式（无序，会被排序）
replacements = {
    x: 1,                    # 简单：后替换
    sin(x): 2,               # 中等
    exp(sin(2*x)): 3,        # 复杂：先替换
}

# 实际处理顺序（按复杂度降序）：
# 1. exp(sin(2*x)) → 3
# 2. sin(x) → 2
# 3. x → 1

# 列表方式（有序，保持原顺序）
replacements_list = [
    (x, 1),
    (sin(x), 2),
]
# 处理顺序：x → 1，然后 sin(x) → 2（但 x 已被替换）
```

### 3. 顺序替换 vs 同时替换

#### 顺序替换（默认）

```python
if not simultaneous:
    rv = self
    for old, new in sequence:
        rv = rv._subs(old, new, **kwargs)
    return rv
```

**特点**：
- 每次替换的结果会影响后续替换
- 前一个替换的 `old` 可能已经被前一个替换修改

#### 同时替换（simultaneous=True）

```python
if simultaneous:
    reps = {}
    rv = self
    kwargs['hack2'] = True
    m = Dummy('subs_m')
    for old, new in sequence:
        com = new.is_commutative
        if com is None:
            com = True
        d = Dummy('subs_d', commutative=com)
        # 使用 d*m 以便 Subs 处理绑定变量
        rv = rv._subs(old, d*m, **kwargs)
        if not isinstance(rv, Basic):
            break
        reps[d] = new
    reps[m] = S.One  # 去除 m
    return rv.xreplace(reps)
```

**原理**：
1. 使用临时 Dummy 符号替换所有 `old`
2. 最后一次性将 Dummy 替换为对应的 `new`
3. 避免中间结果相互影响

### 4. 顺序对结果的影响

#### 示例1: 链式依赖

```python
from sympy import symbols
x, y = symbols('x y')

# 情况1: 列表顺序
expr = x + y

# 顺序1: x→y, y→2
result1 = expr.subs([(x, y), (y, 2)])
print(result1)  # 4 (x→y 得到 y+y, 然后 y→2 得到 2+2)

# 顺序2: y→2, x→y
result2 = expr.subs([(y, 2), (x, y)])
print(result2)  # y + 2 (y→2 得到 x+2, 然后 x→y 得到 y+2)

# 情况2: 字典（排序后）
result3 = expr.subs({x: y, y: 2})
# 排序：x 和 y 复杂度相同，按 default_sort_key 排序
# 结果取决于符号的排序顺序
```

#### 示例2: 相互依赖

```python
from sympy import symbols, exp
x, y = symbols('x y')

# 经典问题：x/y 在 x→0, y→0 时的行为
expr = x / y

# 顺序替换（取决于顺序）
result_seq1 = expr.subs([(x, 0), (y, 0)])
# x→0 得到 0/y, 然后 y→0 得到 0/0 = nan (或 0 取决于具体实现)

result_seq2 = expr.subs([(y, 0), (x, 0)])
# y→0 得到 x/0 = zoo, 然后 x→0 得到 0/0 = nan

# 同时替换
result_sim = expr.subs({x: 0, y: 0}, simultaneous=True)
# 直接得到 0/0 = nan
```

#### 示例3: 子表达式破坏

```python
from sympy import symbols, sin
x, y = symbols('x y')

expr = sin(2*x)

# 替换字典
replacements = {
    x: y,           # 简单
    2*x: 1,         # 复杂（依赖 x）
}

# 字典方式（排序后：2*x 先替换）
result_dict = expr.subs(replacements)
print(result_dict)  # sin(1) (正确：先替换 2*x→1)

# 列表方式（如果 x 先替换）
result_list = expr.subs([(x, y), (2*x, 1)])
print(result_list)  # sin(2*y) (2*x 已被破坏，无法匹配)
```

### 5. 官方推荐的最佳实践

根据代码注释和测试用例：

```python
# 1. 当替换之间有依赖时，使用列表明确指定顺序
expr.subs([(x, y), (y, 2)])  # 明确的顺序

# 2. 当需要"并行"替换时，使用 simultaneous=True
expr.subs({x: 0, y: 0}, simultaneous=True)  # 同时替换

# 3. 当替换涉及复杂子表达式时，使用字典让系统排序
# （复杂表达式优先，避免子表达式被破坏）
expr.subs({
    complex_expr: new1,
    simple_expr: new2,
})

# 4. 注意 _illegal 值（无穷大等）的特殊处理
# 这些值会被移到顺序的最前面
```

### 6. 测试用例验证

```python
# test_subs.py 中的相关测试

def test_dict_ambigous():
    """测试顺序影响"""
    f = x*exp(x)
    
    # 字典方式
    df = {x: y, exp(x): y}
    assert f.subs(df) == y**2  # 排序后：x 和 exp(x) 复杂度相同
    
    # 不同顺序的列表产生不同结果
    assert f.subs(x, y).subs(exp(x), y) == y*exp(y)  # x 先替换
    assert f.subs(exp(x), y).subs(x, y) == y**2       # exp(x) 先替换

def test_simultaneous_subs():
    """测试同时替换"""
    reps = {x: 0, y: 0}
    
    # 顺序替换结果不一致
    assert (x/y).subs(reps) != (y/x).subs(reps)
    
    # 同时替换结果一致（都是 nan）
    assert (x/y).subs(reps, simultaneous=True) == \
           (y/x).subs(reps, simultaneous=True)

def test_guard_against_indeterminate_evaluation():
    """测试不确定求值的防护"""
    eq = x**y
    
    # 列表顺序影响结果
    assert eq.subs([(x, 1), (y, oo)]) == 1       # 1**y == 1
    assert eq.subs([(y, oo), (x, 1)]) is S.NaN   # y→oo 先，得到 x**oo，然后 x→1
    
    # 字典方式（排序不确定）
    assert eq.subs({x: 1, y: oo}) is S.NaN
    
    # 同时替换
    assert eq.subs([(x, 1), (y, oo)], simultaneous=True) is S.NaN
```

---

## 总结与对比

### 1. 三种替换机制的核心差异

| 维度 | subs (数学语义) | xreplace (结构精确) | replace (模式匹配) |
|------|------------------|---------------------|---------------------|
| **设计目标** | 数学替换 | 精确节点替换 | 模式驱动替换 |
| **匹配方式** | 数学等价 + 结构 | 仅结构相等 | Wild 模式匹配 |
| **递归策略** | _subs → _eval_subs → fallback | 简单 args 递归 | walk 底向上遍历 |
| **化简行为** | 可能触发多种化简 | 仅构造器可能化简 | 取决于替换值 |
| **绑定变量** | 区分自由/绑定变量 | 不区分 | 取决于模式 |
| **缓存** | _subs 被缓存 | 无缓存 | 无缓存 |
| **典型用例** | 符号替换、数学变换 | 精确重命名、无副作用操作 | 规则重写、模式识别 |

### 2. 实现路径对比图

```
┌─────────────────────────────────────────────────────────────────┐
│                         替换入口层                                │
├─────────────────┬─────────────────┬─────────────────────────────┤
│      subs       │    xreplace     │         replace             │
├─────────────────┼─────────────────┼─────────────────────────────┤
│  参数处理       │  直接转换规则   │  解析 query/value 类型      │
│  排序(无序时)   │                 │  构建 _query/_value 函数   │
│  顺序/同时替换  │                 │                             │
└────────┬────────┴────────┬────────┴─────────────┬───────────────┘
         │                 │                      │
         ▼                 ▼                      ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────────┐
│    _subs        │ │   _xreplace     │ │     walk (底向上)        │
│  (cacheit 缓存) │ │                 │ │                         │
├─────────────────┤ ├─────────────────┤ ├─────────────────────────┤
│ 1. _aresame 检查 │ │ 1. self in rule │ │ 1. 递归处理子节点        │
│ 2. _eval_subs   │ │ 2. 递归处理 args │ │ 2. 同时模式检查          │
│    (类特定逻辑)  │ │ 3. func(*args)  │ │ 3. 应用 F (rec_replace)  │
│ 3. fallback     │ │    重构         │ │                         │
│    (递归子表达式)│ │                 │ │                         │
└────────┬────────┘ └────────┬────────┘ └───────────┬─────────────┘
         │                 │                      │
         ▼                 ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                      表达式重构层                                 │
│                                                                   │
│  self.func(*args) 是所有替换路径的共同点                        │
│  - 可能触发类构造器的化简（flatten 等）                          │
│  - 这是"隐性化简"的主要来源                                      │
└─────────────────────────────────────────────────────────────────┘
```

### 3. 数学化简触发路径

```
subs 触发化简的可能点：

1. _eval_subs 方法（类特定）
   ├── Add._eval_subs: 理解常数项、子集匹配
   ├── Mul._eval_subs: 幂次分解、符号匹配
   ├── Pow._eval_subs: 指数处理
   └── ... 其他类

2. fallback 重构
   └── self.func(*args) → 触发构造器

3. 构造器本身
   ├── Add.flatten: 合并同类项
   ├── Mul.flatten: 合并因子
   └── ... 规范化处理

xreplace 触发化简的可能点：

1. 仅 self.func(*args) 重构
   └── 构造器可能进行规范化

replace 触发化简的可能点：

1. 同上：self.func(*args) 重构
2. 替换值的 subs（_value 函数中）
   └── value.subs(result) 可能触发 subs 的化简
```

### 4. 缓存影响总结

| 机制 | 缓存位置 | 缓存键 | 失效条件 |
|------|----------|--------|----------|
| subs | `_subs` 方法 | `(self, old, new, **hints)` | 不同的参数、`clear_cache()` |
| xreplace | 无 | - | - |
| replace | 无 | - | - |
| Wild 创建 | `__xnew__` | `(name, exclude, properties, assumptions)` | 不同的参数 |
| Symbol 创建 | `__xnew_cached_` | `(name, assumptions)` | 不同的参数 |

**缓存的实际影响**：
- 大量重复的相同替换会显著加速
- 但可能掩盖一些实现细节（首次执行 vs 缓存命中）
- 调试时可以用 `SYMPY_USE_CACHE=no` 禁用缓存

### 5. 批量替换顺序总结

| 容器类型 | 排序策略 | 使用建议 |
|----------|----------|----------|
| **列表/元组** | 保持原有顺序 | 需要精确控制顺序时使用 |
| **字典/集合** | 复杂度降序 → default_sort_key | 复杂子表达式优先，避免破坏 |

**关键要点**：
1. **列表是有序的**：`[(a, b), (c, d)]` 严格按顺序执行
2. **字典是无序的**：会被排序，复杂表达式优先
3. **无穷大优先**：`_illegal` 值（如 oo, zoo, nan）被移到最前
4. **同时替换**：使用 `simultaneous=True` 避免中间结果影响
5. **依赖链影响**：替换之间有依赖时，顺序决定结果

### 6. 选择正确的替换机制

**何时使用 `subs`**：
- 需要数学语义的替换
- 符号到值的替换
- 需要处理 Add/Mul/Pow 等的数学等价性
- 接受（或期望）化简发生

**何时使用 `xreplace`**：
- 需要精确的节点替换
- 不希望引入额外化简
- 批量重命名符号
- 处理绑定变量（如 Integral 中的积分变量）

**何时使用 `replace`**：
- 需要模式匹配（使用 Wild）
- 需要底向上的遍历策略
- 需要类型转换（type→type）
- 需要函数式的替换逻辑（func→func）

**何时使用 `simultaneous=True`**：
- 替换之间有相互依赖
- 需要"并行"替换的语义
- 避免 0/0 这类不确定结果的顺序依赖

---

## 附录：关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| subs 入口 | `sympy/core/basic.py` | 971-1180 |
| _subs 核心 | `sympy/core/basic.py` | 1182-1292 |
| xreplace | `sympy/core/basic.py` | 1305-1390 |
| replace | `sympy/core/basic.py` | 1548-1802 |
| match/matches | `sympy/core/basic.py` | 1818-1924 |
| Add._eval_subs | `sympy/core/add.py` | 898-933 |
| Mul._eval_subs | `sympy/core/mul.py` | 1709-1850 |
| Wild 类 | `sympy/core/symbol.py` | 545-673 |
| cacheit | `sympy/core/cache.py` | 1-167 |
| _nodes (复杂度) | `sympy/core/sorting.py` | - |
| default_sort_key | `sympy/core/sorting.py` | - |
| 测试用例 | `sympy/core/tests/test_subs.py` | 全文件 |
| 匹配测试 | `sympy/core/tests/test_match.py` | 全文件 |

---

*报告生成时间: 2026-04-29*
*基于 SymPy 版本: 本地代码库 (sympy-10062)*
