
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

| 代码                                          | 在做什么             | 意义                                      |
| ------------------------------------------- | ---------------- | --------------------------------------- |
| `x = torch.tensor([[1.2, 0.5, -0.8, 2.1]])` | 创建 `[1, 4]` 张量   | 模拟一个样本的 4 维原始特征。`1` 是 batch 维，`4` 是特征维。 |
| `W = torch.randn(4, 3)`                     | 创建 `[4, 3]` 权重矩阵 | 表示从 4 维输入空间到 3 维隐空间的线性映射规则。             |
| `output = torch.matmul(x, W)`               | 矩阵乘法             | 计算 y=xW，输出形状 `[1, 3]`。                  |
#### 实验意义

- 这段验证了正文第 4 节“维度跨越的本质：表征转换”。
- 在真实神经网络中，`W` 不是随机初始化的，而是通过反向传播训练出来的。
- 这里用随机 `W` 只是为了演示形状变化和矩阵乘法过程，不代表真实语义。
- 输入是 `[1, 4]`，权重是 `[4, 3]`，输出一定是 `[1, 3]`，这是矩阵乘法维度规则的自然结果。

### A.3 模块二：Prefill 与 Decode 瓶颈

按照原来的代码，结果不够明显，数据没有足够的显著性说明Chapter 02中提到的性能差异和所预想的时间、带宽差别，所以在这里我又单独添加了一个模块的代码来更加显性的说明上述问题，运行在GPU上。但是因为硬件设备有限，所究原理就停到这里。

```import time
import torch

# ============================================================
# 0. 环境检查
# ============================================================
assert torch.cuda.is_available(), "需要 GPU"
device = "cuda"
props = torch.cuda.get_device_properties(0)
print(f"GPU: {props.name}")
print(f"显存: {props.total_memory / 1e9:.2f} GB")
print(f"SM 数量: {props.multi_processor_count}")
print()

# 常见 GPU 的峰值带宽（GB/s），按需修改或查你的显卡规格
PEAK_BW_GBPS = {
    "T4": 320,
    "V100": 900,
    "A100": 1555,
    "H100": 3350,
    "RTX 3090": 936,
    "RTX 4090": 1008,
}
gpu_peak = None
for name, bw in PEAK_BW_GBPS.items():
    if name in props.name:
        gpu_peak = bw
        break
if gpu_peak is None:
    print("⚠️ 未识别 GPU 型号，请手动填入峰值带宽")
    gpu_peak = float(input("请输入峰值带宽 (GB/s): "))
print(f"使用峰值带宽参考值: {gpu_peak} GB/s\n")

# ============================================================
# 1. 通用 benchmark 函数
# ============================================================
def bench_matmul(M, K, N, n_warmup=5, n_iter=30):
    A = torch.randn(M, K, device=device)
    B = torch.randn(K, N, device=device)

    # warmup：让 cuBLAS 选 kernel、分配 workspace、填满 cache
    for _ in range(n_warmup):
        C = torch.matmul(A, B)
    torch.cuda.synchronize()

    # 正式计时：所有 kernel 下发完后同步，取平均
    start = time.perf_counter()
    for _ in range(n_iter):
        C = torch.matmul(A, B)
    torch.cuda.synchronize()
    end = time.perf_counter()

    t_ms = (end - start) / n_iter * 1000

    bytes_moved = 4 * (M * K + K * N + M * N)
    flops = 2 * M * K * N
    bw_eff = bytes_moved / (t_ms / 1000) / 1e9  # GB/s
    ai = flops / bytes_moved                     # FLOPs/Byte

    return {
        "M": M, "K": K, "N": N,
        "t_ms": t_ms,
        "GFLOPs": flops / (t_ms / 1000) / 1e9,
        "bytes_MB": bytes_moved / 1e6,
        "bw_eff": bw_eff,
        "ai": ai,
        "bw_util": bw_eff / gpu_peak * 100,
    }

# ============================================================
# 2. 三种场景对比
# ============================================================
# 为了不让权重被 L2 缓存完全吸收，K、N 取大一点
# 2048x2048 float32 权重 ≈ 16 MB，可能被 L2 缓存吃下
# 4096x4096 float32 权重 ≈ 64 MB，稳超 L2
configs = [
    ("Prefill (M=2048)", 2048, 4096, 4096),
    ("Decode  (M=1)   ", 1,    4096, 4096),
]

print(f"{'场景':<20} {'耗时(ms)':>10} {'GFLOPs':>10} "
      f"{'字节(MB)':>10} {'有效带宽(GB/s)':>15} {'带宽利用率':>10} {'AI':>8}")
print("-" * 95)

for name, M, K, N in configs:
    r = bench_matmul(M, K, N)
    print(f"{name:<20} {r['t_ms']:>10.3f} {r['GFLOPs']:>10.2f} "
          f"{r['bytes_MB']:>10.2f} {r['bw_eff']:>15.2f} "
          f"{r['bw_util']:>9.1f}% {r['ai']:>8.2f}")

```
运行结果如下：
```
GPU: Tesla T4
显存: 15.64 GB
SM 数量: 40

使用峰值带宽参考值: 320 GB/s

场景                       耗时(ms)     GFLOPs     字节(MB)      有效带宽(GB/s)      带宽利用率       
-----------------------------------------------------------------------------------------------
Prefill (M=2048)         20.092    3420.27     134.22            6.68       2.1%   512.00
Decode  (M=1)             0.287     116.81      67.14          233.74      73.0%     0.50
```

大矩阵乘法模拟 prefill，大量token并行计算。M=1的情况模拟Decode过程，单token矩阵乘法。Prefill的计算量上：

$$
2MKN = 2 \times 2048^3 = 2 \times 8.59 \times 10^9 \approx 17.18 \text{ GFLOPs}
$$

单 token 的 Decode：

$$
2 \times 1 \times 2048^2 = 2 \times 4.19 \times 10^6 \approx 8.39 \text{ MFLOPs}
$$

两者正好差 $M = 2048$ 倍，这就是为什么 Prefill 的计算量远大于 Decode。Decode 每次都要读取 `B_large` 的整个权重矩阵，它的算术强度是远低于现代GPU的算力/带宽比的。在数据里可以看到，在控制了相对严格的环境变量之后，Prefill由于计算量较大所以耗时明显高于Decode，而在Decode过程中因为要反复搬运权重所以对于带宽的要求较高。
### A.4 模块三：内存布局

	PyTorch 的 Tensor 底层是一块连续内存 + stride（步长）来描述逻辑形状的。
	连续情况下，CPU/GPU 一次加载一个 cache line,一次内存访问，拿到一整行数据。
	不连续情况，四次内存访问，只用到 4 个 float，有效利用率很低。

内存不连续影响的是读取 A 的访存效率，进而拖慢整个矩阵乘法。因为跳着读会让 cache line 利用率暴跌、SIMD 失效、GPU 合并访存被破坏。**根据这个项目代码的结果和所说明问题，更加说明了Vllm在PagedAttention 设计分页 KV Cache的原因，和在显存里尽量保持访问连续性的必要性。
