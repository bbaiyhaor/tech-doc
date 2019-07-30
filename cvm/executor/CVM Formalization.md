# CVM Formalization

### Predefined Structure

#### TShape

struct `TShape`, a array of `uint32_t`, which is used to standing for data shape.

#### DLTensor

Data format，a wrapper of data content pointer.

*Attributes*

- dtype must be INT8 or INT32.

- shape is `TShape` type, storing data shape, for example (1, 2, 3).

- precision is the max bit of each element take up, -1 stands for non-defined; normal value betweens range (0, 32]; others is non-available, which means a error occurs, either logic error or runtime error.

  *Notes*: we define a `p` precision data is between range $[-\alpha, \alpha], \alpha=2^{p-1}-1$ for all elements, for example: data that precision is 8 means all value is larger than -128 and less than 128, this is strong constraint.

*Public Interface*

- ndim is the dimensions of data, for example 3.

#### Attribute Constant

*max_attr* = 4096

*min_attr* = 0

## Executor

## Ops

本节主要介绍 *cvm executor* 的算子形式化描述，包含但不限于

1. 输入，输出和参数限制 (restriction of inputs and outputs)
2. 源代码引用 (source code reference)
3. 数学形式化描述 (math formalization)

##### Inputs & Outputs

算子输入输出的数据格式为`DLTensor`，precision属性在(0, 32]范围内。

### Reduce Operator

Reduce operator perform the reduction function to input data based on the parameters, and the process logic over all the type-based operators is consistent. We abstract the formalization here and introduce the details as belows:

*Attributes Constraints*

1. axes is `TShape` type, by default (). The axis or axes along which to perform the reduction.
   + no duplicate
   + range [-ndim, ndim). If axis < 0, axis += ndim
2. keepdims is `false` (`boolean` type) by default. If this is set to `True`, the reduced axes are left with 1.
3. exclude is `false` by default. Whether to perform reduction on axis that are NOT in axis instead.

*Math Formalization*

Suppose Input `X`, Output `Y`, attributes `axes`, `exclude` and reduce function `REDUCE_OP`.

```python
real_axes = calculate the real axes depending on (axes, exclude)
```

Reference: https://github.com/CortexFoundation/CortexTheseus/blob/76320455f0769dbf22115d82181b7ba876c5f942/infernet/src/cvm/ops/cpu/reduce.cc#L6

1. len(real_axes) == 0 and exclude == true 

   Math:	$Y = X$

   Reference: https://github.com/CortexFoundation/CortexTheseus/blob/76320455f0769dbf22115d82181b7ba876c5f942/infernet/src/cvm/ops/cpu/reduce.cc#L42

2. len(real_axes) == 0 and exclude == false

   Math: 

   ```python
   Y[0] = X[0] # do reduce op over all axis
   for x in X[1:]:
     Y[0] = REDUCE_OP(x, Y[0])
   ```

   Reference: https://github.com/CortexFoundation/CortexTheseus/blob/76320455f0769dbf22115d82181b7ba876c5f942/infernet/src/cvm/ops/cpu/reduce.cc#L45

3. other case

   Math:

   ```python
   init Y with 0
   for xidx in range(X.shape):
   	yidx = [x if ix not in real_axes for ix in xidx]
   	Y[yidx] = REDUCE_OP(Y[yidx], X[xidx])
   ```

   Reference: https://github.com/CortexFoundation/CortexTheseus/blob/76320455f0769dbf22115d82181b7ba876c5f942/infernet/src/cvm/ops/cpu/reduce.cc#L52


#### max

set `REDUCE_OP` to be `max`.

#### sum

set `REDUCE_OP` to be `sum`.

Example:

```python
data = [[[1, 2], [2, 3], [1, 3]],
        [[1, 4], [4, 3], [5, 2]],
        [[7, 1], [7, 2], [7, 3]]]

sum(data, axis=(1))
[[  4.   8.]
 [ 10.   9.]
 [ 21.   6.]]

sum(data, axis=[1,2])
[ 12.  19.  27.]
```

### Broadcast Operator

Broadcast operator perform the broadcast function to input datas, and the process logic over all the type-based operators is consistent. We abstract the formalization here and introduce the details as belows:

*Math Formalization*

Suppose Input `A`, `B`, Output `Y` and broadcast function `BROADCAST_OP`. Where `A`'s shape is M dimension, exactly $(m_0, m_1, \cdots, m_{M-1})$, `B`'s shape is N dimension, exactly $(n_0, n_1, \cdots, n_{N-1})$.
$$
K = max(M, N) \\
SA[p] = \begin{cases}
m_{p-K+M}, & p \geqslant K - M \\
1, & p < K - M
\end{cases}, p \in [0, K) \\
SB[q] = \begin{cases}
n_{q-K+N}, & q \geqslant K-N \\
1, & q < K - N
\end{cases}, q \in [0, K) \\
\forall \{i \mid i \in [0, K) \}: SA[i]=SB[i] \or SA[i]=1 \or SB[i]=1 \\
\forall (d_0, d_1, \cdots, d_{K-1}), \text{where } d_j \in [0, max(SA[j], SB[j])) \and j \in [0, K): \\
Y[d_0, d_1, \cdots, d_{K-1}] = 
\text{BROADCAST_OP}(A[a_0, a_1, \cdots, a_{M-1}], B[b_0, b_1, \cdots, b_{N-1}]), \\
\text{where } a_i = min(d_{K-M+i}, m_i) \and i \in [0, M) \and \\
b_j = min(d_{K-N+j}, n_j) \and j \in [0, N)
$$


Math:

```python
1. Make A, B shape length equal, 
	suppose A.shape is (a1, a2, ..., aN), B.shape is (b1, b2, ... bM), and N < M,
	extend A.shape to (1, 1, ..., 1, a1, a2, ..., aN), total number of 1 is M-N.
2. Promise the input shape satisfy the constraint,
  foreach i in len(shape): 
  	(A.shape[i] == 1) or (B.shape[i] == 1) or (A.shape[i] == B.shape[i])
3. Calculate output shape,
	oshp = [max(A.shape[i], B.shape[i]) for i in len(shape)]
4. for oidx in range(oshp):
  		# calulate input index by output index
  		aidx = [min(A.shape[i], oidx[i]) for i in oidx]
    	bidx = [min(B.shape[i], oidx[i]) for i in oidx]
      Y[oidx] = BROADCAST_OP(A[aidx], B[bidx])
```

Reference: https://github.com/CortexFoundation/CortexTheseus/blob/76320455f0769dbf22115d82181b7ba876c5f942/infernet/src/cvm/ops/cpu/ops.cc#L575

#### broadcast_add

set `BROADCAST_OP` to `add`.

Example:

```python
x = [[ 1.,  1.,  1.],
     [ 1.,  1.,  1.]]

y = [[ 0.],
     [ 1.]]

broadcast_add(x, y) = [[ 1.,  1.,  1.],
                       [ 2.,  2.,  2.]]
```

#### broadcast_sub

set `BROADCAST_OP` to `sub`.

#### broadcast_mul

set `BROADCAST_OP` to `multiply`.

#### broadcast_max

set `BROADCAST_OP` to `max`.

### NN Operator

#### Convolution

We only supported 2-D convolution operator.

*Math Formalization*

Suppose Input `X`, `W`, `B`, and output `Y`, where `X` format is `NCHW`(batch, in_channels, height, weight), `W` format is `OIHW`(out_channels, in_channels, kernel_h, kernel_w), `B` format is `O`(out_channels)

Math:
$$
Y[n,i,p,q]=\sum_{j=0}^{in\_channels} kernel(X[n,j, p:p+\text{kernel_h}, q:q+\text{kernel_w}], W[i,j,:,:])
+ B[i], \\
\forall \quad n \in N \and i \in C \and
p \in \text{floor}(H - \text{pool_size_h}) + 1 \and
q \in \text{floor}(W - \text{pool_size_w}) + 1
$$
where $kernel$ function is 
$$
kernel(A, B) = \sum_{ki=0}^{kernel\_h} \sum_{kj = 0}^{kernel\_w} A[ki, kj] * B[ki, kj]
$$
Reference: https://github.com/CortexFoundation/CortexTheseus/blob/76320455f0769dbf22115d82181b7ba876c5f942/infernet/src/cvm/ops/cpu/ops.cc#L475

In source code at reference, many attributes are designed for more complicated situation.

1. padding, 2-D array, zero pad for input data at edge of image.
2. stride, 2-D array, the strides of kernel move under input image.
3. dilation, 2-D array, select input image data with dilation for kernel convolution.

 the output shape calculation with attributes above is
$$
f(x) = floor((x+2*\text{padding}-\text{dilation}*(\text{kernel_size}-1)-1)/\text{stride})+1
$$

##### Depth-Wise Convolution

Math:
$$
Y[n,i,p,q]=B[i] + kernel(X[n,i, p:p+\text{kernel_h}, q:q+\text{kernel_w}], W[i,1,:,:])
$$
Reference: https://github.com/CortexFoundation/CortexTheseus/blob/76320455f0769dbf22115d82181b7ba876c5f942/infernet/src/cvm/ops/cpu/ops.cc#L390

#### Dense

*Math Formalization*

Suppose Input `X`, `W`, `B`, where `X` shape is (M * K), `W` shape is (N * K), `B` shape is (N,)

Math:
$$
Y=XW^T+B
$$
Reference: https://github.com/CortexFoundation/CortexTheseus/blob/76320455f0769dbf22115d82181b7ba876c5f942/infernet/src/cvm/ops/cpu/ops.cc#L86

#### MaxPooling

We only supported 2-D max pooling.

*Math Formalization*

Suppose Input `X`, `pool_size`, where `X` format is `NCHW`, `pool_size` is two dimension array

Math:
$$
Y[n,i,p,q] = max\{X[n,i, p:p+\text{pool_size_h}, q:q+\text{pool_size_w}]\}, \\
\forall \quad n \in N \and i \in C \and
p \in \text{floor}(H - \text{pool_size_h}) + 1 \and
q \in \text{floor}(W - \text{pool_size_w}) + 1
$$
Reference: https://github.com/CortexFoundation/CortexTheseus/blob/76320455f0769dbf22115d82181b7ba876c5f942/infernet/src/cvm/ops/cpu/ops.cc#L692

In source code at reference, many attributes are supplied as below:

1. padding, 2-D array, zero pad for input data at edge of image.
2. strides, 2-D array, the strides of kernel move under input image.
3. ceil_mode, boolean, when true, will use ceil instead of floor to compute the output shape.

 the output shape calculation with attributes above is
$$
p \in \text{floor}((H+2*\text{padding}-\text{pool_size_h})/\text{stride})+1 \\
q \in \text{floor}((W + 2 * \text{padding} - \text{pool_size_w})/\text{stride}) + 1
$$

### Elemwise Operator

#### abs

*Math Formalization*

Suppose Input `X`, Output `Y`.

Math:
$$
y = \begin{cases}
x, &  x \in X \and x \geqslant 0  \\
-x, & x \in X \and x < 0 
\end{cases}
$$

#### cvm_precision

*Math Formalization*

Suppose Input `X`, Output `Y`.

Math:
$$
y = \begin{cases}
\lceil log_2(abs(x+1)) \rceil, & x \in X \and x \neq 0\\
1, & x \in X \and  x = 0 
\end{cases}
$$

#### elemwise_add

*Math Formalization*

Suppose Input `A`, `B`, Output `Y`. Where `A` and `B` have the same shape, $(n_0, n_1, \cdots, n_{N-1})$.
$$
Y = A + B
$$

#### elemwise_sub

*Math Formalization*

Suppose Input `A`, `B`, Output `Y`. Where `A` and `B` have the same shape, $(n_0, n_1, \cdots, n_{N-1})$.
$$
Y = A - B
$$

#### negative

*Math Formalization*

Suppose Input `X`, Output `Y`.
$$
Y = -X
$$

#### clip

*Math Formalization*

Suppose Input `X`, Output `Y`, attribute `a_min`, `a_max`.
$$
y = \begin{cases}
\text{a_max}, & x \in X \and x \geqslant \text{a_max} \\
x, & x \in X \and x \in (\text{a_min}, \text{a_max}) \\
\text{a_min}, & x \in X \and x \leqslant \text{a_min}
\end{cases}
$$


#### cvm_clip

*Math Formalization*

Suppose Input `X`, Output `Y`, attribute `precision`.
$$
\alpha = 2^{\text{precision}-1}-1, \quad \text{precision} \in [1, 32] \\
Y = clip(X, \text{a_min}=-\alpha, \text{a_max}=\alpha)
$$


#### cvm_right_shift

*Math Formalization*

Suppose Input `X`, Output `Y`, attribute `precision`, `shift_bit`.
$$
\alpha = 2^{\text{precision} - 1} - 1, \quad \text{precision} \in [1, 32] \\
T = {\left\lfloor 
(\left\lfloor \frac{X}{2^{\text{shift_bit}} - 1} \right\rfloor + 1) 
\div 2 \right\rfloor}, \quad \text{shift_bit} \in [1, 32] \\
Y = clip(T, \text{a_min} = -\alpha, \text{a_max}=\alpha)
$$

#### cvm_left_shift

*Math Formalization*

Suppose Input `X`, Output `Y`, attribute `precision`, `shift_bit`. Where $\text{precision}$ is in range $[1, 32]$, and $\text{shift_bit}$ is in range $[1, 32]$.
$$
\alpha = 2^{\text{precision} - 1} - 1 \\
T = X * 2^\text{shift_bit} \\
Y = clip(T, \text{a_min} = -\alpha, \text{a_max}=\alpha)
$$

### Transform Operator

#### repeat

*Math Formalization*

Suppose Input `X`, Output `Y`, attribute `axis`, `repeats`. Where `X`'s shape is N dimension, exactly  $(n_0, n_1, \cdots, n_{\text{axis}}, \cdots, n_{N-1})$, `Y`'s shape is N dimension, exactly $(n_0, n_1, \cdots, n_{\text{axis}} \cdot repeats, \cdots, n_{N-1})$. Obviously, axis is in range $[0, N)$, repeats is in range $[1, +\infty)$.
$$
\forall \{(d_0, d_1, \cdots, d_{N-1}) \mid d_j \in 
\begin{cases} 
[0, n_j), & j \ne \text{axis} \\
[0, n_j \cdot \text{repeats}), & j = \text{axis}
\end{cases} \and j \in [0, N) \}: \\
Y[d_0, d_1, \cdots, d_\text{axis}, \cdots, d_{N-1}] = 
X[d_0, d_1, \cdots, \left\lfloor{d_\text{axis} \over \text{repeats}}\right\rfloor, \cdots, d_{N-1}]
$$


#### tile

*Math Formalization*

Suppose Input `X`, Output `Y`, attribute `reps`. Where `X`'s shape is N dimension, exactly  $(n_0, n_1, \cdots, n_{N-1})$, `reps` is M dimension, exactly $(m_0, m_1, \cdots, m_{M-1})$.
$$
\forall r \in \text{reps}: r \in [1, max\_attr) \\
\forall (k_0, k_1, \cdots, k_{K-1}), 
\text{where } k_j \in [0, SX[j] \cdot SR[j]) \and j \in [0, K) \and K=max(M, N): \\
Y[k_0, \cdots, k_{K-N-1}, k_{K-N}, k_{K-N+1}, \cdots, k_{K-1}] = \\
X[k_{K-N+0} \text{ mod } n_0, k_{K-N+1} \text{ mod } n_1, \cdots, k_{K-N+N-1} \text{ mod } n_{N-1}], \\
\text{where } SX[p] = \begin{cases}
n_{p-K+N}, & p \in [K-N, K-1) \\
1, & p \in [0, K-N)
\end{cases} \and p \in [0, K) \and \\
SR[q] = \begin{cases}
m_{q-K+M}, & q \in [K-M, K-1) \\
1, & q \in [0, K-M)
\end{cases} \and q \in [0, K)
$$


#### flatten

*Math Formalization*

Suppose Input `X`, Output `Y`. Where `X`'s shape is N dimension, exactly $(n_0, n_1, \cdots, n_{N-1})$.
$$
\forall (d_0, d_1, \cdots, d_{N-1}), \text{where } d_j \in [0, n_j) \and j \in [0, N): \\
Y[flatten\_index(d_0, d_1, \cdots, d_{N-1}, n_0, n_1, \cdots, n_{N-1})]  =  \\
X[d_0, d_1, \cdots, d_{N-1}]
$$
where $flatten\_index$ is 
$$
flatten\_index(d_0, d_1, \cdots, d_{N-1}, n_0, n_1, \cdots, n_{N-1}) = \\
d_0 * n_1 * n_2 * \cdots * n_{N-1} + d_1 * n_2 * n_3 * \cdots * n_{N-1} + 
\cdots + d_{N-2} * n_{N-1} + d_{N-1}
$$


#### concatenate

*Math Formalization*

Suppose `M` Inputs $I^0, I^1, \cdots, I^{M-1}$, Output `Y`, attribute `axis`. Where all inputs' shape is N dimension, exactly $I^i$'s shape is $(n^i_0, n^i_1, \cdots, n^i_{N-1})$, and `axis` is in range $[0, N)$.
$$
\forall i \in [1, M) \; \forall j \in [0, N) \and j \ne \text{axis}: \;
n^i_j = n^{i-1}_j \\
\forall i \in [0, M) \; \forall (d_0, d_1, \cdots, d_{N-1}), \text{where } d_j \in [0, n^i_j) \and j \in [0, N): \\
Y[d_0, d_1, \cdots, d_\text{axis-1}, \text{new_idx}, d_\text{axis+1}, \cdots, d_{N-1}] = I^i[d_0, d_1, \cdots, d_{N-1}], \\
\text{where new_idx} = n^0_\text{axis} + n^1_\text{axis} + \cdots + n^{i-1}_\text{axis} + d_\text{axis}
$$


#### expand_dims

*Math Formalization*

Suppose Input `X`, Output `Y`, attributes `axis`, `num_newaxis`. Where `X`'s shape is N dimension, exactly $(n_0, n_1, \cdots, n_{N-1})$,  `axis` is in range $[-N-1, N+1)$,  and `num_newaxis` is in range $[min\_attr, max\_attr)$.
$$
\forall (d_0, d_1,\cdots, d_{N-1}), \text{where } d_j \in [0, n_j) \and j \in [0, N): \\
Y[d_0,d_1, \cdots, d_{axis-1}, \underbrace{1, 1, \cdots, 1}_{\text{num_newaxis}}, d_\text{real_axis}, \cdots, d_{N-1}] = X[d_0, d_1, \cdots, d_{N-1}], \\ 
\text{where } \text{real_axis} = 
\begin{cases}
\text{axis},& \text{axis} \geqslant 0 \\
\text{axis} + N,& \text{axis} < 0
\end{cases}
$$


#### reshape

*Math Formalization*

Suppose Input `X`, Output `Y`, attributes `target_shape`. Where `X`'s shape is N dimension, exactly $(n_0, n_1, \cdots, n_{N-1})$, and `target_shape` is M dimension, exactly $(m_0, m_1, \cdots,  m_{M-1})$ , and satisfy constraint : $m_0 * m_1 * \cdots * m_{M-1} = n_0 * n_1 * \cdots * n_{N-1}$.
$$
T = flatten(X) \\
\forall (d_0, d_1, \cdots, d_{N-1}), \text{where } d_j \in [0, m_j) \and j \in [0, M): \\
Y[d_0, d_1, \cdots, d_{N-1}] = T[flatten\_index(d_0, d_1, \cdots, d_{N-1}, n_0, n_1, \cdots, n_{N-1})]
$$


#### squeeze

*Math Formalization*

Suppose Input `X`, Output `Y`, attributes `axes`. Where `X`'s shape is N dimension, exactly $(n_0, n_1, \cdots, n_{N-1})$, and `axes` is TShape and dimension is M.
$$
\forall \text{axis} \in \text{axes}: axis \in [-N, N) \\
\text{real_axes} = 
\begin{cases}
\{\text{axis} \mid \text{axis} \geqslant 0 \and \text{axis} \in \text{axes} \} \bigcup
\{\text{axis} + N \mid \text{axis} < 0 \and \text{axis} \in \text{axis}\},
& M > 0 \\
\{\text{axis} \mid n_\text{axis} = 1 \and \text{axis} \in [0, N) \}, & M = 0
\end{cases} \\
\forall \text{axis} \in \text{real_axes}: n_{axis} = 1 \\
\forall (d_0, d_1, \cdots, d_{N-1}), \text{where } d_j \in [0, n_j) \and j \in [0, N): \\
Y[s_0, s_1, \cdots, s_{N-K-1}] = X[d_0, d_1, \cdots, d_{N-1}], \\
\text{where } K = \mathbf|\text{real_axes}\mathbf| \and s_{i-count(i)} = d_i 
\and i \in [0, N) \and i \notin \text{real_axes} \and \\
count(i) = \mathbf|\{\text{axis} \mid \text{axis} < i \and \text{axis} \in \text{real_axes} \}\mathbf|
$$


#### transpose

*Math Formalization*

Suppose Input `X`, Output `Y`, attributes `axes`. Where `X`'s shape is N dimension, exactly $(n_0, n_1, \cdots, n_{N-1})$, and `axes` is TShape and dimension is M, where M is in $\{0, N\}$.
$$
\forall \text{axis} \in \text{axes}: \text{axis} \in [-N, N) \\
\text{real_axes} = \begin{cases}
(r_0, r_1, \cdots, r_{N-1}), \text{where } r_j = 
\begin{cases} 
\text{axes}_j, & \text{axes}_j \geqslant 0 \\
\text{axes}_j + N, & \text{axes}_j < 0 
\end{cases} \and
j \in [0, N), & M=N\\
(n_{N-1}, n_{N-2}, \cdots, n_0), & M = 0
\end{cases} \\
\{i \mid i \in \text{real_axes}\} = \{j \mid j \in [0, N) \} \\
\forall (d_0, d_1, \cdots, d_{N-1}), 
\text{where } d_j \in [0, n_{\text{real_axes}_{j}}) \and j \in [0, N): \\
Y[d_0, d_1, \cdots, d_{N-1}] = X[s_0, s_1, \cdots, s_{N-1}], \\
\text{where } s_{\text{real_axes}[j]} = d_j \and j \in [0, N)
$$


#### slice | strided_slice

#### take

#### cvm_lut

#### slice_like



### NMS Operator



### Common Operator