
> 本附录对应正文**Chapter 02**第 2、4、6 节，用最小 PyTorch 实验验证矩阵乘法在表征转换、推理瓶颈和内存布局中的表现。

### A.1 完整代码

```
import time
import torch

torch.manual_seed(42)

# 1. 验证表征空间转换
x = torch.tensor([[1.2, 0.5, -0.8, 2.1]])
W = torch.randn(4, 3)
output = torch.matmul(x, W)
print(f"原始特征 (4维): {x.tolist()}")
print(f"映射后的表征空间 (3维隐向量): {output.tolist()}\n")

# 2. 模拟 Prefill 与 Decode
M, K, N = 2048, 2048, 2048
A_large = torch.randn(M, K)
B_large = torch.randn(K, N)

start = time.perf_counter()
C_large = torch.matmul(A_large, B_large)
end = time.perf_counter()
print(f"【Prefill 模拟】大矩阵乘法 [{M}x{K}] * [{K}x{N}] 耗时: {(end - start)*1000:.2f} ms")

M_decode = 1
A_small = torch.randn(M_decode, K)

start = time.perf_counter()
C_small = torch.matmul(A_small, B_large)
end = time.perf_counter()
print(f"【Decode 模拟】单Token矩阵乘法 [{M_decode}x{K}] * [{K}x{N}] 耗时: {(end - start)*1000:.2f} ms\n")

# 3. 内存布局
size = 4000
A_base = torch.randn(size, size)
A_non_contiguous = A_base.t()
A_contiguous = A_non_contiguous.clone().contiguous()

print(f"A_non_contiguous 内存连续性: {A_non_contiguous.is_contiguous()}")
print(f"A_contiguous     内存连续性: {A_contiguous.is_contiguous()}")
```

运行结果如下：
```
原始特征 (4维): [[1.2000000476837158, 0.5, -0.800000011920929, 2.0999999046325684]]
映射后的表征空间 (3维隐向量): [[-0.6859293580055237, 1.2268404960632324, 1.5185149908065796]]

【Prefill 模拟】大矩阵乘法 [2048x2048] * [2048x2048] 耗时: 312.11 ms
【Decode 模拟】单Token矩阵乘法 [1x2048] * [2048x2048] 耗时: 1.53 ms

A_non_contiguous 内存连续性: False
A_contiguous     内存连续性: True
```

### A.2 模块一：表征空间转换



### A.3 模块二：Prefill 与 Decode 瓶颈



### A.4 模块三：内存布局