# 性能推演 Demo -- 矩阵乘法

Created: 2022-05-02 11:14

## Notes

以矩阵乘法为例，假设float32类型的矩阵 1000x1000 的矩阵和 1000x1000 的矩阵相乘，

### CPU 时间

理论上的float32计算次数：

- 乘法：1000 * 1000 * 1000 = 1000000000
- 加法：1000 * 1000 * 1000 = 1000000000

即一共有 10^9 次 FMA指令（乘加）

假设系统CPU有10个物理core，每个core运行在2GHz，每个core在每个cycle可以执行2条256bit FMA指令（乘加），则理论上系统每秒可以计算 FMA ：

10 × 2G * 2 *  (256/8bits/4bytes) = 320 G  (注：1个字节是8bits，1个float是4bytes，所以256bit的FMA指令实际上是8个float的FMA指令）

FMA 预期消耗时间： t_cpu = 1000000000 / 320G = 31.25ms

### 访存时间

理论上需要访问内存的次数：

- 读内存：1000*1000*2 × 4Byte =  8000000 Byte
- 写内存：1000*1000*4Byte = 4000000 Byte

假设内存读写带宽为：20GB/s

读内存预期消耗的时间为： t_mem = 8000000Byte /  20Gb/s = 4.0ms

### 最终时间

理论上最少需要时间： t = max(t_cpu, t_mem) = 31.25ms

如果你发现上述过程实际耗时 200ms，那么恭喜，找到了需要优化的点～

小问题：

1. 为什么理论上需要的时间是 t = max(t_cpu, t_mem)，而不是  t =t_cpu + t_mem？

2. 计算矩阵乘法时，矩阵的每个元素会被多次访问，为什么计算访问内存量的时候没有被计算在内？

# References

1.
