# Kernel - Inner Product

<!-- > [!NOTE] -->
> The implementation of computing inner product between binary codes and query vectors heavily affects the efficiency. The best implementation may vary largely across platforms and the dimensionality of datasets. For now, we only include the implementation with `__builtin_popcountll` and expect that compilers will automatically vectorize it.

This part introduces how to compute the inner product between quantization codes and rotated query vectors i.e., $\left< \mathbf{x}_0,\mathbf{q}_r'\right>$ and $\left< \mathbf{x}_u,\mathbf{q}_r'\right>$. The implementation includes two types:

1. The inner product between binary codes and $\mathbf{q}_r'$.
2. The inner product between multi-bit codes and $\mathbf{q}_r'$.


## The Kernel for Binary Codes

### Single Code
We compute the inner product between a single binary vector $\mathbf{x}_0$ and a floating-point vector $\mathbf{q}_r'$ via bitwise-and `&` and `popcnt`. We first quantize $\mathbf{q}_r'$ into a vector of 4-bit unsigned integers (based on a fast version of RaBitQ). 

$$
\mathbf{q}_r'\approx \Delta_q \mathbf{q}_u + v_q\cdot \mathbf{1}_D
$$

Let $\mathbf{q}_u^{(i)}$ be the $i$-th bit of $\mathbf{q}_u$. Then with the quantized vector $\mathbf{q}_u$, we can compute the inner product based on the following formula:

$$
\begin{align}
\left< \mathbf{x}_0,\mathbf{q}_r'\right>
&\approx \left< \mathbf{x}_0,\Delta_q \mathbf{q}_u + v_q\cdot \mathbf{1}_D\right>
\\&=\Delta_q \left< \mathbf{x}_0,\mathbf{q}_u\right> + v_q \cdot \mathrm{popcnt}(\mathbf{x}_0)
\\ &=\Delta_q \sum_{i=0}^{B_q-1} \left( 2^i\left< \mathbf{x}_0,\mathbf{q}_u^{(i)}\right> \right) + v_q \cdot\mathrm{popcnt}(\mathbf{x}_0)
\\ &=\Delta_q \sum_{i=0}^{B_q-1} 2^i\cdot \mathrm{popcnt}(\mathbf{x}_0\ \&\  \mathbf{q}_u^{(i)})  + v_q \cdot \mathrm{popcnt}(\mathbf{x}_0)
\end{align}
$$


### Batch Code
We compute the inner product between a batch of binary vectors $\mathbf{x}_0$ and a floating-point vector $\mathbf{q}_r'$ via `FastScan`. Here we provide a brief introduction and refer readers to a detailed [tutorial by Faiss](https://github.com/facebookresearch/faiss/wiki/Fast-accumulation-of-PQ-and-AQ-codes-(FastScan)).

For a $D$-bit binary, we split it into $M=D/4$ segments. We prepare look-up-tables for each segment.
- $\mathrm{LUT}[m][mask]$: the inner product with $\mathbf{q}_r'$ in the $m$-th segment, i.e., the $(4m)$-th to the $(4m+3)$-th dimensions, when the code in the $m$-th segment equals to $mask$. 

Based on the look-up-tables, we can compute the inner product as follows:

$$
\begin{align}
\left< \mathbf{x}_0,\mathbf{q}_r'\right>= \sum_{m=0}^{M-1}  \mathrm{LUT}[m][\mathbf{x}_{0}[4m:4m+3]] 
\end{align}
$$


## The Kernel for Multi-bit Codes

For the multi-bit codes, we convert the unsigned integer codes to floating point numbers with native instructions of AVX512. 