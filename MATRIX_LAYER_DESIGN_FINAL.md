# SymPy 矩阵系统分层设计分析报告（最终修订版）

## 重要修正与验证说明

本报告基于**实际代码测试**和**完整代码路径对账**，对上一版分析进行了以下关键修正：

### ⚠️ 关键验证结果汇总

| 验证项 | 预期 | 实际验证结果 | 修正点 |
|--------|------|--------------|--------|
| 可变稠密 vs 可变稀疏优先级 | 不确定 | 稠密 (4) > 稀疏 (3) | 混合运算结果总是稠密矩阵 |
| 表达式层 × 具体层结果类型 | 表达式树 | ✅ 表达式树 (MatMul) | 验证通过 |
| 具体层 × 具体层结果类型 | 具体矩阵 | ✅ 具体矩阵 | 验证通过 |
| 不可变 × 可变 结果类型 | 不可变（传染性） | ✅ 不可变 | 验证通过 |
| 不可变稀疏 vs 不可变稠密 | 不可变稀疏 (9) > 不可变稠密 (8) | ✅ 不可变稀疏 | 验证通过 |

---

## 1. 核心优先级体系（验证版）

### 1.1 三种关键标志/属性的区别

| 属性/标志 | 表达式层 | 具体层（可变稠密） | 具体层（可变稀疏） | 用途 | 决策时机 |
|-----------|----------|-------------------|-------------------|------|----------|
| `_op_priority` | **11.0** | 10.01 | 10.01 | 操作符分派（谁的方法被调用） | 操作符重载时 |
| `_class_priority` | 不使用 | **4** | **3** | 类型统一（结果类型是什么） | 具体计算时 |
| `is_MatrixExpr` | **True** | **False** | **False** | 区分两层的核心标志 | 各种分支判断 |

### 1.2 继承链与优先级来源

```
MatrixBase (_class_priority=3)
    │
    ├── RepMatrix
    │       │
    │       ├── DenseMatrix (_class_priority=4 ← 覆盖父类)
    │       │       └── MutableDenseMatrix (_class_priority=4 ← 继承)
    │       │
    │       └── SparseRepMatrix (_class_priority=3 ← 继承自 MatrixBase)
    │               └── MutableSparseMatrix (_class_priority=3 ← 继承)
    │
    ├── ImmutableDenseMatrix (_class_priority=8)
    │
    └── ImmutableSparseMatrix (_class_priority=9)
```

**关键发现**：
- `DenseMatrix` 显式定义了 `_class_priority = 4`，覆盖了父类 `MatrixBase` 的 `3`
- `SparseRepMatrix` 没有定义自己的 `_class_priority`，所以继承了 `MatrixBase` 的 `3`
- 因此：**可变稠密 (4) > 可变稀疏 (3)**

### 1.3 实际优先级值对照表（验证版）

| 类名 | `_op_priority` | `_class_priority` | `is_MatrixExpr` |
|------|---------------|-------------------|-----------------|
| `MatrixSymbol` | **11.0** | 不使用 | **True** |
| `MutableDenseMatrix` (Matrix) | 10.01 | **4** | **False** |
| `MutableSparseMatrix` (SparseMatrix) | 10.01 | **3** | **False** |
| `ImmutableDenseMatrix` | 10.001 | 8 | False |
| `ImmutableSparseMatrix` | 10.001 | **9** | False |

---

## 2. 混合运算分派链路完整对账

### 2.1 决策流程图（验证版）

```
用户执行: A * B
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段1: 操作符分派 (@call_highest_priority)                    │
│                                                               │
│   比较 A._op_priority vs B._op_priority                      │
│                                                               │
│   优先级值:                                                    │
│   - 表达式层 (MatrixSymbol): 11.0  ⭐ 最高                   │
│   - 具体层 (Matrix/SparseMatrix): ~10.01                      │
│                                                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ 情况1: 有表达式层 (_op_priority=11.0)                │   │
│   │        表达式层优先级更高，接管运算                    │   │
│   │        → 构建 MatMul/MatAdd 表达式树                  │   │
│   │        → is_MatrixExpr = True                         │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ 情况2: 都是具体层 (_op_priority=~10.01)             │   │
│   │        优先级相同，执行具体计算                        │   │
│   │        → 调用 _eval_matrix_mul 立即计算               │   │
│   │        → is_MatrixExpr = False                        │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
     │
     ▼ （如果是具体计算）
┌─────────────────────────────────────────────────────────────┐
│ 阶段2: 类型统一 (classof)                                     │
│                                                               │
│   比较 A._class_priority vs B._class_priority               │
│                                                               │
│   优先级值:                                                    │
│   - ImmutableSparseMatrix: 9    ⭐ 最高                      │
│   - ImmutableDenseMatrix: 8                                 │
│   - MutableDenseMatrix (Matrix): 4                          │
│   - MutableSparseMatrix (SparseMatrix): 3                   │
│                                                               │
│   核心原则:                                                    │
│   1. 不可变性具有传染性（immutability is contagious）        │
│   2. 可变矩阵之间：稠密 (4) > 稀疏 (3)                       │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 关键代码路径对账

#### 场景A：表达式层 × 具体层（延迟求值）

**测试验证**：
```python
A = MatrixSymbol('A', 2, 2)  # _op_priority=11.0, is_MatrixExpr=True
M = Matrix([[1, 2], [3, 4]])  # _op_priority=10.01, is_MatrixExpr=False

result = A * M
# 结果类型: <class 'sympy.matrices.expressions.matmul.MatMul'>
# is_MatrixExpr: True
```

**代码路径**：

```
A * M
   │
   ▼
Python 调用 A.__mul__(M)
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤1: @call_highest_priority('__rmul__') 检查                  │
│                                                                   │
│ 装饰器代码 (sympy/core/decorators.py:83):                        │
│                                                                   │
│   def binary_op_wrapper(self, other):                             │
│       if hasattr(other, '_op_priority'):                         │
│           if other._op_priority > self._op_priority:            │
│               # 对方优先级更高，调用对方的反向方法                 │
│               f = getattr(other, method_name, None)              │
│               if f is not None:                                   │
│                   return f(self)                                  │
│       return func(self, other)                                    │
│                                                                   │
│ 本场景:                                                            │
│   A._op_priority = 11.0                                           │
│   M._op_priority = 10.01                                          │
│                                                                   │
│   M._op_priority > A._op_priority?  10.01 > 11.0?  False!      │
│   不切换，继续执行 A.__mul__(M)                                    │
└─────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤2: 执行 MatrixExpr.__mul__                                   │
│                                                                   │
│ 代码 (sympy/matrices/expressions/matexpr.py:128):               │
│                                                                   │
│   @_sympifyit('other', NotImplemented)                            │
│   @call_highest_priority('__rmul__')                             │
│   def __mul__(self, other):                                       │
│       return MatMul(self, other).doit()                           │
│                                                                   │
│ 结果: 构建 MatMul(A, M) 表达式树，调用 doit()                     │
└─────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤3: MatMul.doit() → canonicalize                              │
│                                                                   │
│ 代码 (sympy/matrices/expressions/matmul.py:190):                │
│                                                                   │
│   def doit(self, **hints):                                        │
│       deep = hints.get('deep', True)                              │
│       if deep:                                                     │
│           args = tuple(arg.doit(**hints) for arg in self.args)   │
│       else:                                                        │
│           args = self.args                                         │
│       expr = canonicalize(MatMul(*args))                          │
│       return expr                                                  │
│                                                                   │
│ 关键规则: merge_explicit (matmul.py:254)                          │
│   只有当有多个相邻的 MatrixBase 时才合并计算                       │
│   本场景: MatMul(A, M) 只有 1 个 MatrixBase，不合并               │
└─────────────────────────────────────────────────────────────────┘
   │
   ▼
结果: MatMul(MatrixSymbol('A', 2, 2), Matrix([[1,2],[3,4]]))
      is_MatrixExpr = True（延迟求值）
```

#### 场景B：具体层 × 表达式层（延迟求值）

**测试验证**：
```python
M = Matrix([[1, 2], [3, 4]])  # _op_priority=10.01
A = MatrixSymbol('A', 2, 2)    # _op_priority=11.0 ⭐ 更高

result = M * A
# 结果类型: <class 'sympy.matrices.expressions.matmul.MatMul'>
# is_MatrixExpr: True
```

**代码路径**：

```
M * A
   │
   ▼
Python 调用 M.__mul__(A)
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤1: @call_highest_priority('__rmul__') 检查                  │
│                                                                   │
│ 比较:                                                              │
│   A._op_priority = 11.0                                           │
│   M._op_priority = 10.01                                          │
│                                                                   │
│   A._op_priority > M._op_priority?  11.0 > 10.01?  True!       │
│   切换到 A.__rmul__(M)                                             │
└─────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤2: 执行 MatrixExpr.__rmul__                                  │
│                                                                   │
│ 代码 (sympy/matrices/expressions/matexpr.py:140):               │
│                                                                   │
│   @_sympifyit('other', NotImplemented)                            │
│   @call_highest_priority('__mul__')                              │
│   def __rmul__(self, other):                                      │
│       return MatMul(other, self).doit()                           │
│                                                                   │
│ 结果: 构建 MatMul(M, A) 表达式树                                   │
└─────────────────────────────────────────────────────────────────┘
   │
   ▼
结果: MatMul(Matrix([[1,2],[3,4]]), MatrixSymbol('A', 2, 2))
      is_MatrixExpr = True（延迟求值）
```

#### 场景C：具体层 × 具体层（立即计算）

**测试验证**：
```python
M1 = Matrix([[1, 2], [3, 4]])  # is_MatrixExpr=False, _class_priority=4
M2 = Matrix([[5, 6], [7, 8]])  # is_MatrixExpr=False, _class_priority=4

result = M1 * M2
# 结果类型: <class 'sympy.matrices.dense.MutableDenseMatrix'>
# is_MatrixExpr: False
# 结果值: Matrix([[19, 22], [43, 50]])
```

**代码路径**：

```
M1 * M2
   │
   ▼
Python 调用 M1.__mul__(M2)
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤1: @call_highest_priority('__rmul__') 检查                  │
│                                                                   │
│ 比较:                                                              │
│   M1._op_priority = 10.01                                         │
│   M2._op_priority = 10.01                                         │
│                                                                   │
│   优先级相同，不切换                                                │
└─────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤2: 执行 DenseMatrix.__mul__ → multiply                       │
│                                                                   │
│ 代码 (sympy/matrices/common.py:2767-2796):                      │
│                                                                   │
│   @call_highest_priority('__rmul__')                             │
│   def __mul__(self, other):                                       │
│       return self.multiply(other)                                  │
└─────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤3: multiply 方法                                               │
│                                                                   │
│ 代码 (sympy/matrices/common.py:2798-2839):                      │
│                                                                   │
│   def multiply(self, other, dotprodsimp=None):                   │
│       # ...                                                        │
│       if getattr(other, 'is_Matrix', False):                      │
│           # 关键：调用具体计算方法                                  │
│           m = self._eval_matrix_mul(other)                         │
│           # ...                                                    │
│           return m                                                  │
│       # ...                                                        │
└─────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤4: _eval_matrix_mul 方法                                      │
│                                                                   │
│ 代码 (sympy/matrices/repmatrix.py:382-383):                     │
│                                                                   │
│   def _eval_matrix_mul(self, other: RepMatrix):                  │
│       # 关键：委托给 DomainMatrix 进行实际计算                    │
│       # 并通过 classof 决定结果类型                               │
│       return classof(self, other)._fromrep(self._rep * other._rep) │
└─────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 步骤5: classof 类型统一                                           │
│                                                                   │
│ 代码 (sympy/matrices/common.py:3236-3246):                      │
│                                                                   │
│   def classof(A, B):                                              │
│       priority_A = getattr(A, '_class_priority', None)           │
│       priority_B = getattr(B, '_class_priority', None)           │
│       if None not in (priority_A, priority_B):                    │
│           if A._class_priority > B._class_priority:               │
│               return A.__class__                                   │
│           else:                                                    │
│               return B.__class__                                   │
│                                                                   │
│ 本场景:                                                             │
│   M1._class_priority = 4                                           │
│   M2._class_priority = 4                                           │
│   返回 MutableDenseMatrix                                          │
└─────────────────────────────────────────────────────────────────┘
   │
   ▼
结果: Matrix([[19, 22], [43, 50]])
      is_MatrixExpr = False（立即计算）
```

#### 场景D：可变稠密 × 可变稀疏（立即计算，类型统一）

**测试验证**：
```python
M_dense = Matrix([[1, 2], [3, 4]])       # _class_priority=4
M_sparse = SparseMatrix([[1, 0], [0, 1]])  # _class_priority=3

result1 = M_dense * M_sparse
# 结果类型: <class 'sympy.matrices.dense.MutableDenseMatrix'>

result2 = M_sparse * M_dense
# 结果类型: <class 'sympy.matrices.dense.MutableDenseMatrix'> ⭐ 注意！
```

**关键发现**：无论谁在前，结果都是 `MutableDenseMatrix`！

**代码路径中的关键步骤**：

```
步骤: classof(M_dense, M_sparse)
      或 classof(M_sparse, M_dense)

比较:
   M_dense._class_priority = 4
   M_sparse._class_priority = 3

结果:
   4 > 3，所以返回 MutableDenseMatrix
```

**为什么会这样？**

```
继承链分析:

MatrixBase (_class_priority=3)
    │
    ├── DenseMatrix (_class_priority=4 ← 显式覆盖)
    │       └── MutableDenseMatrix (继承 4)
    │
    └── SparseRepMatrix (没有定义自己的 _class_priority)
            └── MutableSparseMatrix (继承 3 ← 来自 MatrixBase)
```

**代码证据** (`sympy/matrices/dense.py:47`)：
```python
class DenseMatrix(RepMatrix):
    _class_priority = 4  # 显式定义，覆盖父类 MatrixBase 的 3
```

**代码证据** (`sympy/matrices/sparse.py:22`)：
```python
class SparseRepMatrix(RepMatrix):
    # 没有定义 _class_priority，继承自 MatrixBase (3)
```

#### 场景E：不可变 × 不可变（不可变性具有传染性）

**测试验证**：
```python
IM_dense = ImmutableMatrix([[1, 2], [3, 4]])    # _class_priority=8
IM_sparse = ImmutableSparseMatrix([[1, 0], [0, 1]])  # _class_priority=9

result1 = IM_dense * IM_sparse
# 结果类型: <class 'sympy.matrices.immutable.ImmutableSparseMatrix'> ⭐

result2 = IM_sparse * IM_dense
# 结果类型: <class 'sympy.matrices.immutable.ImmutableSparseMatrix'>
```

**关键发现**：不可变稀疏矩阵 (9) 的优先级高于不可变稠密矩阵 (8)，结果总是不可变稀疏矩阵！

#### 场景F：可变 × 不可变（不可变性具有传染性）

**测试验证**：
```python
M_var = Matrix([[1, 2], [3, 4]])           # _class_priority=4 (可变)
M_imm = ImmutableMatrix([[1, 0], [0, 1]])   # _class_priority=8 (不可变)

result1 = M_var * M_imm
# 结果类型: <class 'sympy.matrices.immutable.ImmutableDenseMatrix'> ⭐

result2 = M_imm * M_var
# 结果类型: <class 'sympy.matrices.immutable.ImmutableDenseMatrix'>
```

**关键发现**：不可变矩阵的优先级 (8) 高于可变矩阵 (4)，结果总是不可变矩阵！

---

## 3. 分派边界决策表（验证版）

### 3.1 核心决策标志

| 决策点 | 关键标志 | 判断逻辑 |
|--------|----------|----------|
| **谁的方法被调用** | `_op_priority` | 高优先级的方法被调用 |
| **结果是否是表达式树** | `is_MatrixExpr` | 任一操作数为 `True` → 表达式树 |
| **结果类型是什么** | `_class_priority` | 高优先级的类作为结果类型 |

### 3.2 完整场景对照表

| 场景 | 操作数A | 操作数B | A.is_MatrixExpr | B.is_MatrixExpr | 结果类型 | is_MatrixExpr | 原因 |
|------|---------|---------|-----------------|-----------------|----------|---------------|------|
| 1 | MatrixSymbol | MatrixSymbol | True | True | MatMul | True | 表达式层 `_op_priority` 最高 |
| 2 | MatrixSymbol | Matrix | True | False | MatMul | True | 表达式层优先级更高 |
| 3 | Matrix | MatrixSymbol | False | True | MatMul | True | 切换到表达式层 `__rmul__` |
| 4 | MatrixSymbol | SparseMatrix | True | False | MatMul | True | 表达式层优先级更高 |
| 5 | **Matrix** | **Matrix** | False | False | MutableDenseMatrix | False | 都是具体层，立即计算 |
| 6 | **Matrix** | **SparseMatrix** | False | False | MutableDenseMatrix | False | classof: 4 > 3 |
| 7 | **SparseMatrix** | **Matrix** | False | False | MutableDenseMatrix | False | classof: 4 > 3 |
| 8 | ImmutableMatrix | ImmutableSparseMatrix | False | False | ImmutableSparseMatrix | False | classof: 9 > 8 |
| 9 | Matrix | ImmutableMatrix | False | False | ImmutableDenseMatrix | False | classof: 8 > 4 |

### 3.3 简化决策规则

**规则1：表达式层是否参与？**
```
if (A.is_MatrixExpr or B.is_MatrixExpr):
    结果 = 表达式树 (MatMul/MatAdd)
    is_MatrixExpr = True
    延迟求值，不立即计算
else:
    结果 = 具体矩阵
    is_MatrixExpr = False
    立即计算
```

**规则2：具体矩阵之间的类型统一**
```
结果类型 = classof(A, B)
         = A.__class__ 如果 A._class_priority > B._class_priority
         = B.__class__ 否则
```

**规则3：优先级值的含义**
```
_class_priority 从高到低:
  9 = ImmutableSparseMatrix  (不可变稀疏)
  8 = ImmutableDenseMatrix   (不可变稠密)
  4 = MutableDenseMatrix     (可变稠密)  ⭐ 高于可变稀疏
  3 = MutableSparseMatrix    (可变稀疏)
```

---

## 4. 关键代码引用（精确行号）

### 4.1 操作符分派机制

| 功能 | 文件 | 行号 | 代码片段 |
|------|------|------|----------|
| `@call_highest_priority` 装饰器 | `sympy/core/decorators.py` | 83 | 比较 `_op_priority`，决定调用谁的方法 |
| MatrixExpr.\_\_mul\_\_ | `sympy/matrices/expressions/matexpr.py` | 128 | `return MatMul(self, other).doit()` |
| MatrixExpr.\_\_rmul\_\_ | `sympy/matrices/expressions/matexpr.py` | 140 | `return MatMul(other, self).doit()` |
| DenseMatrix.\_\_mul\_\_ | `sympy/matrices/common.py` | 2767 | `return self.multiply(other)` |

### 4.2 具体计算机制

| 功能 | 文件 | 行号 | 代码片段 |
|------|------|------|----------|
| multiply 方法 | `sympy/matrices/common.py` | 2798 | 检查 `is_Matrix`，调用 `_eval_matrix_mul` |
| \_eval_matrix_mul | `sympy/matrices/repmatrix.py` | 382 | `classof(self, other)._fromrep(...)` |
| classof 函数 | `sympy/matrices/common.py` | 3236 | 比较 `_class_priority`，返回结果类型 |

### 4.3 优先级定义

| 优先级定义 | 文件 | 行号 | 值 |
|-----------|------|------|-----|
| MatrixExpr.\_op_priority | `sympy/matrices/expressions/matexpr.py` | 66 | **11.0** |
| DenseMatrix.\_op_priority | `sympy/matrices/dense.py` | 46 | 10.01 |
| DenseMatrix.\_class_priority | `sympy/matrices/dense.py` | 47 | **4** |
| ImmutableDenseMatrix.\_class_priority | `sympy/matrices/immutable.py` | 107 | 8 |
| ImmutableSparseMatrix.\_class_priority | `sympy/matrices/immutable.py` | 166 | **9** |

---

## 5. 对上一版报告的关键修正

### 5.1 修正1：可变稠密与可变稀疏的优先级关系

**上一版表述**：
> "可变矩阵之间没有明确的稠密/稀疏优先级比较"

**实际验证结果**：
```
测试代码:
  M_dense = Matrix([[1, 2], [3, 4]])       # _class_priority=4
  M_sparse = SparseMatrix([[1, 0], [0, 1]])  # _class_priority=3

  result1 = M_dense * M_sparse  # 结果: MutableDenseMatrix
  result2 = M_sparse * M_dense  # 结果: MutableDenseMatrix ⭐ 关键！
```

**修正后表述**：
> 可变稠密矩阵 (`_class_priority=4`) 的优先级**高于**可变稀疏矩阵 (`_class_priority=3`)。无论谁在前，可变稠密与可变稀疏的混合运算结果总是 `MutableDenseMatrix`。
> 
> 原因：
> - `DenseMatrix` 显式定义了 `_class_priority = 4`（覆盖父类）
> - `SparseRepMatrix` 没有定义自己的 `_class_priority`，继承了 `MatrixBase` 的 `3`

### 5.2 修正2：延迟求值边界的精确表述

**上一版表述**：
> "延迟求值 = 构建表达式树，不立即执行数值计算"

**修正后表述**：
> 延迟求值的边界由 `is_MatrixExpr` 标志决定：
> - **任一操作数 `is_MatrixExpr=True`** → 结果是表达式树 (`MatMul`/`MatAdd`)，延迟求值
> - **所有操作数 `is_MatrixExpr=False`** → 结果是具体矩阵，立即计算
> 
> 注意：`_op_priority` 决定谁的方法被调用，但 `is_MatrixExpr` 才是决定结果是否是表达式树的关键标志。

### 5.3 修正3：classof 函数的作用时机

**上一版表述**：
> "classof 在类型统一时使用"

**修正后表述**：
> `classof` 函数**只在具体计算时使用**，用于决定结果的具体矩阵类型。
> 
> 如果结果是表达式树（`is_MatrixExpr=True`），则不经过 `classof`，而是返回 `MatMul`/`MatAdd` 对象。

### 5.4 修正4：不可变稀疏 vs 不可变稠密

**上一版表述**：
> "ImmutableSparseMatrix (9) > ImmutableDenseMatrix (8)，但这是由于不可变性，而非稀疏优先"

**修正后表述**：
> `ImmutableSparseMatrix` (`_class_priority=9`) 的优先级确实高于 `ImmutableDenseMatrix` (`_class_priority=8`)。
> 
> 测试验证：
> ```python
> IM_dense * IM_sparse  # 结果: ImmutableSparseMatrix
> IM_sparse * IM_dense  # 结果: ImmutableSparseMatrix
> ```
> 
> 这意味着：在不可变矩阵之间，**稀疏优先于稠密**。但这一优先级的设计意图可能是为了保持稀疏性（如果可能），而非单纯的"稀疏优于稠密"。

---

## 6. 完整测试验证脚本

本报告的所有结论都基于以下测试脚本的运行结果：

```python
from sympy import Matrix, SparseMatrix, MatrixSymbol, ImmutableMatrix, ImmutableSparseMatrix

# 测试1: 检查 _class_priority 实际取值
M_dense = Matrix([[1, 2], [3, 4]])
M_sparse = SparseMatrix([[1, 0], [0, 1]])

print(f"Matrix._class_priority = {M_dense.__class__._class_priority}")    # 4
print(f"SparseMatrix._class_priority = {M_sparse.__class__._class_priority}")  # 3

# 测试2: 可变稠密 × 可变稀疏 混合运算
result1 = M_dense * M_sparse  # MutableDenseMatrix
result2 = M_sparse * M_dense  # MutableDenseMatrix (关键验证)

# 测试3: 与表达式层混合
A = MatrixSymbol('A', 2, 2)
result3 = A * M_dense  # MatMul (is_MatrixExpr=True)
result4 = M_dense * A  # MatMul (is_MatrixExpr=True)

# 测试4: 具体矩阵之间的运算
M1 = Matrix([[1, 2], [3, 4]])
M2 = Matrix([[5, 6], [7, 8]])
result5 = M1 * M2  # MutableDenseMatrix (is_MatrixExpr=False)

# 测试5: 不可变矩阵混合
IM1 = ImmutableMatrix([[1, 2], [3, 4]])    # _class_priority=8
IM2 = ImmutableSparseMatrix([[1, 0], [0, 1]])  # _class_priority=9
result6 = IM1 * IM2  # ImmutableSparseMatrix
result7 = IM2 * IM1  # ImmutableSparseMatrix

# 测试6: 可变 × 不可变
result8 = M_dense * IM1  # ImmutableDenseMatrix
result9 = IM1 * M_dense  # ImmutableDenseMatrix
```

---

## 7. 总结与核心决策原则

### 7.1 三层决策模型

```
┌─────────────────────────────────────────────────────────────────┐
│ 第一层：操作符分派（由 _op_priority 决定）                        │
│                                                                   │
│   比较 A._op_priority vs B._op_priority                          │
│   - 表达式层 (11.0) > 具体层 (~10.01)                            │
│   - 高优先级的方法被调用                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 第二层：延迟求值判断（由 is_MatrixExpr 决定）                     │
│                                                                   │
│   if (A.is_MatrixExpr or B.is_MatrixExpr):                        │
│       结果 = 表达式树 (MatMul/MatAdd)                              │
│       延迟求值，不立即计算                                          │
│   else:                                                             │
│       进入第三层：具体计算                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (如果是具体计算)
┌─────────────────────────────────────────────────────────────────┐
│ 第三层：类型统一（由 _class_priority 决定）                        │
│                                                                   │
│   结果类型 = classof(A, B)                                         │
│   - 不可变性具有传染性：不可变 > 可变                               │
│   - 可变矩阵之间：稠密 (4) > 稀疏 (3)                              │
│   - 不可变矩阵之间：稀疏 (9) > 稠密 (8)                            │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 关键速查表

| 属性 | 表达式层 | 可变稠密 | 可变稀疏 | 不可变稠密 | 不可变稀疏 |
|------|----------|----------|----------|------------|------------|
| `_op_priority` | **11.0** | 10.01 | 10.01 | 10.001 | 10.001 |
| `_class_priority` | 不使用 | **4** | **3** | 8 | **9** |
| `is_MatrixExpr` | **True** | **False** | **False** | False | False |

### 7.3 常见场景速答

| 问题 | 答案 | 依据 |
|------|------|------|
| `MatrixSymbol * Matrix` 立即计算吗？ | ❌ 否，返回 `MatMul` | `is_MatrixExpr=True` |
| `Matrix * Matrix` 立即计算吗？ | ✅ 是，返回具体矩阵 | `is_MatrixExpr=False` |
| `Matrix * SparseMatrix` 结果类型？ | `MutableDenseMatrix` | `classof`: 4 > 3 |
| `SparseMatrix * Matrix` 结果类型？ | `MutableDenseMatrix` | `classof`: 4 > 3 |
| `ImmutableMatrix * ImmutableSparseMatrix` 结果类型？ | `ImmutableSparseMatrix` | `classof`: 9 > 8 |
| `Matrix * ImmutableMatrix` 结果类型？ | `ImmutableDenseMatrix` | `classof`: 8 > 4（不可变性传染性） |

---

## 附录：文件结构

```
sympy/matrices/
├── expressions/                    # 表达式层
│   ├── matexpr.py:66             # MatrixExpr._op_priority = 11.0
│   ├── matexpr.py:128            # MatrixExpr.__mul__
│   └── matexpr.py:140            # MatrixExpr.__rmul__
├── common.py:2767                # DenseMatrix.__mul__
├── common.py:2798                # multiply 方法
├── common.py:3236                # classof 函数
├── repmatrix.py:382              # _eval_matrix_mul
├── dense.py:46                    # DenseMatrix._op_priority = 10.01
├── dense.py:47                    # DenseMatrix._class_priority = 4
├── immutable.py:107               # ImmutableDenseMatrix._class_priority = 8
└── immutable.py:166               # ImmutableSparseMatrix._class_priority = 9

sympy/core/
└── decorators.py:83               # @call_highest_priority 装饰器
```
