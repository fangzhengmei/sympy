# SymPy 打印子系统深度分析

## 目录

1. [概述](#1-概述)
2. [核心 Printer 基类与方法分发机制](#2-核心-printer-基类与方法分发机制)
3. [多种打印后端的共享配置与特化](#3-多种打印后端的共享配置与特化)
4. [代码生成类打印器的语言差异化处理](#4-代码生成类打印器的语言差异化处理)
5. [表达式类型回退机制](#5-表达式类型回退机制)
6. [架构总结与设计模式](#6-架构总结与设计模式)

---

## 1. 概述

SymPy 的打印子系统是一个高度可扩展的架构，用于将数学表达式转换为多种输出格式。该系统采用了**访问者模式**的变体，通过方法名分发机制实现了表达式类型与打印处理器的解耦。

### 1.1 打印后端分类

| 类别 | 后端类型 | 主要用途 |
|------|----------|----------|
| **数学渲染** | LaTeX, MathML | 高质量数学文档和网页展示 |
| **字符艺术** | Pretty Printer | 控制台/终端的 2D 排版 |
| **代码生成** | C, Fortran, Julia, Python, Rust | 数值计算代码生成 |
| **基础格式** | Str, Repr | 调试和序列化 |

### 1.2 模块文件结构

```
sympy/printing/
├── printer.py          # 核心 Printer 基类（方法分发核心）
├── str.py              # StrPrinter 基础字符串打印
├── repr.py             # ReprPrinter 可执行字符串
├── latex.py            # LatexPrinter LaTeX 格式
├── mathml.py           # MathMLPrinter 数学标记语言
├── pretty/             # 字符艺术打印模块
│   ├── pretty.py       # PrettyPrinter 主类
│   ├── pretty_symbology.py  # Unicode/ASCII 符号表
│   └── stringpict.py   # 2D 字符串排版引擎
├── codeprinter.py      # CodePrinter 代码生成基类
├── c.py                # C89/C99 代码打印
├── cxx.py              # C++ 代码打印
├── fortran.py          # Fortran 代码打印
├── julia.py            # Julia 代码打印
├── rust.py             # Rust 代码打印
├── python.py           # Python 代码打印
├── jscode.py           # JavaScript 代码打印
├── rcode.py            # R 语言代码打印
├── octave.py           # Octave/Matlab 代码打印
├── mathematica.py      # Mathematica 语法
├── maple.py            # Maple 语法
├── glsl.py             # OpenGL Shader Language
├── smtlib.py           # SMT-LIB 格式（定理证明）
├── numpy.py            # NumPy 特定代码
├── pytorch.py          # PyTorch 特定代码
├── tensorflow.py       # TensorFlow 特定代码
├── pycode.py           # 高级 Python 代码生成
├── lambdarepr.py       # Lambda 表达式表示
├── precedence.py       # 运算符优先级定义
├── conventions.py      # 命名约定工具
├── defaults.py         # 默认配置
├── preview.py          # LaTeX 预览渲染
├── tree.py             # 表达式树可视化
├── dot.py              # Graphviz DOT 格式
└── tableform.py        # 表格格式化
```

---

## 2. 核心 Printer 基类与方法分发机制

### 2.1 Printer 基类核心架构

`Printer` 类定义在 `sympy/printing/printer.py:235-432`，是所有打印器的抽象基类。

#### 核心属性

```python
class Printer:
    _global_settings: dict[str, Any] = {}      # 全局设置（影响所有实例）
    _default_settings: dict[str, Any] = {}     # 类级默认设置
    printmethod: str = None                      # 对象自打印方法名（如 _latex）
```

#### 三级分发策略

打印器在 `_print` 方法（`printer.py:295-336`）中实现了三级分发机制：

```
┌─────────────────────────────────────────────────────────────────┐
│                    doprint(expr) 入口点                          │
│                         ↓                                         │
│              self._str(self._print(expr))                        │
│                         ↓                                         │
├─────────────────────────────────────────────────────────────────┤
│  第一级：对象自打印（优先级最高）                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ if self.printmethod and hasattr(expr, self.printmethod): │  │
│  │     return getattr(expr, self.printmethod)(self, **kwargs)│  │
│  └───────────────────────────────────────────────────────────┘  │
│                         ↓（如果不存在）                           │
├─────────────────────────────────────────────────────────────────┤
│  第二级：方法名分发（沿 MRO 继承链）                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ for cls in type(expr).__mro__:                             │  │
│  │     printmethodname = '_print_' + cls.__name__             │  │
│  │     printmethod = getattr(self, printmethodname, None)     │  │
│  │     if printmethod is not None:                             │  │
│  │         return printmethod(expr, **kwargs)                  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                         ↓（如果所有都不存在）                     │
├─────────────────────────────────────────────────────────────────┤
│  第三级：兜底回退                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ return self.emptyPrinter(expr)                              │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 第一级：对象自打印机制

**设计意图**：允许自定义表达式类型控制自身的打印行为。

每个打印器定义 `printmethod` 属性，指定对象可以实现的方法名：

| 打印器 | printmethod 值 | 对象需实现方法 |
|--------|-----------------|----------------|
| `StrPrinter` | `"_sympystr"` | `def _sympystr(self, printer):` |
| `LatexPrinter` | `"_latex"` | `def _latex(self, printer):` |
| `MathMLContentPrinter` | `"_mathml_content"` | `def _mathml_content(self, printer):` |
| `PrettyPrinter` | `"_pretty"` | `def _pretty(self, printer):` |
| `C89CodePrinter` | `"_ccode"` | `def _ccode(self, printer):` |
| `FCodePrinter` | `"_fcode"` | `def _fcode(self, printer):` |
| `JuliaCodePrinter` | `"_julia"` | `def _julia(self, printer):` |

**示例：自定义 LaTeX 打印**

```python
from sympy import Mod, Symbol, print_latex

class ModOp(Mod):
    def _latex(self, printer):
        a, b = [printer._print(i) for i in self.args]
        return r"\operatorname{Mod}{\left(%s, %s\right)}" % (a, b)

x = Symbol('x')
m = Symbol('m')
print(print_latex(Mod(x, m)))       # 输出: x \bmod m
print(print_latex(ModOp(x, m)))     # 输出: \operatorname{Mod}{\left(x, m\right)}
```

### 2.3 第二级：方法名分发机制（核心）

这是打印子系统最精妙的设计，采用**基于类名的方法约定**。

#### 分发算法详解

```python
# printer.py:316-332
def _print(self, expr, **kwargs) -> str:
    # ... 第一级分发已尝试 ...
    
    # 获取表达式类型的方法解析顺序（MRO）
    classes = type(expr).__mro__
    
    # 特殊处理：忽略 UndefinedFunction 子类
    if AppliedUndef in classes:
        classes = classes[classes.index(AppliedUndef):]
    if UndefinedFunction in classes:
        classes = classes[classes.index(UndefinedFunction):]
    
    # 特殊处理：用户自定义函数的名称匹配
    if Function in classes:
        i = classes.index(Function)
        classes = tuple(c for c in classes[:i] if 
            c.__name__ == classes[0].__name__ or 
            c.__name__.endswith("Base")) + classes[i:]
    
    # 沿 MRO 链依次查找 _print_<ClassName> 方法
    for cls in classes:
        printmethodname = '_print_' + cls.__name__
        printmethod = getattr(self, printmethodname, None)
        if printmethod is not None:
            return printmethod(expr, **kwargs)
    
    # 第三级回退
    return self.emptyPrinter(expr)
```

#### 继承链分发示例

假设有如下类层次结构：

```
        Basic
          │
        Atom
          │
        Number
          │
       Rational
```

当打印 `Rational(1, 2)` 时，打印器按以下顺序查找方法：

1. `_print_Rational(expr)` → 找到则调用
2. `_print_Number(expr)` → 找到则调用
3. `_print_Atom(expr)` → 找到则调用
4. `_print_Basic(expr)` → 找到则调用
5. 否则调用 `emptyPrinter(expr)`

#### 为什么这种设计优秀？

| 优势 | 说明 |
|------|------|
| **零配置注册** | 新增表达式类型只需添加 `_print_<Type>` 方法，无需注册表 |
| **继承复用** | 子类可复用父类的打印逻辑，符合面向对象设计 |
| **特化覆盖** | 子类可定义更具体的打印方法覆盖父类行为 |
| **运行时动态** | 支持猴子补丁（monkey patching）扩展打印器 |

### 2.4 设置管理机制

Printer 实现了**三层设置优先级**：

```python
# printer.py:252-276
@classmethod
def _get_initial_settings(cls):
    settings = cls._default_settings.copy()           # 第一层：类默认
    for key, val in cls._global_settings.items():     # 第二层：全局覆盖
        if key in cls._default_settings:
            settings[key] = val
    return settings

def __init__(self, settings=None):
    self._settings = self._get_initial_settings()
    self._context = {}  # 临时上下文（打印过程中可变）
    
    if settings is not None:                          # 第三层：实例覆盖
        self._settings.update(settings)
        # 验证：拒绝未知设置
        if len(self._settings) > len(self._default_settings):
            for key in self._settings:
                if key not in self._default_settings:
                    raise TypeError("Unknown setting '%s'." % key)
```

**设置优先级顺序**：
```
实例 settings 参数 > 类 _global_settings > 类 _default_settings
```

**上下文管理器支持**：

```python
# printer.py:225-232
@contextmanager
def printer_context(printer, **kwargs):
    original = printer._context.copy()
    try:
        printer._context.update(kwargs)
        yield
    finally:
        printer._context = original
```

---

## 3. 多种打印后端的共享配置与特化

### 3.1 打印后端类层次

```
┌─────────────────────────────────────────────────────────────────┐
│                         Printer (基类)                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ _print() 分发机制                                            │  │
│  │ doprint() 入口                                               │  │
│  │ emptyPrinter() 兜底                                          │  │
│  │ _default_settings 配置                                       │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
           ▲                    ▲                    ▲
           │                    │                    │
┌──────────┴─────────┐  ┌──────┴────────┐  ┌──────┴─────────┐
│    StrPrinter      │  │  PrettyPrinter │  │ LatexPrinter    │
│  (基础字符串)       │  │  (字符艺术)     │  │  (LaTeX 格式)    │
└────────────────────┘  └─────────────────┘  └──────────────────┘
           ▲
           │
┌──────────┴─────────────────────────────────────────────────────┐
│                      CodePrinter                                 │
│  (代码生成基类 - C/Fortran/Julia/Python/Rust/... 的公共父类)    │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 各打印后端的 `_default_settings` 对比

#### StrPrinter (`str.py:27-36`)
```python
_default_settings = {
    "order": None,           # 项排序方式
    "full_prec": "auto",     # 浮点数精度
    "sympy_integers": False, # 使用 SymPy Integer 语法
    "abbrev": False,         # 缩写表示
    "perm_cyclic": True,     # 循环置换表示
    "min": None, "max": None, "dps": None  # 数值显示范围
}
```

#### LatexPrinter (`latex.py:145-155`)
```python
_default_settings = {
    "full_prec": False,
    "fold_frac_powers": False,      # 折叠分数幂 a^{1/2} → sqrt(a)
    "fold_func_brackets": False,     # 折叠函数括号
    "fold_short_frac": None,         # 短分数使用 \frac 还是 /
    "inv_trig_style": "abbreviated", # 反三角函数样式
    "itex": False,                    # iTeX 兼容模式
    "ln_notation": False,             # 使用 ln 而非 log
    "long_frac_ratio": None,          # 长分数阈值
    "mat_delim": "[",                 # 矩阵分隔符
    "mode": "plain",                  # 输出模式
    "mul_symbol": None,               # 乘法符号
    "order": None,
    "root_notation": True,            # 使用 sqrt, root 表示
    "symbol_names": {},               # 符号名映射
    "imaginary_unit": "i",            # 虚数单位
    "gothic_re_im": True,             # 使用哥特体表示 Re/Im
    "mat_symbol_style": "plain",      # 矩阵符号样式
    "interv_rev": False,              # 区间表示反转
    "decimal_separator": ".",         # 小数点分隔符
    "perm_cyclic": True,
    "min": None, "max": None,
    "full_prec": "auto",
}
```

#### PrettyPrinter (`pretty/pretty.py:35-46`)
```python
_default_settings = {
    "order": None,
    "full_prec": "auto",
    "use_unicode": None,           # 使用 Unicode 字符
    "wrap_line": True,             # 自动换行
    "num_columns": None,           # 列数限制
    "use_unicode_sqrt_char": True, # Unicode 根号字符
    "root_notation": True,
    "mat_symbol_style": "plain",
    "imaginary_unit": "i",
    "perm_cyclic": True
}
```

### 3.3 打印方法的特化示例

#### 乘法打印的差异化处理

**C 语言** (`codeprinter.py:550-606`)：
```python
def _print_Mul(self, expr):
    # 分离分子分母，生成 a*b/(c*d) 形式
    # 使用 * 作为乘法运算符
    # 处理负数系数的负号位置
    ...
    if not b:
        return sign + '*'.join(a_str)
    elif len(b) == 1:
        return sign + '*'.join(a_str) + "/" + b_str[0]
    else:
        return sign + '*'.join(a_str) + "/(%s)" % '*'.join(b_str)
```

**Fortran** (`fortran.py` - 使用 `**` 而非 `^`，数组使用 `()` 而非 `[]`)

**Julia** (`julia.py:117-150`)：
```python
def _print_Mul(self, expr):
    # 特殊处理虚数单位：3*I → 3im
    if (expr.is_number and expr.is_imaginary and
            expr.as_coeff_Mul()[0].is_integer):
        return "%sim" % self._print(-S.ImaginaryUnit*expr)
    # ... 其余逻辑类似
```

**PrettyPrinter**：返回 `prettyForm` 对象而非字符串，支持 2D 排版。

---

## 4. 代码生成类打印器的语言差异化处理

### 4.1 CodePrinter 基类架构

`CodePrinter` 定义在 `codeprinter.py:53-644`，是所有代码生成打印器的基类。

#### 核心属性与方法

```python
class CodePrinter(StrPrinter):
    _operators = {              # 运算符映射（可被子类覆盖）
        'and': '&&',
        'or': '||',
        'not': '!',
    }
    
    _default_settings = {       # 代码生成特有设置
        'order': None,
        'full_prec': 'auto',
        'error_on_reserved': False,      # 保留字处理策略
        'reserved_word_suffix': '_',      # 保留字后缀
        'human': True,                     # 人类可读格式
        'inline': False,                   # 内联常数
        'allow_unknown_functions': False,  # 允许未知函数
        'strict': None                     # 严格模式
    }
    
    # 可重写函数表：不支持的函数尝试重写为支持的形式
    _rewriteable_functions = {
        'cot': ('tan', []),        # cot → tan
        'csc': ('sin', []),        # csc → sin
        'Mod': ('floor', []),      # Mod → floor
        'factorial': ('gamma', []),# factorial → gamma
        'Max': ('Piecewise', []),  # Max → Piecewise
        # ... 更多
    }
```

#### 语言级抽象方法（必须由子类实现）

```python
# codeprinter.py:311-346
def _rate_index_position(self, p):
    """计算数组索引位置的权重（用于循环优化排序）"""
    raise NotImplementedError()

def _get_statement(self, codestring):
    """格式化语句（添加行尾符号如 ; 或 空）"""
    raise NotImplementedError()

def _get_comment(self, text):
    """格式化注释（C: // 或 /* */, Fortran: !, Julia: #）"""
    raise NotImplementedError()

def _declare_number_const(self, name, value):
    """声明数值常量"""
    raise NotImplementedError()

def _format_code(self, lines):
    """格式化代码行（缩进、换行等）"""
    raise NotImplementedError()

def _get_loop_opening_ending(self, indices):
    """返回循环的开始和结束语句"""
    raise NotImplementedError()
```

### 4.2 各语言打印器差异化对比

#### 4.2.1 C 语言 (`c.py`)

| 特性 | 实现 |
|------|------|
| **语句终止** | `;` |
| **注释** | `// text` |
| **数组索引** | `[i]` |
| **类型映射** | `float64→'double'`, `int32→'int32_t'` |
| **头文件依赖** | `bool_→{'stdbool.h'}`, `int8→{'stdint.h'}` |
| **标准版本** | C89/C99 两套函数表 |

**C89 vs C99 函数差异**：
```python
known_functions_C89 = {
    "sin": "sin", "cos": "cos", "tan": "tan",
    # C89 没有 exp2, log2, asinh, acosh, atanh 等
}

known_functions_C99 = dict(known_functions_C89, **{
    'exp2': 'exp2', 'log2': 'log2',
    'asinh': 'asinh', 'acosh': 'acosh', 'atanh': 'atanh',
    'Cbrt': 'cbrt', 'hypot': 'hypot', 'fma': 'fma',
})
```

**类型映射系统** (`c.py:168-195`)：
```python
type_mappings = {
    real: 'double',
    intc: 'int',
    float32: 'float',
    float64: 'double',
    bool_: 'bool',
    int8: 'int8_t',
    int16: 'int16_t',
    int32: 'int32_t',
    int64: 'int64_t',
}

type_headers = {
    bool_: {'stdbool.h'},
    int8: {'stdint.h'},  # 需要 #include <stdint.h>
}
```

#### 4.2.2 Fortran (`fortran.py`)

| 特性 | 实现 |
|------|------|
| **不区分大小写** | 变量名自动添加后缀区分 `x` 和 `X` |
| **数组索引** | `(i)` 而非 `[i]` |
| **运算符** | `.and.`, `.or.`, `.not.`, `.eqv.`, `/=` |
| **源格式** | 固定格式（前 6 列保留）vs 自由格式 |
| **标准版本** | 66, 77, 90, 95, 2003, 2008 |

**运算符映射** (`fortran.py:108-118`)：
```python
_operators = {
    'and': '.and.',
    'or': '.or.',
    'xor': '.neqv.',
    'equivalent': '.eqv.',
    'not': '.not. ',
}

_relationals = {
    '!=': '/=',  # Fortran 使用 /= 而非 !=
}
```

**名称 mangling 机制** (`fortran.py:149-175`)：
```python
def _print_Symbol(self, expr):
    if self._settings['name_mangling'] == True:
        # Fortran 不区分大小写，需要处理 name 和 Name 冲突
        name = expr.name.lower()
        if name in self.mangled_symbols:
            return self.mangled_symbols[name]
        # ... 生成唯一名称
```

**固定格式 vs 自由格式**：
```python
@property
def _lead(self):
    if self._settings['source_format'] == 'fixed':
        return {
            'code': "      ",    # 前 6 列保留
            'cont': "     @ ",   # 第 6 列续行标记
            'comment': "C     "  # 第 1 列 C 表示注释
        }
    elif self._settings['source_format'] == 'free':
        return {
            'code': "",
            'cont': "      ",
            'comment': "! "
        }
```

#### 4.2.3 Julia (`julia.py`)

| 特性 | 实现 |
|------|------|
| **语句终止** | 无（可选分号） |
| **注释** | `# text` |
| **数组索引** | `[i]`，从 1 开始 |
| **数组顺序** | 列优先（Fortran 顺序） |
| **虚数单位** | `im` 后缀：`3*I → 3im` |
| **方法调用风格** | 支持 `x.sin()` 或 `sin(x)` |

**循环语法** (`julia.py:105-114`)：
```python
def _get_loop_opening_ending(self, indices):
    open_lines = []
    close_lines = []
    for i in indices:
        # Julia 数组从 1 开始
        var, start, stop = map(self._print,
                [i.label, i.lower + 1, i.upper + 1])
        open_lines.append("for %s = %s:%s" % (var, start, stop))
        close_lines.append("end")
    return open_lines, close_lines
```

**矩阵遍历顺序** (`julia.py:99-102`)：
```python
def _traverse_matrix_indices(self, mat):
    # Julia 使用 Fortran 顺序（列优先）
    rows, cols = mat.shape
    return ((i, j) for j in range(cols) for i in range(rows))
```

### 4.3 函数打印机制

`_print_Function` 是代码生成中最复杂的方法之一，实现了**多层函数解析**：

```python
# codeprinter.py:437-463
def _print_Function(self, expr):
    # 第一层：已知函数表（支持条件分支）
    if expr.func.__name__ in self.known_functions:
        cond_func = self.known_functions[expr.func.__name__]
        if isinstance(cond_func, str):
            # 简单映射："sin" → "sin"
            return "%s(%s)" % (cond_func, self.stringify(expr.args, ", "))
        else:
            # 条件映射：[(condition1, func1), (condition2, func2)]
            for cond, func in cond_func:
                if cond(*expr.args):
                    break
            if func is not None:
                try:
                    return func(*[self.parenthesize(item, 0) for item in expr.args])
                except TypeError:
                    return "%s(%s)" % (func, self.stringify(expr.args, ", "))
    
    # 第二层：内联函数（使用 Lambda 定义的）
    elif hasattr(expr, '_imp_') and isinstance(expr._imp_, Lambda):
        return self._print(expr._imp_(*expr.args))
    
    # 第三层：可重写函数（如 cot → tan）
    elif expr.func.__name__ in self._rewriteable_functions:
        target_f, required_fs = self._rewriteable_functions[expr.func.__name__]
        if self._can_print(target_f) and all(self._can_print(f) for f in required_fs):
            return '(' + self._print(expr.rewrite(target_f)) + ')'
    
    # 第四层：允许未知函数（或报错）
    if expr.is_Function and self._settings.get('allow_unknown_functions', False):
        return '%s(%s)' % (self._print(expr.func), ', '.join(map(self._print, expr.args)))
    else:
        return self._print_not_supported(expr)
```

### 4.4 赋值与控制流打印

`_print_Assignment` 处理复杂的赋值场景：

```python
# codeprinter.py:369-404
def _print_Assignment(self, expr):
    lhs = expr.lhs
    rhs = expr.rhs
    
    # 特殊情况 1: Piecewise 赋值 → if-else 语句
    if isinstance(expr.rhs, Piecewise):
        expressions = []
        conditions = []
        for (e, c) in rhs.args:
            expressions.append(Assignment(lhs, e))
            conditions.append(c)
        temp = Piecewise(*zip(expressions, conditions))
        return self._print(temp)
    
    # 特殊情况 2: MatrixSymbol 赋值 → 逐元素赋值
    elif isinstance(lhs, MatrixSymbol):
        lines = []
        for (i, j) in self._traverse_matrix_indices(lhs):
            temp = Assignment(lhs[i, j], rhs[i, j])
            code0 = self._print(temp)
            lines.append(code0)
        return "\n".join(lines)
    
    # 特殊情况 3: IndexedBase（数组）+ contract=True → 生成循环
    elif self._settings.get("contract", False) and (lhs.has(IndexedBase) or
            rhs.has(IndexedBase)):
        return self._doprint_loops(rhs, lhs)
    
    # 普通赋值
    else:
        lhs_code = self._print(lhs)
        rhs_code = self._print(rhs)
        return self._get_statement("%s = %s" % (lhs_code, rhs_code))
```

### 4.5 保留字处理机制

```python
# codeprinter.py:420-430
def _print_Symbol(self, expr):
    name = super()._print_Symbol(expr)
    
    if name in self.reserved_words:
        if self._settings['error_on_reserved']:
            # 策略 1: 报错
            msg = ('This expression includes the symbol "{}" which is a '
                   'reserved keyword in this language.')
            raise ValueError(msg.format(name))
        # 策略 2: 添加后缀
        return name + self._settings['reserved_word_suffix']
    else:
        return name
```

**各语言保留字示例**：
- **C**: `auto`, `break`, `case`, `char`, `const`, `continue`, `default`, `do`, `double`, `else`, `enum`, `extern`, `float`, `for`, `goto`, `if`, `int`, `long`, `register`, `return`, `short`, `signed`, `sizeof`, `static`, `struct`, `switch`, `typedef`, `union`, `unsigned`, `void`, `volatile`, `while`
- **C99 新增**: `inline`, `restrict`

---

## 5. 表达式类型回退机制

### 5.1 回退机制概览

当打印器遇到没有对应 `_print_*` 方法的表达式类型时，系统会按以下层级回退：

```
┌─────────────────────────────────────────────────────────────────┐
│                    表达式类型打印回退链                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  优先级 1: 专用方法（最具体）                                      │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ _print_Rational(expr)    ← Rational 实例首先尝试           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓（不存在）                              │
│  优先级 2: 父类方法                                                │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ _print_Number(expr)      ← 沿 MRO 向上查找                │  │
│  │ _print_Atom(expr)                                            │  │
│  │ _print_Basic(expr)                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓（都不存在）                            │
│  优先级 3: emptyPrinter 兜底                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ def emptyPrinter(self, expr):                               │  │
│  │     return str(expr)          ← 默认行为                    │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 不同打印器的 `emptyPrinter` 实现

#### Printer 基类默认 (`printer.py:338-339`)
```python
def emptyPrinter(self, expr):
    return str(expr)
```

#### StrPrinter 特化 (`str.py:49-55`)
```python
def emptyPrinter(self, expr):
    if isinstance(expr, str):
        return expr
    elif isinstance(expr, Basic):
        return repr(expr)  # Basic 实例使用 repr
    else:
        return str(expr)
```

#### PrettyPrinter 特化 (`pretty/pretty.py:56-57`)
```python
def emptyPrinter(self, expr):
    return prettyForm(str(expr))  # 包装为 prettyForm 对象
```

#### CodePrinter 特化（不支持的表达式）

CodePrinter 显式将某些类型标记为不支持：

```python
# codeprinter.py:621-643
_print_Basic = _print_not_supported
_print_ComplexInfinity = _print_not_supported
_print_ExprCondPair = _print_not_supported
_print_GeometryEntity = _print_not_supported
_print_Infinity = _print_not_supported
_print_Integral = _print_not_supported
_print_Interval = _print_not_supported
_AccumulationBounds = _print_not_supported
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

#### `_print_not_supported` 实现 (`codeprinter.py:608-619`)
```python
def _print_not_supported(self, expr):
    if self._settings.get('strict', False):
        # 严格模式：抛出异常
        raise PrintMethodNotImplementedError(
            f"Unsupported by {type(self)}: {type(expr)}" +
            "\nSet the printer option 'strict' to False in order to generate partially printed code."
        )
    try:
        self._not_supported.add(expr)  # 记录不支持的表达式
    except TypeError:
        pass  # 不可哈希
    return self.emptyPrinter(expr)  # 兜底
```

### 5.3 函数重写回退机制

当函数不被直接支持时，CodePrinter 尝试**自动重写**为等价形式：

```python
# codeprinter.py:78-114
_rewriteable_functions = {
    # 三角函数
    'cot': ('tan', []),        # cot(x) = 1/tan(x)
    'csc': ('sin', []),        # csc(x) = 1/sin(x)
    'sec': ('cos', []),        # sec(x) = 1/cos(x)
    'acot': ('atan', []),      # 反三角函数
    'acsc': ('asin', []),
    'asec': ('acos', []),
    
    # 双曲函数
    'coth': ('exp', []),
    'csch': ('exp', []),
    'sech': ('exp', []),
    'acoth': ('log', []),
    'acsch': ('log', []),
    'asech': ('log', []),
    
    # 特殊函数
    'catalan': ('gamma', []),
    'fibonacci': ('sqrt', []),
    'lucas': ('sqrt', []),
    'beta': ('gamma', []),
    'sinc': ('sin', ['Piecewise']),
    'Mod': ('floor', []),
    'factorial': ('gamma', []),
    'factorial2': ('gamma', ['Piecewise']),
    'subfactorial': ('uppergamma', []),
    'RisingFactorial': ('gamma', ['Piecewise']),
    'FallingFactorial': ('gamma', ['Piecewise']),
    'binomial': ('gamma', []),
    'frac': ('floor', []),
    
    # 条件函数
    'Max': ('Piecewise', []),
    'Min': ('Piecewise', []),
    'Heaviside': ('Piecewise', []),
    
    # 特殊函数变体
    'erf2': ('erf', []),
    'erfc': ('erf', []),
    'Li': ('li', []),
    'Ei': ('li', []),
    'dirichlet_eta': ('zeta', []),
    'riemann_xi': ('zeta', ['gamma']),
    
    # 奇异函数
    'SingularityFunction': ('Piecewise', []),
}
```

**重写逻辑** (`codeprinter.py:454-458`)：
```python
elif expr.func.__name__ in self._rewriteable_functions:
    target_f, required_fs = self._rewriteable_functions[expr.func.__name__]
    if self._can_print(target_f) and all(self._can_print(f) for f in required_fs):
        return '(' + self._print(expr.rewrite(target_f)) + ')'
```

### 5.4 回退机制示例

#### 示例 1: 打印未定义的自定义类

```python
from sympy import Basic, Symbol
from sympy.printing.str import StrPrinter

class MyCustomExpr(Basic):
    def __new__(cls, x):
        return Basic.__new__(cls, x)
    
    @property
    def x(self):
        return self.args[0]

x = Symbol('x')
expr = MyCustomExpr(x)

printer = StrPrinter()
result = printer.doprint(expr)
# 结果: MyCustomExpr(x) - 来自 emptyPrinter → repr(expr)
```

#### 示例 2: C 代码打印不支持的表达式

```python
from sympy import Integral, Symbol, ccode
from sympy.printing.c import C89CodePrinter

x = Symbol('x')
expr = Integral(x**2, x)

# 默认模式（human=True, strict=False）
printer = C89CodePrinter()
result = printer.doprint(expr)
# 结果包含注释:
# // Not supported in C:
# // Integral
# Integral(x**2, x)

# 严格模式
printer_strict = C89CodePrinter({'strict': True})
try:
    printer_strict.doprint(expr)
except PrintMethodNotImplementedError as e:
    print(e)
    # Unsupported by <class 'C89CodePrinter'>: <class 'sympy.integrals.integrals.Integral'>
    # Printer has no method: _print_Derivative_Integral
    # Set the printer option 'strict' to False in order to generate partially printed code.
```

---

## 6. 架构总结与设计模式

### 6.1 核心设计模式

| 设计模式 | 应用位置 | 实现方式 |
|----------|----------|----------|
| **访问者模式 (Visitor)** | 方法分发核心 | `_print_<ClassName>` 方法约定 |
| **模板方法模式** | CodePrinter 抽象方法 | `_get_statement`, `_get_comment` 等 |
| **策略模式** | 设置与配置 | 三层设置优先级 + 上下文管理器 |
| **责任链模式** | 回退机制 | 沿 MRO 链查找方法 + 多层兜底 |
| **组合模式** | PrettyPrinter | `stringPict`, `prettyForm` 2D 排版 |
| **工厂模式** | 打印器实例化 | `ccode()`, `fcode()` 等工厂函数 |

### 6.2 架构优势

1. **高度可扩展**
   - 新增表达式类型：只需添加 `_print_<Type>` 方法
   - 新增打印后端：继承 `Printer` 或 `CodePrinter`
   - 用户自定义类：实现 `_latex`, `_sympystr` 等方法

2. **代码复用最大化**
   - 继承链：`Printer → StrPrinter → CodePrinter → C89CodePrinter`
   - 设置继承：`_default_settings` 字典更新机制
   - 方法复用：子类可调用 `super()._print_*(expr)`

3. **运行时灵活性**
   - 全局设置影响所有打印器
   - 实例设置覆盖默认
   - 上下文管理器临时修改
   - 猴子补丁动态扩展

4. **健壮的错误处理**
   - 多层回退机制
   - 可配置的严格/宽松模式
   - 保留字自动处理
   - 不支持表达式记录

### 6.3 关键数据结构

#### 设置系统
```python
# 三层配置
_global_settings    # 类变量，全局影响
_default_settings   # 类变量，子类可扩展
_settings           # 实例变量，构造时传入
_context            # 临时上下文，打印过程可变
```

#### 函数映射表
```python
known_functions = {
    # 简单映射
    "sin": "sin",
    "cos": "cos",
    
    # 条件映射列表
    "Abs": [
        (lambda x: not x.is_integer, "fabs"),
        (lambda x: x.is_integer, "abs")
    ],
    
    # 自定义格式化函数
    "Pow": [
        (lambda b, e: b == 2, lambda b, e: 'exp2(%s)' % e),
        (lambda b, e: b != 2, 'pow')
    ]
}
```

#### 类型映射系统
```python
type_aliases = {
    real: float64,
    complex_: complex128,
    integer: int32,
}

type_mappings = {
    float64: 'double',    # C
    float64: 'real*8',    # Fortran
    int32: 'int32_t',     # C
    int32: 'integer*4',   # Fortran
}

type_headers = {
    bool_: {'stdbool.h'},
    int8: {'stdint.h'},
}
```

### 6.4 扩展指南

#### 添加新的表达式类型支持

```python
from sympy.printing.latex import LatexPrinter

# 方法 1: 继承打印器（推荐）
class MyLatexPrinter(LatexPrinter):
    def _print_MyCustomType(self, expr):
        return r"\custom{%s}" % self._print(expr.arg)

# 方法 2: 猴子补丁（运行时扩展）
def _print_MyCustomType(self, expr):
    return r"\custom{%s}" % self._print(expr.arg)

LatexPrinter._print_MyCustomType = _print_MyCustomType
```

#### 添加新的代码生成后端

```python
from sympy.printing.codeprinter import CodePrinter

class MyLangPrinter(CodePrinter):
    printmethod = "_mylang"
    language = "MyLang"
    
    # 必需实现的抽象方法
    def _get_statement(self, codestring):
        return codestring + ";"  # 类似 C
    
    def _get_comment(self, text):
        return f"// {text}"
    
    def _declare_number_const(self, name, value):
        return f"const {name} = {value};"
    
    def _format_code(self, lines):
        return self.indent_code(lines)
    
    def _get_loop_opening_ending(self, indices):
        open_lines = []
        close_lines = []
        for i in indices:
            var, start, stop = map(self._print, 
                [i.label, i.lower + 1, i.upper + 1])
            open_lines.append(f"for {var} in {start}..{stop} {{")
            close_lines.append("}")
        return open_lines, close_lines
    
    # 可选：覆盖默认设置
    _default_settings = dict(CodePrinter._default_settings, **{
        'precision': 15,
        'contract': True,
    })
    
    # 可选：运算符映射
    _operators = {
        'and': '&&',
        'or': '||',
        'not': '!',
    }
```

---

## 附录：关键类参考

### A. 类继承关系图

```
Printer (基类)
├── StrPrinter
│   └── CodePrinter
│       ├── C89CodePrinter
│       │   └── C99CodePrinter
│       ├── CXXCodePrinter
│       ├── FCodePrinter (Fortran)
│       ├── JuliaCodePrinter
│       ├── RustCodePrinter
│       ├── PythonCodePrinter
│       ├── NumPyPrinter
│       ├── PyTorchPrinter
│       ├── TensorFlowPrinter
│       ├── OctavePrinter
│       ├── MathematicaPrinter
│       ├── MaplePrinter
│       ├── GLSLPrinter
│       ├── JavascriptPrinter
│       ├── RCodePrinter
│       └── SMTLibPrinter
├── LatexPrinter
├── ReprPrinter
├── PrettyPrinter
├── MathMLPrinterBase
│   ├── MathMLContentPrinter
│   └── MathMLPresentationPrinter
├── DotPrinter
├── TreePrinter
└── TableFormPrinter
```

### B. 关键文件位置参考

| 文件 | 路径 | 说明 |
|------|------|------|
| 核心分发机制 | `sympy/printing/printer.py` | `Printer` 基类，`_print()` 方法 |
| 代码生成基类 | `sympy/printing/codeprinter.py` | `CodePrinter`，函数重写 |
| C 代码生成 | `sympy/printing/c.py` | `C89CodePrinter`, `C99CodePrinter` |
| Fortran 代码生成 | `sympy/printing/fortran.py` | `FCodePrinter`，名称 mangling |
| Julia 代码生成 | `sympy/printing/julia.py` | `JuliaCodePrinter`，1 基索引 |
| LaTeX 打印 | `sympy/printing/latex.py` | `LatexPrinter` |
| MathML 打印 | `sympy/printing/mathml.py` | `MathMLPrinter` |
| 字符艺术打印 | `sympy/printing/pretty/pretty.py` | `PrettyPrinter` |
| 运算符优先级 | `sympy/printing/precedence.py` | `PRECEDENCE` 常量 |

---

## 参考文献

1. SymPy 官方文档: https://docs.sympy.org/latest/modules/printing.html
2. 源代码位置: `sympy/printing/` 目录
3. 设计模式参考: Gamma 等《设计模式》

---

**报告生成时间**: 2026-05-01  
**分析版本**: SymPy (当前开发版本)  
**分析范围**: `sympy/printing/` 模块及相关依赖
