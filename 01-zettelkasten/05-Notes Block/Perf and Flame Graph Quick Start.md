# Perf & 火焰图 Quick Start

Created: 2022-05-01 00:52

## Perf 简介

Linux 系统原生提供的性能分析工具，会返回 CPU 正在执行的函数名以及调用栈（stack）。

## 火焰图简介

[Flame Graphs](Flame%20Graphs.md)

火焰图是基于 perf 结果产生的 SVG 图片，用来展示 CPU 的调用栈。可读性好于 perf report。

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h1t4tzsiyjj20rs0hf43d.jpg)

y 轴表示调用栈，每一层都是一个函数。调用栈越深，火焰就越高，顶部就是正在执行的函数，下方都是它的父函数。

x 轴表示抽样数，如果一个函数在 x 轴占据的宽度越宽，就表示它被抽到的次数多，即执行的时间长。注意，x 轴不代表时间，而是所有的调用栈合并后，按字母顺序排列的。

火焰图就是看顶层的哪个函数占据的宽度最大。只要有"平顶"（plateaus），就表示该函数可能存在性能问题。

颜色没有特殊含义，因为火焰图表示的是 CPU 的繁忙程度，所以一般选择暖色调。

## Perf 安装

perf 的版本应该与 linux kernel 一致

```bash
perf  --version
# perf version 5.11.22
uname -r
# 5.11.0-38-generic
```

安装命令

```bash
sudo apt update
sudo apt install linux-tools-common linux-tools-generic linux-tools-`uname -r`
perf list # 确认 perf 安装成功，并查看 perf 所支持的事件和 trace
```

如果看不到符号表，比如 arm 上可能默认不带，需要安装 libc6-dbg，详见 [[trace glibc using gdb]]

## Perf 用法

### Perf Record

假设，被 profile 的程序是 ./test，3 种典型的用法如下
```bash
# simple
sudo perf record -g --call-graph dwarf ./test
# watch processes
sudo perf record -F 99 -p ${pid} -g -- sleep 30
sudo perf record -g --call-graph dwarf -p ${pid} [--cpu ${cpuid}]
```

说明：
1. -F 99：执行频率是 99Hz（每秒99次）
3. -p ${pid}：profile 的进程号，
4. -g：记录调用栈
5. --call-graph dwarf: Allows specifying "fp" (frame pointer) or "dwarf" (DWARF's CFI - Call Frame Information) or "lbr" (Hardware Last Branch Record facility) as the method to collect the information used to show the call graphs.
6. sleep 30：持续 30s。也可以运行一段时间后，手动掐掉。
7. --cpu: Collect samples only on the list of CPUs provided. a comma-separated list with no space: 0,1. Ranges of CPUs are specified with -: 0-2

运行后会产生一个庞大的文本文件。如果一台服务器有 16 个 CPU，每秒抽样 99 次，持续 30 秒，就得到 47,520 个调用栈，长达几十万甚至上百万行。

### Perf Report

为了便于阅读，perf report 命令可以统计每个调用栈出现的百分比，然后从高到低排列。

进入report界面:

```bash
sudo perf report -i perf.data
```

## 火焰图用法

```bash
# install (copy the code)
git clone https://github.com/brendangregg/FlameGraph
# gen graph file
cd FlameGraph
sudo perf script -i ${perf_data_filename} | ./stackcollapse-perf.pl | ./flamegraph.pl --title="CPU-clock Flame Graph" > cpu-clock.svg
```

我封装后的脚本 [flame_graph.sh](https://github.com/JackonYang/projects-in-one/blob/b6dadf5ec93d5d2297976db99fb1547cd24b3ec0/projects/j-commands/plugins/flame_graph.sh)

用如下的命令加入  `.bashrc` (`.zshrc`)

```bash
alias flame_graph="bash path/to/flame_graph.sh"
```

用法：

```bash
flame_graph <perf_data_file>
```

火焰图是SVG 图片，可以与用户互动。

（1）鼠标悬浮

火焰的每一层都会标注函数名，鼠标悬浮时会显示完整的函数名、抽样抽中的次数、占据总抽样次数的百分比。下面是一个例子。

> mysqld'JOIN::exec (272,959 samples, 78.34 percent)

（2）点击放大

在某一层点击，火焰图会水平放大，该层会占据所有宽度，显示详细信息。

## 练手的例子

代码：[cpu-and-memory-bound-cpp](../../02-tutorial-code/02-profile-examples/02-cpu-and-memory-bound-cpp/README.md)

1. 提供了 1 个 c++ 文件，内含 1 个 cpu bound 的代码，一个 memory bound 的代码，依次运行。
2. perf 命令的例子

我在 Mac M1 上跑出的火焰图如下

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h1t5c99cghj20x90bl3zb.jpg)

## 浏览器的火焰图

Chrome 浏览器可以生成页面脚本的火焰图，用来进行 CPU 分析。

打开开发者工具，切换到 Performance 面板。然后，点击"录制"按钮，开始记录数据。这时，可以在页面进行各种操作，然后停止"录制"。

这时，开发者工具会显示一个时间轴。它的下方就是火焰图。

浏览器的火焰图与标准火焰图有两点差异：它是倒置的（即调用栈最顶端的函数在最下方）；x 轴是时间轴，而不是抽样次数。

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h1sazacb0ej20ku0h074z.jpg)

## Perf 的缺陷

1. 运行时间越多的函数，被中断击中的机会越大，从而推测出这个函数 (或者 pid 等) 的 CPU 占用率就越高
2. 如果某个函数 (或者进程) 运气特别好，每次都刚好能够躲过发起探测的位置，那么统计结果可能是完全错误的。这是所有采样统计都有可能遇到的问题。
3. 调用栈不完整. 当调用栈过深时，某些系统只返回前面的一部分（比如前10层）。
4. 函数名缺失. 有些函数没有名字，编译器只用内存地址来表示（比如匿名函数）。

## References

1. http://oliveryang.net/2018/03/linux-CPI-flamegraph/
2. [http://www.brendangregg.com/perf.html](http://www.brendangregg.com/perf.html)
3. [https://yq.aliyun.com/articles/65255](https://yq.aliyun.com/articles/65255)
4. [https://www.ibm.com/developerworks/cn/linux/l-cn-perf1/index.html](https://www.ibm.com/developerworks/cn/linux/l-cn-perf1/index.html)
5. [https://www.ibm.com/developerworks/cn/linux/l-cn-perf2/index.html](https://www.ibm.com/developerworks/cn/linux/l-cn-perf2/index.html)
6. [https://github.com/freelancer-leon/notes/blob/master/kernel/profiling/perf.md](https://github.com/freelancer-leon/notes/blob/master/kernel/profiling/perf.md)
7. [https://stackoverflow.com/questions/10933408/how-can-i-get-perf-to-find-symbols-in-my-program](https://stackoverflow.com/questions/10933408/how-can-i-get-perf-to-find-symbols-in-my-program)
8. [https://stackoverflow.com/questions/19719911/getting-user-space-stack-information-from-perf](https://stackoverflow.com/questions/19719911/getting-user-space-stack-information-from-perf)
