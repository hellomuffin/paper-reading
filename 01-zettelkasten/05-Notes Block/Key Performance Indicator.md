# Key Performance Indicator

Created: 2022-05-02 12:42

## Notes

统性能指标包括CPU利用率、内存使用率，swap分区、带宽使用率；处理器性能指标包括CPI、浮点运算的峰值、向量化比例、cache miss、DTLB miss等。

- CPU利用率，比如，usr/sys/wa 等分别是多少。

- CPI（Cycles Pre Instruction），就是每条指令需要多少个时钟周期完成。现在主流处理器的理论值是0.25，因为它有4个算术逻辑运算单元。CPI偏高的可能原因：1. 大量的访存指令stall了流水线；2. 分支预测失败等；

- 浮点运算的峰值，每秒钟执行多少浮点型运算。对于科学计算相关的优化，这是判断我们距离理论预期的

- 向量化比例，现代 CPU 为了提高浮点计算的吞吐量，一般都支持向量化指令，比如，Intel 上的 SSE、AVX 指令集，ARM 包含了 NEON 指令集，可以在一个cycle中处理多个数据（参见SIMD）

- Cache miss比例，一般访问L1 cache 的时间在几个cycle，访问L2 cache的时间在几十个Cycle，访问内存的 cycle 在几百个cycle。一旦缓存miss，会带来较高的代价。一般可以通过对数据做 blocking 解决。

- DTLB miss，进程访问的地址空间是逻辑地址空间，比如64位的程序，地址范围从0到2的64次方减一，然而这么大的地址空间，实际上不可能配置这么多物理内存的，需要建立一个从逻辑内存到物理内存的映射表，这个映射表的管理是按照页为最小单位的，缺省的页大小是4K。但是页大小可调，越大的页，页表条目越少，页大小为4M的跟4k的页表条目就差了1024倍。这些页表不可能完全放在CPU里面，CPU只能放在一个称为DTLB的缓冲区中，更多的部分放在内存里面，当映射相应条目的时候，如果在缓冲区找不到就会产生DTLB miss的异常，这个异常处理会从内存中找到相关条目，然后放入DTLB缓冲区。这个操作极其耗时，经常发生时对性能影响极大，通常可以通过调整页大小，或者在程序中使用内存池等内存管理技术减少耗时。

- 内存带宽，内存带宽体现了内存访问的忙闲程度，这个值高到一定程度，会导致内存延迟迅速增加，有一些工具比如并行的Paramon和Intel的VTune可以帮助测量这个值。

- 网络，网络分为千兆网、万兆网、infiniband网络，这个千兆网就是1000比特每秒，万兆就是10000比特每秒。infiniband是一个非常快速的网络，它能达到40G到100G。

## References

1.
