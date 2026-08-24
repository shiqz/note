# NumPy 使用手册

> 本手册系统整理 NumPy 常用函数与属性，先以速查表总览，再按分类详细讲解每个函数的用法、参数、返回值及注意事项。基于 NumPy 2.x 版本编写。

```python
import numpy as np
```

---

## 一、函数与属性速查表

### 1.1 数组创建

| 函数/属性 | 常用参数 | 简要作用 |
|-----------|----------|----------|
| `np.array()` | object(输入), dtype(类型), copy(是否复制), ndmin(最小维度) | 从Python列表/元组创建数组 |
| `np.zeros()` | shape(形状), dtype(类型), order('C'/'F') | 创建全0数组 |
| `np.ones()` | shape(形状), dtype(类型), order('C'/'F') | 创建全1数组 |
| `np.full()` | shape(形状), fill_value(填充值), dtype | 创建填充指定值的数组 |
| `np.empty()` | shape(形状), dtype(类型), order('C'/'F') | 分配内存但不初始化（内容随机） |
| `np.eye()` | N(行), M(列,默认=N), k(对角线偏移), dtype | 创建单位矩阵 |
| `np.identity()` | n(阶数), dtype | 创建单位矩阵（方阵） |
| `np.arange()` | start(起始), stop(结束,不含), step(步长), dtype | 创建等间隔序列（类似range） |
| `np.linspace()` | start, stop, num(点数,默认50), endpoint(含stop), retstep(返回步长) | 创建指定数量的均匀间隔样本 |
| `np.logspace()` | start, stop(指数范围), num, base(底数,默认10), endpoint | 创建对数均匀间隔序列 |
| `np.geomspace()` | start, stop, num, endpoint | 创建几何级数间隔序列 |
| `np.zeros_like()` | a(参考数组), dtype, order | 创建与另一数组shape/dtype相同的全0数组 |
| `np.ones_like()` | a(参考数组), dtype, order | 创建与另一数组shape/dtype相同的全1数组 |
| `np.full_like()` | a(参考数组), fill_value, dtype, order | 创建与另一数组shape/dtype相同的填充数组 |
| `np.empty_like()` | a(参考数组), dtype, order | 创建与另一数组shape/dtype相同的未初始化数组 |
| `np.copy()` / `.copy()` | order('C'/'F') | 创建数组深拷贝 |
| `np.fromfunction()` | function(坐标函数), shape, dtype | 通过函数生成数组 |
| `np.fromiter()` | iter(迭代器), dtype, count(数量) | 从可迭代对象创建数组 |
| `np.tile()` | A(数组), reps(重复次数) | 重复构造数组 |
| `np.repeat()` | a, repeats(重复次数), axis(轴) | 重复元素 |
| `np.diag()` | v(数组), k(偏移) | 提取对角线或构造对角矩阵 |
| `np.tri()` | N(行), M(列), k, dtype | 构造下三角矩阵（含对角线为1） |
| `np.vander()` | x(1D数组), N(列数), increasing(递增) | 构造范德蒙德矩阵 |

### 1.2 数组属性

| 属性 | 常用参数 | 简要作用 |
|------|----------|----------|
| `.ndim` | 只读 | 数组维度（轴数） |
| `.shape` | 可赋值改变形状 | 数组各轴长度的元组 |
| `.size` | 只读 | 数组元素总数 |
| `.dtype` | dtype对象 | 元素数据类型 |
| `.itemsize` | 只读 | 单个元素占用字节数 |
| `.nbytes` | 只读 | 数组数据区总字节数 |
| `.T` | 等价于transpose() | 转置视图 |
| `.flat` | 迭代器 | 一维迭代器 |
| `.flatten()` | order(可选) | 展平为一维（返回副本） |
| `.ravel()` | order | 展平为一维（尽可能返回视图） |
| `.imag` | 只读 | 复数的虚部 |
| `.real` | 只读 | 复数的实部 |
| `.strides` | 只读 | 各轴步进字节数 |
| `.flags` | 只读 | 内存布局信息 |
| `.base` | 只读 | 视图所指向的原数组（若为视图） |
| `.ctypes` | 只读 | 用于与C交互的接口对象 |

### 1.3 数据类型与类型转换

| 函数/属性 | 常用参数 | 简要作用 |
|-----------|----------|----------|
| `.astype()` | dtype, copy, casting(安全级别) | 转换数组元素类型 |
| `np.dtype()` | obj(类型描述) | 创建/查询数据类型对象 |
| `np.iinfo()` | type(类型), 返回.min/.max/.eps | 查询整数类型的取值范围 |
| `np.finfo()` | type(类型), 返回.min/.max/.eps | 查询浮点类型的精度与范围 |
| `np.can_cast()` | from, to, casting | 判断类型转换是否安全 |
| `np.result_type()` | *arrays_and_dtypes | 推断多个数组运算后的结果类型 |
| `np.common_type()` | *arrays | 找多个数组的公共类型 |
| `np.min_scalar_type()` | a(值) | 找能容纳指定值的最小类型 |
| `.item()` | *args(索引) | 将单元素数组转为Python标量 |
| `.tolist()` | 无参数 | 将数组转为Python嵌套列表 |
| `np.bool_` | - | 布尔类型 |
| `np.int8/16/32/64` | - | 有符号整数类型 |
| `np.uint8/16/32/64` | - | 无符号整数类型 |
| `np.float16/32/64` | - | 浮点类型 |
| `np.complex64/128` | - | 复数类型 |
| `np.str_` / `np.bytes_` | - | 字符串/字节类型 |

### 1.4 形状变换与轴操作

| 函数/方法 | 常用参数 | 简要作用 |
|-----------|----------|----------|
| `.reshape()` | *shape(新形状,-1自动推导), order | 改变数组形状（不改变数据） |
| `.resize()` | new_shape, refcheck | 原地改变数组形状（可改变size） |
| `np.resize()` | a, new_shape | 返回改变形状的新数组（可填充重复） |
| `.transpose()` | *axes(轴顺序) | 转置（可指定轴顺序） |
| `.swapaxes()` | axis1, axis2 | 交换两个轴 |
| `.moveaxis()` | source, destination | 移动轴的位置 |
| `np.expand_dims()` | a, axis(位置) | 扩展维度（增加长度为1的轴） |
| `np.squeeze()` | a, axis(要移除的轴) | 移除长度为1的轴 |
| `np.newaxis` / `None` | 索引时使用 | 在指定位置增加一个维度 |
| `np.concatenate()` | arrays, axis(默认0), dtype | 沿已有轴连接多个数组 |
| `np.stack()` | arrays, axis(新轴位置,默认0) | 沿新轴堆叠数组 |
| `np.vstack()` | arrays | 垂直堆叠（沿axis=0） |
| `np.hstack()` | arrays | 水平堆叠（沿axis=1） |
| `np.dstack()` | arrays | 按深度堆叠（沿axis=2） |
| `np.column_stack()` | tup | 将一维数组作为列堆叠成二维 |
| `np.row_stack()` | tup | 将一维数组作为行堆叠成二维 |
| `np.split()` | ary, indices_or_sections(N或列表), axis | 沿指定轴分割数组 |
| `np.vsplit()` | ary, indices_or_sections | 垂直分割（按行） |
| `np.hsplit()` | ary, indices_or_sections | 水平分割（按列） |
| `np.dsplit()` | ary, indices_or_sections | 按深度分割（按axis=2） |
| `np.array_split()` | ary, indices_or_sections, axis(允许不等分) | 分割数组（允许不等分） |
| `np.tile()` | A, reps | 平铺重复数组 |
| `np.repeat()` | a, repeats, axis | 重复元素 |

### 1.5 索引与切片

| 方式/函数 | 常用参数 | 简要作用 |
|-----------|----------|----------|
| 基础索引 `a[i]` / `a[i,j]` | - | 选取单个元素或子数组 |
| 切片 `a[start:stop:step]` | - | 选取连续范围（返回视图） |
| 多维切片 `a[:, 1:3]` | - | 每个轴独立切片 |
| 布尔索引 `a[mask]` | - | 用布尔数组筛选元素（返回副本） |
| 花式索引 `a[[0,2,4]]` | - | 用整数数组指定位置选取（返回副本） |
| `np.where()` | condition, x(True值), y(False值); 单参返回索引 | 条件筛选，返回索引或选择值 |
| `np.take()` | a, indices, axis, mode | 沿指定轴按索引取元素 |
| `np.put()` | a, ind(索引), v(值), mode | 沿指定轴按索引赋值 |
| `np.choose()` | a(索引数组), choices(候选), mode | 按索引数组从多个数组中选择值 |
| `np.select()` | condlist(条件列表), choicelist(值列表), default | 多条件选择值 |
| `np.piecewise()` | x, condlist, funclist | 分段定义函数 |
| `np.clip()` | a, a_min(下界), a_max(上界), out | 将值裁剪到指定范围 |
| `np.nonzero()` | a | 返回非零元素的索引 |
| `np.argwhere()` | a | 返回非零元素的坐标 |
| `np.extract()` | condition, arr | 按条件提取元素 |
| `np.delete()` | arr, obj(索引), axis | 删除指定位置元素 |
| `np.insert()` | arr, obj(位置), values, axis | 在指定位置插入元素 |
| `np.append()` | arr, values, axis(⚠️性能差) | 在末尾追加元素（每次复制，性能差） |
| `np.trim_zeros()` | filt(1D数组), trim('f'/'b'/'fb') | 去除首尾的零 |

### 1.6 数学运算（逐元素 ufunc）

| 函数 | 常用参数 | 简要作用 |
|------|----------|----------|
| `np.add(a, b)` / `a + b` | x1,x2,out=,where= | 逐元素加法 |
| `np.subtract(a, b)` / `a - b` | x1,x2,out=,where= | 逐元素减法 |
| `np.multiply(a, b)` / `a * b` | x1,x2,out=,where= | 逐元素乘法 |
| `np.divide(a, b)` / `a / b` | x1,x2,out=,where= | 逐元素真除法 |
| `np.floor_divide(a, b)` / `a // b` | x1,x2,out=,where= | 向下取整除法 |
| `np.power(a, b)` / `a ** b` | x1,x2,out=,where= | 逐元素幂运算 |
| `np.mod(a, b)` / `a % b` | x1,x2,out=,where= | 取模（余数） |
| `np.remainder(a, b)` | x1,x2,out=,where= | 同 `mod` |
| `np.fmod(a, b)` | x1,x2,out=,where= | C语言风格取模（商截断向零） |
| `np.abs(a)` / `np.absolute(a)` | x,out=,where= | 绝对值 |
| `np.sqrt(a)` | x,out=,where= | 平方根 |
| `np.cbrt(a)` | x,out=,where= | 立方根 |
| `np.square(a)` | x,out=,where= | 平方 |
| `np.exp(a)` | x,out=,where= | 自然指数 eˣ |
| `np.exp2(a)` | x,out=,where= | 2ˣ |
| `np.expm1(a)` | x,out=,where= | eˣ - 1（x很小时更精确） |
| `np.log(a)` | x,out=,where= | 自然对数 ln(x) |
| `np.log2(a)` | x,out=,where= | 以2为底对数 |
| `np.log10(a)` | x,out=,where= | 以10为底对数 |
| `np.log1p(a)` | x,out=,where= | ln(1+x)（x很小时更精确） |
| `np.sign(a)` | x,out=,where= | 符号函数（-1/0/1） |
| `np.ceil(a)` | x,out=,where= | 向上取整 |
| `np.floor(a)` | x,out=,where= | 向下取整 |
| `np.trunc(a)` / `np.fix(a)` | x,out=,where= | 截断取整（向零取整） |
| `np.rint(a)` | x,out=,where= | 四舍六入五取偶（银行家舍入） |
| `np.round(a, decimals=0)` / `.round()` | a, decimals(小数位), out= | 四舍五入到指定小数位 |
| `np.around(a, decimals=0)` | a, decimals(小数位), out= | 同 `np.round` |
| `np.sin(a)` / `np.cos(a)` / `np.tan(a)` | x,out=,where= | 三角函数（弧度制） |
| `np.arcsin(a)` / `np.arccos(a)` / `np.arctan(a)` | x,out=,where= | 反三角函数 |
| `np.arctan2(y, x)` | x1(y坐标), x2(x坐标), out= | 考虑象限的反正切 |
| `np.deg2rad(a)` / `np.radians(a)` | x, out= | 角度转弧度 |
| `np.rad2deg(a)` / `np.degrees(a)` | x, out= | 弧度转角度 |
| `np.sinh(a)` / `np.cosh(a)` / `np.tanh(a)` | x,out=,where= | 双曲函数 |
| `np.arcsinh(a)` / `np.arccosh(a)` / `np.arctanh(a)` | x,out=,where= | 反双曲函数 |
| `np.maximum(a, b)` | x1,x2,out=,where= | 逐元素取大值 |
| `np.minimum(a, b)` | x1,x2,out=,where= | 逐元素取小值 |
| `np.fmax(a, b)` | x1,x2,out=,where= | 逐元素取大值（忽略NaN） |
| `np.fmin(a, b)` | x1,x2,out=,where= | 逐元素取小值（忽略NaN） |
| `np.logical_and(a, b)` / `a & b` | x1,x2,out=,where= | 逐元素逻辑与 |
| `np.logical_or(a, b)` / `a \| b` | x1,x2,out=,where= | 逐元素逻辑或 |
| `np.logical_not(a)` / `~a` | x,out= | 逐元素逻辑非 |
| `np.logical_xor(a, b)` / `a ^ b` | x1,x2,out=,where= | 逐元素逻辑异或 |
| `np.bitwise_and(a, b)` | x1,x2,out=,where= | 逐元素按位与 |
| `np.bitwise_or(a, b)` | x1,x2,out=,where= | 逐元素按位或 |
| `np.bitwise_xor(a, b)` | x1,x2,out=,where= | 逐元素按位异或 |
| `np.invert(a)` / `~a` | x,out= | 逐元素按位取反 |
| `np.left_shift(a, b)` / `a << b` | x1,x2,out=,where= | 左移 |
| `np.right_shift(a, b)` / `a >> b` | x1,x2,out=,where= | 右移 |
| `np.isnan(a)` | x,out= | 判断是否为NaN |
| `np.isinf(a)` | x,out= | 判断是否为无穷大 |
| `np.isfinite(a)` | x,out= | 判断是否有限（非NaN非Inf） |
| `np.isneginf(a)` | x,out= | 判断是否为负无穷 |
| `np.isposinf(a)` | x,out= | 判断是否为正无穷 |
| `np.allclose(a, b)` | a,b,rtol(1e-5),atol(1e-8),equal_nan | 所有元素在容限内近似相等 |
| `np.isclose(a, b)` | a,b,rtol,atol,equal_nan | 逐元素判断是否近似相等 |
| `np.greater(a, b)` / `a > b` | x1,x2,out=,where= | 逐元素大于 |
| `np.greater_equal(a, b)` / `a >= b` | x1,x2,out=,where= | 逐元素大于等于 |
| `np.less(a, b)` / `a < b` | x1,x2,out=,where= | 逐元素小于 |
| `np.less_equal(a, b)` / `a <= b` | x1,x2,out=,where= | 逐元素小于等于 |
| `np.equal(a, b)` / `a == b` | x1,x2,out=,where= | 逐元素等于 |
| `np.not_equal(a, b)` / `a != b` | x1,x2,out=,where= | 逐元素不等于 |
| `np.negative(a)` / `-a` | x,out=,where= | 逐元素取负 |
| `np.positive(a)` / `+a` | x,out=,where= | 逐元素正号（无变化） |
| `np.absolute(a)` / `np.abs(a)` | x,out=,where= | 绝对值（复数为模长） |
| `np.conj(a)` / `np.conjugate(a)` | x,out=,where= | 共轭复数 |
| `np.real(a)` | x,out= | 复数实部 |
| `np.imag(a)` | x,out= | 复数虚部 |

### 1.7 聚合统计

| 函数/方法 | 常用参数 | 简要作用 |
|-----------|----------|----------|
| `.sum()` / `np.sum()` | a,axis,dtype,keepdims,initial/where | 求和 |
| `.prod()` / `np.prod()` | a,axis,dtype,keepdims,initial/where | 求积 |
| `.mean()` / `np.mean()` | a,axis,dtype,keepdims,where | 算术平均值 |
| `.std()` / `np.std()` | a,axis,dtype,keepdims,ddof,where | 标准差 |
| `.var()` / `np.var()` | a,axis,dtype,keepdims,ddof,where | 方差 |
| `.min()` / `np.min()` | a,axis,keepdims,initial/where | 最小值 |
| `.max()` / `np.max()` | a,axis,keepdims,initial/where | 最大值 |
| `.argmin()` / `np.argmin()` | a,axis,keepdims | 最小值索引 |
| `.argmax()` / `np.argmax()` | a,axis,keepdims | 最大值索引 |
| `.ptp()` / `np.ptp()` | a,axis,keepdims | 峰峰值（max-min） |
| `.median()` / `np.median()` | a,axis,keepdims,overwrite_input | 中位数 |
| `np.average()` | a,axis,weights,returned | 加权平均 |
| `np.quantile()` | a,q(分位点0~1),axis,method | 分位数 |
| `np.percentile()` | a,q(百分位0~100),axis,method | 百分位数 |
| `np.nanmean()` / `np.nanstd()` / `np.nansum()` 等 | 同普通版本 | 忽略NaN的聚合统计 |
| `np.cumsum()` / `.cumsum()` | a,axis,dtype,out | 累积和 |
| `np.cumprod()` / `.cumprod()` | a,axis,dtype,out | 累积积 |
| `np.cummin()` / `.cummin()` | a,axis,dtype,out | 累积最小值 |
| `np.cummax()` / `.cummax()` | a,axis,dtype,out | 累积最大值 |
| `np.diff()` | a,n(阶数,默认1),axis,prepend/append | 相邻元素差 |
| `np.ediff1d()` | ary, to_begin/to_end | 展平后相邻元素差 |
| `np.gradient()` | f,*varargs(间距),axis,edge_order | 数值梯度 |
| `np.trapz()` / `np.trapezoid()` | y,x(采样点),dx(间距),axis | 梯形积分 |
| `np.bincount()` | x(非负整数),weights,minlength | 统计非负整数出现次数 |
| `np.histogram()` | a,bins(箱数/边界),range,density | 直方图统计 |
| `np.histogram2d()` | x,y,bins,range,density | 二维直方图 |
| `np.cov()` | m,y,rowvar,ddof,fweights/aweights | 协方差矩阵 |
| `np.corrcoef()` | x,y,rowvar,ddof | 相关系数矩阵 |
| `np.count_nonzero()` | a,axis,keepdims | 统计非零元素个数 |
| `np.all()` / `.all()` | a,axis,keepdims,where | 是否所有元素为真 |
| `np.any()` / `.any()` | a,axis,keepdims,where | 是否有任意元素为真 |

### 1.8 排序与集合操作

| 函数/方法 | 常用参数 | 简要作用 |
|-----------|----------|----------|
| `.sort()` | axis,kind('quicksort'等),order | 原地排序 |
| `np.sort()` | a,axis,kind,order | 返回排序后的副本 |
| `.argsort()` / `np.argsort()` | a,axis,kind,order | 返回排序后的索引 |
| `np.lexsort()` | keys,axis,kind,order | 多关键字排序（返回索引） |
| `np.msort()` | a | 沿第0轴排序 |
| `np.sort_complex()` | a | 按先实部后虚部排序 |
| `np.partition()` | a,kth,axis,kind | 部分排序（第k小位置正确） |
| `np.argpartition()` | a,kth,axis,kind | 部分排序索引 |
| `np.searchsorted()` | a(有序),v(待插),side('left'/'right'),sorter | 在有序数组中查找插入位置 |
| `np.unique()` | ar,return_index,return_counts,return_inverse,axis | 去重并返回唯一值 |
| `np.in1d()` / `np.isin()` | element,test_elements,assume_unique,invert | 检查元素是否在另一数组中 |
| `np.intersect1d()` | ar1,ar2,assume_unique | 两个数组的交集 |
| `np.union1d()` | ar1,ar2 | 两个数组的并集 |
| `np.setdiff1d()` | ar1,ar2,assume_unique | 差集（在A不在B） |
| `np.setxor1d()` | ar1,ar2,assume_unique | 对称差（在A或B但不同时在） |

### 1.9 线性代数 (`np.linalg`)

| 函数 | 常用参数 | 简要作用 |
|------|----------|----------|
| `np.dot(a, b)` / `a.dot(b)` | a,b,out= | 点积/矩阵乘法 |
| `np.matmul(a, b)` / `a @ b` | a,b,out= | 矩阵乘法（推荐方式） |
| `np.vdot(a, b)` | a,b | 向量点积（共轭） |
| `np.inner(a, b)` | a,b | 内积 |
| `np.outer(a, b)` | a,b,out= | 外积 |
| `np.tensordot()` | a,b,axes(缩并轴) | 张量缩并 |
| `np.kron(a, b)` | a,b | 克罗内克积 |
| `np.einsum()` | subscripts(下标串),*operands,optimize | 爱因斯坦求和（灵活的多维运算） |
| `np.linalg.matrix_power()` | a,n(幂次) | 矩阵幂 |
| `np.linalg.inv()` | a | 矩阵求逆 |
| `np.linalg.pinv()` | a,rcond | 摩尔-彭若斯伪逆 |
| `np.linalg.det()` | a | 行列式 |
| `np.linalg.matrix_rank()` | M,tol,hermitian | 矩阵秩 |
| `np.linalg.slogdet()` | a | 行列式的符号和对数绝对值 |
| `np.linalg.solve()` | a(方阵),b(常数项) | 求解线性方程组 Ax=b |
| `np.linalg.lstsq()` | a,b,rcond(推荐None) | 最小二乘解 |
| `np.linalg.norm()` | x,ord(范数阶),axis,keepdims | 向量/矩阵范数 |
| `np.linalg.cond()` | x,p(范数阶) | 条件数 |
| `np.linalg.eig()` | a(方阵) | 特征值和特征向量 |
| `np.linalg.eigh()` | a(对称阵),UPLO | 对称/厄米矩阵特征分解 |
| `np.linalg.eigvals()` | a(方阵) | 特征值 |
| `np.linalg.eigvalsh()` | a(对称阵),UPLO | 对称矩阵特征值 |
| `np.linalg.svd()` | a,full_matrices,compute_uv | 奇异值分解(SVD) |
| `np.linalg.cholesky()` | a(对称正定) | Cholesky分解 |
| `np.linalg.qr()` | a,mode | QR分解 |
| `np.trace()` / `.trace()` | a,offset,axis1,axis2 | 矩阵的迹（对角线之和） |
| `np.diag()` | v,k | 提取对角线/构造对角矩阵 |
| `np.diagflat()` | v,k | 用展平的输入构造对角矩阵 |
| `np.tril()` | m,k | 下三角矩阵 |
| `np.triu()` | m,k | 上三角矩阵 |
| `np.cross(a, b)` | a,b,axisa/axisb/axisc,axis | 叉积（三维向量） |

### 1.10 随机数 (`np.random.Generator`)

| 方法/函数 | 常用参数 | 简要作用 |
|-----------|----------|----------|
| `np.random.default_rng(seed)` | seed(种子) | 创建推荐的新随机数生成器 |
| `rng.random(size)` | size,dtype,out | [0,1)均匀分布采样 |
| `rng.integers(low, high, size)` | low,high,size,dtype,endpoint | 整数均匀采样 |
| `rng.choice(a, size, replace, p)` | a,size,replace(有放回),p(概率),axis,shuffle | 从数组中随机抽样 |
| `rng.normal(loc, scale, size)` | loc/mean,scale/sigma,size | 正态分布采样 |
| `rng.uniform(low, high, size)` | low,high,size | 均匀分布采样 |
| `rng.binomial(n, p, size)` | n,p,size | 二项分布采样 |
| `rng.poisson(lam, size)` | lam,size | 泊松分布采样 |
| `rng.exponential(scale, size)` | scale,size | 指数分布采样 |
| `rng.beta(a, b, size)` | a,b,size | Beta分布采样 |
| `rng.gamma(shape, scale, size)` | shape,scale,size | Gamma分布采样 |
| `rng.chisquare(df, size)` | df,size | 卡方分布采样 |
| `rng.standard_normal(size)` | size,dtype,out | 标准正态分布(0,1) |
| `rng.lognormal(mean, sigma, size)` | mean,sigma,size | 对数正态分布 |
| `rng.multinomial(n, pvals, size)` | n,pvals,size | 多项分布 |
| `rng.multivariate_normal(mean, cov, size)` | mean,cov,size,tol | 多元正态分布 |
| `rng.dirichlet(alpha, size)` | alpha,size | Dirichlet分布 |
| `rng.permutation(x)` | x | 返回随机排列副本 |
| `rng.shuffle(x)` | x,axis | 原地打乱 |
| `rng.permuted(x, axis)` | x,axis,out | 沿指定轴随机排列 |
| `rng.standard_exponential(size)` | size,dtype,out | 标准指数分布 |
| `rng.standard_gamma(shape, size)` | shape,size,dtype,out | 标准Gamma分布 |
| `rng.bytes(length)` | length | 生成随机字节 |

### 1.11 文件读写

| 函数 | 常用参数 | 简要作用 |
|------|----------|----------|
| `np.save(file, arr)` | file,arr,allow_pickle | 保存单个数组到 `.npy` 二进制格式 |
| `np.savez(file, **kwds)` | file,*args/**kwds | 保存多个数组到未压缩 `.npz` |
| `np.savez_compressed(file, **kwds)` | file,*args/**kwds | 保存多个数组到压缩 `.npz` |
| `np.load(file)` | file,mmap_mode,allow_pickle | 加载 `.npy` / `.npz` 文件 |
| `np.savetxt(fname, X)` | fname,X,fmt,delimiter,header,footer,comments | 保存数组到文本文件(CSV等) |
| `np.loadtxt(fname)` | fname,dtype,comments,delimiter,skiprows,usecols,unpack,encoding | 从文本文件加载数组 |
| `np.genfromtxt(fname)` | fname,dtype,comments,delimiter,skip_header/skip_footer,missing_values,filling_values,names | 从文本加载（可处理缺失值） |
| `np.fromfile(file, dtype)` | file,dtype,count,sep,offset | 从二进制文件读取 |
| `ndarray.tofile(fid)` | fid,sep,format | 写入到二进制文件 |
| `np.frombuffer(buffer)` | buffer,dtype,count,offset | 从缓冲区解析数组 |
| `np.memmap()` | filename,dtype,mode('r'/'r+'/'w+'),offset,shape,order | 内存映射文件（处理超大数据） |

### 1.12 视图与内存相关

| 函数/属性 | 常用参数 | 简要作用 |
|-----------|----------|----------|
| `.copy()` / `np.copy()` | order('C'/'F') | 创建深拷贝 |
| `.view()` | dtype/type | 创建视图（同一数据不同解释） |
| `np.shares_memory(a, b)` | a,b,max_work | 两数组是否共享内存 |
| `np.may_share_memory(a, b)` | a,b,max_work | 快速判断是否可能共享内存 |
| `np.ascontiguousarray()` | a,dtype | 转为C顺序连续数组 |
| `np.asfortranarray()` | a,dtype | 转为Fortran顺序连续数组 |
| `np.require(a, requirements)` | a,dtype,requirements | 满足指定内存要求的数组 |
| `np.isfortran(a)` | a | 是否Fortran顺序 |
| `np.iscomplexobj(a)` | a | 是否复数类型 |
| `np.isrealobj(a)` | a | 是否实数类型 |
| `.flags` | 只读 | 内存布局标志（C_CONTIGUOUS等） |

### 1.13 其他实用函数

| 函数 | 常用参数 | 简要作用 |
|------|----------|----------|
| `np.array_equal()` | a1,a2,equal_nan | 严格判断数组是否完全相等 |
| `np.array_equiv()` | a1,a2 | 判断是否广播后相等 |
| `np.isscalar()` | num | 判断是否标量 |
| `np.shape()` | a | 以函数形式获取shape |
| `np.atleast_1d()` / `np.atleast_2d()` / `np.atleast_3d()` | *arys | 确保至少N维 |
| `np.broadcast_to()` | array,shape,subok | 广播到指定形状 |
| `np.broadcast_arrays()` | *args,subok | 广播多个数组互相兼容 |
| `np.ix_()` | *args(1D索引) | 构造开放网格（用于花式索引笛卡尔组合） |
| `np.ogrid` | 切片索引对象 | 开放网格：返回可广播的稀疏向量 |
| `np.mgrid` | 切片索引对象 | 稠密网格：返回完整坐标数组 |
| `np.meshgrid()` | *xi,indexing('xy'/'ij'),sparse,copy | 生成坐标网格矩阵 |
| `np.indices()` | dimensions,dtype,sparse | 生成网格索引 |
| `np.ndenumerate()` | arr | 多维数组枚举 |
| `np.ndindex()` | *shape | 多维索引迭代器 |
| `np.apply_along_axis()` | func1d,axis,arr,*args,**kwargs | 沿指定轴应用函数 |
| `np.apply_over_axes()` | func,a,axes | 在多个轴上重复应用函数 |
| `np.vectorize()` | pyfunc,otypes,signature,cache | 将Python函数转为ufunc（便利但不快） |
| `np.frompyfunc()` | func,nin,nout | 从Python函数创建通用ufunc |
| `np.piecewise()` | x,condlist,funclist | 分段函数求值 |
| `np.asarray()` | a,dtype,order | 转为数组（已是ndarray时不复制） |
| `np.asanyarray()` | a,dtype,order | 转为数组（允许子类通过） |
| `np.asfarray()` | a,dtype | 转为浮点数组 |
| `np.asarray_chkfinite()` | a,dtype,order | 转为数组并检查非NaN/Inf |
| `np.nan_to_num()` | x,copy,nan(0),posinf,neginf | 将NaN替换为0，Inf替换为大数 |
| `np.errstate()` | kwargs(divide/invalid/over/under) | 上下文管理器，控制浮点错误处理 |
| `np.set_printoptions()` | precision,threshold,edgeitems,linewidth,suppress,formatter | 设置打印格式选项 |
| `np.get_printoptions()` | 无参 | 获取当前打印格式选项 |
| `np.who()` | vardict | 打印当前NumPy数组信息（调试用） |

---

## 二、函数详细用法

### 2.1 数组创建函数详解

#### `np.array(object, dtype=None, copy=True, order='K', subok=False, ndmin=0)`

**作用**：将Python序列转换为ndarray，最基础的构造函数。

**参数**：
- `object`：类数组对象（list、tuple等）
- `dtype`：数据类型，省略则自动推断
- `copy`：True（默认）总是复制；False则尽量不复制
- `order`：内存布局，`'C'`行优先，`'F'`列优先，`'K'`保留输入顺序
- `subok`：True则返回子类（如matrix），False强制返回ndarray
- `ndmin`：最少维度数

**示例**：
```python
np.array([1, 2, 3])                          # array([1, 2, 3])
np.array([1, 2, 3], dtype=np.float32)       # 指定类型
np.array([[1, 2], [3, 4]])                  # 二维
np.array([1, 2, 3], ndmin=2)                # shape (1, 3)
```

**注意**：传入不等长嵌套列表可能产生 `dtype=object` 数组，失去数值计算优势。

---

#### `np.zeros(shape, dtype=float, order='C')`

**作用**：创建全0数组。

**参数**：
- `shape`：整数或整数元组
- `dtype`：默认 `float64`

**示例**：
```python
np.zeros(5)                  # [0. 0. 0. 0. 0.]
np.zeros((2, 3), int)        # [[0 0 0] [0 0 0]]
```

---

#### `np.ones(shape, dtype=float, order='C')`

**作用**：创建全1数组，参数同 `zeros`。

**示例**：
```python
np.ones((3, 3))              # 3×3 全1（float64）
```

---

#### `np.full(shape, fill_value, dtype=None, order='C')`

**作用**：创建所有元素为 `fill_value` 的数组。

**参数**：
- `fill_value`：填充值，可以是任意类型

**示例**：
```python
np.full((2, 3), 7)          # [[7 7 7] [7 7 7]]
np.full(3, np.nan)          # [nan nan nan]
np.full((2,2), np.inf)      # 全inf
```

---

#### `np.empty(shape, dtype=float, order='C')`

**作用**：分配内存但**不初始化**，元素值是内存中的旧数据。只在后续会完全覆盖所有元素时使用，速度比 `zeros` 快。

**⚠️ 警告**：读取未赋值位置结果不可预测。使用前必须填充。

**示例**：
```python
buf = np.empty(1000, dtype=np.float64)
buf[:] = compute_values()  # 先填满再读取
```

---

#### `np.eye(N, M=None, k=0, dtype=float)`

**作用**：创建二维数组，对角线位置为1，其余为0。

**参数**：
- `N`：行数
- `M`：列数，默认等于N
- `k`：对角线偏移，0为主对角线，1上偏，-1下偏

**示例**：
```python
np.eye(3)        # 3×3单位矩阵
np.eye(3, k=1)   # 主对角线上方一条线为1
np.eye(3, 4)     # 3×4，主对角线为1
```

---

#### `np.identity(n, dtype=float)`

**作用**：创建 n×n 方阵单位矩阵，等价于 `np.eye(n)`。

---

#### `np.arange([start,] stop, [step,] dtype=None)`

**作用**：创建等间隔数值的一维数组，类似Python `range`。

**参数**：
- `start`：起始值（包含），默认0
- `stop`：结束值（**不包含**）
- `step`：步长，默认1

**示例**：
```python
np.arange(10)            # [0 1 2 3 4 5 6 7 8 9]
np.arange(2, 10, 2)      # [2 4 6 8]
np.arange(10, 0, -1)     # [10  9  8  7  6  5  4  3  2  1]
np.arange(0, 1, 0.1)     # ⚠️ 浮点步长可能精度不准
```

**⚠️ 注意**：浮点步长会因累积误差导致终点不确定。浮点等间隔采样优先用 `np.linspace`。

---

#### `np.linspace(start, stop, num=50, endpoint=True, retstep=False, dtype=None)`

**作用**：在闭区间内生成指定数量均匀分布的点。

**参数**：
- `start`：起点
- `stop`：终点（`endpoint=True`时包含）
- `num`：样本点数，默认50
- `endpoint`：True（默认）包含stop，False不包含
- `retstep`：True时同时返回步长

**示例**：
```python
np.linspace(0, 1, 5)              # [0.   0.25 0.5  0.75 1.  ]
np.linspace(0, 1, 5, endpoint=False)  # [0.  0.2 0.4 0.6 0.8]
x, dx = np.linspace(0, 2*np.pi, 100, retstep=True)  # 同时返回步长
```

---

#### `np.logspace(start, stop, num=50, endpoint=True, base=10.0, dtype=None)`

**作用**：在对数尺度上生成均匀分布的点，等价于 `base ** linspace(start, stop, num)`。

**示例**：
```python
np.logspace(0, 3, 4)    # [1. 10. 100. 1000.]，即10^0到10^3
np.logspace(1, 5, 3, base=2)  # [2. 8. 32.]，即2^1到2^5
```

---

#### `np.zeros_like(a, dtype=None, order='K')` / `np.ones_like(a, ...)` / `np.empty_like(a, ...)` / `np.full_like(a, fill_value, ...)`

**作用**：创建与参考数组a形状和类型相同的新数组。

**示例**：
```python
a = np.array([[1,2],[3,4]], dtype=np.float32)
np.zeros_like(a)   # shape(2,2) float32 全0
np.ones_like(a)    # 全1
np.empty_like(a)   # 未初始化
np.full_like(a, -1) # 全-1
```

---

#### `np.fromfunction(function, shape, dtype=float)`

**作用**：通过坐标函数生成数组，每个位置的值为 `function(i,j,...)`。

**示例**：
```python
np.fromfunction(lambda i, j: i + j, (3, 3))
# [[0. 1. 2.]
#  [1. 2. 3.]
#  [2. 3. 4.]]
```

---

#### `np.tile(A, reps)`

**作用**：将数组A沿各轴重复reps次构造新数组。

**参数**：`reps`为各轴重复次数元组。

**示例**：
```python
np.tile([1,2], 3)           # [1 2 1 2 1 2]
np.tile([[1],[2]], (1,3))   # [[1 1 1] [2 2 2]]
np.tile([[1,2]], (3,1))     # [[1 2] [1 2] [1 2]]
```

---

#### `np.repeat(a, repeats, axis=None)`

**作用**：重复数组元素。与tile不同，repeat逐个元素重复。

**参数**：
- `repeats`：每个元素重复次数（标量或数组）
- `axis`：沿哪个轴重复，None则展平

**示例**：
```python
np.repeat([1,2,3], 2)            # [1 1 2 2 3 3]
np.repeat([[1,2],[3,4]], 2, axis=0)
# [[1 2] [1 2] [3 4] [3 4]]
np.repeat([1,2,3], [1,2,3])      # [1 2 2 3 3 3]
```

---

#### `np.diag(v, k=0)`

**作用**：
- v是一维数组：以v为对角线构造对角矩阵
- v是二维数组：提取其第k条对角线

**示例**：
```python
np.diag([1,2,3])        # [[1 0 0] [0 2 0] [0 0 3]]
np.diag([1,2], k=1)     # [[0 1 0] [0 0 2] [0 0 0]]
np.diag(np.array([[1,2],[3,4]]))  # [1 4]，主对角线
```

---

### 2.2 数组属性详解

| 属性 | 类型 | 说明 |
|------|------|------|
| `a.ndim` | int | 维度数，如矩阵为2 |
| `a.shape` | tuple | 各轴长度，如 `(2,3)` |
| `a.size` | int | 元素总数，即shape各维度乘积 |
| `a.dtype` | dtype | 元素类型对象 |
| `a.itemsize` | int | 单个元素字节数（float64为8） |
| `a.nbytes` | int | 数据区总字节数 = size × itemsize |
| `a.T` | ndarray | 转置视图，等价于 `a.transpose()` |
| `a.flat` | iterator | 一维迭代器，可 `for x in a.flat` |
| `a.imag` | ndarray | 复数虚部（实数数组返回全0视图） |
| `a.real` | ndarray | 复数实部 |
| `a.strides` | tuple | 每个轴步进的字节数，如连续float64矩阵为`(24,8)` |
| `a.flags` | obj | 内存标志：`C_CONTIGUOUS`、`F_CONTIGUOUS`、`OWNDATA`、`WRITEABLE`、`ALIGNED` |
| `a.base` | ndarray/None | 视图所基于的原数组，自己拥有数据则为None |

---

### 2.3 类型转换详解

#### `.astype(dtype, copy=True, casting='unsafe')`

**作用**：转换数组类型，返回新数组。

**参数**：
- `casting`：`'no'`（不允许）、`'equiv'`（字节序变化）、`'safe'`（安全转换，如int→float）、`'same_kind'`（如int32→int64）、`'unsafe'`（任意，默认）

**示例**：
```python
a = np.array([1, 2, 3])
a.astype(np.float32)        # [1. 2. 3.] float32
a.astype(np.complex128)     # 复数
```

**⚠️ 注意**：
- float→int会截断小数部分，不是四舍五入
- 大范围类型转小范围可能溢出：`np.array([300], dtype=int16)` → `[44]`

---

#### `np.iinfo(type)` / `np.finfo(type)`

**作用**：查询类型的取值限制。

**返回字段**：
- 整数：`.min`, `.max`, `.bits`
- 浮点：`.min`（最小正正规数）, `.max`（最大有限值）, `.eps`（机器精度）, `.tiny`（最小正非零）, `.precision`（有效十进位数）

**示例**：
```python
np.iinfo(np.int32).max     # 2147483647
np.iinfo(np.uint8).max     # 255
np.finfo(np.float64).eps   # 2.22e-16
np.finfo(np.float32).eps   # 1.19e-7
```

---

#### `.item(*args)`

**作用**：将单元素数组转为Python标量。

**示例**：
```python
a = np.array([42])
a.item()       # 42 (Python int)
a[()]          # 也可以，等价
int(a)         # 也可以
```

**注意**：对多元素数组调用 `.item()` 会报错。

---

#### `.tolist()`

**作用**：将数组转为嵌套Python列表，元素转为Python原生类型。

**示例**：
```python
np.array([1,2,3]).tolist()        # [1, 2, 3]
np.array([[1,2],[3,4]]).tolist()  # [[1,2],[3,4]]
```

---

### 2.4 形状变换详解

#### `.reshape(*shape, order='C')`

**作用**：在不改变数据的前提下重新解释形状。某个维度写`-1`时自动推导。

**示例**：
```python
a = np.arange(12)
a.reshape(3, 4)         # (3,4)
a.reshape(3, -1)        # -1自动推导为4
a.reshape(-1, 6)        # (2,6)
a.reshape(2, 3, -1)     # (2,3,2)
```

**⚠️ 注意**：总元素数必须不变。内存连续时返回视图，否则返回副本。

---

#### `.transpose(*axes)` / `.T`

**作用**：调换轴顺序。

**示例**：
```python
a = np.arange(24).reshape(2,3,4)
a.T                     # 等价于 a.transpose(2,1,0)
a.transpose(1,0,2)      # 交换前两轴
a.swapaxes(0, 2)        # 交换第0轴和第2轴
```

---

#### `np.expand_dims(a, axis)`

**作用**：在指定位置插入长度为1的新轴。

**示例**：
```python
a = np.array([1,2,3])          # shape (3,)
np.expand_dims(a, axis=0)      # shape (1, 3)
np.expand_dims(a, axis=1)      # shape (3, 1)
a[np.newaxis, :]               # 等价于 axis=0
a[:, np.newaxis]               # 等价于 axis=1
```

---

#### `np.squeeze(a, axis=None)`

**作用**：移除长度为1的轴。指定axis则只移除该位置。

**示例**：
```python
a = np.zeros((1,3,1,4))
np.squeeze(a).shape          # (3,4)，移除所有长度1轴
np.squeeze(a, axis=0).shape  # (3,1,4)
```

---

#### `np.concatenate(arrays, axis=0, dtype=None)`

**作用**：沿已存在的轴连接多个数组。其他轴shape必须一致。

**示例**：
```python
a = np.array([[1,2],[3,4]])
b = np.array([[5,6]])
np.concatenate([a, b], axis=0)  # [[1 2] [3 4] [5 6]]
np.concatenate([a, b.T], axis=1) # 沿axis=1
```

---

#### `np.stack(arrays, axis=0)`

**作用**：沿**新**轴堆叠多个数组（输入数组shape必须完全相同）。

**示例**：
```python
a = np.array([1,2,3])
b = np.array([4,5,6])
np.stack([a, b])         # [[1 2 3] [4 5 6]] shape(2,3)
np.stack([a, b], axis=1) # [[1 4] [2 5] [3 6]] shape(3,2)
```

**区别**：`concatenate` 在已有轴上连接；`stack` 创建新轴堆叠。`vstack`/`hstack`/`dstack`是便捷写法。

---

#### `np.split(ary, indices_or_sections, axis=0)`

**作用**：沿指定轴分割数组。

**参数**：
- `indices_or_sections`：整数表示等分成N份；列表表示分割位置

**示例**：
```python
a = np.arange(9)
np.split(a, 3)                # [array([0,1,2]), array([3,4,5]), array([6,7,8])]
np.split(a, [2,5])            # [array([0,1]), array([2,3,4]), array([5,6,7,8])]
```

---

### 2.5 索引与切片详解

#### 基础索引与切片

```python
a = np.arange(12).reshape(3, 4)
a[1]          # 第1行，shape(4,)
a[1, 2]       # 第1行第2列的元素（标量）
a[-1]         # 最后一行
a[:, 1]       # 第1列
a[0:2, 1:3]   # 前2行、第1-2列
a[::-1]       # 行倒序
a[:, ::-1]    # 列倒序
```

**视图语义**：基础切片**返回视图**，修改切片会影响原数组。

---

#### 布尔索引

```python
a = np.array([18, 25, 30, 15, 22])
mask = a >= 20
print(mask)              # [False  True  True False  True]
print(a[mask])           # [25 30 22]

# 常用组合：条件赋值
a[a < 20] = 20           # 原地修改，小于20的设为20

# 多条件（必须用括号和位运算符）
a[(a > 18) & (a < 30)]   # 且
a[(a < 18) | (a > 30)]   # 或
```

**⚠️ 注意**：布尔索引**返回副本**。必须用 `&`、`|`、`~`（不是 `and`、`or`、`not`）。

---

#### 花式索引（整数数组索引）

```python
a = np.array([10,20,30,40,50])
a[[0,2,4]]      # [10 30 50]
a[[3,0,0]]      # [40 10 10]，可重复选
a[np.array([0,-1])]  # 支持负数索引
```

多维花式索引：
```python
a = np.arange(12).reshape(3,4)
a[[0,2], [1,3]]  # [a[0,1], a[2,3]] = [1, 11]，坐标对
```

**注意**：花式索引**返回副本**。

---

#### `np.where(condition, [x, y])`

**作用**：三参数时条件选择值；单参数时返回满足条件的索引。

**三参数形式**：
```python
a = np.array([1,2,3,4,5])
np.where(a > 3, a, 0)         # [0 0 0 4 5]
np.where(a > 3, '大', '小')   # ['小' '小' '小' '大' '大']
```

**单参数形式**：
```python
np.where(a > 3)     # (array([3, 4]),)，行索引（一维时）
```

---

#### `np.clip(a, a_min, a_max, out=None)`

**作用**：将值裁剪到 `[a_min, a_max]` 范围。

**示例**：
```python
np.clip([80, 200, -5, 150, 0], 0, 100)  # [ 80 100   0 100   0]
np.clip([80, 200, -5], a_min=50, a_max=None)  # [ 80 200  50]
```

---

#### `np.nonzero(a)` / `np.argwhere(a)`

**作用**：返回非零/满足条件元素的索引。
- `nonzero` 返回元组，每个轴一个索引数组
- `argwhere` 返回形状 `(N, ndim)` 的坐标数组

```python
a = np.array([[0,1,0],[2,0,3]])
np.nonzero(a)              # (array([0,1,1]), array([1,0,2]))
np.argwhere(a)             # [[0 1] [1 0] [1 2]]
```

---

### 2.6 数学运算详解

#### 逐元素运算与广播

所有ufunc都支持广播机制：从尾部维度开始比较，维度相等或其中之一为1即可广播。

```python
# 标量广播
a = np.array([1,2,3])
a + 10         # [11 12 13]

# 二维与一维广播
m = np.array([[1,2,3],[4,5,6]])
v = np.array([10,20,30])
m + v          # 每行加v：[[11 22 33] [14 25 36]]

# 列向量与行向量广播
col = np.array([[1],[2],[3]])     # (3,1)
row = np.array([[10,20,30,40]])   # (1,4)
col + row                        # (3,4)
```

#### 常用ufunc注意事项

```python
# 除法：用where避免除零
num = np.array([1.0, 2.0, 3.0])
den = np.array([2.0, 0.0, 5.0])
result = np.zeros_like(num)
np.divide(num, den, out=result, where=den!=0)
# [5. 0. 6.]

# log/exp注意数值范围
np.exp(1000)   # inf，超出float64范围
np.log(0)      # -inf

# out参数避免临时分配
result = np.empty_like(a)
np.sqrt(a, out=result)
```

---

### 2.7 聚合统计详解

#### 通用参数

所有聚合函数都支持以下参数：
- `axis`：指定聚合轴，None聚合所有元素
- `keepdims`：True时保留聚合轴（长度为1），便于广播
- `dtype`：指定累加类型（避免溢出）

```python
a = np.array([[1,2],[3,4]])
a.sum()                  # 10
a.sum(axis=0)            # [4 6]
a.sum(axis=1)            # [3 7]
a.sum(axis=0, keepdims=True)  # [[4 6]] shape(1,2)
```

#### 常用聚合函数

| 函数 | 说明 | 注意 |
|------|------|------|
| `sum`/`prod` | 和/积 | 大整数求和注意dtype溢出 |
| `mean`/`median` | 均值/中位数 | median会复制排序 |
| `std`/`var` | 标准差/方差 | `ddof=1`为样本标准差 |
| `min`/`max`/`ptp` | 最小/最大/极差 | |
| `argmin`/`argmax` | 最值索引 | 多维时返回展平索引 |
| `percentile`/`quantile` | 百分位数/分位数 | `q`为0~100(percentile)或0~1(quantile) |
| `cumsum`/`cumprod` | 累积和/积 | 返回相同shape |
| `bincount` | 非负整数计数 | 输入必须非负整数 |
| `all`/`any` | 全为真/存在真 | 常用于条件判断 |

#### NaN安全聚合

以 `nan` 开头的版本会忽略NaN值：
```python
a = np.array([1.0, np.nan, 3.0])
np.mean(a)        # nan
np.nanmean(a)     # 2.0
# 还有 nanstd, nansum, nanmax, nanmin, nanmedian 等
```

#### 其他统计
```python
np.cov(X)               # 协方差矩阵
np.corrcoef(X)          # 相关系数矩阵
np.histogram(a, bins=10) # 直方图，返回(频数,边界)
```

---

### 2.8 排序与集合详解

#### 排序

```python
a = np.array([3,1,4,1,5,9])
np.sort(a)              # 返回排序副本：[1 1 3 4 5 9]
np.argsort(a)           # 排序后原索引：[1 3 0 2 4 5]
a[np.argsort(a)]        # 用argsort结果排序

# 原地排序
a.sort()

# 按行/列排序
m = np.array([[3,1],[2,4]])
np.sort(m, axis=0)      # 按列排序
np.sort(m, axis=1)      # 按行排序

# 部分排序（找Top-K最快）
np.partition(a, 3)      # 第3小位置正确，左边都更小，右边都更大（但内部无序）
np.argpartition(a, 3)
```

#### 搜索
```python
a = np.array([1,3,5,7,9])
np.searchsorted(a, 4)   # 2，查找插入位置（数组须有序）
np.searchsorted(a, [2,6]) # [1 3]
```

#### 集合操作
```python
a = np.array([1,2,3,3])
b = np.array([3,4,5])

np.unique(a)                    # [1 2 3] 去重
vals, counts = np.unique(a, return_counts=True)  # [1 2 3], [2 1 1]
np.intersect1d(a, b)            # [3] 交集
np.union1d(a, b)                # [1 2 3 4 5] 并集
np.setdiff1d(a, b)              # [1 2] 差集
np.setxor1d(a, b)               # [1 2 4 5] 对称差
np.isin([1,3,5], a)             # [ True True False] 是否在a中
```

---

### 2.9 线性代数详解

#### 矩阵乘法

```python
A = np.array([[1,2],[3,4]])
B = np.array([[5,6],[7,8]])

A * B          # 逐元素乘法（Hadamard积）
A @ B          # 矩阵乘法（推荐），等价于 np.matmul(A,B)
np.dot(A, B)   # 二维等价于matmul；一维为点积
```

**注意**：`*` 是逐元素，`@`/`matmul`/`dot` 才是矩阵乘法。

#### 核心线性代数函数

```python
# 解线性方程组 Ax = b
A = np.array([[2.0, 1.0], [1.0, 3.0]])
b = np.array([8.0, 13.0])
x = np.linalg.solve(A, b)      # 优先使用，数值稳定
# 不要用 inv(A) @ b！

# 行列式与逆
np.linalg.det(A)               # 5.0
inv_A = np.linalg.inv(A)
A @ inv_A                      # 近似单位矩阵

# 范数
v = np.array([3.0, 4.0])
np.linalg.norm(v)              # 5.0，L2范数（默认）
np.linalg.norm(v, ord=1)       # 7.0，L1范数
np.linalg.norm(v, ord=np.inf)  # 4.0，无穷范数（max绝对值）

# 特征分解
eigenvalues, eigenvectors = np.linalg.eig(A)
# 对称矩阵用eigh更快更稳定
eigvals, eigvecs = np.linalg.eigh(A)

# SVD奇异值分解
U, S, VT = np.linalg.svd(A, full_matrices=False)

# 最小二乘
x, residuals, rank, s = np.linalg.lstsq(X, y, rcond=None)

# QR分解
Q, R = np.linalg.qr(A)

# Cholesky分解（A必须对称正定）
L = np.linalg.cholesky(A)
```

#### 常用矩阵构造
```python
np.trace(A)                    # 迹（对角线之和）
np.diag(A)                     # 提取对角线
np.diag([1,2,3])               # 构造对角矩阵
np.tril(A)                     # 下三角
np.triu(A)                     # 上三角
np.outer([1,2,3], [4,5,6])     # 外积
np.cross([1,0,0], [0,1,0])     # 叉积：[0,0,1]
```

---

### 2.10 随机数详解

> 推荐使用 `np.random.default_rng()` 创建独立生成器，避免全局状态。

```python
rng = np.random.default_rng(seed=42)  # 固定种子可复现

# 基本分布
rng.random((2,3))                    # [0,1)均匀分布
rng.uniform(-1, 1, size=10)          # 指定范围均匀分布
rng.integers(0, 10, size=5)          # [low, high)整数
rng.normal(0, 1, size=1000)          # 正态分布(均值, 标准差)
rng.standard_normal(1000)            # 标准正态(0,1)
rng.poisson(lam=5, size=100)         # 泊松分布
rng.exponential(scale=2, size=100)   # 指数分布
rng.binomial(n=10, p=0.5, size=10)   # 二项分布
rng.multivariate_normal(mean, cov, size=100)  # 多元正态

# 抽样
rng.choice([1,2,3,4], size=2, replace=False)  # 无放回
rng.choice(['A','B'], size=100, p=[0.7,0.3])  # 加权抽样
rng.permutation(10)                 # 0-9的排列
rng.permutation(arr)                # 副本排列
rng.shuffle(arr)                    # 原地打乱
rng.permuted(arr, axis=0)           # 沿轴随机排列（副本）

# 随机字节
rng.bytes(16)                       # 16字节随机数据
```

**⚠️ 最佳实践**：
- 在程序入口创建一个 `rng`，将其作为参数传入需要随机性的函数
- 不要在每个函数内部固定seed
- 避免使用旧API：`np.random.seed()`、`np.random.rand()`、`np.random.randn()`

---

### 2.11 文件读写详解

```python
# NumPy原生二进制（.npy/.npz）
arr = np.arange(12).reshape(3,4)
np.save("data.npy", arr)            # 保存单个数组
loaded = np.load("data.npy")        # 加载

np.savez("data.npz", X=arr, y=np.array([0,1,1]))  # 多数组，未压缩
np.savez_compressed("data.npz", X=arr, y=np.array([0,1,1]))  # 压缩
with np.load("data.npz") as data:
    X = data["X"]
    y = data["y"]

# 文本（CSV/TSV等）
np.savetxt("data.csv", arr, delimiter=",", fmt="%.3f", header="a,b,c,d")
arr = np.loadtxt("data.csv", delimiter=",", skiprows=1)
arr = np.genfromtxt("data.csv", delimiter=",", filling_values=0)  # 处理缺失值

# 原始二进制
arr.tofile("data.bin")
arr = np.fromfile("data.bin", dtype=np.float64).reshape(3,4)

# 内存映射（处理大于内存的数据）
mm = np.memmap("huge.dat", dtype=np.float32, mode="w+", shape=(1_000_000, 512))
mm[0] = 1.0           # 写入
chunk = mm[100:200]   # 读取，按需加载
mm.flush()            # 写回磁盘
```

**⚠️ 安全提示**：加载不可信来源的 `.npy/.npz` 时，保持 `allow_pickle=False`（默认），禁用pickle。

---

### 2.12 高级实用函数详解

#### `np.einsum(subscripts, *operands)` — 爱因斯坦求和

**作用**：用统一符号表达任意多维线性代数运算，极度灵活。

```python
a = np.array([1,2,3])
b = np.array([4,5,6])
A = np.array([[1,2],[3,4]])
B = np.array([[5,6],[7,8]])

np.einsum('i->', a)           # 求和: 6
np.einsum('i,i->', a, b)      # 点积: 32
np.einsum('i,j->ij', a, b)    # 外积 (3,3)
np.einsum('ij,jk->ik', A, B)  # 矩阵乘法
np.einsum('ii->', A)          # 迹: 5
np.einsum('ij->ji', A)        # 转置
```

规则：重复索引表示相乘并求和；输出索引保留。

---

#### `np.meshgrid(*xi, indexing='xy')`

**作用**：生成坐标网格，用于3D绘图或函数求值。

```python
x = np.array([1,2,3])
y = np.array([4,5])
xx, yy = np.meshgrid(x, y)
# xx = [[1 2 3]
#       [1 2 3]]
# yy = [[4 4 4]
#       [5 5 5]]
```

---

#### `np.nan_to_num(x, copy=True, nan=0.0, posinf=None, neginf=None)`

**作用**：将NaN替换为0，Inf替换为大的有限值。

```python
a = np.array([1.0, np.nan, np.inf, -np.inf])
np.nan_to_num(a)              # [ 1.00e+00  0.00e+00  1.80e+308 -1.80e+308]
```

---

#### `np.apply_along_axis(func1d, axis, arr, *args)`

**作用**：沿指定轴对每个一维切片应用函数。

```python
a = np.array([[1,2,3],[4,5,6]])
np.apply_along_axis(lambda x: x.max() - x.min(), axis=1, arr=a)
# [2 2]，每行的极差
```

**注意**：这不是向量化运算，本质是循环，仅为便利。

---

#### `np.vectorize(pyfunc)`

**作用**：将Python标量函数包装为可作用于数组的ufunc。

```python
def my_func(x):
    return x*2 if x > 0 else x*3

vfunc = np.vectorize(my_func)
vfunc(np.array([-1, 0, 1, 2]))  # [-3  0  2  4]
```

**⚠️ 注意**：`vectorize` 是语法糖，**不提供性能提升**，内部仍为Python循环。性能敏感时改写为真正的向量化表达式。

---

#### `np.broadcast_to(array, shape, subok=False)`

**作用**：将数组广播到指定形状，返回只读视图。

```python
a = np.array([1,2,3])
np.broadcast_to(a, (2,3))  # [[1 2 3] [1 2 3]]
```

---

#### `np.allclose(a, b, rtol=1e-05, atol=1e-08)` / `np.isclose(a, b, ...)`

**作用**：浮点近似相等比较。

- `|a - b| ≤ atol + rtol * |b|`
- `isclose` 返回布尔数组，`allclose` 返回单个布尔

```python
0.1 + 0.2 == 0.3                  # False
np.allclose(0.1+0.2, 0.3)         # True
```

**⚠️ 永远不要用 `==` 比较浮点数。**

---

#### 打印控制

```python
np.set_printoptions(
    precision=3,           # 小数位数
    suppress=True,         # 禁用科学计数法
    linewidth=120,         # 每行字符数
    threshold=1000,        # 超过多少元素省略显示
    edgeitems=3,           # 省略时两端显示几个
)
```

---

## 三、常用数据类型速查

| 类型 | 字节 | 范围/精度 | 典型用途 |
|------|------|-----------|----------|
| `bool_` | 1 | True/False | 掩码、条件 |
| `int8` | 1 | -128 ~ 127 | 图像像素、小整数 |
| `int16` | 2 | -32768 ~ 32767 | 音频采样 |
| `int32` | 4 | -2³¹ ~ 2³¹-1 | 索引、一般整数 |
| `int64` | 8 | -2⁶³ ~ 2⁶³-1 | 大整数计数、ID |
| `uint8/16/32/64` | 1/2/4/8 | 0 ~ 2^n-1 | 非负数据、位运算 |
| `float16` | 2 | ~3位十进制精度 | 深度学习混合精度 |
| `float32` | 4 | ~7位十进制精度 | 深度学习、图像、GPU计算 |
| `float64` | 8 | ~16位十进制精度 | 科学计算默认 |
| `complex64` | 8 | 两个float32 | 频域信号处理 |
| `complex128` | 16 | 两个float64 | 科学计算复数 |

**类型选择建议**：
- 索引和计数：`int64`（或平台相关的 `intp`）
- 通用科学计算：`float64`（默认）
- 深度学习/大量数据：`float32`（节省内存，GPU更快）
- 图像/音频：`uint8`/`int16`

---

## 四、广播规则速记

广播三规则（从后往前比较维度）：
1. 维度数不同时，短的shape在左侧补1
2. 任一维度为1时，沿该维度复制以匹配
3. 维度既不相等也不为1时报错

```text
(3,)      → 可广播到 (2,3)、(1,3)、(3,)
(3,1)     → 可广播到 (3,4)、(2,3,4)
(2,3)     → 可广播到 (1,2,3)、(5,2,3)
(3,4)+(4,)→ ✓ (4)视为(1,4)→广播到(3,4)
(2,3)+(3,2)→ ✗ 尾部维度既不等也不为1
```

---

## 五、常见错误与陷阱

| 错误现象 | 原因 | 解决方案 |
|----------|------|----------|
| `a * b` 结果不对 | `*` 是逐元素乘 | 矩阵乘用 `@` |
| 修改切片原数组变了 | 切片是视图共享内存 | 需要隔离用 `.copy()` |
| 浮点相等判断为False | 二进制浮点精度有限 | 用 `np.allclose`/`np.isclose` |
| 整数溢出无声出错 | dtype范围不够 | 用更宽的类型（如int32→int64） |
| 广播报错 | shape不兼容 | 检查shape，用 `newaxis`/`keepdims` 调整 |
| `np.append` 很慢 | 每次都复制整个数组 | 预分配或先存list最后转array |
| `arange(0,1,0.1)` 长度不对 | 浮点步长累积误差 | 用 `linspace` |
| 布尔索引用`and/or/not`报错 | Python关键字无法重载 | 用 `&`/`|`/`~` 并加括号 |
| 修改切片副本Warning | 链式索引如 `a[a>0][0]=1` | 一次性索引 `a[a>0] = 1` |
| `np.nan == np.nan` 为False | NaN不等于任何值包括自己 | 用 `np.isnan()` 判断 |
| 视图修改导致数据错乱 | `ravel()`/`reshape`可能返回视图 | 不确定时用 `.copy()` 保证独立 |
| `load`时pickle错误 | 默认禁用pickle防安全风险 | 受信任来源才设 `allow_pickle=True` |
