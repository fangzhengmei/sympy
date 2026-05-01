# SymPy 打印子系统调用链深度分析

> 分析日期：2026-05-01
> 目标：对比同一表达式在三条打印路径下的分发、回退和最终产出差异

---

## 目录

1. [示例表达式设计](#1-示例表达式设计)
2. [StrPrinter 路径完整调用链](#2-strprinter-路径完整调用链)
3. [MathML Content 路径完整调用链](#3-mathml-content-路径完整调用链)
4. [MathML Presentation 路径完整调用链](#4-mathml-presentation-路径完整调用链)
5. [C99CodePrinter 路径完整调用链](#5-c99codeprinter-路径完整调用链)
6. [三条路径对比分析](#6-三条路径对比分析)
7. [共享配置与后端独有配置边界](#7-共享配置与后端独有配置边界)
8. [附录：关键代码位置](#8-附录关键代码位置)

---

## 1. 示例表达式设计

为了完整展示三条打印路径的差异，我们选择一个包含多种类型的复杂表达式：

```python
from sympy import symbols, sin, cos, cot, Integral, pi
x, y = symbols('x y')
expr = sin(x) + cos(x)**2 + Integral(cot(y), (y, 0, pi))
```

### 表达式结构解析

| 组件 | 类型 | 说明 |
|------|------|------|
| `sin(x)` | `sin` (Function 子类) | 三角函数 |
| `cos(x)` | `cos` (Function 子类) | 三角函数 |
| `cos(x)**2` | `Pow` | 幂运算 |
| `cot(y)` | `cot` (Function 子类) | 余切函数（代码生成可能触发重写） |
| `(y, 0, pi)` | `Tuple` | 积分上下限 |
| `Integral(...)` | `Integral` | 积分（代码生成不支持） |
| `sin(x) + cos(x)**2 + ...` | `Add` | 加法（根节点） |

### 表达式类型 MRO 链预览

```
Add:      Add -> Expr -> Basic -> object
Pow:      Pow -> Expr -> Basic -> object
Integral: Integral -> Expr -> Basic -> object
sin:      sin -> Function -> Application -> Basic -> object
Symbol:   Symbol -> Atom -> Expr -> Basic -> object
Integer:  Integer -> Rational -> Number -> Atom -> Expr -> Basic -> object
```

---

## 2. StrPrinter 路径完整调用链

### 2.1 打印器初始化

```python
from sympy.printing.str import StrPrinter
sp = StrPrinter()
result = sp.doprint(expr)
```

### 2.2 类层次结构

```
Printer (基类)
    │
    └── StrPrinter (当前使用)
            printmethod = "_sympystr"
            _default_settings = {
                "order": None,
                "full_prec": "auto",
                "sympy_integers": False,
                "abbrev": False,
                "perm_cyclic": True,
                "min": None,
                "max": None,
                "dps": None
            }
```

### 2.3 完整调用流程

#### Step 1: `doprint(expr)` 入口

**文件**: `printer.py:291-293`
```python
def doprint(self, expr):
    return self._str(self._print(expr))
```

调用 `_print(expr)` 开始分发。

---

#### Step 2: `_print(expr)` 三级分发

**文件**: `printer.py:295-336`

```python
def _print(self, expr, **kwargs) -> str:
    self._print_level += 1
    try:
        # 第一级：对象自打印
        if self.printmethod and hasattr(expr, self.printmethod):
            if not (isinstance(expr, type) and issubclass(expr, Basic)):
                return getattr(expr, self.printmethod)(self, **kwargs)
        
        # 第二级：沿 MRO 链查找 _print_<ClassName>
        classes = type(expr).__mro__
        # ... 特殊处理 AppliedUndef、UndefinedFunction、Function
        for cls in classes:
            printmethodname = '_print_' + cls.__name__
            printmethod = getattr(self, printmethodname, None)
            if printmethod is not None:
                return printmethod(expr, **kwargs)
        
        # 第三级：兜底
        return self.emptyPrinter(expr)
    finally:
        self._print_level -= 1
```

---

#### Step 3: 根节点 `Add` 的分发

表达式 `expr = sin(x) + cos(x)**2 + Integral(...)` 是 `Add` 类型。

**MRO 链**: `Add -> Expr -> Basic -> object`

**查找顺序**:
1. `_print_Add` → ✅ 找到（StrPrinter 有定义）
2. 不再继续向上查找

**文件**: `str.py:57-79`
```python
def _print_Add(self, expr, order=None):
    terms = self._as_ordered_terms(expr, order=order)
    # ... 处理符号和括号
    l = []
    for term in terms:
        t = self._print(term)  # 递归打印每个项
        # ... 处理加减号
    # ...
    return sign + ' '.join(l)
```

**关键行为**: 对每个项递归调用 `self._print(term)`

---

#### Step 4: 递归打印 `sin(x)`

`sin(x)` 是 `sin` 类型，继承自 `Function`。

**MRO 链**: `sin -> Function -> Application -> Basic -> object`

**查找顺序**:
1. `_print_sin` → ❌ 未找到
2. `_print_Function` → ✅ 找到（StrPrinter 有定义）

**文件**: `str.py:169-170`
```python
def _print_Function(self, expr):
    return expr.func.__name__ + "(%s)" % self.stringify(expr.args, ", ")
```

**输出**: `"sin(x)"`

---

#### Step 5: 递归打印 `cos(x)**2`

这是 `Pow` 类型：`base=cos(x), exp=2`

**MRO 链**: `Pow -> Expr -> Basic -> object`

**查找顺序**:
1. `_print_Pow` → ✅ 找到（StrPrinter 有定义）

递归处理 `base=cos(x)`:
- `cos(x)` 是 `cos` 类型
- `_print_cos` → ❌ 未找到
- `_print_Function` → ✅ 找到
- 输出: `"cos(x)"`

递归处理 `exp=2`:
- `2` 是 `Integer` 类型
- `_print_Integer` → 通常在基类或 `_print_Number` 处理

**最终输出**: `"cos(x)**2"`

---

#### Step 6: 递归打印 `Integral(cot(y), (y, 0, pi))`

这是 `Integral` 类型。

**MRO 链**: `Integral -> Expr -> Basic -> object`

**查找顺序**:
1. `_print_Integral` → ✅ 找到（StrPrinter 有定义）

**文件**: `str.py:189-196`
```python
def _print_Integral(self, expr):
    def _xab_tostr(xab):
        if len(xab) == 1:
            return self._print(xab[0])
        else:
            return self._print((xab[0],) + tuple(xab[1:]))
    L = ', '.join([_xab_tostr(l) for l in expr.limits])
    return 'Integral(%s, %s)' % (self._print(expr.function), L)
```

**递归处理**:
- `expr.function = cot(y)`: `_print_Function` → `"cot(y)"`
- `expr.limits = [(y, 0, pi)]`: 处理为 `"(y, 0, pi)"`

**输出**: `"Integral(cot(y), (y, 0, pi))"`

---

#### Step 7: 最终组装

`_print_Add` 将所有项拼接：

```
"sin(x) + cos(x)**2 + Integral(cot(y), (y, 0, pi))"
```

### 2.4 回退机制分析

**StrPrinter 的回退机制相对简单**，因为大多数类型都有专用处理器：

#### StrPrinter.emptyPrinter

**文件**: `str.py:49-55`
```python
def emptyPrinter(self, expr):
    if isinstance(expr, str):
        return expr
    elif isinstance(expr, Basic):
        return repr(expr)  # 对 Basic 类型调用 repr()
    else:
        return str(expr)
```

**回退触发场景**:
- 表达式类型不在 `_print_*` 方法覆盖范围内
- 沿 MRO 链向上也没有找到任何处理器

**示例**: 如果有一个自定义类 `MyExpr(Basic)` 没有专用处理器：
1. `_print_MyExpr` → ❌
2. `_print_Basic` → ✅ 找到！（StrPrinter 有 `_print_Basic`）

等等，StrPrinter 实际上有 `_print_Basic`：

**文件**: `str.py:108-110`
```python
def _print_Basic(self, expr):
    l = [self._print(o) for o in expr.args]
    return expr.__class__.__name__ + "(%s)" % ", ".join(l)
```

**分析**: 对于任何 `Basic` 子类，都会沿 MRO 链找到 `_print_Basic`，因此 `emptyPrinter` 在 StrPrinter 中**几乎不会被触发**，除非表达式不是 `Basic` 类型。

---

## 3. MathML Content 路径完整调用链

### 3.1 打印器初始化

```python
from sympy.printing.mathml import MathMLContentPrinter
mcp = MathMLContentPrinter()
result = mcp.doprint(expr)
```

### 3.2 类层次结构

```
Printer (基类)
    │
    └── MathMLPrinterBase (共享基类)
            │
            └── MathMLContentPrinter (语义表示)
                    printmethod = "_mathml_content"
                    _default_settings = MathMLPrinterBase._default_settings
```

### 3.3 关键特性：返回 DOM 节点，不是字符串

**这是 MathML 打印器与其他打印器最大的区别**。

#### doprint 覆盖

**文件**: `mathml.py:66-74`
```python
def doprint(self, expr):
    """
    Prints the expression as MathML.
    """
    mathML = Printer._print(self, expr)  # 返回 DOM 节点！
    unistr = mathML.toxml()              # DOM → XML 字符串
    xmlbstr = unistr.encode('ascii', 'xmlcharrefreplace')
    res = xmlbstr.decode()
    return res
```

**关键差异**:
1. `Printer._print()` 返回的不是字符串，而是 **DOM 元素对象**
2. `doprint` 调用 `toxml()` 将 DOM 节点序列化为 XML 字符串

---

### 3.4 完整调用流程

#### Step 1: 根节点 `Add` 的分发

**MRO 链**: `Add -> Expr -> Basic -> object`

**查找顺序**:
1. `_print_Add` → ✅ 找到

**文件**: `mathml.py:191-217`
```python
def _print_Add(self, expr, order=None):
    args = self._as_ordered_terms(expr, order=order)
    lastProcessed = self._print(args[0])  # 返回 DOM 节点
    plusNodes = []
    for arg in args[1:]:
        if arg.could_extract_minus_sign():
            # use minus
            x = self.dom.createElement('apply')
            x.appendChild(self.dom.createElement('minus'))
            x.appendChild(lastProcessed)
            x.appendChild(self._print(-arg))
            lastProcessed = x
        else:
            plusNodes.append(lastProcessed)
            lastProcessed = self._print(arg)
    # ...
    x = self.dom.createElement('apply')
    x.appendChild(self.dom.createElement('plus'))
    while plusNodes:
        x.appendChild(plusNodes.pop(0))
    return x  # 返回 DOM 节点！
```

**输出类型**: `xml.dom.minidom.Element` (DOM 节点)

---

#### Step 2: 递归打印 `sin(x)`

`sin(x)` 是 `sin` 类型。

**MRO 链**: `sin -> Function -> Application -> Basic -> object`

**查找顺序**:
1. `_print_sin` → ❌ 未找到
2. `_print_Function` → ❌ 可能没有
3. 沿 MRO 继续向上...

**关键**: MathMLContentPrinter 使用 `mathml_tag()` 方法来确定标签名。

**文件**: `mathml.py:90-154`
```python
def mathml_tag(self, e):
    """Returns the MathML tag for an expression."""
    translate = {
        'Add': 'plus',
        'Mul': 'times',
        'Number': 'cn',
        'Symbol': 'ci',
        'sin': 'sin',
        'cos': 'cos',
        'tan': 'tan',
        'cot': 'cot',
        # ...
    }
    for cls in e.__class__.__mro__:
        n = cls.__name__
        if n in translate:
            return translate[n]
    return n.lower()
```

对于 `sin`:
- MRO 链中 `'sin'` 在 `translate` 字典中
- 返回 `'sin'` 作为标签名

**Content MathML 输出**（语义标签）:
```xml
<apply>
    <sin/>
    <ci>x</ci>
</apply>
```

---

#### Step 3: 递归打印 `Integral(...)`

**MRO 链**: `Integral -> Expr -> Basic -> object`

**查找顺序**:
1. `_print_Integral` → ✅ 找到

**文件**: `mathml.py:321-348`
```python
def _print_Integral(self, e):
    def lime_recur(limits):
        x = self.dom.createElement('apply')
        x.appendChild(self.dom.createElement(self.mathml_tag(e)))
        bvar_elem = self.dom.createElement('bvar')
        bvar_elem.appendChild(self._print(limits[0][0]))
        x.appendChild(bvar_elem)
        
        if len(limits[0]) == 3:
            low_elem = self.dom.createElement('lowlimit')
            low_elem.appendChild(self._print(limits[0][1]))
            x.appendChild(low_elem)
            up_elem = self.dom.createElement('uplimit')
            up_elem.appendChild(self._print(limits[0][2]))
            x.appendChild(up_elem)
        # ...
        if len(limits) == 1:
            x.appendChild(self._print(e.function))
        else:
            x.appendChild(lime_recur(limits[1:]))
        return x
    
    limits = list(e.limits)
    limits.reverse()
    return lime_recur(limits)
```

**关键**：
- 调用 `self.mathml_tag(e)` 获取标签
- 对于 `Integral`，`mathml_tag()` 返回 `'int'`

**Content MathML 输出**:
```xml
<apply>
    <int/>
    <bvar><ci>y</ci></bvar>
    <lowlimit><cn>0</cn></lowlimit>
    <uplimit><pi/></uplimit>
    <apply>
        <cot/>
        <ci>y</ci>
    </apply>
</apply>
```

---

#### Step 4: 最终 XML 序列化

`doprint` 调用 `toxml()` 将 DOM 树序列化为 XML 字符串：

```xml
<apply>
    <plus/>
    <apply><sin/><ci>x</ci></apply>
    <apply><power/><apply><cos/><ci>x</ci></apply><cn>2</cn></apply>
    <apply><int/><bvar><ci>y</ci></bvar>...<apply><cot/><ci>y</ci></apply></apply>
</apply>
```

### 3.5 回退机制分析

**MathMLContentPrinter 的空打印机**继承自基类：

**文件**: `printer.py:338-339`
```python
def emptyPrinter(self, expr):
    return str(expr)
```

**但实际上**，MathML 打印器有更复杂的处理：

1. `_print_Basic` 可能不存在或行为不同
2. 如果某个类型没有 `_print_*` 方法，会沿 MRO 链查找
3. 如果都没有，`emptyPrinter` 返回 `str(expr)`，但这会导致问题因为 `doprint` 期望 DOM 节点

**潜在问题**：如果 `_print()` 返回字符串而不是 DOM 节点，`toxml()` 会报错。

---

## 4. MathML Presentation 路径完整调用链

### 4.1 类层次结构

```
Printer (基类)
    │
    └── MathMLPrinterBase (共享基类)
            │
            ├── MathMLContentPrinter (语义表示)
            │
            └── MathMLPresentationPrinter (视觉渲染)
                    printmethod = "_mathml_presentation"
```

### 4.2 Content vs Presentation 核心差异

| 维度 | Content MathML | Presentation MathML |
|------|----------------|---------------------|
| 用途 | 语义表示（定理证明、语义网） | 视觉渲染（网页、文档） |
| 数字标签 | `<cn>` (content number) | `<mn>` (math number) |
| 标识符标签 | `<ci>` (content identifier) | `<mi>` (math identifier) |
| 应用标签 | `<apply>` | `<mrow>` |
| 运算符标签 | `<plus>`, `<times>`, `<sin>` 等 | `<mo>` (math operator) |

### 4.3 调用流程差异

以 `sin(x) + cos(x)**2` 为例：

#### Content MathML 输出（语义）:
```xml
<apply>
    <plus/>
    <apply><sin/><ci>x</ci></apply>
    <apply><power/><apply><cos/><ci>x</ci></apply><cn>2</cn></apply>
</apply>
```

#### Presentation MathML 输出（视觉）:
```xml
<mrow>
    <mrow>
        <mi>sin</mi>
        <mo>&ApplyFunction;</mo>
        <mfenced><mi>x</mi></mfenced>
    </mrow>
    <mo>+</mo>
    <msup>
        <mrow>
            <mi>cos</mi>
            <mo>&ApplyFunction;</mo>
            <mfenced><mi>x</mi></mfenced>
        </mrow>
        <mn>2</mn>
    </msup>
</mrow>
```

### 4.4 关键差异原因

1. **标签体系不同**:
   - Content: 语义标签（`<plus>`, `<sin>` 等）
   - Presentation: 布局标签（`<mrow>`, `<mo>`, `<msup>` 等）

2. **目标不同**:
   - Content: 机器理解（可以被计算机代数系统解析）
   - Presentation: 人类阅读（控制字体大小、位置、样式）

3. **方法名分发相同，但实现不同**:
   - 都使用 `_print_<ClassName>` 分发机制
   - 但 `_print_Add`、`_print_Pow` 等方法的实现完全不同

---

## 5. C99CodePrinter 路径完整调用链

### 5.1 打印器初始化

```python
from sympy.printing.c import C99CodePrinter
cp = C99CodePrinter()
result = cp.doprint(expr)  # 可能抛出异常或触发回退
```

### 5.2 类层次结构

```
Printer (基类)
    │
    └── StrPrinter
            │
            └── CodePrinter (代码生成基类)
                    │
                    └── C89CodePrinter
                            │
                            └── C99CodePrinter (当前使用)
```

### 5.3 关键特性：显式不支持映射

**这是代码生成打印器最特殊的地方**：大量类型被显式标记为不支持。

**文件**: `codeprinter.py:621-643`
```python
# The following can not be simply translated into C or Fortran
_print_Basic = _print_not_supported
_print_ComplexInfinity = _print_not_supported
_print_ExprCondPair = _print_not_supported
_print_GeometryEntity = _print_not_supported
_print_Infinity = _print_not_supported
_print_Integral = _print_not_supported    # 重点关注！
_print_Interval = _print_not_supported
_print_AccumulationBounds = _print_not_supported
_print_Limit = _print_not_supported
_print_MatrixBase = _print_not_supported
_print_DeferredVector = _print_not_supported
_print_NaN = _print_not_supported
_print_NegativeInfinity = _print_not_supported
_print_Order = _print_not_supported
_print_RootOf = _print_not_supported
_print_RootsOf = _print_not_supported
_print_RootSum = _print_not_supported
_print_Uniform = _print_not_supported
_print_Unit = _print_not_supported
_print_Wild = _print_not_supported
_print_WildFunction = _print_not_supported
_print_Relational = _print_not_supported
```

**关键理解**：
- `_print_Integral = _print_not_supported` 是**显式映射**
- 打印 `Integral` 时，会找到 `_print_Integral` 方法
- **不会沿 MRO 链继续向上查找**
- 直接调用 `_print_not_supported`

---

### 5.4 `_print_not_supported` 实现

**文件**: `codeprinter.py:608-619`
```python
def _print_not_supported(self, expr):
    if self._settings.get('strict', False):
        raise PrintMethodNotImplementedError(
            f"Unsupported by {type(self)}: {type(expr)}" +
            "\nSet the printer option 'strict' to False in order to generate partially printed code."
        )
    try:
        self._not_supported.add(expr)
    except TypeError:
        # not hashable
        pass
    return self.emptyPrinter(expr)  # 非严格模式下调用 emptyPrinter
```

**两种模式**：
1. **严格模式** (`strict=True`): 抛出 `PrintMethodNotImplementedError` 异常
2. **宽松模式** (`strict=False`): 调用 `emptyPrinter(expr)` 兜底

---

### 5.5 完整调用流程

#### 示例表达式（不含 Integral）:

```python
simple_expr = sin(x) + cos(x)**2
```

#### Step 1: 根节点 `Add` 的分发

**MRO 链**: `Add -> Expr -> Basic -> object`

**查找顺序**:
1. `_print_Add` → ✅ 找到（CodePrinter 继承自 StrPrinter）

**但 CodePrinter 可能覆盖了 `_print_Add` 或使用父类实现**。

---

#### Step 2: 递归打印 `sin(x)`

**MRO 链**: `sin -> Function -> Application -> Basic -> object`

**查找顺序**:
1. `_print_sin` → ❌ 未找到
2. `_print_Function` → ✅ 找到（CodePrinter 有定义）

**文件**: `codeprinter.py:437-463`
```python
def _print_Function(self, expr):
    if expr.func.__name__ in self.known_functions:
        # 已知函数，直接映射
        cond_func = self.known_functions[expr.func.__name__]
        if isinstance(cond_func, str):
            return "%s(%s)" % (cond_func, self.stringify(expr.args, ", "))
        # ... 条件函数处理
    elif hasattr(expr, '_imp_') and isinstance(expr._imp_, Lambda):
        # inlined function
        return self._print(expr._imp_(*expr.args))
    elif expr.func.__name__ in self._rewriteable_functions:
        # 函数重写机制！
        target_f, required_fs = self._rewriteable_functions[expr.func.__name__]
        if self._can_print(target_f) and all(self._can_print(f) for f in required_fs):
            return '(' + self._print(expr.rewrite(target_f)) + ')'
    
    if expr.is_Function and self._settings.get('allow_unknown_functions', False):
        return '%s(%s)' % (self._print(expr.func), ', '.join(map(self._print, expr.args)))
    else:
        return self._print_not_supported(expr)  # 不支持的函数
```

**对于 `sin(x)`**:
- `'sin'` 在 `known_functions` 中
- C99 代码生成映射为 `sin`
- **输出**: `"sin(x)"`

---

#### Step 3: 函数重写机制 - `cot(y)` 示例

**文件**: `codeprinter.py:78-114`
```python
_rewriteable_functions = {
    'cot': ('tan', []),        # cot(x) → 1/tan(x)
    'csc': ('sin', []),         # csc(x) → 1/sin(x)
    'sec': ('cos', []),         # sec(x) → 1/cos(x)
    'acot': ('atan', []),
    'coth': ('exp', []),
    # ... 更多
}
```

**调用流程**（打印 `cot(y)`）:
1. `_print_cot` → ❌ 未找到
2. `_print_Function` → ✅ 找到
3. `'cot'` 不在 `known_functions` 中
4. `'cot'` 在 `_rewriteable_functions` 中
5. 检查 `'tan'` 是否可打印 ✅
6. 调用 `expr.rewrite('tan')` → `1/tan(y)`
7. 递归打印重写后的表达式
8. **输出**: `"(1/tan(y))"`

---

#### Step 4: 打印 `Integral(...)` - 显式不支持

**MRO 链**: `Integral -> Expr -> Basic -> object`

**查找顺序**:
1. `_print_Integral` → ✅ 找到！（但映射到 `_print_not_supported`）

**关键**：
- `_print_Integral = _print_not_supported` 是显式赋值
- **不会沿 MRO 链继续向上查找 `_print_Expr` 或 `_print_Basic`**
- 直接调用 `_print_not_supported`

**严格模式下**:
```
PrintMethodNotImplementedError: Unsupported by <class 'C99CodePrinter'>: <class 'sympy.integrals.integrals.Integral'>
Set the printer option 'strict' to False in order to generate partially printed code.
```

**宽松模式下**:
- 调用 `emptyPrinter(expr)`
- 继承自 StrPrinter
- 输出类似 `"Integral(cot(y), (y, 0, pi))"`（但这不是有效的 C 代码）

---

### 5.6 最终产出对比

| 表达式 | StrPrinter | C99CodePrinter (strict=False) |
|--------|------------|-------------------------------|
| `sin(x)` | `"sin(x)"` | `"sin(x)"` |
| `cos(x)**2` | `"cos(x)**2"` | `"pow(cos(x), 2)"` 或 `"(cos(x))*(cos(x))"` |
| `cot(y)` | `"cot(y)"` | `"(1/tan(y))"` (重写后) |
| `Integral(...)` | `"Integral(...)"` | 异常 或 `"Integral(...)"` (宽松) |

---

## 6. 三条路径对比分析

### 6.1 核心差异汇总表

| 维度 | StrPrinter | MathML Content | MathML Presentation | C99CodePrinter |
|------|------------|----------------|---------------------|----------------|
| **输出类型** | 字符串 | XML 字符串 (DOM→toxml) | XML 字符串 (DOM→toxml) | 字符串 (C 代码) |
| **目标** | 人类可读的 SymPy 表达式 | 机器可解析的语义表示 | 视觉渲染格式 | 可编译的 C 代码 |
| **printmethod** | `"_sympystr"` | `"_mathml_content"` | `"_mathml_presentation"` | 继承自 StrPrinter |
| **回退策略** | `_print_Basic` 兜底 | 可能出错（期望 DOM） | 可能出错（期望 DOM） | `_print_not_supported` 严格/宽松 |
| **函数重写** | 无 | 无 | 无 | 有（`_rewriteable_functions`） |

### 6.2 同一表达式在三条路径下的详细对比

**示例表达式**: `sin(x) + cos(x)**2`

#### StrPrinter 输出:
```
"sin(x) + cos(x)**2"
```

**分发过程**:
1. `Add` → `_print_Add` → 递归处理子项
2. `sin(x)` → `_print_Function` → `"sin(x)"`
3. `cos(x)` → `_print_Function` → `"cos(x)"`
4. `Pow(cos(x), 2)` → `_print_Pow` → `"cos(x)**2"`
5. 拼接 → `"sin(x) + cos(x)**2"`

#### MathML Content 输出:
```xml
<apply>
    <plus/>
    <apply><sin/><ci>x</ci></apply>
    <apply><power/><apply><cos/><ci>x</ci></apply><cn>2</cn></apply>
</apply>
```

**分发过程**:
1. `Add` → `_print_Add` → 创建 `<apply><plus/>...`
2. `sin(x)` → 沿 MRO 查找 → 使用 `mathml_tag()` 获取标签
3. 创建 `<apply><sin/><ci>x</ci></apply>`
4. `Pow` → `_print_Pow` → 创建 `<apply><power/>...`
5. 返回 DOM 节点 → `toxml()` 序列化

#### C99CodePrinter 输出 (strict=False):
```c
"sin(x) + pow(cos(x), 2)"
```

**分发过程**:
1. `Add` → 继承自 StrPrinter 的 `_print_Add`
2. `sin(x)` → `_print_Function` → `known_functions` 中有 → `"sin(x)"`
3. `cos(x)` → `_print_Function` → `known_functions` 中有 → `"cos(x)"`
4. `Pow(cos(x), 2)` → `_print_Pow` → 可能映射为 `pow()` 函数
5. 拼接 → `"sin(x) + pow(cos(x), 2)"`

### 6.3 回退触发场景对比

#### 场景 1: 自定义类型 `MyExpr(Basic)` 无专用处理器

**StrPrinter**:
1. `_print_MyExpr` → ❌
2. `_print_Basic` → ✅ 找到！
3. 输出: `"MyExpr(arg1, arg2, ...)"`
4. **不会触发 emptyPrinter**

**MathML**:
1. `_print_MyExpr` → ❌
2. 沿 MRO 向上...
3. 可能没有 `_print_Basic`
4. `emptyPrinter` → 返回 `str(expr)`（字符串）
5. **问题**: `doprint` 期望 DOM 节点，可能出错

**C99CodePrinter**:
1. `_print_MyExpr` → ❌
2. 沿 MRO 向上...
3. `_print_Basic = _print_not_supported` → ✅ 找到！
4. 调用 `_print_not_supported`
5. **严格模式**: 抛出异常
6. **宽松模式**: `emptyPrinter` → `"MyExpr(...)"`（不是有效 C 代码）

#### 场景 2: 积分 `Integral(f(x), x)`

**StrPrinter**:
1. `_print_Integral` → ✅ 找到
2. 输出: `"Integral(f(x), x)"`

**MathML Content**:
1. `_print_Integral` → ✅ 找到
2. 输出: `<apply><int/>...</apply>`

**C99CodePrinter**:
1. `_print_Integral = _print_not_supported` → ✅ 找到
2. **不会沿 MRO 向上查找**
3. 调用 `_print_not_supported`
4. 结果: 异常 或 `"Integral(...)"`（宽松）

### 6.4 关键差异原因深度分析

#### 原因 1: 设计目标不同

| 打印器 | 设计目标 | 影响 |
|--------|----------|------|
| **StrPrinter** | 显示 SymPy 表达式的字符串形式 | 覆盖所有 Basic 类型（`_print_Basic` 兜底） |
| **MathML** | 遵循 MathML 标准 | 必须返回 DOM 节点，语义/展示标签分离 |
| **CodePrinter** | 生成可编译的代码 | 必须严格检查类型，不支持的显式拒绝 |

#### 原因 2: 继承链导致的行为差异

```
Printer (emptyPrinter = str(expr))
    │
    ├── StrPrinter
    │       _print_Basic = 构造 "ClassName(args)"
    │       emptyPrinter = 对 Basic 用 repr()
    │
    ├── MathMLPrinterBase
    │       doprint = 调用 toxml()
    │       _print_* 返回 DOM 节点
    │
    └── CodePrinter (继承 StrPrinter)
            _print_Integral = _print_not_supported
            _print_Basic = _print_not_supported  (覆盖 StrPrinter!)
            _print_not_supported = 严格/宽松模式
```

**关键发现**: `CodePrinter` **覆盖**了 `_print_Basic`：
```python
# codeprinter.py:621
_print_Basic = _print_not_supported
```

这意味着：
- `StrPrinter._print_Basic` 提供兜底
- `CodePrinter._print_Basic` 显式标记为不支持
- 这是**两条路径最根本的差异之一**

#### 原因 3: 函数重写机制的必要性

**为什么 CodePrinter 需要 `_rewriteable_functions`？**

```python
# C 标准库
# - 有 sin(), cos(), tan()
# - 没有 cot(), csc(), sec()！

# 解决方案：cot(x) = 1/tan(x)
_rewriteable_functions = {
    'cot': ('tan', []),  # cot(x) → 1/tan(x)
    # ...
}
```

**StrPrinter 不需要**：
- 目标是显示 SymPy 表达式
- `cot(y)` 直接显示为 `"cot(y)"` 即可

**CodePrinter 需要**：
- 目标是生成可编译的 C 代码
- C 标准库没有 `cot()`
- 必须重写为支持的形式

---

## 7. 共享配置与后端独有配置边界

### 7.1 配置系统核心机制

**文件**: `printer.py:251-271`
```python
@classmethod
def _get_initial_settings(cls):
    settings = cls._default_settings.copy()
    for key, val in cls._global_settings.items():
        if key in cls._default_settings:
            settings[key] = val
    return settings

def __init__(self, settings=None):
    self._settings = self._get_initial_settings()
    # ...
    if settings is not None:
        self._settings.update(settings)  # 实例参数覆盖
```

**配置优先级**:
1. **最高**: 实例化时传入的 `settings` 参数
2. **中间**: 类 `_global_settings`（全局设置）
3. **最低**: 类 `_default_settings`（默认设置）

### 7.2 共享配置（Printer/StrPrinter 定义）

#### Printer 基类

```python
# printer.py:244-246
_global_settings: dict[str, Any] = {}
_default_settings: dict[str, Any] = {}
```

Printer 基类**没有定义默认配置**，完全由子类定义。

#### StrPrinter 配置

**文件**: `str.py:27-36`
```python
_default_settings: dict[str, Any] = {
    "order": None,           # 项的排序方式
    "full_prec": "auto",     # 浮点数精度
    "sympy_integers": False,
    "abbrev": False,
    "perm_cyclic": True,
    "min": None,
    "max": None,
    "dps": None
}
```

### 7.3 CodePrinter 独有配置

**文件**: `codeprinter.py:64-73`
```python
_default_settings: dict[str, Any] = {
    'order': None,                    # 继承自 StrPrinter
    'full_prec': 'auto',              # 继承自 StrPrinter
    'error_on_reserved': False,       # 独有：遇到保留字是否报错
    'reserved_word_suffix': '_',      # 独有：保留字后缀
    'human': True,                     # 独有：人类可读格式
    'inline': False,                   # 独有：内联数字符号
    'allow_unknown_functions': False, # 独有：允许未知函数
    'strict': None                     # 独有：严格模式（不支持类型的处理）
}
```

### 7.4 MathML 独有配置

**文件**: `mathml.py:25-41`
```python
_default_settings: dict[str, Any] = {
    "order": None,                     # 共享
    "encoding": "utf-8",               # 独有：XML 编码
    "fold_frac_powers": False,         # 独有：折叠分数幂
    "fold_func_brackets": False,       # 独有：折叠函数括号
    "fold_short_frac": None,           # 独有：短分数折叠
    "inv_trig_style": "abbreviated",   # 独有：反三角函数风格
    "ln_notation": False,              # 独有：ln 表示法
    "long_frac_ratio": None,           # 独有：长分数比例
    "mat_delim": "[",                  # 独有：矩阵分隔符
    "mat_symbol_style": "plain",       # 独有：矩阵符号风格
    "mul_symbol": None,                # 独有：乘法符号
    "root_notation": True,              # 独有：根号表示法
    "symbol_names": {},                 # 独有：符号名称映射
    "mul_symbol_mathml_numbers": '&#xB7;',  # 独有：数字乘法符号
    "disable_split_super_sub": False,  # 独有：禁用上下标拆分
}
```

### 7.5 配置边界清晰划分

#### 共享配置（所有打印器通用）

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `order` | `None \| str` | 项的排序方式（'old', 'none' 等） |
| `full_prec` | `'auto' \| bool` | 浮点数精度控制 |

#### 代码生成独有配置

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `error_on_reserved` | `bool` | 遇到 C 保留字（如 `if`, `for`）是否报错 |
| `reserved_word_suffix` | `str` | 保留字后添加的后缀（如 `if_`） |
| `human` | `bool` | 生成人类可读的代码格式 |
| `inline` | `bool` | 是否内联 `pi`, `E` 等数字符号 |
| `allow_unknown_functions` | `bool` | 是否允许未定义的函数调用 |
| `strict` | `bool \| None` | 不支持类型是否抛出异常 |

#### MathML 独有配置

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `encoding` | `str` | XML 文档编码 |
| `fold_frac_powers` | `bool` | 折叠 `x^(1/2)` 为 `sqrt(x)` |
| `inv_trig_style` | `str` | 反三角函数显示风格（'abbreviated', 'full'） |
| `mat_delim` | `str` | 矩阵分隔符（'[', '(' 等） |
| `mul_symbol` | `str \| None` | 乘法符号显示 |

### 7.6 配置继承示例

#### C99CodePrinter 的配置继承

**文件**: `c.py:154-...`
```python
_default_settings: dict[str, Any] = dict(CodePrinter._default_settings, **{
    'precision': 17,
    'user_functions': {},
    'enable_cse': False,
    'restrict': False,
    'language': 'C99',
    'contract': False,
    'standard': 'c99',
    'include_math_h': True,
})
```

**继承链**:
```
Printer._default_settings = {}
    │
    └── StrPrinter._default_settings
            {'order': None, 'full_prec': 'auto', ...}
            │
            └── CodePrinter._default_settings
                    {'order': None, 'full_prec': 'auto',  # 继承
                     'error_on_reserved': False,          # 新增
                     'strict': None, ...}                 # 新增
                    │
                    └── C99CodePrinter._default_settings
                            {'order': None, ...,           # 继承
                             'precision': 17,              # 新增
                             'standard': 'c99', ...}      # 新增
```

### 7.7 配置使用示例

#### 严格模式 vs 宽松模式

```python
from sympy import Integral, sin, symbols
from sympy.printing.c import C99CodePrinter

x = symbols('x')
expr = Integral(sin(x), x)

# 严格模式（默认）
cp_strict = C99CodePrinter(strict=True)
cp_strict.doprint(expr)  # 抛出 PrintMethodNotImplementedError

# 宽松模式
cp_lax = C99CodePrinter(strict=False)
cp_lax.doprint(expr)  # 返回 "Integral(sin(x), x)"（但不是有效 C 代码）
```

#### 保留字处理

```python
from sympy import symbols
from sympy.printing.c import C99CodePrinter

# 'if' 是 C 保留字
if_var = symbols('if')

# 默认：添加后缀
cp1 = C99CodePrinter()
cp1.doprint(if_var)  # "if_"

# 报错模式
cp2 = C99CodePrinter(error_on_reserved=True)
cp2.doprint(if_var)  # 抛出 ValueError
```

---

## 8. 附录：关键代码位置

### 8.1 核心分发机制

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| `_print()` 三级分发 | `sympy/printing/printer.py` | 295-336 |
| `doprint()` 入口 | `sympy/printing/printer.py` | 291-293 |
| 配置优先级 | `sympy/printing/printer.py` | 251-271 |

### 8.2 StrPrinter

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 类定义 | `sympy/printing/str.py` | 25-36 |
| `_print_Add` | `sympy/printing/str.py` | 57-79 |
| `_print_Function` | `sympy/printing/str.py` | 169-170 |
| `_print_Integral` | `sympy/printing/str.py` | 189-196 |
| `_print_Basic` | `sympy/printing/str.py` | 108-110 |
| `emptyPrinter` | `sympy/printing/str.py` | 49-55 |

### 8.3 MathML

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| `MathMLPrinterBase` 定义 | `sympy/printing/mathml.py` | 20-65 |
| `doprint()` DOM→XML | `sympy/printing/mathml.py` | 66-74 |
| `MathMLContentPrinter` 定义 | `sympy/printing/mathml.py` | 83-88 |
| `mathml_tag()` 标签映射 | `sympy/printing/mathml.py` | 90-154 |
| `_print_Add` (Content) | `sympy/printing/mathml.py` | 191-217 |
| `_print_Integral` (Content) | `sympy/printing/mathml.py` | 321-348 |

### 8.4 CodePrinter

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| `CodePrinter` 定义 | `sympy/printing/codeprinter.py` | 53-73 |
| 显式不支持映射 | `sympy/printing/codeprinter.py` | 621-643 |
| `_print_not_supported` | `sympy/printing/codeprinter.py` | 608-619 |
| `_print_Function` | `sympy/printing/codeprinter.py` | 437-463 |
| `_rewriteable_functions` | `sympy/printing/codeprinter.py` | 78-114 |

---

## 9. 关键结论

### 9.1 调用链差异核心原因

1. **输出目标不同**:
   - StrPrinter: 显示 SymPy 表达式
   - MathML: 遵循 XML DOM 结构
   - CodePrinter: 生成可编译代码

2. **继承链覆盖不同**:
   - StrPrinter: `_print_Basic` 提供兜底
   - CodePrinter: `_print_Basic = _print_not_supported` 显式拒绝

3. **回退策略不同**:
   - StrPrinter: 几乎不会触发 `emptyPrinter`
   - CodePrinter: 显式不支持映射 → `_print_not_supported`
   - MathML: 可能出错（期望 DOM 节点）

### 9.2 配置边界清晰

| 配置类型 | 所属打印器 | 核心目的 |
|----------|------------|----------|
| 共享配置 | Printer/StrPrinter | 表达式排序、浮点数精度 |
| 代码生成独有 | CodePrinter | 保留字处理、严格/宽松模式 |
| MathML 独有 | MathMLPrinterBase | XML 格式、数学符号渲染 |

### 9.3 函数重写机制的必要性

- **StrPrinter 不需要**: `cot(x)` → `"cot(x)"` 直接显示
- **CodePrinter 需要**: C 标准库没有 `cot()` → 重写为 `1/tan(x)`
- **MathML 不需要**: MathML 标准定义了 `<cot>` 标签

---

> 报告完成日期：2026-05-01
> 分析基于 SymPy 代码提交版本
