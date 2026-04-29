# SymPy 假设系统（Assumptions）深度分析

## 1. 系统概述

SymPy 采用**双轨假设系统**设计，同时维护两套假设机制：

- **旧式假设系统**（Core Assumptions）：位于 `sympy/core/assumptions.py` 和 `sympy/core/facts.py`
- **新式假设系统**（New Assumptions）：位于 `sympy/assumptions/` 目录

这两套系统并行工作，各自有不同的设计目标和使用场景。

---

## 2. 符号属性查询机制与推理引擎协同工作

### 2.1 旧式假设的查询流程

旧式假设系统通过**属性访问**方式查询符号属性，如 `x.is_real`、`x.is_positive` 等。这个过程并非简单的字典查找，而是涉及复杂的推理链条。

#### 核心查询函数：`_ask(fact, obj)`

文件位置：`sympy/core/assumptions.py:518-620`

**工作原理：**

1. **缓存优先检查**：首先检查 `obj._assumptions` 字典中是否已有该事实的缓存值
2. **处理器调用**：如果存在 `_eval_is_<fact>` 处理器方法，调用该方法进行计算
3. **事实演绎**：如果获得新的事实值，通过 `deduce_all_facts` 进行前向推理
4. **前提扩展**：如果当前事实无法确定，根据 `prereq` 表扩展查询相关的前提事实
5. **结果缓存**：最终结果存入 `_assumptions` 字典

**代码分析：**

```python
def _ask(fact, obj):
    assumptions = obj._assumptions  # FactKB 实例
    handler_map = obj._prop_handler  # _eval_is_* 方法映射
    
    facts_to_check = [fact]
    facts_queued = {fact}
    
    for fact_i in facts_to_check:
        if fact_i in assumptions:
            continue  # 已有缓存，跳过
        
        # 1. 调用处理器方法
        fact_i_value = None
        handler_i = handler_map.get(fact_i)
        if handler_i is not None:
            fact_i_value = handler_i(obj)
        
        # 2. 如果获得新值，进行演绎推理
        if fact_i_value is not None:
            assumptions.deduce_all_facts(((fact_i, fact_i_value),))
        
        # 3. 检查目标事实是否已被推断出
        fact_value = assumptions.get(fact)
        if fact_value is not None:
            return fact_value
        
        # 4. 扩展查询前提事实
        new_facts_to_check = list(_assume_rules.prereq[fact_i] - facts_queued)
        shuffle(new_facts_to_check)  # 随机化顺序以检测非确定性
        facts_to_check.extend(new_facts_to_check)
        facts_queued.update(new_facts_to_check)
    
    # 5. 无法确定，缓存 None
    assumptions._tell(fact, None)
    return None
```

#### 推理引擎：`deduce_all_facts`

文件位置：`sympy/core/facts.py:600-635`

这是推理系统的核心工作horse，实现了**前向链推理**：

**推理过程：**

1. **Alpha 规则**（单条件推理）：如 `integer -> rational`，直接通过查表实现
2. **Beta 规则**（多条件推理）：如 `integer & !odd -> even`，需要检查所有条件是否满足

**代码分析：**

```python
def deduce_all_facts(self, facts):
    full_implications = self.rules.full_implications
    beta_triggers = self.rules.beta_triggers
    beta_rules = self.rules.beta_rules
    
    while facts:
        beta_maytrigger = set()
        
        # Alpha 链：单条件推理
        for k, v in facts:
            if not self._tell(k, v) or v is None:
                continue
            
            # 查表获取所有直接蕴含的事实
            for key, value in full_implications[k, v]:
                self._tell(key, value)
            
            # 记录可能触发的 Beta 规则
            beta_maytrigger.update(beta_triggers[k, v])
        
        # Beta 链：多条件推理
        facts = []
        for bidx in beta_maytrigger:
            bcond, bimpl = beta_rules[bidx]
            # 检查所有条件是否满足
            if all(self.get(k) is v for k, v in bcond):
                facts.append(bimpl)  # 结论作为新事实加入
```

### 2.2 三值逻辑系统

假设系统采用**三值逻辑**：`True`、`False`、`None`（未知）。这对于符号计算至关重要，因为很多属性在缺乏足够信息时无法确定。

**示例：**
```python
from sympy import Symbol
x = Symbol('x')
x.is_positive  # None - 无法确定
y = Symbol('y', positive=True)
y.is_positive  # True - 明确为正
y.is_negative  # False - 由 positive 蕴含
```

---

## 3. 旧式假设与新式假设的设计动机与接口差异

### 3.1 设计动机对比

| 特性 | 旧式假设 | 新式假设 |
|------|---------|---------|
| **设计目标** | 表达式类型系统的一部分，属性静态绑定 | 灵活的逻辑推理系统，支持动态假设上下文 |
| **绑定方式** | 符号构造时固定，不可更改 | 通过上下文动态添加/移除 |
| **推理方式** | 规则编译 + 前向链推理 | SAT 求解器 + 多分发处理器 |
| **适用场景** | 日常符号计算，性能优先 | 复杂逻辑推理，需要条件假设 |

### 3.2 接口差异

#### 旧式假设接口

**符号构造时传入：**
```python
from sympy import Symbol

# 构造时绑定假设
x = Symbol('x', real=True, positive=True)

# 属性访问查询
print(x.is_real)        # True
print(x.is_positive)    # True
print(x.is_integer)     # None - 无法确定

# 完整假设集合
print(x.assumptions0)
# {'commutative': True, 'complex': True, 'extended_negative': False, ...}
```

**关键数据结构：**
- `_assumptions_orig`：用户传入的原始假设（紧凑存储）
- `_assumptions0`：演绎后的完整假设集合（用于符号比较）
- `_assumptions`：运行时查询缓存（`StdFactKB` 实例）

#### 新式假设接口

**谓词系统 + `ask()` 函数：**
```python
from sympy import ask, Q, assuming
from sympy.abc import x, y

# 基本查询
print(ask(Q.integer(3)))                    # True
print(ask(Q.rational(x)))                   # None

# 带局部假设的查询
print(ask(Q.even(x*y), Q.even(x) & Q.integer(y)))  # True

# 全局假设上下文
with assuming(Q.real(x), Q.positive(y)):
    print(ask(Q.real(x + y)))              # True
    print(ask(Q.positive(x)))               # None - 局部只知道 y 是正的
```

**关键组件：**
- `Q`（`AssumptionKeys` 实例）：谓词访问入口
- `Predicate`：谓词基类，支持多分发处理器
- `AppliedPredicate`：谓词应用于参数后的表达式
- `ask()`：核心查询函数
- `global_assumptions`：全局假设上下文
- `assuming()`：上下文管理器，用于临时假设

### 3.3 新式假设的查询流程

文件位置：`sympy/assumptions/ask.py:405-554`

**查询策略（按优先级）：**

1. **快速检查**：`_ask_single_fact` - 检查是否有直接的单位子句蕴含
2. **直接解析**：`_eval_ask` - 调用谓词的处理器方法
3. **SAT 求解**：`satask` - 使用可满足性求解器进行逻辑推理
4. **线性实数算术**：`lra_satask` - 针对线性实数约束的专用求解器

**代码分析：**

```python
def ask(proposition, assumptions=True, context=global_assumptions):
    # 1. 转换为 CNF 范式
    assump_cnf = CNF.from_prop(assumptions)
    assump_cnf.extend(context)
    
    # 2. 提取相关事实
    local_facts = _extract_all_facts(assump_cnf, args)
    
    # 3. 加入已知事实公理
    known_facts_cnf = get_all_known_facts()
    enc_cnf = EncodedCNF()
    enc_cnf.from_cnf(CNF(known_facts_cnf))
    enc_cnf.add_from_cnf(local_facts)
    
    # 4. 检查一致性
    if local_facts.clauses and satisfiable(enc_cnf) is False:
        raise ValueError(f"inconsistent assumptions {assumptions}")
    
    # 5. 快速检查
    res = _ask_single_fact(key, local_facts)
    if res is not None:
        return res
    
    # 6. 直接解析
    res = key(*args)._eval_ask(assumptions)
    if res is not None:
        return bool(res)
    
    # 7. SAT 求解
    res = satask(proposition, assumptions=assumptions, context=context)
    if res is not None:
        return res
    
    # 8. 线性实数算术求解
    try:
        res = lra_satask(proposition, assumptions=assumptions, context=context)
    except UnhandledInput:
        return None
    
    return res
```

### 3.4 SAT 求解方法：`satask`

文件位置：`sympy/assumptions/satask.py:18-107`

**核心思想：** 通过检查命题及其否定的可满足性来确定真值：

- 如果 `P ∧ 假设` 不可满足 → `P` 为 `False`
- 如果 `¬P ∧ 假设` 不可满足 → `P` 为 `True`
- 如果两者都可满足 → 无法确定（返回 `None`）
- 如果两者都不可满足 → 假设矛盾（抛出 `ValueError`）

**代码分析：**

```python
def check_satisfiability(prop, _prop, factbase):
    sat_true = factbase.copy()
    sat_false = factbase.copy()
    
    sat_true.add_from_cnf(prop)    # 假设命题为真
    sat_false.add_from_cnf(_prop)  # 假设命题为假
    
    can_be_true = satisfiable(sat_true)
    can_be_false = satisfiable(sat_false)
    
    if can_be_true and can_be_false:
        return None      # 无法确定
    
    if can_be_true and not can_be_false:
        return True      # 必为真
    
    if not can_be_true and can_be_false:
        return False     # 必为假
    
    if not can_be_true and not can_be_false:
        raise ValueError("Inconsistent assumptions")  # 矛盾
```

---

## 4. 假设事实库的构造过程与冲突解决策略

### 4.1 规则编译过程

文件位置：`sympy/core/facts.py:381-469`

**规则定义语法：**

```python
# 蕴含规则
'integer -> rational'           # integer=True → rational=True
'zero -> even & finite'         # zero=True → even=True, finite=True

# 等价规则
'positive == nonnegative & nonzero'  # 双向蕴含

# 否定规则
'imaginary -> !extended_real'         # imaginary=True → extended_real=False

# Beta 规则（多条件）
'odd == integer & !even'        # integer=True 且 even=False → odd=True
```

**编译阶段：**

1. **解析规则**：将字符串规则转换为逻辑表达式
2. **证明器处理**：`Prover` 类分析规则间的蕴含关系
3. **Alpha 规则提取**：单条件蕴含，构建 `full_implications` 表
4. **Beta 规则提取**：多条件蕴含，构建 `beta_rules` 表
5. **前提表构建**：`prereq` 表记录每个事实可能由哪些事实推导得出

### 4.2 标准事实库：`StdFactKB`

文件位置：`sympy/core/assumptions.py:473-495`

```python
class StdFactKB(FactKB):
    def __init__(self, facts=None):
        super().__init__(_assume_rules)  # 使用预编译的规则
        
        if not facts:
            self._generator = {}
        elif not isinstance(facts, FactKB):
            self._generator = facts.copy()
        else:
            self._generator = facts.generator
        
        if facts:
            self.deduce_all_facts(facts)  # 初始演绎
```

### 4.3 冲突解决策略

文件位置：`sympy/core/facts.py:583-595`

**`_tell` 方法的冲突检测：**

```python
def _tell(self, k, v):
    """Add fact k=v to the knowledge base."""
    if k in self and self[k] is not None:
        if self[k] == v:
            return False  # 已有相同值，无更新
        else:
            # 冲突！
            raise InconsistentAssumptions(self, k, v)
    else:
        self[k] = v
        return True
```

**冲突示例：**

```python
from sympy import Symbol

# 旧式假设：构造时检测
try:
    x = Symbol('x', even=True, odd=True)
except ValueError as e:
    print(f"旧式假设冲突: {e}")

# 新式假设：查询时检测
from sympy import ask, Q
try:
    ask(Q.integer(x), Q.even(x) & Q.odd(x))
except ValueError as e:
    print(f"新式假设冲突: {e}")
```

**注意事项：**
- 旧式假设在**符号构造时**立即检测冲突
- 新式假设在**实际查询时**才检测一致性

---

## 5. 假设在表达式树重组时的传播与合并

### 5.1 处理器方法机制

每个表达式类通过实现 `_eval_is_<property>` 方法来定义该运算如何推断假设属性。

**处理器注册：**

文件位置：`sympy/core/assumptions.py:623-677`

```python
def _prepare_class_assumptions(cls):
    # 收集类中定义的 _eval_is_* 方法
    cls._prop_handler = {}
    for k in _assume_defined:
        eval_is_meth = getattr(cls, '_eval_is_%s' % k, None)
        if eval_is_meth is not None:
            cls._prop_handler[k] = eval_is_meth
```

### 5.2 乘法节点（Mul）的假设传播

文件位置：`sympy/core/mul.py:1297-1539`

**示例：`_eval_is_zero` 的实现**

```python
def _eval_is_zero_infinite_helper(self):
    """
    零和无穷的三值逻辑推理：
    - Mul(0, 有限) → 零
    - Mul(∞, 非零) → 无穷
    - Mul(0, ∞) → 未定式（返回 None）
    """
    seen_zero = seen_infinite = False
    
    for a in self.args:
        if a.is_zero:
            if seen_infinite is not False:
                return None, None  # 0 * ∞ = 未定式
            seen_zero = True
        elif a.is_infinite:
            if seen_zero is not False:
                return None, None  # ∞ * 0 = 未定式
            seen_infinite = True
        else:
            # 处理 None（未知）情况
            if seen_zero is False and a.is_zero is None:
                if seen_infinite is not False:
                    return None, None
                seen_zero = None
            if seen_infinite is False and a.is_infinite is None:
                if seen_zero is not False:
                    return None, None
                seen_infinite = None
    
    return seen_zero, seen_infinite
```

**更多乘法属性推断：**

| 属性 | 推断逻辑 |
|------|---------|
| `is_integer` | 所有因子都是整数 → 是整数；如有一个是无理数 → 非整数 |
| `is_rational` | 所有因子都是有理数且非零 → 是有理数 |
| `is_positive` | 负因子个数为偶数且无零 → 正数 |
| `is_commutative` | 所有因子可交换 → 可交换 |

### 5.3 加法节点（Add）的假设传播

文件位置：`sympy/core/add.py:646-888`

**示例：`_eval_is_extended_positive` 的实现**

```python
def _eval_is_extended_positive(self):
    # 策略：分析各项的符号
    pos = nonneg = nonpos = unknown_sign = False
    saw_INF = set()
    
    args = [a for a in self.args if not a.is_zero]
    if not args:
        return False  # 0 不是正数
    
    for a in args:
        ispos = a.is_extended_positive
        infinite = a.is_infinite
        
        # 处理无穷项
        if infinite:
            saw_INF.add(fuzzy_or((ispos, a.is_extended_nonnegative)))
            if True in saw_INF and False in saw_INF:
                return  # ∞ + (-∞) = 未定式
        
        # 符号分类
        if ispos:
            pos = True
        elif a.is_extended_nonnegative:
            nonneg = True
        elif a.is_extended_nonpositive:
            nonpos = True
        else:
            if infinite is None:
                return
            unknown_sign = True
    
    # 综合判断
    if saw_INF:
        if len(saw_INF) > 1:
            return
        return saw_INF.pop()
    elif unknown_sign:
        return
    elif pos and not nonpos:
        return True  # 有正数且无非正数
    elif not pos and not nonneg:
        return False  # 既无正数也无非负 → 全负
```

### 5.4 模糊逻辑辅助函数

文件位置：`sympy/core/logic.py`

**三值逻辑操作：**

```python
def fuzzy_and(args):
    """三值逻辑与运算"""
    rv = True
    for arg in args:
        v = arg if callable(arg) else arg
        if v is False:
            return False
        elif v is None:
            rv = None
    return rv

def fuzzy_or(args):
    """三值逻辑或运算"""
    rv = False
    for arg in args:
        v = arg if callable(arg) else arg
        if v is True:
            return True
        elif v is None:
            rv = None
    return rv

def fuzzy_not(arg):
    """三值逻辑非运算"""
    if arg is True:
        return False
    elif arg is False:
        return True
    else:
        return None

def _fuzzy_group(args, quick_exit=False):
    """
    用于判断所有参数是否具有相同属性
    - 全 True → True
    - 全 False → False
    - 混合 → None（除非 quick_exit）
    """
    result = None
    for a in args:
        if a is None:
            if quick_exit:
                return None
            continue
        if result is None:
            result = a
        elif a != result:
            if quick_exit:
                return None
            result = None
    return result
```

### 5.5 新式假设中的运算处理器

文件位置：`sympy/assumptions/handlers/sets.py`

**使用多分发机制：**

```python
@IntegerPredicate.register(Add)
def _(expr, assumptions):
    """
    * Integer + Integer       -> Integer
    * Integer + !Integer      -> !Integer
    * !Integer + !Integer -> ?
    """
    if expr.is_number:
        return _IntegerPredicate_number(expr, assumptions)
    return test_closed_group(expr, assumptions, Q.integer)

@IntegerPredicate.register(Mul)
def _(expr, assumptions):
    """
    * Integer*Integer      -> Integer
    * Integer*Irrational   -> !Integer
    * Odd/Even             -> !Integer
    * Integer*Rational     -> ?
    """
    if expr.is_number:
        return _IntegerPredicate_number(expr, assumptions)
    _output = True
    for arg in expr.args:
        if not ask(Q.integer(arg), assumptions):
            if arg.is_Rational:
                if arg.q == 2:
                    return ask(Q.even(2*expr), assumptions)
                if ~(arg.q & 1):
                    return None
            elif ask(Q.irrational(arg), assumptions):
                if _output:
                    _output = False
                else:
                    return
            else:
                return
    return _output
```

**闭包测试函数：**

```python
def test_closed_group(expr, assumptions, predicate):
    """
    测试运算是否在谓词下封闭：
    - 所有参数满足 P → 结果满足 P
    - 恰好一个参数不满足 P → 结果不满足 P
    - 其他情况 → 无法确定
    """
    result = True
    count = 0
    for arg in expr.args:
        if ask(predicate(arg), assumptions):
            continue
        elif ask(~predicate(arg), assumptions):
            result = False
            count += 1
        else:
            return None
    return result if count <= 1 else None
```

---

## 6. 查询缓存的维护与失效时机

### 6.1 旧式假设的实例级缓存

**缓存结构：**

每个 `Basic` 实例维护自己的 `_assumptions` 字典（`StdFactKB` 实例）。

文件位置：`sympy/core/assumptions.py:503-515`

```python
def make_property(fact):
    """创建自动属性"""
    def getit(self):
        try:
            return self._assumptions[fact]
        except KeyError:
            # 首次访问，复制默认假设
            if self._assumptions is self.default_assumptions:
                self._assumptions = self.default_assumptions.copy()
            return _ask(fact, self)
    
    getit.func_name = as_property(fact)
    return property(getit)
```

**缓存生命周期：**

| 阶段 | 缓存状态 |
|------|---------|
| 实例创建 | `_assumptions` 指向类的 `default_assumptions`（共享只读） |
| 首次查询 | 复制 `default_assumptions`，开始独立缓存 |
| 查询过程 | 新事实通过 `deduce_all_facts` 自动加入 |
| 实例销毁 | 缓存随实例一起被回收 |

### 6.2 类级默认假设

文件位置：`sympy/core/assumptions.py:623-677`

**准备过程：**

```python
def _prepare_class_assumptions(cls):
    # 1. 收集类定义的显式假设
    local_defs = {}
    for k in _assume_defined:
        attrname = as_property(k)
        v = cls.__dict__.get(attrname, '')
        if isinstance(v, (bool, int, type(None))):
            local_defs[k] = v
    
    # 2. 继承基类假设
    defs = {}
    for base in reversed(cls.__bases__):
        assumptions = getattr(base, '_explicit_class_assumptions', None)
        if assumptions is not None:
            defs.update(assumptions)
    defs.update(local_defs)
    
    # 3. 创建类级默认 FactKB
    cls._explicit_class_assumptions = defs
    cls.default_assumptions = StdFactKB(defs)
    
    # 4. 收集处理器方法
    cls._prop_handler = {}
    for k in _assume_defined:
        eval_is_meth = getattr(cls, '_eval_is_%s' % k, None)
        if eval_is_meth is not None:
            cls._prop_handler[k] = eval_is_meth
```

### 6.3 全局函数缓存

文件位置：`sympy/core/cache.py`

**`@cacheit` 装饰器：**

```python
def __cacheit(maxsize):
    def func_wrapper(func):
        cfunc = lru_cache(maxsize, typed=True)(func)
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            try:
                retval = cfunc(*args, **kwargs)
            except TypeError as e:
                # 不可哈希参数时跳过缓存
                if not e.args or not e.args[0].startswith('unhashable type:'):
                    raise
                retval = func(*args, **kwargs)
            return retval
        
        wrapper.cache_info = cfunc.cache_info
        wrapper.cache_clear = cfunc.cache_clear
        CACHE.append(wrapper)  # 注册到全局缓存列表
        return wrapper
    
    return func_wrapper
```

**缓存控制：**

```python
# 环境变量配置
USE_CACHE = _getenv('SYMPY_USE_CACHE', 'yes').lower()
# 'yes' - 启用缓存
# 'no' - 禁用缓存
# 'debug' - 启用缓存并验证一致性

SYMPY_CACHE_SIZE = int(_getenv('SYMPY_CACHE_SIZE', '1000'))
# 0 - 无缓存
# None - 无界缓存
```

### 6.4 缓存失效机制

**旧式假设缓存：**
- **无主动失效**：缓存与实例生命周期绑定
- **隐含失效**：当 `_eval_is_*` 方法返回新值时，会触发重新演绎

**全局函数缓存：**

```python
# 手动清除
from sympy.core.cache import clear_cache
clear_cache()  # 清除所有 @cacheit 装饰的函数缓存

# 查看缓存状态
from sympy.core.cache import print_cache
print_cache()
```

**Symbol 构造缓存：**

文件位置：`sympy/core/symbol.py:357-375`

```python
@staticmethod
@cacheit
def _canonical_assumptions(**assumptions):
    """假设集合的规范缓存"""
    assumptions_orig = assumptions.copy()
    assumptions.setdefault('commutative', True)
    assumptions_kb = StdFactKB(assumptions)
    assumptions0 = dict(assumptions_kb)
    return assumptions_kb, assumptions_orig, assumptions0

@staticmethod
@cacheit
def __xnew_cached_(cls, name, **assumptions):
    """符号实例缓存（按名称和假设）"""
    return Symbol.__xnew__(cls, name, **assumptions)
```

**设计意图：**
- 相同名称和假设的 `Symbol` 是同一个对象（用于结构比较）
- 这意味着：`Symbol('x', real=True) == Symbol('x', real=True)` 但 `is` 也为 `True`
- 缓存失效：只能通过 `clear_cache()` 全局清除

### 6.5 新式假设的缓存策略

新式假设系统的缓存相对简单：

1. **谓词单例化**：`@memoize_property` 确保每个谓词只有一个实例
2. **SAT 求解**：无持久化缓存，每次查询重新计算
3. **处理器结果**：依赖处理器自身的实现（可能有内部缓存）

**谓词单例化：**

文件位置：`sympy/assumptions/ask.py:30-286`

```python
class AssumptionKeys:
    @memoize_property
    def real(self):
        from .handlers.sets import RealPredicate
        return RealPredicate()  # 每次访问返回同一实例
    
    @memoize_property
    def integer(self):
        from .handlers.sets import IntegerPredicate
        return IntegerPredicate()
```

---

## 7. 关键模块与文件索引

### 7.1 旧式假设系统

| 文件 | 主要内容 |
|------|---------|
| `sympy/core/assumptions.py` | 核心查询逻辑、属性生成、`StdFactKB` |
| `sympy/core/facts.py` | 规则编译、推理引擎、`FactKB`/`FactRules` |
| `sympy/core/symbol.py` | 符号类定义、假设构造与缓存 |
| `sympy/core/basic.py` | 基类假设初始化、`__init_subclass__` 钩子 |
| `sympy/core/assumptions_generated.py` | 预编译的规则数据 |

### 7.2 新式假设系统

| 文件 | 主要内容 |
|------|---------|
| `sympy/assumptions/ask.py` | `ask()` 主函数、`Q` 谓词入口 |
| `sympy/assumptions/assume.py` | `Predicate`/`AppliedPredicate` 定义、上下文管理 |
| `sympy/assumptions/satask.py` | SAT 求解器集成 |
| `sympy/assumptions/lra_satask.py` | 线性实数算术求解 |
| `sympy/assumptions/handlers/*.py` | 各类运算的处理器实现 |
| `sympy/assumptions/ask_generated.py` | 预生成的事实公理 |

### 7.3 表达式运算中的假设传播

| 文件 | 主要内容 |
|------|---------|
| `sympy/core/mul.py` | 乘法节点的 `_eval_is_*` 方法 |
| `sympy/core/add.py` | 加法节点的 `_eval_is_*` 方法 |
| `sympy/core/power.py` | 幂运算节点的 `_eval_is_*` 方法 |
| `sympy/core/expr.py` | 通用表达式的 `_eval_is_*` 方法 |
| `sympy/core/function.py` | 函数应用的假设传播 |

---

## 8. 设计模式与架构洞察

### 8.1 双轨设计的权衡

**为什么需要两套系统？**

1. **性能考量**：旧式假设是属性访问，开销极低；新式假设涉及 SAT 求解，开销大
2. **使用场景**：
   - 日常计算：`x.is_real` 快速检查类型
   - 定理证明：`ask(Q.prime(x), Q.integer(x) & x > 1)` 复杂推理
3. **历史原因**：旧式假设是早期设计，新式假设是后来的增强

**两套系统的交互：**

```python
from sympy import Symbol, ask, Q
x = Symbol('x', positive=True)

# 旧式假设可用于新式假设的处理器
print(ask(Q.real(x)))  # True - 处理器会检查 x.is_real

# 但反之不然：旧式假设不访问全局假设上下文
from sympy.assumptions import assuming
y = Symbol('y')
with assuming(Q.real(y)):
    print(y.is_real)  # None - 旧式假设不知道上下文
    print(ask(Q.real(y)))  # True - 新式假设知道
```

### 8.2 关键设计模式

**1. 模板方法模式**
- `_ask()` 定义查询骨架
- `_eval_is_*` 由子类实现具体计算

**2. 享元模式**
- `Symbol` 按 `(name, assumptions0)` 缓存
- 相同假设的符号是同一对象

**3. 解释器模式**
- `FactRules` 编译规则为可执行的查找表
- `deduce_all_facts` 解释执行规则

**4. 策略模式**
- `ask()` 尝试多种求解策略（快速检查 → 直接解析 → SAT → LRA）
- 每种策略是可替换的算法

**5. 多分发模式**
- 新式假设的处理器使用 `multipledispatch`
- 按参数类型选择不同的处理逻辑

### 8.3 性能优化技巧

**1. 惰性计算**
- 假设只在访问时计算
- `_assumptions` 初始时指向共享的 `default_assumptions`

**2. 前向推理**
- 一次查询可能推导出多个相关事实
- 全部存入缓存，避免重复计算

**3. 前提引导**
- `prereq` 表指导查询方向
- 不盲目枚举所有可能的前提

**4. 规则预编译**
- 规则在模块加载时编译为查找表
- 运行时只需查表，无需解析

---

## 9. 实际使用建议

### 9.1 何时使用哪套系统？

**使用旧式假设的场景：**
```python
# 1. 符号定义时已知属性
x = Symbol('x', real=True, positive=True)

# 2. 快速类型检查
if expr.is_real:
    # 实数特化处理
    pass

# 3. 表达式构造和简化（自动利用假设）
from sympy import sqrt
print(sqrt(x**2))  # x （因为 x 是正的）
```

**使用新式假设的场景：**
```python
# 1. 条件性推理
if ask(Q.prime(n), Q.integer(n) & Q.gt(n, 1)):
    # 基于假设的推理
    pass

# 2. 临时假设上下文
with assuming(Q.positive(x), Q.real(y)):
    result = ask(Q.positive(x + y))

# 3. 复杂逻辑组合
from sympy import And, Or, Implies
theorem = Implies(Q.even(x) & Q.even(y), Q.even(x + y))
print(ask(theorem))  # True
```

### 9.2 常见陷阱

**陷阱 1：混淆两套系统**
```python
x = Symbol('x')
with assuming(Q.real(x)):
    print(x.is_real)  # None！旧式假设不知道上下文
    # 正确做法：
    print(ask(Q.real(x)))  # True
```

**陷阱 2：假设矛盾不总是立即报错**
```python
# 旧式假设：构造时报错
try:
    x = Symbol('x', even=True, odd=True)
except ValueError:
    print("旧式：立即报错")

# 新式假设：查询时才检测
from sympy import ask, Q
from sympy.abc import x
# 这行不会报错：
assumptions = Q.even(x) & Q.odd(x)
# 只有查询时才报错：
try:
    ask(Q.integer(x), assumptions)
except ValueError:
    print("新式：查询时报错")
```

**陷阱 3：缓存导致的意外行为**
```python
from sympy import Symbol, clear_cache

# 创建符号
x1 = Symbol('x', real=True)
print(x1.is_real)  # True

# 清除缓存
clear_cache()

# 同名但不同假设的符号现在是不同对象
x2 = Symbol('x', real=False)
print(x1 == x2)  # False
print(x1.is_real)  # True（x1 的缓存不受影响）
print(x2.is_real)  # False
```

---

## 10. 总结

SymPy 的假设系统是一个精心设计的**双轨推理架构**：

1. **旧式假设**作为表达式类型系统的一部分，提供**快速属性访问**
   - 基于规则编译和前向链推理
   - 三值逻辑处理不确定性
   - 实例级缓存最小化重复计算

2. **新式假设**作为灵活的**逻辑推理引擎**，支持条件假设
   - 基于 SAT 求解器和多分发处理器
   - 动态上下文管理（`assuming`）
   - 更强大但计算开销更大

3. **假设传播**通过 `_eval_is_*` 方法和多分发处理器实现
   - 加法、乘法等基本运算都有精细的推断逻辑
   - 模糊逻辑函数处理三值逻辑组合

4. **缓存策略**分层设计
   - 实例级缓存：随对象生命周期
   - 类级缓存：共享默认假设
   - 全局函数缓存：可手动清除

这套设计在**性能**和**表达能力**之间取得了良好的平衡，既支持日常符号计算的高效类型检查，也为复杂的数学定理证明提供了逻辑推理基础。
