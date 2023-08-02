#  State-of-the-Art

## Smart Scheduling

### AFLFast
stress low-frequency paths to explore more branches and find more bugs

在调度方面需要关心以下两个问题：

1. In which order should the fuzzer pick the seeds, in order
to stress low-frequency paths? 如何选取合适的seed

2. Can we tune the amount of generated inputs from each
seed (the energy)? 如何将好的seed fuzz更多次

## Bypassing Roadblocks

## Mutate Structured Inputs

# 3 A New Baseline for Fuzzing

## 3.1 Seed Scheduling

AFL++包含了AFLFast的fast, coe, explore, quad, lin, exploit的六种模式，它还新增了mmopt和rare模式

Mmopt模式：更关注最新的种子，它将帮助发现最新的路径

Rare模式：忽略运行时间，关注罕见边的种子

## 3.2 Mutators

### 3.2.1 Custom Mutator API

### 3.2.2 Input-To-State Mutator

+ colorization：在原有的基础上增加运行时间限制，保持在原有运行时间的两倍以内
+ bypass a comparison：If the fuzzer fails to generate an interesting input
when trying to bypass a comparison, the next time this comparison will be fuzzed with a lower probability.
+ CmpLog Instrumentation：Each comparison logs the operands of its last 256 executions in a 256 MB table shared between fuzzer and target.

### 3.2.3 MOpt Mutator

包含 MOpt 的 Core 和 Pilot 模式，并且可以结合 Input-To-State

## 3.3 Instrumentations

if an edge is hit in multiples of 256 — overflowing the corresponding bitmap entry to 0 — the fuzzer is in an inconsistent state.

提出了两种解决办法：

+ NeverZero：adding the carry flag to the bitmap entry and so, if an edge is executed at least one time, the entry is never 0
+ Saturated Counters：freezes the counter when it reaches the value of 255

默认使用 NeverZero

### 3.3.1 LLVM

两种统计边覆盖率的方法：
+ Context-sensitive Edge Coverage： XORing the assigned ID of each block with the unique ID of the callee.
+ Ngram： the fuzzer considers the destination block and the N-1 previous blocks where N is a number between 2 and 16.

除了 stdin 和 file 的方式传递测试用例，还提供了一种共享内存的方式，提高运行速率

实现了 INSTRIM 避免无用的插桩

### 3.3.2 GCC

### 3.3.3 QEMU

为了减少源码级别和二进制级别fuzz的差距提出：
+ CompareCoverage: the code is not modified, but all comparisons are hooked and each byte of each operand is compared, increasing a different bitmap entry if equal.

AFL++ 支持 Persistent Mode
+ Looping around a functionLooping：the user can specify the address of a function, and automatically the fuzzer will use it in a persistent loop patching the return address.
+ Specify entry and exit pointspoints：a user can specify the address of the first and the last instruction of the loop and QEMU will emit code a runtime to generate a loop between those addresses

### 3.3.4 Unicornafl

adds a low-level C API, Rust and Python bindings

### 3.3.5 QBDI

AFL++ can fuzz Android libraries with compiler instrumentation using LLVMLLVM

## 3.4 Platform Support

GNU/Linux, the fuzzer runs on Android, iOS, macOS, FreeBSD, OpenBSD, NetBSD and is packaged in several popular distributions like Debian, Ubuntu, NixOS, Arch Linux, FreeBSD, Kali Linux and more.

## 3.5 Snapshot LKM

fork() is wellknown to be a performance bottleneck for a large number of targets

为了避免 fork 的瓶颈，利用 Snapshot LKM 技术