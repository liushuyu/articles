+++
title = "修复 RISC-V 上 Bionic 的浮点数行为"
date = "2022-03-09"
+++

我在 [PLCT Lab](https://plctlab.github.io/) 当实习生有段时间了。其间我学到了不少 RISC-V 有关的东西。

## 背景

近段时间，我们 AOSP 移植小分队在尝试移植 Bionic（安卓的 `libc` 实现）到 RISC-V。我的导师汪辰老师已经基本上把 Bionic 移植到了 RISC-V 上。不过最近他要匀出时间搞 QEMU 的事情，所以他让我来处理一点[浮点有关的问题](https://github.com/aosp-riscv/working-group/issues/36)。

## 准备工作

首先，我们需要把开发环境搞起来。我们目前的 Bionic 实现在这里：[https://github.com/aosp-riscv/platform_bionic](https://github.com/aosp-riscv/platform_bionic).

由于 Bionic 的单元测试和各种依赖需要使用 AOSP 的构建工具，我们需要在 AOSP 的构建环境里面构建它们。
我们小队有一些备忘录性质的构建教程，不妨一试：

- [如何设置 AOSP 开发环境](https://github.com/aosp-riscv/working-group/blob/9a8b450471a72cb92dbf274c9d054568ca3682ba/docs/howto-setup-build-env.md)
- [如何设置 RISC-V Bionic 仓库](https://github.com/aosp-riscv/test-riscv/blob/4cdc228de846220e4603e1b80dabcaa4c491d98d/docs/howto-setup-test-env.md)

有些部分写得可能不太清楚，如有不懂的地方，你可以在 GitHub 上联系我们。不过你可以在[这个仓库](https://github.com/aosp-riscv/platform-prebuilts-build-tools/tree/f0e2377d3c29d1e9942dd861a8050b65cf04032c)还有[那个仓库](https://github.com/aosp-riscv/test-riscv/tree/4cdc228de846220e4603e1b80dabcaa4c491d98d/bin/qemu/install)里面找到教程里面提到的二进制文件。

然后我们准备一下环境变量并且开始构建：

```bash
source ./build/envsetup.sh
export TARGET_ARCH=riscv64
export TARGET_PRODUCT=aosp_riscv64
mmm bionic external/icu
```

## 调查

构建完成后，我们可以跑一下测试：

```bash
pushd test/riscv
source ./envsetup
cd bionic/host
./run.sh
```

……然后等它爆炸：

```
[... 略 ...]
bionic/tests/math_test.cpp:1017: Failure
Expected equality of these values:
  1235
  lrintl(1234.01L)
    Which is: 1234
[  FAILED  ] math_h.lrint (3 ms)
```

哎呀，看来舍入模式（rounding mode）的处理上有点问题。我们首先了解一下[单元测试代码](https://github.com/aosp-riscv/platform_bionic/blob/0dde4734fa01e36dee2ce6372c84b32d1523a48d/tests/math_test.cpp#L1011-L1021)是怎么写的吧：

```c++,linenos,linenostart=1011
TEST(MATH_TEST, lrint) {
  auto guard = android::base::make_scope_guard([]() { fesetenv(FE_DFL_ENV); });

  fesetround(FE_UPWARD); // lrint/lrintf/lrintl obey the rounding mode.
  ASSERT_EQ(1235, lrint(1234.01));
  ASSERT_EQ(1235, lrintf(1234.01f));
  ASSERT_EQ(1235, lrintl(1234.01L));
  fesetround(FE_TOWARDZERO); // lrint/lrintf/lrintl obey the rounding mode.
  ASSERT_EQ(1234, lrint(1234.01));
  ASSERT_EQ(1234, lrintf(1234.01f));
  ASSERT_EQ(1234, lrintl(1234.01L));
```

我们可以看出，单元测试里面倒是正确地设置了舍入模式，只是我们的舍入结果不太对。
这样的话，我们需要瞅瞅 `fesetround` 这个函数在搞什么鬼。

[这个函数](https://github.com/aosp-riscv/platform_bionic/blob/0dde4734fa01e36dee2ce6372c84b32d1523a48d/libm/riscv64/fenv.c#L96-L101)的实现如下：

```c,linenos,linenostart=96
int fesetround(int round)
{
  round &= FE_UPWARD;
  asm volatile ("fsrm %z0" : : "r" (round));
  return 0;
}
```

这段代码里面有一小坨汇编。不过没关系，我们有 [RISC-V 汇编手册](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf)的帮助！

让我们把书翻到第 11.2 节：_浮点控制和状态寄存器_（Floating-Point Control and Status Register）。我们会发现针对 `fsrm` 这条指令，这个小册子是这么讲的（译文）：

> [...] FRRM 指令会读出舍入模式 `frm` 字段的值并复制到整数寄存器 _rd_ 的最低 3 位，其他位填零。FSRM 指令会读出 `frm` 到整数寄存器 _rd_ 中，并将 _rs1_ 寄存器中最低 3 位作为新值写入到 `frm` 中。

那这其实就证明了 `fesetround` 实现得没有问题。

那究竟是什么问题呢？呃…… 现在这个实现在用浮点寄存器么？说不定这玩意在用软浮点。让我们看看其他架构是咋实现这堆东西的吧。`arm64` 跟 RISC-V 都是 RISC 架构，我们看看 [`arm64` 是咋做的](https://github.com/aosp-riscv/platform_bionic/tree/0dde4734fa01e36dee2ce6372c84b32d1523a48d/libm/arm64)：哎，这里面有一堆汇编文件来着。[构建文件](https://github.com/aosp-riscv/platform_bionic/blob/0dde4734fa01e36dee2ce6372c84b32d1523a48d/libm/Android.bp#L320-L343)里面也差不多能证明有手写的浮点实现。


## 初入兔子洞

我们总算是知道问题的所在了，那现在来解决它吧。对于 `aarch64` 架构，Bionic 直接用的编译器内建函数（compiler intrinsics）做的舍入实现。
让我们写个简单小程序试试：

```c
#include <stdio.h>

int main() {
  printf("%f\n", __builtin_rint(100.1));
  return 0;
}
```

由于没有能用的 `libc`，我们这里看一眼反汇编就好了：

```
0000000000000014 <.LBB0_1>:
  14:   00000517                auipc   a0,0x0
  18:   00050513                mv      a0,a0
  1c:   2108                    fld     fa0,0(a0)
  1e:   00000097                auipc   ra,0x0
  22:   000080e7                jalr    ra # 1e <.LBB0_1+0xa>
  26:   e20505d3                fmv.x.d a1,fa0
```

编译器在这里用了 `fmv.x.d a1, fa0` 这条指令将 FPU 寄存器 `fa0` 里面的整数值复制到通用寄存器（GPR） `a0` 里面。

我们可以从汇编手册上得知，`fmv` 这个指令比较特殊：它并不吃舍入模式的标记。汇编手册上的编码示意图也证实了这一点：

![FMV 编码示意图](/fmv-diagram.drawio.svg)

啊这，那我们直接手撸汇编吧。
手边有一本（电子版）汇编手册就是好，我们可以直接搜索一下舍入指令是哪一个。
哎？怎么会**没有**，这也太奇怪了。俗话说“兵来将挡，水来土掩”，我们想想有什么别的实现办法。
我们可以用 `fcvt` 把浮点值从浮点寄存器复制到整数寄存器，然后再从整数寄存器复制回来。

这办法听起来特别的脏，但是我们似乎也没有什么别的办法。

那么我们就这么写一下 `rint` 家族的函数吧：

```bash
rint:
  # 从 FPU 寄存器 fa0 使用动态舍入模式（dyn）复制数值到 GPR a0
  FCVT.L.D a0,  fa0, dyn
  # 从 GPR a0 使用动态舍入模式（dyn）复制数值到 FPU 寄存器 ft0
  FCVT.D.L ft0,  a0, dyn
  # 从 ft0 复制数值到 fa0，并使用来自 fa0 的正负号（矫正复制 -0.0 的行为）
  FSGNJ.D  fa0, ft0, fa0
  RET

rintf:
  FCVT.W.S a0,  fa0, dyn
  FCVT.S.W ft0,  a0, dyn
  FSGNJ.S  fa0, ft0, fa0
  RET
```

## 无尽深渊

这边还有一个问题，就是 `rintl` 这一类函数的实现。`rintl` 的函数签名是这样的：

```c
long double rintl(long double x);
```

我们能根据以前的经验照着葫芦画个瓢么？*\*叹气\** 好像不太行……

### `long double` 的那些事

在说明这个问题之前，我们需要先理解一下 `long double` 这个概念。

`x86` 设备上面 `long double` 是拓展精度浮点数，长度为 80 比特（但是由于对齐原因，会占用 128 位）。`arm64` 上面 `long double` 跟 `double` 的定义一致，都是 64 位。

不过呢，RISC-V 就不一样了，根据 [RISC-V C ABI 规范](https://riscv.org/wp-content/uploads/2015/01/riscv-calling.pdf)，`long double` 在 RISC-V 上面是…… **128 位**！好家伙，这可是四精度浮点（quadruple precision）。

这种先进的设计带来了一个大麻烦，就是硬件 `long double` 支持仅能在 RISC-V 的 `Q` 拓展下使用。但是，Q 拓展不在 RISC-V `riscv64gc` 的范畴中。

我去 GNU LibC (GLibc) 那边看了一眼，发现他/她们也在使用软件四精度浮点（同样也不遵守舍入模式）。

### 未完待续……？

这个情况下，我们是直接撞到墙上了。我们还在思考怎么*比较*优雅地解决这个问题。
且听吾等下回分解。
