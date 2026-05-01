# SymPy 矩阵系统分层设计分析报告（修正版）

## 重要修正说明

本报告对上一版分析进行了以下关键修正：

1. **延迟求值范围**：明确区分 `_op_priority`（操作符分派）与延迟求值的关系
2. **稠密/稀疏优先关系**：澄清 `_class_priority` 实际反映的是**不可变性**，而非直接的稠密/稀疏偏好
3. **分派链路**：完整追溯从操作符调用到具体计算的完整流程
4. **`is_MatrixExpr` 标志**：强调这是区分表达式层与计算层的关键

---

## 1. 核心优先级体系

### 1.1 两种优先级的区别

SymPy 矩阵系统使用两种完全不同的优先级机制：

| 机制 | 属性 | 用途 | 决策点 |
|------|------|------|--------|
| 操作符分派优先级 | `_op_priority` | 决定谁的 `__mul__`/`__add__` 被调用 | 操作符重载时 |
| 类型统一优先级 | `_class_priority` | 决定混合运算结果的类型 | 具体矩阵计算时 |

### 1.2 `_op_priority` 值（操作符分派）

| 类型 | `_op_priority` | `is_MatrixExpr` | 层面 |
|------|---------------|-----------------|------|
| `MatrixExpr` | **11.0** | `True` | 表达式层 |
| `DenseMatrix` | 10.01 | `False` | 具体计算层 |
| `ImmutableDenseMatrix` | 10.001 | `False` | 具体计算层 |

**关键发现**：表达式层的 `_op_priority` (11.0) **高于所有具体矩阵类型**！

### 1.3 `_class_priority` 值（类型统一）

| 类型 | `_class_priority` | 特性 |
|------|-------------------|------|
| `MatrixBase` / `MatrixArithmetic` | 3 | 基类 |
| `DenseMatrix` | 4 | 可变稠密矩阵 |
| `ImmutableDenseMatrix` | 8 | **不可变**稠密矩阵 |
| `ImmutableSparseMatrix` | 9 | **不可变**稀疏矩阵 |

**关键发现**：
- 优先级反映的是**不可变性**（"immutability is contagious"）
- 不可变稀疏矩阵 (9) > 不可变稠密矩阵 (8)，但这是由于**不可变性**，而非"稀疏优先"
- 可变矩阵之间没有明确的稠密/稀疏优先级比较

### 1.4 `is_MatrixExpr` 标志的关键作用

这是区分两层的**核心标志**：

```python
# 表达式层 (sympy/matrices/expressions/matexpr.py:69)
class MatrixExpr(Expr):
    is_MatrixExpr: bool = True

# 具体计算层 (sympy/matrices/dense.py:44)
class DenseMatrix(RepMatrix):
    is_MatrixExpr: bool = False
```

**使用场景**：
1. **构造后处理器**：检测是否需要路由到 `MatMul`/`MatAdd`
2. **`merge_explicit` 规则**：检测是否是具体矩阵（`is_MatrixExpr=False`）
3. **标量吸收逻辑**：决定如何处理标量与矩阵的混合

---

## 2. 操作符分派链路完整分析

### 2.1 `@call_highest_priority` 装饰器机制

定义在 `sympy/core/decorators.py:83`：

```python
def call_highest_priority(method_name: str):
    """A decorator for binary special methods to handle _op_priority."""
    def priority_decorator(func):
        @wraps(func)
        def binary_op_wrapper(self, other):
            if hasattr(other, '_op_priority'):
                # 关键：比较优先级
                if other._op_priority > self._op_priority:
                    # 对方优先级更高，调用对方的反向方法
                    f = getattr(other, method_name, None)
                    if f is not None:
                        return f(self)
            # 优先级相同或对方无优先级，执行自身方法
            return func(self, other)
        return binary_op_wrapper
    return priority_decorator
```

**工作原理**：
1. 当执行 `A * B` 时，Python 首先调用 `A.__mul__(B)`
2. 装饰器检查 `B._op_priority > A._op_priority`
3. 如果是，转而调用 `B.__rmul__(A)`
4. 否则继续执行 `A.__mul__(B)`

### 2.2 场景1：表达式层 × 具体矩阵

```python
from sympy import MatrixSymbol, Matrix

A = MatrixSymbol('A', 2, 2)  # 表达式层，_op_priority=11.0
B = Matrix([[1, 2], [3, 4]])  # 具体层，_op_priority=10.01

result = A * B
```

**完整分派流程**：

```
┌─────────────────────────────────────────────────────────────────┐
│ 步骤1: Python 调用 A.__mul__(B)                                  │
│        (MatrixSymbol.__mul__)                                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤2: @call_highest_priority('__rmul__') 检查                  │
│                                                                   │
│        B._op_priority = 10.01                                    │
│        A._op_priority = 11.0                                     │
│                                                                   │
│        10.01 > 11.0?  False!                                     │
│        不切换到 B.__rmul__                                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤3: 执行 MatrixExpr.__mul__ (sympy/matrices/expressions/matexpr.py:128) │
│                                                                   │
│        @_sympifyit('other', NotImplemented)                      │
│        @call_highest_priority('__rmul__')                        │
│        def __mul__(self, other):                                 │
│            return MatMul(self, other).doit()                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤4: 构建 MatMul(A, B) 表达式树                                │
│                                                                   │
│        MatMul.args = (MatrixSymbol('A', 2, 2),                  │
│                        Matrix([[1,2],[3,4]]))                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤5: 调用 MatMul.doit() (sympy/matrices/expressions/matmul.py:190) │
│                                                                   │
│        def doit(self, **hints):                                   │
│            deep = hints.get('deep', True)                        │
│            if deep:                                                │
│                args = tuple(arg.doit(**hints) for arg in self.args) │
│            else:                                                   │
│                args = self.args                                    │
│            expr = canonicalize(MatMul(*args))                     │
│            return expr                                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤6: 规范化 canonicalize (应用规则链)                           │
│                                                                   │
│        规则链: distribute_monom → any_zeros → remove_ids → ...  │
│                   → merge_explicit → factor_in_front → ...      │
│                                                                   │
│        关键规则: merge_explicit (matmul.py:254)                  │
│        ┌──────────────────────────────────────────────────────┐  │
│        │ def merge_explicit(matmul):                            │  │
│        │     # 检查是否有多个相邻的 MatrixBase                   │  │
│        │     if not any(isinstance(arg, MatrixBase)             │  │
│        │            for arg in matmul.args):                     │  │
│        │         return matmul  # 不处理                          │  │
│        │                                                          │  │
│        │     # 只有当有多个相邻的 MatrixBase 时才合并             │  │
│        │     # 例如: MatMul(A, Matrix1, Matrix2, B)             │  │
│        │     #       → MatMul(A, Matrix1*Matrix2, B)            │  │
│        └──────────────────────────────────────────────────────┘  │
│                                                                   │
│        本场景: MatMul(A, B) 只有 1 个 MatrixBase                  │
│              merge_explicit 不触发合并计算                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 结果: 返回 MatMul(MatrixSymbol('A', 2, 2), Matrix([[1,2],[3,4]])) │
│        仍然是表达式树，未立即计算                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 场景2：具体矩阵 × 表达式层

```python
from sympy import MatrixSymbol, Matrix

B = Matrix([[1, 2], [3, 4]])  # 具体层，_op_priority=10.01
A = MatrixSymbol('A', 2, 2)  # 表达式层，_op_priority=11.0

result = B * A
```

**完整分派流程**：

```
┌─────────────────────────────────────────────────────────────────┐
│ 步骤1: Python 调用 B.__mul__(A)                                  │
│        (DenseMatrix.__mul__ → multiply)                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤2: @call_highest_priority('__rmul__') 检查                  │
│                                                                   │
│        A._op_priority = 11.0                                     │
│        B._op_priority = 10.01                                    │
│                                                                   │
│        11.0 > 10.01?  True!                                      │
│        切换到 A.__rmul__(B)                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤3: 执行 MatrixExpr.__rmul__ (sympy/matrices/expressions/matexpr.py:140) │
│                                                                   │
│        @_sympifyit('other', NotImplemented)                      │
│        @call_highest_priority('__mul__')                         │
│        def __rmul__(self, other):                                │
│            return MatMul(other, self).doit()                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤4-6: 与场景1相同                                              │
│        构建 MatMul(B, A) → 调用 doit() → 规范化                  │
│        由于只有 1 个 MatrixBase，merge_explicit 不触发合并        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 结果: 返回 MatMul(Matrix([[1,2],[3,4]]), MatrixSymbol('A', 2, 2)) │
│        仍然是表达式树，未立即计算                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.4 场景3：具体矩阵 × 具体矩阵

```python
from sympy import Matrix

B1 = Matrix([[1, 2], [3, 4]])  # 具体层，_op_priority=10.01
B2 = Matrix([[5, 6], [7, 8]])  # 具体层，_op_priority=10.01

result = B1 * B2
```

**完整分派流程**：

```
┌─────────────────────────────────────────────────────────────────┐
│ 步骤1: Python 调用 B1.__mul__(B2)                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤2: @call_highest_priority 检查                                │
│                                                                   │
│        B1._op_priority = 10.01                                   │
│        B2._op_priority = 10.01                                   │
│                                                                   │
│        优先级相同，不切换                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤3: 执行 DenseMatrix.__mul__ → multiply (sympy/matrices/common.py:2798) │
│                                                                   │
│        def multiply(self, other, dotprodsimp=None):              │
│            # ...                                                  │
│            if getattr(other, 'is_Matrix', False):                │
│                # 关键：调用具体计算方法                            │
│                m = self._eval_matrix_mul(other)                   │
│                return m                                            │
│            # ...                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤4: 执行 _eval_matrix_mul (sympy/matrices/repmatrix.py:382)  │
│                                                                   │
│        def _eval_matrix_mul(self, other: RepMatrix):             │
│            # 关键：委托给 DomainMatrix 进行实际计算                │
│            return classof(self, other)._fromrep(self._rep * other._rep) │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤5: 类型统一 classof (sympy/matrices/matrixbase.py:5653)     │
│                                                                   │
│        def classof(A, B):                                         │
│            priority_A = getattr(A, '_class_priority', None)      │
│            priority_B = getattr(B, '_class_priority', None)      │
│            if A._class_priority > B._class_priority:              │
│                return A.__class__                                  │
│            else:                                                   │
│                return B.__class__                                  │
│                                                                   │
│        本场景: B1._class_priority = 4, B2._class_priority = 4    │
│              返回 DenseMatrix                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 结果: 返回具体的 Matrix 结果                                      │
│        Matrix([[19, 22], [43, 50]])                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 延迟求值的精确范围

### 3.1 延迟求值的定义

**延迟求值 = 构建表达式树，不立即执行数值计算**

### 3.2 延迟求值场景对照表

| 场景 | 示例 | 是否延迟 | 原因 |
|------|------|----------|------|
| 表达式 × 表达式 | `MatrixSymbol * MatrixSymbol` | ✅ 是 | 表达式层优先级高，构建 `MatMul` |
| 表达式 × 具体 | `MatrixSymbol * Matrix` | ✅ 是 | 表达式层优先级高，构建 `MatMul` |
| 具体 × 表达式 | `Matrix * MatrixSymbol` | ✅ 是 | 表达式层优先级高，切换到 `__rmul__` |
| 具体 × 具体 | `Matrix * Matrix` | ❌ 否 | 优先级相同，立即调用 `_eval_matrix_mul` |
| `doit()` 中有多个相邻具体矩阵 | `MatMul(A, M1, M2, B).doit()` | 部分计算 | `merge_explicit` 合并 `M1*M2` |

### 3.3 延迟求值边界的关键代码

**表达式层操作符重载** (`sympy/matrices/expressions/matexpr.py:108-156`)：

```python
@_sympifyit('other', NotImplemented)
@call_highest_priority('__radd__')
def __add__(self, other):
    return MatAdd(self, other).doit()  # 构建表达式树

@_sympifyit('other', NotImplemented)
@call_highest_priority('__rmul__')
def __mul__(self, other):
    return MatMul(self, other).doit()  # 构建表达式树
```

**具体层操作符重载** (`sympy/matrices/common.py:2767-2839`)：

```python
@call_highest_priority('__rmul__')
def __mul__(self, other):
    return self.multiply(other)

def multiply(self, other, dotprodsimp=None):
    # ...
    if getattr(other, 'is_Matrix', False):
        # 关键：立即调用具体计算
        m = self._eval_matrix_mul(other)
        return m
    # ...
```

### 3.4 `merge_explicit` 规则的精确行为

**MatMul.merge_explicit** (`sympy/matrices/expressions/matmul.py:254`)：

```python
def merge_explicit(matmul):
    """ Merge explicit MatrixBase arguments
    
    只有当有多个相邻的 MatrixBase 时才合并计算
    """
    if not any(isinstance(arg, MatrixBase) for arg in matmul.args):
        return matmul
    
    newargs = []
    last = matmul.args[0]
    for arg in matmul.args[1:]:
        # 关键：只有当 last 和 arg 都是 MatrixBase 或 Number 时才合并
        if isinstance(arg, (MatrixBase, Number)) and isinstance(last, (MatrixBase, Number)):
            last = last * arg  # 立即计算！
        else:
            newargs.append(last)
            last = arg
    newargs.append(last)
    
    return MatMul(*newargs)
```

**示例分析**：

```python
from sympy import MatrixSymbol, Matrix

A = MatrixSymbol('A', 2, 2)
M1 = Matrix([[1, 2], [3, 4]])
M2 = Matrix([[5, 6], [7, 8]])
B = MatrixSymbol('B', 2, 2)

# 场景1: 只有一个具体矩阵
expr1 = MatMul(A, M1, B)
result1 = merge_explicit(expr1)
# 结果: MatMul(A, M1, B) - 不变，没有相邻的具体矩阵

# 场景2: 有多个相邻的具体矩阵
expr2 = MatMul(A, M1, M2, B)
result2 = merge_explicit(expr2)
# 结果: MatMul(A, M1*M2, B) - M1*M2 被计算为具体矩阵
```

---

## 4. 稠密/稀疏矩阵的优先关系

### 4.1 关键澄清

**上一版表述问题**：暗示存在直接的"稠密优先"或"稀疏优先"机制

**实际情况**：
- `_class_priority` 主要反映的是**不可变性**，而非直接的稠密/稀疏偏好
- 稠密/稀疏的选择由用户显式指定或 `DomainMatrix` 内部处理

### 4.2 `_class_priority` 实际含义

| 类型 | `_class_priority` | 实际特性 |
|------|-------------------|----------|
| `MatrixBase` | 3 | 抽象基类 |
| `DenseMatrix` | 4 | 可变稠密矩阵 |
| `ImmutableDenseMatrix` | 8 | **不可变**稠密矩阵 |
| `ImmutableSparseMatrix` | 9 | **不可变**稀疏矩阵 |

**核心原则**：**不可变性具有传染性**（"immutability is contagious"）

```python
# sympy/matrices/matrixbase.py:5653
def classof(A: Tmat, B: Tmat) -> type[Tmat]:
    """
    Get the type of the result when combining matrices of different types.
    
    Currently the strategy is that immutability is contagious.
    """
    priority_A = getattr(A, '_class_priority', None)
    priority_B = getattr(B, '_class_priority', None)
    if None not in (priority_A, priority_B):
        if A._class_priority > B._class_priority:
            return A.__class__
        else:
            return B.__class__
    # ...
```

### 4.3 稠密/稀疏混合运算的实际行为

```python
from sympy import Matrix, SparseMatrix, ImmutableMatrix, ImmutableSparseMatrix

# 场景1: 可变稠密 × 可变稀疏
M_dense = Matrix([[1, 2], [3, 4]])
M_sparse = SparseMatrix([[1, 0], [0, 1]])

# 两者 _class_priority 相同（都继承自可变基类）
# 结果类型取决于具体实现...

# 场景2: 不可变稠密 × 不可变稀疏
IM_dense = ImmutableMatrix([[1, 2], [3, 4]])  # _class_priority=8
IM_sparse = ImmutableSparseMatrix([[1, 0], [0, 1]])  # _class_priority=9

result = IM_dense * IM_sparse
# 根据 classof: 9 > 8，返回 ImmutableSparseMatrix
```

### 4.4 稠密/稀疏的实际选择机制

**用户显式指定**：
```python
from sympy import Matrix, SparseMatrix

# 用户明确选择稠密矩阵
M1 = Matrix([[1, 2], [3, 4]])

# 用户明确选择稀疏矩阵
M2 = SparseMatrix([[1, 0], [0, 1]])
```

**`DomainMatrix` 内部处理**：

`DomainMatrix` 是 SymPy 内部使用的高效矩阵实现，它：
- 支持稠密和稀疏两种存储格式
- 根据数据特性自动选择或保持格式
- 在 `sympy/polys/matrices/` 目录下实现

**操作结果的稀疏性**：

某些操作可能改变矩阵的稀疏性：
- `稀疏 × 稀疏` → 可能更稠密或更稀疏
- `稠密 × 稀疏` → 通常是稠密
- `稀疏 + 稀疏` → 取决于非零元素位置

---

## 5. 构造后处理器的角色

### 5.1 机制概述

`Basic._constructor_postprocessor_mapping` 确保即使是通过核心 `Mul`/`Add` 构造的表达式也能正确路由到矩阵类。

**注册代码** (`sympy/matrices/expressions/matexpr.py:528`)：

```python
Basic._constructor_postprocessor_mapping[MatrixExpr] = {
    "Mul": [get_postprocessor(Mul)],
    "Add": [get_postprocessor(Add)],
}
```

### 5.2 后处理器实现

```python
def get_postprocessor(cls):
    def _postprocessor(expr):
        mat_class = {Mul: MatMul, Add: MatAdd}[cls]
        nonmatrices = []
        matrices = []
        
        # 分离矩阵参数和非矩阵参数
        for term in expr.args:
            if isinstance(term, MatrixExpr):
                matrices.append(term)
            else:
                nonmatrices.append(term)
        
        if not matrices:
            return cls._from_args(nonmatrices)
        
        # 处理标量因子
        if nonmatrices:
            if cls == Mul:
                for i in range(len(matrices)):
                    if not matrices[i].is_MatrixExpr:  # 关键：检查 is_MatrixExpr
                        # 如果是具体矩阵，将标量吸收进去
                        matrices[i] = matrices[i].__mul__(cls._from_args(nonmatrices))
                        nonmatrices = []
                        break
            else:
                # Add 的情况
                return cls._from_args(nonmatrices + [mat_class(*matrices).doit(deep=False)])
        
        # 路由到矩阵类
        if mat_class == MatAdd:
            return mat_class(*matrices).doit(deep=False)
        return mat_class(cls._from_args(nonmatrices), *matrices).doit(deep=False)
    
    return _postprocessor
```

### 5.3 触发场景

```python
from sympy import MatrixSymbol, symbols

# 场景: 通过 Mul 构造（可能来自其他算法）
A = MatrixSymbol('A', 2, 2)
x = symbols('x')

# 如果某些算法内部使用了 Mul(2, A)
# 后处理器会将其转换为 MatMul(2, A)
```

---

## 6. 完整分派链路总结

### 6.1 决策流程图

```
用户执行: A * B
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段1: 操作符分派 (@call_highest_priority)                    │
│                                                               │
│   比较 A._op_priority vs B._op_priority                      │
│                                                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ 表达式层 (_op_priority=11.0)                         │   │
│   │ 优先级最高，接管所有包含表达式层的运算                  │   │
│   │ → 构建 MatMul/MatAdd 表达式树                        │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ 具体矩阵之间 (_op_priority=10.01)                   │   │
│   │ 优先级相同，执行具体计算                              │   │
│   │ → 调用 _eval_matrix_mul 立即计算                     │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
     │
     ▼ （如果构建了表达式树）
┌─────────────────────────────────────────────────────────────┐
│ 阶段2: 表达式树规范化 (doit() → canonicalize)                │
│                                                               │
│   应用规则链:                                                  │
│   - remove_ids: 移除单位矩阵                                  │
│   - any_zeros: 检测零矩阵                                    │
│   - merge_explicit: 合并相邻的具体矩阵 ⭐ 关键               │
│     └─ 只有当有多个相邻的 MatrixBase 时才触发实际计算         │
└─────────────────────────────────────────────────────────────┘
     │
     ▼ （如果执行具体计算）
┌─────────────────────────────────────────────────────────────┐
│ 阶段3: 类型统一与具体计算                                     │
│                                                               │
│   1. classof(A, B): 根据 _class_priority 选择结果类型        │
│      └─ 主要考虑不可变性，而非直接的稠密/稀疏                 │
│                                                               │
│   2. _eval_matrix_mul: 委托给 DomainMatrix 实际计算          │
│      └─ DomainMatrix 内部处理稠密/稀疏存储格式                │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 关键标志对照表

| 标志/属性 | 表达式层 | 具体计算层 | 用途 |
|-----------|----------|------------|------|
| `is_MatrixExpr` | `True` | `False` | 区分两层的核心标志 |
| `_op_priority` | 11.0 | ~10.01 | 操作符分派决策 |
| `_class_priority` | 不使用 | 3~9 | 具体矩阵类型统一 |

---

## 7. 代码引用速查

### 7.1 核心机制

| 机制 | 文件位置 | 关键代码 |
|------|----------|----------|
| `@call_highest_priority` | `sympy/core/decorators.py:83` | 操作符分派装饰器 |
| `is_MatrixExpr` 标志 | `sympy/matrices/expressions/matexpr.py:69` | 表达式层标志 |
| `is_MatrixExpr = False` | `sympy/matrices/dense.py:44` | 具体层标志 |
| `_op_priority` 表达式层 | `sympy/matrices/expressions/matexpr.py:66` | `_op_priority = 11.0` |
| `_op_priority` 具体层 | `sympy/matrices/dense.py:46` | `_op_priority = 10.01` |
| `_class_priority` | `sympy/matrices/matrixbase.py:137` | 类型统一优先级 |
| `classof` 函数 | `sympy/matrices/matrixbase.py:5653` | 类型统一决策 |

### 7.2 操作符重载

| 操作 | 表达式层实现 | 具体层实现 |
|------|-------------|------------|
| `__add__` | `matexpr.py:108` | `common.py:2727` / `matrixbase.py:3005` |
| `__mul__` | `matexpr.py:128` | `common.py:2768` / `matrixbase.py:3047` |
| `__rmul__` | `matexpr.py:140` | `common.py:2984` / `matrixbase.py:3264` |

### 7.3 规则系统

| 规则 | 文件位置 | 作用 |
|------|----------|------|
| `merge_explicit` (MatMul) | `matmul.py:254` | 合并相邻具体矩阵 |
| `merge_explicit` (MatAdd) | `matadd.py:124` | 合并相邻具体矩阵 |
| `canonicalize` (MatMul) | `matmul.py:448` | 规则链应用 |
| `canonicalize` (MatAdd) | `matadd.py:159` | 规则链应用 |

---

## 8. 修正要点总结

### 8.1 对上一版的关键修正

| 主题 | 上一版表述 | 修正后表述 |
|------|-----------|-----------|
| 延迟求值范围 | 不够精确 | 明确：表达式层 `_op_priority` 最高，所有包含表达式层的运算都构建表达式树；只有具体矩阵之间的运算才立即计算 |
| 稠密/稀疏优先 | 未明确说明 | 澄清：`_class_priority` 主要反映**不可变性**，而非直接的稠密/稀疏偏好；不可变稀疏矩阵 (9) > 不可变稠密矩阵 (8) 是由于不可变性，而非"稀疏优先" |
| 分派链路 | 不够完整 | 完整追溯：`@call_highest_priority` → 优先级比较 → 方法切换 → 表达式树构建或具体计算 |
| `is_MatrixExpr` 标志 | 未强调其关键作用 | 强调：这是区分表达式层与计算层的**核心标志**，用于构造后处理器、`merge_explicit` 规则、标量吸收逻辑 |

### 8.2 核心决策原则

1. **操作符分派**：由 `_op_priority` 决定，表达式层 (11.0) > 具体层 (~10.01)
2. **类型统一**：由 `_class_priority` 决定，主要考虑不可变性
3. **延迟求值边界**：由 `is_MatrixExpr` 标志区分，表达式层构建树，具体层立即计算
4. **`merge_explicit` 触发条件**：需要**多个相邻**的具体矩阵 (`is_MatrixExpr=False`)

### 8.3 常见场景速答

| 问题 | 答案 |
|------|------|
| `MatrixSymbol * Matrix` 立即计算吗？ | ❌ 否，表达式层优先级高，构建 `MatMul` |
| `Matrix * MatrixSymbol` 立即计算吗？ | ❌ 否，切换到 `MatrixSymbol.__rmul__`，构建 `MatMul` |
| `Matrix * Matrix` 立即计算吗？ | ✅ 是，优先级相同，立即调用 `_eval_matrix_mul` |
| 稀疏矩阵和稠密矩阵运算，结果类型？ | 取决于 `_class_priority`，主要考虑不可变性 |
| `doit()` 会触发所有具体矩阵计算吗？ | ❌ 不会，`merge_explicit` 只合并**相邻**的具体矩阵 |
