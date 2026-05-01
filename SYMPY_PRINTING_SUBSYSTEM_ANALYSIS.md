# SymPy 打印子系统深度分析（修正版）

## 目录

1. [概述](#1-概述)
2. [真实类层次结构](#2-真实类层次结构)
3. [核心 Printer 基类与方法分发机制](#3-核心-printer-基类与方法分发机制)
4. [MathML 打印器深度分析](#4-mathml-打印器深度分析)
5. [各后端共享配置与特化差异](#5-各后端共享配置与特化差异)
6. [回退机制的真实流程](#6-回退机制的真实流程)
7. [设计模式与架构总结](#7-设计模式与架构总结)

---

## 1. 概述

SymPy 的打印子系统采用**访问者模式**的变体，通过方法名分发机制实现表达式类型与打印处理器的解耦。本报告修正了之前分析中的错误，特别是关于类层次结构、MathML 实现和回退机制的细节。

### 1.1 模块文件结构

```
sympy/printing/
├── printer.py              # 核心 Printer 基类（分发机制核心）
├── str.py                  # StrPrinter - 基础字符串打印
├── repr.py                 # ReprPrinter - 可执行字符串
├── latex.py                # LatexPrinter - LaTeX 格式
├── mathml.py               # MathML 打印器（含 Content 和 Presentation 两种）
├── pretty/                 # 字符艺术打印
│   ├── pretty.py           # PrettyPrinter
│   ├── pretty_symbology.py # Unicode/ASCII 符号表
│   └── stringpict.py       # 2D 字符串排版
├── codeprinter.py          # CodePrinter - 代码生成基类（继承自 StrPrinter）
├── c.py                    # C89/C99/C11 代码打印
├── cxx.py                  # C++ 代码打印（使用混入模式）
├── fortran.py              # Fortran 代码打印
├── julia.py                # Julia 代码打印
├── rust.py                 # Rust 代码打印
├── pycode.py               # Python 代码生成（AbstractPythonCodePrinter, PythonCodePrinter）
├── python.py               # PythonPrinter - 带符号声明的可执行代码
├── numpy.py                # NumPy/SciPy/CuPy/JAX 打印器（多继承）
├── pytorch.py              # PyTorch 打印器
├── tensorflow.py           # TensorFlow 打印器
├── jscode.py               # JavaScript 打印器
├── rcode.py               # R 语言打印器
├── octave.py              # Octave/Matlab 打印器
├── mathematica.py         # Mathematica 打印器
├── maple.py               # Maple 打印器
├── glsl.py                # GLSL 打印器
├── smtlib.py              # SMT-LIB 格式
├── lambdarepr.py          # Lambda 表达式表示
├── llvmjitcode.py         # LLVM JIT 打印器
├── aesaracode.py          # Aesara 打印器
├── theanocode.py          # Theano 打印器
├── precedence.py          # 运算符优先级定义
├── conventions.py         # 命名约定工具
├── defaults.py            # 默认配置
├── preview.py             # LaTeX 预览
├── tree.py                # 表达式树可视化
├── dot.py                 # Graphviz DOT 格式
└── tableform.py           # 表格格式化
```

---

## 2. 真实类层次结构

### 2.1 核心继承链（修正版）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Printer (最顶层基类)                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ printmethod: str = None                                                  │ │
│  │ _default_settings: dict                                                  │ │
│  │ _global_settings: dict                                                   │ │
│  │ _print(expr) - 三级分发核心                                              │ │
│  │ doprint(expr) - 入口方法                                                 │ │
│  │ emptyPrinter(expr) - 兜底方法                                            │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
         ▲                    ▲                    ▲                    ▲
         │                    │                    │                    │
┌────────┴────────┐  ┌──────┴────────┐  ┌──────┴────────┐  ┌──────┴────────┐
│   StrPrinter    │  │  ReprPrinter  │  │  LatexPrinter │  │ PrettyPrinter │
│  (基础字符串)    │  │  (可执行 repr) │  │  (LaTeX 格式)  │  │ (字符艺术 2D)  │
└─────────────────┘  └─────────────────┘  └─────────────────┘  └─────────────────┘
         ▲
         │
┌────────┴─────────────────────────────────────────────────────────────────────┐
│                          CodePrinter                                            │
│  (代码生成基类 - 继承自 StrPrinter，非直接继承 Printer)                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ _operators: dict          # 运算符映射（如 'and' → '&&'）                │ │
│  │ _rewriteable_functions: dict  # 可重写函数表                            │ │
│  │ _print_not_supported(expr)  # 不支持表达式处理                           │ │
│  │ 抽象方法（必须实现）：                                                     │ │
│  │   _get_statement(), _get_comment(), _format_code()                       │ │
│  │   _get_loop_opening_ending(), _declare_number_const()                    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
         ▲
         ├──────────────────────┬──────────────────────┬──────────────────────┐
         │                      │                      │                      │
┌────────┴────────┐  ┌──────────┴──────────┐  ┌─────┴──────────┐  ┌──────┴─────────┐
│ C89CodePrinter  │  │ AbstractPythonCode   │  │ FCodePrinter   │  │ JuliaCodePrinter│
│ (C89 标准)       │  │ Printer (Python 基类)│  │ (Fortran)       │  │ (Julia)         │
└─────────────────┘  └──────────────────────┘  └─────────────────┘  └─────────────────┘
         ▲                      ▲
         │                      │
┌────────┴────────┐  ┌──────────┴──────────┐
│ C99CodePrinter  │  │   PythonCodePrinter  │
│ (继承 C89)       │  │   (完整 Python 代码)  │
└─────────────────┘  └──────────┬──────────┘
         ▲                       │
         │                       ├─────────────────────────────────┐
┌────────┴────────┐    ┌────────┴────────┐              ┌────────┴────────┐
│ C11CodePrinter  │    │  ArrayPrinter    │              │ LambdaPrinter    │
│ (继承 C99)       │    │  (数组操作混入)   │              │ (Lambda 表达式)  │
└─────────────────┘    └────────┬────────┘              └─────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
┌────────┴────────┐    ┌────────┴────────┐    ┌────────┴────────┐
│  NumPyPrinter   │    │  TorchPrinter   │    │TensorflowPrinter│
│ (多继承: Array  │    │ (多继承: Array   │    │ (多继承: Array   │
│  + PythonCode)  │    │  + AbstractPy)   │    │  + AbstractPy)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         ▲
         ├──────────────┬──────────────┐
         │              │              │
┌────────┴──────┐ ┌─────┴──────┐ ┌─────┴──────┐
│ SciPyPrinter  │ │ CuPyPrinter │ │ JaxPrinter  │
│  (继承 NumPy)  │ │ (继承 NumPy) │ │ (继承 NumPy) │
└───────────────┘ └─────────────┘ └─────────────┘
```

### 2.2 多继承与混入模式

#### PythonPrinter（`python.py:11`）
```python
class PythonPrinter(ReprPrinter, StrPrinter):
    """A printer which converts an expression into its Python interpretation."""
```
- **多继承目的**：
  - 从 `ReprPrinter` 继承可执行代码生成逻辑
  - 从 `StrPrinter` 继承基础打印方法
  - 在 `__init__` 中动态将 `StrPrinter` 的部分方法绑定到自身

#### NumPyPrinter（`numpy.py:38`）
```python
class NumPyPrinter(ArrayPrinter, PythonCodePrinter):
    """
    Numpy printer which handles vectorized piecewise functions,
    logical operators, etc.
    """
```
- **多继承目的**：
  - `ArrayPrinter`：提供数组操作的打印方法（矩阵乘法、逆矩阵等）
  - `PythonCodePrinter`：提供基础 Python 代码生成

#### C++ 打印器（`cxx.py:123-160`）- 混入模式
```python
class _CXXCodePrinterBase:
    """C++ 特有功能的混入类"""
    printmethod = "_cxxcode"
    language = 'C++'
    _ns = 'std::'  # 命名空间前缀

class CXX98CodePrinter(_CXXCodePrinterBase, C89CodePrinter):
    standard = 'C++98'

class CXX11CodePrinter(_CXXCodePrinterBase, C99CodePrinter):
    standard = 'C++11'

class CXX17CodePrinter(_CXXCodePrinterBase, C99CodePrinter):
    standard = 'C++17'
```

### 2.3 MathML 类层次

```
┌─────────────────────────────────────────────────────────────────┐
│                    MathMLPrinterBase (Printer)                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ _default_settings: dict                                    │  │
│  │ self.dom: Document (XML DOM)                               │  │
│  │ doprint(expr) - 覆盖基类，返回 XML 字符串                   │  │
│  │ _split_super_sub(name) - 处理上下标                        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
         ▲                    ▲
         │                    │
┌────────┴────────┐  ┌────────┴────────────────┐
│MathMLContent    │  │ MathMLPresentation       │
│Printer          │  │ Printer                 │
│ (内容 MathML)    │  │ (展示 MathML)          │
│ printmethod =   │  │ printmethod =           │
│ "_mathml_content"│  │ "_mathml_presentation"  │
│ 用于语义网/      │  │ 用于网页渲染            │
│ 定理证明         │  │                        │
└─────────────────┘  └─────────────────────────┘
```

---

## 3. 核心 Printer 基类与方法分发机制

### 3.1 三级分发机制（准确版）

`_print` 方法定义在 `printer.py:295-336`，是整个打印系统的核心：

```python
def _print(self, expr, **kwargs) -> str:
    """Internal dispatcher
    三级分发机制：
        1. Let the object print itself if it knows how.
        2. Take the best fitting method defined in the printer.
        3. As fall-back use the emptyPrinter method for the printer.
    """
    self._print_level += 1
    try:
        # ========== 第一级：对象自打印（优先级最高） ==========
        # 如果打印器定义了 printmethod 且对象有该方法
        if self.printmethod and hasattr(expr, self.printmethod):
            if not (isinstance(expr, type) and issubclass(expr, Basic)):
                return getattr(expr, self.printmethod)(self, **kwargs)

        # ========== 第二级：方法名分发（沿 MRO 继承链） ==========
        # 获取表达式类型的方法解析顺序
        classes = type(expr).__mro__
        
        # 特殊处理 1：忽略 UndefinedFunction 子类
        # 例如 Function('gamma') 不会错误地分发到 _print_gamma
        if AppliedUndef in classes:
            classes = classes[classes.index(AppliedUndef):]
        if UndefinedFunction in classes:
            classes = classes[classes.index(UndefinedFunction):]
        
        # 特殊处理 2：用户自定义函数的名称匹配
        # 如果有人子类化已知函数（如 gamma）但改了名，忽略 _print_gamma
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
        
        # ========== 第三级：兜底回退 ==========
        return self.emptyPrinter(expr)
    finally:
        self._print_level -= 1
```

### 3.2 各打印器的 printmethod 值

| 打印器类 | printmethod 值 | 对象需实现方法 | 用途 |
|----------|----------------|----------------|------|
| `StrPrinter` | `"_sympystr"` | `def _sympystr(self, printer):` | 基础字符串 |
| `ReprPrinter` | `"_sympyrepr"` | `def _sympyrepr(self, printer):` | 可执行 repr |
| `LatexPrinter` | `"_latex"` | `def _latex(self, printer):` | LaTeX |
| `PrettyPrinter` | `"_pretty"` | `def _pretty(self, printer):` | 字符艺术 |
| `MathMLContentPrinter` | `"_mathml_content"` | `def _mathml_content(self, printer):` | 内容 MathML |
| `MathMLPresentationPrinter` | `"_mathml_presentation"` | `def _mathml_presentation(self, printer):` | 展示 MathML |
| `CodePrinter` | 未定义 | - | 代码生成基类 |
| `C89CodePrinter` | `"_ccode"` | `def _ccode(self, printer):` | C 代码 |
| `FCodePrinter` | `"_fcode"` | `def _fcode(self, printer):` | Fortran |
| `AbstractPythonCodePrinter` | `"_pythoncode"` | `def _pythoncode(self, printer):` | Python 代码 |
| `JuliaCodePrinter` | `"_julia"` | `def _julia(self, printer):` | Julia |
| `RustCodePrinter` | 未定义 | - | Rust |
| `CXXCodePrinter` | `"_cxxcode"` | `def _cxxcode(self, printer):` | C++ |

### 3.3 设置管理机制（三层优先级）

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
    self._context = {}  # 打印过程中的临时上下文
    
    if settings is not None:                          # 第三层：实例覆盖
        self._settings.update(settings)
        # 验证：拒绝未知设置
        if len(self._settings) > len(self._default_settings):
            for key in self._settings:
                if key not in self._default_settings:
                    raise TypeError("Unknown setting '%s'." % key)
```

**优先级顺序**：
```
实例 settings 参数 > 类 _global_settings > 类 _default_settings
```

---

## 4. MathML 打印器深度分析

### 4.1 MathMLPrinterBase 基类架构

定义在 `mathml.py:20-81`：

```python
class MathMLPrinterBase(Printer):
    """Contains common code required for MathMLContentPrinter and
    MathMLPresentationPrinter.
    """

    _default_settings = {
        "order": None,
        "encoding": "utf-8",
        "fold_frac_powers": False,
        "fold_func_brackets": False,
        "fold_short_frac": None,
        "inv_trig_style": "abbreviated",
        "ln_notation": False,
        "long_frac_ratio": None,
        "mat_delim": "[",
        "mat_symbol_style": "plain",
        "mul_symbol": None,
        "root_notation": True,
        "symbol_names": {},
        "mul_symbol_mathml_numbers": '&#xB7;',
        "disable_split_super_sub": False,
    }

    def __init__(self, settings=None):
        Printer.__init__(self, settings)
        from xml.dom.minidom import Document, Text

        self.dom = Document()  # XML DOM 文档对象

        # 自定义 Text 节点类，避免字符串转义
        class RawText(Text):
            def writexml(self, writer, indent='', addindent='', newl=''):
                if self.data:
                    writer.write('{}{}{}'.format(indent, self.data, newl))

        def createRawTextNode(data):
            r = RawText()
            r.data = data
            r.ownerDocument = self.dom
            return r

        self.dom.createTextNode = createRawTextNode

    def doprint(self, expr):
        """
        覆盖基类 doprint，返回 XML 字符串而非普通字符串
        """
        mathML = Printer._print(self, expr)  # 调用基类分发
        unistr = mathML.toxml()              # DOM 转 XML 字符串
        xmlbstr = unistr.encode('ascii', 'xmlcharrefreplace')
        res = xmlbstr.decode()
        return res
```

### 4.2 MathMLContentPrinter（内容 MathML）

定义在 `mathml.py:83-544`，用于**语义表示**（定理证明、语义网）：

```python
class MathMLContentPrinter(MathMLPrinterBase):
    printmethod = "_mathml_content"

    def mathml_tag(self, e):
        """SymPy 类名到 MathML 标签的映射"""
        translate = {
            'Add': 'plus',
            'Mul': 'times',
            'Derivative': 'diff',
            'Number': 'cn',        # 内容数字
            'Symbol': 'ci',        # 内容标识符
            'Integral': 'int',
            'Sum': 'sum',
            'sin': 'sin',
            'cos': 'cos',
            'log': 'ln',
            'Equality': 'eq',
            'Unequality': 'neq',
            'GreaterThan': 'geq',
            'LessThan': 'leq',
            # ... 更多映射
        }

        # 沿 MRO 查找第一个匹配的标签
        for cls in e.__class__.__mro__:
            n = cls.__name__
            if n in translate:
                return translate[n]
        # 兜底：小写类名
        return e.__class__.__name__.lower()
```

**Content MathML 示例**：
```python
from sympy import symbols, sin
from sympy.printing.mathml import MathMLContentPrinter

x = symbols('x')
expr = sin(x) + 1

printer = MathMLContentPrinter()
print(printer.doprint(expr))
```

输出（内容 MathML，语义化）：
```xml
<apply>
  <plus/>
  <apply>
    <sin/>
    <ci>x</ci>
  </apply>
  <cn>1</cn>
</apply>
```

### 4.3 MathMLPresentationPrinter（展示 MathML）

定义在 `mathml.py:546-1430+`，用于**视觉渲染**（网页、文档）：

```python
class MathMLPresentationPrinter(MathMLPrinterBase):
    printmethod = "_mathml_presentation"

    def mathml_tag(self, e):
        """展示 MathML 使用不同的标签映射"""
        translate = {
            'Number': 'mn',        # 展示数字
            'Symbol': 'mi',        # 展示标识符
            'Integral': '&int;',   # 积分符号实体
            'Sum': '&#x2211;',     # 求和符号 Unicode
            'Derivative': '&dd;',  # 微分符号
            'Equality': '=',
            'Unequality': '&#x2260;',
            # ... 更多展示标签
        }
        
        # 乘法符号可配置
        def mul_symbol_selection():
            if (self._settings["mul_symbol"] is None or
                    self._settings["mul_symbol"] == 'None'):
                return '&InvisibleTimes;'  # 不可见乘号
            elif self._settings["mul_symbol"] == 'times':
                return '&#xD7;'      # ×
            elif self._settings["mul_symbol"] == 'dot':
                return '&#xB7;'      # ·
            # ...
        
        for cls in e.__class__.__mro__:
            n = cls.__name__
            if n in translate:
                return translate[n]
        # 特殊：Mul 使用可配置的乘号
        if e.__class__.__name__ == "Mul":
            return mul_symbol_selection()
        return e.__class__.__name__.lower()
```

**Presentation MathML 示例**：
```python
from sympy.printing.mathml import MathMLPresentationPrinter

printer = MathMLPresentationPrinter()
print(printer.doprint(expr))
```

输出（展示 MathML，可视化）：
```xml
<mrow>
  <mrow>
    <mi>sin</mi>
    <mo>(</mo>
    <mi>x</mi>
    <mo>)</mo>
  </mrow>
  <mo>+</mo>
  <mn>1</mn>
</mrow>
```

### 4.4 Content vs Presentation 对比

| 特性 | Content MathML | Presentation MathML |
|------|----------------|----------------------|
| **目的** | 语义表示、机器可理解 | 视觉渲染、人类可读 |
| **标签** | `<cn>`, `<ci>`, `<apply>` | `<mn>`, `<mi>`, `<mrow>` |
| **结构** | 函数式：`<apply><sin/><ci>x</ci></apply>` | 流式：`<mi>sin</mi><mo>(</mo><mi>x</mi><mo>)</mo>` |
| **用途** | 定理证明、语义网、计算机代数 | 网页渲染、文档出版 |
| **printmethod** | `"_mathml_content"` | `"_mathml_presentation"` |

### 4.5 _print 方法返回类型的特殊性

**注意**：MathML 打印器的 `_print_*` 方法返回的不是字符串，而是 **DOM 节点对象**：

```python
# mathml.py:414-435 (Content Printer 的 _print_Pow)
def _print_Pow(self, e):
    # 使用 root 而非 power 如果指数是整数的倒数
    if (self._settings['root_notation'] and e.exp.is_Rational
            and e.exp.p == 1):
        x = self.dom.createElement('apply')
        x.appendChild(self.dom.createElement('root'))
        if e.exp.q != 2:
            xmldeg = self.dom.createElement('degree')
            xmlcn = self.dom.createElement('cn')
            xmlcn.appendChild(self.dom.createTextNode(str(e.exp.q)))
            xmldeg.appendChild(xmlcn)
            x.appendChild(xmldeg)
        x.appendChild(self._print(e.base))
        return x  # 返回 DOM 元素，不是字符串！

    x = self.dom.createElement('apply')
    x_1 = self.dom.createElement(self.mathml_tag(e))
    x.appendChild(x_1)
    x.appendChild(self._print(e.base))
    x.appendChild(self._print(e.exp))
    return x  # 返回 DOM 元素
```

这就是为什么 `MathMLPrinterBase.doprint` 需要覆盖基类：
```python
def doprint(self, expr):
    mathML = Printer._print(self, expr)  # 返回 DOM 节点
    unistr = mathML.toxml()              # DOM → XML 字符串
    # ...
```

---

## 5. 各后端共享配置与特化差异

### 5.1 _default_settings 继承与覆盖机制

配置通过字典更新实现继承：
```python
# 子类使用 dict() 包装父类配置
_default_settings = dict(
    CodePrinter._default_settings,  # 继承父类
    user_functions={},               # 新增或覆盖
    precision=17,
    inline=True,
)
```

### 5.2 各打印器配置对比表

| 配置项 | Printer (基类) | StrPrinter | LatexPrinter | CodePrinter | AbstractPythonCodePrinter |
|--------|---------------|------------|--------------|-------------|---------------------------|
| **order** | ❌ 无 | `None` | `None` | `None` | `None` |
| **full_prec** | ❌ 无 | `"auto"` | `False` | `"auto"` | - |
| **precision** | ❌ 无 | ❌ 无 | ❌ 无 | `17` | `17` |
| **inline** | ❌ 无 | ❌ 无 | ❌ 无 | `False` | `True` |
| **human** | ❌ 无 | ❌ 无 | ❌ 无 | `True` | - |
| **strict** | ❌ 无 | ❌ 无 | ❌ 无 | `None` | - |
| **error_on_reserved** | ❌ 无 | ❌ 无 | ❌ 无 | `False` | - |
| **user_functions** | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | `{}` |
| **contract** | ❌ 无 | ❌ 无 | ❌ 无 | `True` | `False` |
| **mode** | ❌ 无 | ❌ 无 | `"plain"` | ❌ 无 | - |
| **fold_frac_powers** | ❌ 无 | ❌ 无 | `False` | ❌ 无 | - |
| **use_unicode** | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | - |
| **imaginary_unit** | ❌ 无 | ❌ 无 | `"i"` | ❌ 无 | - |

### 5.3 各语言特化属性对比

| 属性 | C | Fortran | Julia | Python | C++ |
|------|---|---------|-------|--------|-----|
| **language** | `"C"` | `"Fortran"` | `"Julia"` | `"Python"` | `"C++"` |
| **printmethod** | `"_ccode"` | `"_fcode"` | `"_julia"` | `"_pythoncode"` | `"_cxxcode"` |
| **运算符 and** | `"&&"` | `".and."` | `"&&"` | `"and"` | `"&&"` |
| **运算符 or** | `"\|\|"` | `".or."` | `"\|\|"` | `"or"` | `"\|\|"` |
| **运算符 not** | `"!"` | `".not. "` | `"!"` | `"not"` | `"!"` |
| **语句结尾** | `";"` | `""` | `""` | `""` | `";"` |
| **注释前缀** | `"// "` | `"! "` | `"# "` | `"  # "` | `"// "` |
| **数组索引** | `"[i]"` | `"(i)"` | `"[i]"` | `"[i]"` | `"[i]"` |
| **数组起始** | 0 | 1 | 1 | 0 | 0 |
| **命名空间** | 无 | 无 | 无 | 模块导入 | `"std::"` |
| **保留字处理** | 后缀 `_` | 名称 mangling | - | 后缀 `_` | 后缀 `_` |

### 5.4 已知函数表机制

```python
# 简单映射
known_functions_C89 = {
    "sin": "sin",
    "cos": "cos",
    "sqrt": "sqrt",
}

# 条件映射（根据参数选择不同函数）
known_functions_C89 = {
    "Abs": [
        (lambda x: not x.is_integer, "fabs"),  # 浮点数用 fabs
        (lambda x: x.is_integer, "abs")         # 整数用 abs
    ],
}

# 自定义格式化函数
user_functions = {
    "Pow": [
        (lambda b, e: b == 2, lambda b, e: 'exp2(%s)' % e),  # 2^x → exp2(x)
        (lambda b, e: b != 2, 'pow')                          # 其他用 pow
    ]
}
```

### 5.5 函数调用链示例

以 `CodePrinter._print_Function` 为例（`codeprinter.py:437-463`）：

```
┌─────────────────────────────────────────────────────────────────┐
│              _print_Function(expr) 执行流程                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. 检查 known_functions 表                                       │
│     ├─ 简单映射: "sin" → "sin"                                   │
│     ├─ 条件映射: [(cond1, func1), (cond2, func2)]              │
│     └─ 自定义函数: 可调用格式化函数                               │
│                           ↓                                       │
│  2. 检查内联函数 (Lambda 定义的)                                  │
│     hasattr(expr, '_imp_') and isinstance(expr._imp_, Lambda)  │
│                           ↓                                       │
│  3. 检查可重写函数表 _rewriteable_functions                       │
│     如: 'cot' → 'tan', 'Mod' → 'floor'                          │
│                           ↓                                       │
│  4. 允许未知函数或报错                                            │
│     allow_unknown_functions=True → 直接打印                      │
│     否则 → _print_not_supported(expr)                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 回退机制的真实流程

### 6.1 三级回退概览

```
┌─────────────────────────────────────────────────────────────────┐
│                    表达式打印回退链（准确版）                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  优先级 1: 对象自打印（第一级分发）                                │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ if self.printmethod and hasattr(expr, self.printmethod):  │  │
│  │     return getattr(expr, self.printmethod)(self)          │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓（不存在）                              │
│  优先级 2: 方法名分发（沿 MRO 链）                                │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ for cls in type(expr).__mro__:                            │  │
│  │     printmethod = getattr(self, '_print_' + cls.__name__) │  │
│  │     if printmethod: return printmethod(expr)              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓（都不存在）                            │
│  优先级 3: emptyPrinter 兜底                                      │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ return self.emptyPrinter(expr)                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 各打印器的 emptyPrinter 实现

#### Printer 基类默认（`printer.py:338-339`）
```python
def emptyPrinter(self, expr):
    return str(expr)
```

#### StrPrinter 特化（`str.py:49-55`）
```python
def emptyPrinter(self, expr):
    if isinstance(expr, str):
        return expr
    elif isinstance(expr, Basic):
        return repr(expr)  # Basic 实例使用 repr
    else:
        return str(expr)
```

#### ReprPrinter 特化（`repr.py:33-49`）- **最复杂的兜底**
```python
def emptyPrinter(self, expr):
    """
    ReprPrinter 的多层兜底策略
    """
    if isinstance(expr, str):
        return expr
    elif hasattr(expr, "__srepr__"):
        return expr.__srepr__()  # 优先使用对象的 __srepr__
    elif hasattr(expr, "args") and hasattr(expr.args, "__iter__"):
        # 有 args 属性：尝试重构 ClassName(arg1, arg2, ...)
        l = []
        for o in expr.args:
            l.append(self._print(o))
        return expr.__class__.__name__ + '(%s)' % ', '.join(l)
    elif hasattr(expr, "__module__") and hasattr(expr, "__name__"):
        # 是类或模块：返回 '<module.Class>'
        return "<'%s.%s'>" % (expr.__module__, expr.__name__)
    else:
        return str(expr)  # 最终兜底
```

#### PrettyPrinter 特化（`pretty/pretty.py:56-57`）
```python
def emptyPrinter(self, expr):
    return prettyForm(str(expr))  # 包装为 prettyForm 对象
```

#### CodePrinter 特化 - 显式不支持的类型

**关键修正**：CodePrinter 不是通过继承链回退，而是**显式将某些类型映射到 `_print_not_supported`**：

```python
# codeprinter.py:621-643
# 这些类型被显式标记为不支持
_print_Basic = _print_not_supported
_print_ComplexInfinity = _print_not_supported
_print_ExprCondPair = _print_not_supported
_print_GeometryEntity = _print_not_supported
_print_Infinity = _print_not_supported
_print_Integral = _print_not_supported
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

**重要**：这意味着当打印 `Integral` 时，会找到 `_print_Integral`（映射到 `_print_not_supported`），不会继续沿 MRO 向上查找！

### 6.3 _print_not_supported 实现

```python
# codeprinter.py:608-619
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
    return self.emptyPrinter(expr)  # 最终兜底
```

### 6.4 函数重写回退机制

当函数不被直接支持时，尝试自动重写：

```python
# codeprinter.py:78-114
_rewriteable_functions = {
    # 三角函数重写
    'cot': ('tan', []),        # cot(x) = 1/tan(x)
    'csc': ('sin', []),        # csc(x) = 1/sin(x)
    'sec': ('cos', []),        # sec(x) = 1/cos(x)
    
    # 反三角函数
    'acot': ('atan', []),
    'acsc': ('asin', []),
    'asec': ('acos', []),
    
    # 双曲函数
    'coth': ('exp', []),
    'csch': ('exp', []),
    'sech': ('exp', []),
    
    # 特殊函数
    'Mod': ('floor', []),
    'factorial': ('gamma', []),
    'factorial2': ('gamma', ['Piecewise']),
    
    # 条件函数
    'Max': ('Piecewise', []),
    'Min': ('Piecewise', []),
    'Heaviside': ('Piecewise', []),
    
    # 更多...
}

# 重写逻辑（codeprinter.py:454-458）
elif expr.func.__name__ in self._rewriteable_functions:
    target_f, required_fs = self._rewriteable_functions[expr.func.__name__]
    # 检查目标函数和依赖函数是否都可打印
    if self._can_print(target_f) and all(self._can_print(f) for f in required_fs):
        return '(' + self._print(expr.rewrite(target_f)) + ')'
```

### 6.5 回退机制完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        完整回退流程图（以 CodePrinter 为例）                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  打印 expr = Integral(x**2, x)                                               │
│                           ↓                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 第一级：对象自打印                                                    │   │
│  │ 检查 hasattr(expr, '_ccode') → 不存在                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                           ↓                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 第二级：方法名分发（沿 MRO）                                          │   │
│  │ for cls in [Integral, Expr, Basic, object]:                         │   │
│  │   ├─ _print_Integral → 找到了！但它映射到 _print_not_supported     │   │
│  │   └─ 不会继续向上查找（因为已找到方法）                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                           ↓                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 执行 _print_not_supported(expr)                                      │   │
│  │   ├─ strict=True → 抛出 PrintMethodNotImplementedError             │   │
│  │   └─ strict=False → 记录到 _not_supported，调用 emptyPrinter       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                           ↓                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ emptyPrinter(expr)                                                    │   │
│  │ CodePrinter 没有覆盖 emptyPrinter，继承自 StrPrinter                 │   │
│  │   ├─ isinstance(expr, str)? → No                                    │   │
│  │   ├─ isinstance(expr, Basic)? → Yes                                 │   │
│  │   └─ return repr(expr) → "Integral(x**2, x)"                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  最终输出: "// Not supported in C:\n// Integral\nIntegral(x**2, x)"        │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.6 另一个回退示例：函数重写

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        函数重写回退示例                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  打印 expr = cot(x) (余切函数)                                               │
│                           ↓                                                   │
│  1. 检查 known_functions 中是否有 'cot' → C89 没有                          │
│                           ↓                                                   │
│  2. 检查 _rewriteable_functions['cot'] → 找到 ('tan', [])                   │
│                           ↓                                                   │
│  3. 检查 _can_print('tan') → 是，tan 在 known_functions 中                  │
│                           ↓                                                   │
│  4. 执行 expr.rewrite('tan') → cot(x) 重写为 1/tan(x)                      │
│                           ↓                                                   │
│  5. 递归打印 1/tan(x) → "1.0/tan(x)"                                         │
│                           ↓                                                   │
│  最终输出: "(1.0/tan(x))"                                                     │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 设计模式与架构总结

### 7.1 设计模式应用

| 设计模式 | 应用位置 | 实现方式 |
|----------|----------|----------|
| **访问者模式** | 核心分发机制 | `_print_<ClassName>` 方法约定 |
| **模板方法模式** | CodePrinter 抽象方法 | `_get_statement()`, `_get_comment()` 等 |
| **策略模式** | 设置管理 | 三层配置优先级 + 上下文管理器 |
| **责任链模式** | 回退机制 | MRO 方法查找 + 多层兜底 |
| **组合模式** | PrettyPrinter | `stringPict`, `prettyForm` 2D 排版 |
| **混入模式** | C++ 打印器 | `_CXXCodePrinterBase` + C89/C99 |
| **多继承** | NumPy/PythonPrinter | `ArrayPrinter` + `PythonCodePrinter` |
| **工厂模式** | 打印器实例化 | `ccode()`, `fcode()`, `latex()` 等 |

### 7.2 关键数据结构

#### 1. 设置系统
```python
_global_settings    # 类变量，全局影响所有实例
_default_settings   # 类变量，子类可通过 dict() 扩展
_settings           # 实例变量，构造时传入
_context            # 临时上下文，打印过程中可变
```

#### 2. 函数映射表
```python
# 简单映射
"sin": "sin"

# 条件映射列表
"Abs": [
    (lambda x: not x.is_integer, "fabs"),
    (lambda x: x.is_integer, "abs")
]

# 自定义格式化函数
"Pow": [
    (lambda b, e: b == 2, lambda b, e: 'exp2(%s)' % e),
    (lambda b, e: b != 2, 'pow')
]
```

#### 3. 可重写函数表
```python
_rewriteable_functions = {
    'cot': ('tan', []),           # 函数名: (目标函数, [依赖函数])
    'factorial2': ('gamma', ['Piecewise']),
}
```

### 7.3 扩展指南

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

# 方法 3: 对象自打印（让表达式控制自身打印）
class MyCustomExpr(Basic):
    def _latex(self, printer):
        return r"\custom{%s}" % printer._print(self.arg)
    
    def _sympystr(self, printer):
        return "MyCustomExpr(%s)" % printer._print(self.arg)
```

#### 添加新的代码生成后端

```python
from sympy.printing.codeprinter import CodePrinter

class MyLangPrinter(CodePrinter):
    printmethod = "_mylang"
    language = "MyLang"
    reserved_words = {'if', 'else', 'while', 'for'}
    
    # 覆盖默认设置
    _default_settings = dict(CodePrinter._default_settings, **{
        'precision': 15,
        'inline': True,
    })
    
    # 运算符映射
    _operators = {
        'and': '&&',
        'or': '||',
        'not': '!',
    }
    
    # 必须实现的抽象方法
    def _get_statement(self, codestring):
        return codestring + ";"  # 类似 C
    
    def _get_comment(self, text):
        return "// " + text
    
    def _declare_number_const(self, name, value):
        return "const %s = %s;" % (name, value)
    
    def _format_code(self, lines):
        return self.indent_code(lines)
    
    def _get_loop_opening_ending(self, indices):
        open_lines = []
        close_lines = []
        for i in indices:
            var, start, stop = map(self._print,
                [i.label, i.lower, i.upper])
            open_lines.append("for (%s = %s; %s < %s; %s++) {" % 
                (var, start, var, stop, var))
            close_lines.append("}")
        return open_lines, close_lines
```

### 7.4 类继承关系完整图（修正版）

```
Printer
├── StrPrinter
│   └── CodePrinter
│       ├── C89CodePrinter
│       │   └── C99CodePrinter
│       │       └── C11CodePrinter
│       ├── FCodePrinter (Fortran)
│       ├── JuliaCodePrinter
│       ├── RustCodePrinter
│       ├── GLSLPrinter
│       ├── JavascriptCodePrinter
│       ├── RCodePrinter
│       ├── OctaveCodePrinter
│       ├── MCodePrinter (Mathematica)
│       ├── MapleCodePrinter
│       └── AbstractPythonCodePrinter
│           ├── PythonCodePrinter
│           │   ├── LambdaPrinter
│           │   │   ├── NumExprPrinter
│           │   │   └── MpmathPrinter
│           │   │       └── IntervalPrinter (多继承)
│           │   ├── CmathPrinter
│           │   └── SymPyPrinter
│           ├── TorchPrinter (多继承: ArrayPrinter)
│           └── TensorflowPrinter (多继承: ArrayPrinter)
│
├── ReprPrinter
│
├── PythonPrinter (多继承: ReprPrinter, StrPrinter)
│
├── LatexPrinter
│
├── PrettyPrinter
│
├── MathMLPrinterBase
│   ├── MathMLContentPrinter
│   └── MathMLPresentationPrinter
│
├── SMTLibPrinter
│
├── LLVMJitPrinter
│   └── LLVMJitCallbackPrinter
│
├── AesaraPrinter
│
└── TheanoPrinter


# 混入类（不单独使用）
_CXXCodePrinterBase
├── CXX98CodePrinter (多继承: _CXXCodePrinterBase, C89CodePrinter)
├── CXX11CodePrinter (多继承: _CXXCodePrinterBase, C99CodePrinter)
└── CXX17CodePrinter (多继承: _CXXCodePrinterBase, C99CodePrinter)

ArrayPrinter
├── NumPyPrinter (多继承: ArrayPrinter, PythonCodePrinter)
│   ├── SciPyPrinter
│   ├── CuPyPrinter
│   └── JaxPrinter
├── TorchPrinter (同时继承 AbstractPythonCodePrinter)
└── TensorflowPrinter (同时继承 AbstractPythonCodePrinter)
```

---

## 附录：关键修正点汇总

### 之前报告中的错误

| 错误内容 | 正确事实 |
|----------|----------|
| `CodePrinter` 直接继承 `Printer` | `CodePrinter` 继承自 `StrPrinter` |
| `PythonPrinter` 是简单继承 | `PythonPrinter` 是多继承：`ReprPrinter, StrPrinter` |
| MathML 只有一个打印器 | MathML 有两个：`ContentPrinter` 和 `PresentationPrinter` |
| C++ 打印器简单继承 | C++ 使用混入模式：`_CXXCodePrinterBase` + C89/C99 |
| 回退机制简单描述 | `ReprPrinter.emptyPrinter` 有 5 层回退，`CodePrinter` 显式映射不支持类型 |
| 函数重写是可选的 | `_rewriteable_functions` 是自动机制，`_print_Function` 会主动检查 |
| MathML 返回字符串 | MathML 返回 DOM 节点，`doprint` 覆盖基类转 XML |

---

**报告生成时间**: 2026-05-01  
**分析版本**: SymPy (当前开发版本)  
**修正版**: 第 2 版（修正类层次、MathML、回退机制）
