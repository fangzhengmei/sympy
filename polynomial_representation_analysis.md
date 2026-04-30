# SymPy 多项式内部表示与运算机制分析

## 1. 多项式内部表示形式

### 1.1 稠密表示 (Dense Representation)

#### 核心实现类
- **DMP** (Dense Multivariate Polynomials): 基类，定义在 `sympy/polys/polyclasses.py:180`
- **DMP_Python**: Python 实现的稠密多项式，定义在 `sympy/polys/polyclasses.py:1384`
- **DUP_Flint**: 基于 FLINT 库的优化实现（当可用时），定义在 `sympy/polys/polyclasses.py`

#### 数据结构
稠密表示使用**嵌套列表**来存储多项式系数：
- 单变量多项式：`[a_n, a_{n-1}, ..., a_0]`，表示 $a_n x^n + a_{n-1} x^{n-1} + \dots + a_0$
- 多变量多项式：递归嵌套列表，例如二元多项式 $x^2 + 2xy + y^2$ 表示为 `[[1], [2], [1]]`

#### 关键属性
- `_rep`: 存储系数的嵌套列表
- `dom`: 系数所在的数域 (Domain)
- `lev`: 多项式的"级别"，即变量个数减 1

### 1.2 稀疏表示 (Sparse Representation)

#### 核心实现类
- **PolyRing**: 多项式环，定义在 `sympy/polys/rings.py:246`
- **PolyElement**: 稀疏多项式元素，定义在 `sympy/polys/rings.py:790`

#### 数据结构
稀疏表示使用**字典**来存储非零项：
- 键：单项式的指数元组，例如 `(2, 1)` 表示 $x^2 y^1$
- 值：对应的系数

例如，多项式 $3x^2 y + 2xy + 5$ 表示为：
```python
{(2, 1): 3, (1, 1): 2, (0, 0): 5}
```

#### 关键属性
- `ring`: 所属的多项式环 (PolyRing)
- 继承自 `dict`，直接存储单项式到系数的映射

### 1.3 矩阵表示 (Matrix Representation)

#### 核心实现类
- **DomainMatrix**: 域矩阵，定义在 `sympy/polys/matrices/domainmatrix.py:90`
- **DDM** (Dense Domain Matrix): 稠密矩阵表示，定义在 `sympy/polys/matrices/ddm.py`
- **SDM** (Sparse Domain Matrix): 稀疏矩阵表示，定义在 `sympy/polys/matrices/sdm.py`
- **DFM** (Dense Flint Matrix): 基于 FLINT 的优化稠密矩阵（当可用时）

#### 数据结构
- **稠密矩阵 (DDM)**: 列表的列表，`[[a11, a12, ...], [a21, a22, ...], ...]`
- **稀疏矩阵 (SDM)**: 字典的字典，`{row: {col: value}}`，只存储非零元素

#### 关键属性
- `rep`: 内部表示 (DDM/SDM/DFM)
- `shape`: 矩阵形状 `(rows, cols)`
- `domain`: 元素所在的数域

### 1.4 表示形式之间的转换

#### 稠密 ↔ 稀疏
```python
# DMP (稠密) 转字典表示
dmp.to_dict()  # 返回 dict[monom, coefficient]

# 从字典创建 DMP
DMP.from_dict(rep_dict, lev, dom)

# PolyElement (稀疏) 从列表创建
PolyElement.from_list(dmp_list)

# PolyElement 转字典
poly_element.as_expr_dict()
```

#### 矩阵稠密 ↔ 稀疏
```python
# DomainMatrix 方法
dm.to_sparse()   # 转换为稀疏表示 (SDM)
dm.to_dense()    # 转换为稠密表示 (DDM/DFM)

# DDM 与 SDM 互转
ddm.to_sdm()
sdm.to_ddm()
```

#### 矩阵与 SymPy Matrix 互转
```python
# DomainMatrix 转 SymPy Matrix
dm.to_Matrix()

# SymPy Matrix 转 DomainMatrix
DomainMatrix.from_Matrix(matrix, fmt='sparse' or 'dense')
```

## 2. 数域系统与选择机制

### 2.1 数域层次结构

SymPy 的数域系统定义在 `sympy/polys/domains/` 目录下，核心基类是 `Domain`。

#### 基础数域
| 数域 | 类名 | 说明 | 示例 |
|------|------|------|------|
| ZZ | `IntegerRing` | 整数环 | `ZZ(5)` |
| QQ | `RationalField` | 有理数域 | `QQ(1, 2)` |
| ZZ_I | `GaussianIntegerRing` | 高斯整数环 | `ZZ_I(1, 2)` |
| QQ_I | `GaussianRationalField` | 高斯有理数域 | `QQ_I(1, 2)` |
| GF(p) | `FiniteField` | 有限域 | `GF(7)` |
| RR | `RealField` | 实数域（浮点近似） | `RR(3.14)` |
| CC | `ComplexField` | 复数域（浮点近似） | `CC(1, 2)` |
| QQ(a) | `AlgebraicField` | 代数数域 | `QQ.algebraic_field(sqrt(2))` |

#### 复合数域
| 数域 | 类名 | 说明 | 示例 |
|------|------|------|------|
| K[x] | `PolynomialRing` | 多项式环 | `ZZ[x, y]` |
| K(x) | `FractionField` | 有理函数域 | `QQ(x, y)` |
| EX | `ExpressionDomain` | 表达式域（通用 fallback） | `EX(sin(x))` |

### 2.2 最小覆盖域选择

#### `construct_domain` 函数
定义在 `sympy/polys/constructor.py:268`，根据输入表达式自动选择最小覆盖域。

#### 选择策略
1. **简单域检测** (`_construct_simple`):
   - 检测系数类型：整数、有理数、浮点数、复数、代数数
   - 优先级：整数 → 有理数 → 浮点数 → 复数 → 代数数

2. **复合域检测** (`_construct_composite`):
   - 检测是否包含符号变量
   - 检测是否有负指数（需要分式域）
   - 构建多项式环或有理函数域

3. **表达式域 fallback** (`_construct_expression`):
   - 当以上都失败时，使用 EX 域

#### 选择优先级（从低到高）
```
GF(p) < ZZ < QQ < ZZ_I < QQ_I < RR < CC < ALG < K[x] < K(x) < EX
```

#### 示例
```python
from sympy import construct_domain, S, sqrt
from sympy.abc import x, y

# 整数系数 → ZZ
construct_domain([S(2), S(3), S(4)])  # (ZZ, [2, 3, 4])

# 有理数系数 → QQ
construct_domain([S(1)/2, S(3)/4])   # (QQ, [1/2, 3/4])

# 含变量 → 多项式环
construct_domain([2*x + 1, y])         # (ZZ[x,y], [2*x + 1, y])

# 含负指数 → 有理函数域
construct_domain([y/x, x/(1 - y)])     # (ZZ(x,y), ...)

# 含代数数（默认）→ EX
construct_domain([sqrt(2)])             # (EX, ...)

# 含代数数（extension=True）→ 代数数域
construct_domain([sqrt(2)], extension=True)  # (QQ<sqrt(2)>, ...)
```

### 2.3 数域统一机制

#### `Domain.unify` 方法
定义在 `sympy/polys/domains/domain.py:889`，找到能同时表示两个域元素的**最小公共域**。

#### 统一规则
1. **相同域**: 直接返回
2. **特征不同**: 抛出 `UnificationFailed`
3. **复合域统一** (`unify_composite`):
   - 统一基域
   - 合并符号变量
   - 决定使用多项式环还是分式域

4. **简单域统一优先级**:
   ```
   ZZ.unify(QQ) → QQ
   ZZ.unify(RR) → RR
   QQ.unify(CC) → CC
   ZZ[x].unify(QQ) → QQ[x]
   ZZ[x].unify(QQ[y]) → QQ[x,y]
   ```

#### 代码实现要点
```python
# domain.py:889-1016
def unify(K0, K1, symbols=None):
    if symbols is not None:
        return K0.unify_with_symbols(K1, symbols)
    
    if K0 == K1:
        return K0
    
    # 特征检查
    if not (K0.has_CharacteristicZero and K1.has_CharacteristicZero):
        if K0.characteristic() != K1.characteristic():
            raise UnificationFailed(...)
    
    # EX/EXRAW 域优先
    if K0.is_EXRAW or K1.is_EXRAW:
        return K0 if K0.is_EXRAW else K1
    if K0.is_EX or K1.is_EX:
        return K0 if K0.is_EX else K1
    
    # 代数扩域处理
    if K0.is_FiniteExtension or K1.is_FiniteExtension:
        # ... 扩域合并逻辑
    
    # 复合域处理
    if K0.is_Composite or K1.is_Composite:
        return K0.unify_composite(K1)
    
    # 数值域优先级
    if K0.is_ComplexField:
        # ... CC 统一逻辑
    if K0.is_RealField:
        # ... RR 统一逻辑
    if K0.is_AlgebraicField:
        # ... 代数域统一逻辑
    
    # 高斯数域
    if K0.is_GaussianField or K1.is_GaussianField:
        return QQ_I if K0.is_GaussianField else K1
    
    # 有理数/整数
    if K0.is_RationalField or K1.is_RationalField:
        return QQ
    if K0.is_IntegerRing or K1.is_IntegerRing:
        return ZZ
    
    # 最终 fallback
    return EX
```

## 3. 运算时的数域处理

### 3.1 多项式运算前的统一

#### `DMP.unify_DMP` 方法
定义在 `sympy/polys/polyclasses.py:328`，在二元运算前统一两个多项式的数域。

```python
def unify_DMP(f, g: DMP[Es]) -> tuple[DMP[Et], DMP[Et]]:
    """Unify and return DMP instances of f and g."""
    if not isinstance(g, DMP) or f.lev != g.lev:
        raise UnificationFailed("Cannot unify %s with %s" % (f, g))
    
    if f.dom == g.dom:
        return f, g
    else:
        dom: Domain[Et] = f.dom.unify(g.dom)
        return f.convert(dom), g.convert(dom)
```

#### 运算中的统一调用
```python
# polyclasses.py:549-552
def add(f, g: Self, /) -> Self:
    """Add two multivariate polynomials f and g."""
    F, G = f.unify_DMP(g)
    return F._add(G)
```

### 3.2 矩阵运算前的统一

#### `DomainMatrix.unify` 方法
定义在 `sympy/polys/matrices/domainmatrix.py:778`，同时统一域和表示格式。

```python
def unify(self, *others, fmt=None):
    """
    Unifies the domains and the format of self and other matrices.
    """
    matrices = (self,) + others
    matrices = DomainMatrix._unify_domain(*matrices)
    if fmt is not None:
        matrices = DomainMatrix._unify_fmt(*matrices, fmt=fmt)
    return matrices
```

#### 域统一实现
```python
# domainmatrix.py:752-758
@classmethod
def _unify_domain(cls, *matrices):
    """Convert matrices to a common domain"""
    domains = {matrix.domain for matrix in matrices}
    if len(domains) == 1:
        return matrices
    domain = reduce(lambda x, y: x.unify(y), domains)
    return tuple(matrix.convert_to(domain) for matrix in matrices)
```

### 3.3 不同数域运算的差异

#### 精确域 vs 近似域
| 特性 | 精确域 (ZZ, QQ, ALG) | 近似域 (RR, CC) |
|------|---------------------|-----------------|
| 误差 | 无误差 | 有舍入误差 |
| 比较 | 精确相等 (`==`) | 近似相等 (`almosteq`) |
| GCD | 精确计算 | 不支持/不稳定 |
| 因式分解 | 精确分解 | 不支持 |
| 根隔离 | 精确区间 | 数值近似 |

#### 环 vs 域
| 操作 | 环 (ZZ, K[x]) | 域 (QQ, K(x)) |
|------|--------------|---------------|
| 除法 | 地板除 (`//`, `%`) | 精确除 (`/`) |
| 可逆元 | 仅 ±1 | 所有非零元 |
| 线性方程组 | 需特殊处理 | 直接求解 |

### 3.4 降级处理机制

#### 降级场景
1. **代数数 + 浮点数** → EX 域
   ```python
   # constructor.py:33-35, 53-55
   if coeff.is_Float:
       if algebraics:
           # 同时有浮点数和代数数 → EX
           return False
   ```

2. **无法表示的表达式** → EX 域
   - 含超越函数：`sin(x)`, `log(x)`
   - 含未定义的运算

3. **复合域中无法识别的关系** → EX 域
   ```python
   # constructor.py:152-158
   for gen in gens:
       symbols = gen.free_symbols
       if all_symbols & symbols:
           # 符号间可能存在代数关系 → EX
           return None
   ```

#### 降级路径
```
精确域 (ZZ/QQ/ALG) → 近似域 (RR/CC) → 表达式域 (EX)
```

#### 示例
```python
from sympy import construct_domain, sqrt, S

# 代数数 + 浮点数 → EX
construct_domain([sqrt(2), S(3.14)])  # (EX, ...)

# 超越函数 → EX
construct_domain([sin(x)])  # (EX, ...)
```

## 4. 关键代码位置总结

### 4.1 表示层
| 功能 | 文件位置 | 类/函数 |
|------|---------|---------|
| 稠密多项式 | `polyclasses.py` | `DMP`, `DMP_Python`, `DUP_Flint` |
| 稀疏多项式 | `rings.py` | `PolyRing`, `PolyElement` |
| 域矩阵 | `matrices/domainmatrix.py` | `DomainMatrix` |
| 稠密矩阵 | `matrices/ddm.py` | `DDM` |
| 稀疏矩阵 | `matrices/sdm.py` | `SDM` |

### 4.2 数域层
| 功能 | 文件位置 | 类/函数 |
|------|---------|---------|
| 域基类 | `domains/domain.py` | `Domain` |
| 整数环 | `domains/integerring.py` | `IntegerRing` |
| 有理数域 | `domains/rationalfield.py` | `RationalField` |
| 有限域 | `domains/finitefield.py` | `FiniteField` |
| 实数/复数域 | `domains/realfield.py`, `complexfield.py` | `RealField`, `ComplexField` |
| 多项式环 | `domains/polynomialring.py` | `PolynomialRing` |
| 分式域 | `domains/fractionfield.py` | `FractionField` |
| 表达式域 | `domains/expressiondomain.py` | `ExpressionDomain` |

### 4.3 构造与统一
| 功能 | 文件位置 | 函数 |
|------|---------|------|
| 域构造 | `constructor.py` | `construct_domain` |
| 域统一 | `domains/domain.py` | `Domain.unify` |
| 多项式统一 | `polyclasses.py` | `DMP.unify_DMP` |
| 矩阵统一 | `matrices/domainmatrix.py` | `DomainMatrix.unify` |

## 5. 设计要点总结

### 5.1 表示选择策略
1. **稠密表示**适合：
   - 低次、项密集的多项式
   - 数值计算（浮点）
   - 矩阵运算（大多数情况）

2. **稀疏表示**适合：
   - 高次、项稀疏的多项式
   - 符号计算
   - 大型稀疏矩阵

3. **自动选择**：
   - FLINT 库可用时自动使用优化实现
   - 矩阵根据元素密度自动选择稠密/稀疏

### 5.2 数域设计原则
1. **最小覆盖原则**：选择能精确表示所有元素的最小域
2. **层次化统一**：按优先级逐步升级域
3. **安全降级**：当精确表示不可能时，降级到通用域
4. **精确性优先**：优先使用精确域，近似域作为备选

### 5.3 运算流程
```
输入表达式
    ↓
construct_domain() → 选择最小覆盖域
    ↓
创建多项式/矩阵（选择稠密/稀疏表示）
    ↓
运算前：unify() → 统一到公共域
    ↓
执行运算
    ↓
（必要时）降级到更通用的域
    ↓
输出结果
```
