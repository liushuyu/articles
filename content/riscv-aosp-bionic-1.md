+++
title = "Fixing Floating-point handling in Bionic on RISC-V"
date = "2022-03-09"
+++

It's been a while I have been an intern at the [PLCT Lab](https://plctlab.github.io/). I have learned a lot about the RISC-V ecosystem during the past several weeks.

## Background

As I am currently with the AOSP Porting Squad, the first things to do is to get Bionic (the `libc` implementation on Android) work correctly. My mentor Mr. Wang (汪辰) has already done a stellar job on porting most of the Bionic to the RISC-V architecture. However, he needs to squeeze some time to work on the RISC-V QEMU support recently, so he tasked me with the job of fixing the [floating-point handling issues](https://github.com/aosp-riscv/working-group/issues/36).

## Preparations

Obviously, the first thing to do is to get up to the speed with my mentor's existing work and continue from there.
You can find the existing implementation here: [https://github.com/aosp-riscv/platform_bionic](https://github.com/aosp-riscv/platform_bionic).

To build Bionic, together with its dependencies and unit tests, you have to build it in an AOSP development environment. Our porting squad has already provided instructions on how to set up the environment, and you can find them below:

- [How to set up AOSP environment](https://github.com/aosp-riscv/working-group/blob/9a8b450471a72cb92dbf274c9d054568ca3682ba/docs/howto-setup-build-env.md)
- [How to set up RISC-V Bionic repository](https://github.com/aosp-riscv/test-riscv/blob/4cdc228de846220e4603e1b80dabcaa4c491d98d/docs/howto-setup-test-env.md)

Some of the instructions may appear a little bit vague. For instance, to obtain prebuilt binaries for RISC-V GCC and QEMU, you have to navigate to [this repository](https://github.com/aosp-riscv/platform-prebuilts-build-tools/tree/f0e2377d3c29d1e9942dd861a8050b65cf04032c) and [this other repository](https://github.com/aosp-riscv/test-riscv/tree/4cdc228de846220e4603e1b80dabcaa4c491d98d/bin/qemu/install).

Now source the necessary scripts and start the build process:

```bash
source ./build/envsetup.sh
export TARGET_ARCH=riscv64
export TARGET_PRODUCT=aosp_riscv64
mmm bionic external/icu
```

## Identify the Cause

After the build process finishes, we can now run the tests:

```bash
pushd test/riscv
source ./envsetup
cd bionic/host
./run.sh
```

And we can see the following errors:

```
[... snip ...]
bionic/tests/math_test.cpp:1017: Failure
Expected equality of these values:
  1235
  lrintl(1234.01L)
    Which is: 1234
[  FAILED  ] math_h.lrint (3 ms)
```

Hmm... I think the cause might be the rounding mode was not respected, especially if we examine this part of the [unit-test code](https://github.com/aosp-riscv/platform_bionic/blob/0dde4734fa01e36dee2ce6372c84b32d1523a48d/tests/math_test.cpp#L1011-L1021):

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

As you can see, the test clearly sets the rounding mode but our rounding result is not correct.
The next thing to look at is the function that sets the rounding mode, more specifically, the `fesetround` function.

We will find the implementation [like this](https://github.com/aosp-riscv/platform_bionic/blob/0dde4734fa01e36dee2ce6372c84b32d1523a48d/libm/riscv64/fenv.c#L96-L101):

```c,linenos,linenostart=96
int fesetround(int round)
{
  round &= FE_UPWARD;
  asm volatile ("fsrm %z0" : : "r" (round));
  return 0;
}
```

Huh, we can see there is a piece of assembly in here. Don't worry, let's look it up in the [RISC-V Assembly Manual](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf) and see how it works.

In Chapter 11.2: _Floating-Point Control and Status Register_, we can see `fsrm` instruction accepts a rounding mode flag:

> [...] The FRRM instruction reads the Rounding Mode field `frm` and copies it into the least-significant three bits of integer register _rd_, with zero in all other bits. FSRM swaps the value in `frm` by copying the original value into integer register _rd_, and then writing a new value obtained from the three least-significant bits of integer register _rs1_ into `frm`.

And thus, we can verify that `fesetround` was indeed implemented correctly.

So what went wrong? Hmm... Is the current implementation really using the FPU though? Maybe it's using the software float-point implementation instead. Let's look at the implementations for other architectures under the `libm` folder, [like `arm64`](https://github.com/aosp-riscv/platform_bionic/tree/0dde4734fa01e36dee2ce6372c84b32d1523a48d/libm/arm64): Yep, there are some assembly files lying around. This could be further proved by the [build files](https://github.com/aosp-riscv/platform_bionic/blob/0dde4734fa01e36dee2ce6372c84b32d1523a48d/libm/Android.bp#L320-L343).


## Down the Rabbit Hole

Now that we have identified the issue, let's come up with a solution. For `aarch64`, Bionic used compiler intrinsics to round the numbers, let's try that with a short example program:

```c
#include <stdio.h>

int main() {
  printf("%f\n", __builtin_rint(100.1));
  return 0;
}
```

Since we are not going to run it (it would need a `libc`), let's examine the disassembly instead:

```
0000000000000014 <.LBB0_1>:
  14:   00000517                auipc   a0,0x0
  18:   00050513                mv      a0,a0
  1c:   2108                    fld     fa0,0(a0)
  1e:   00000097                auipc   ra,0x0
  22:   000080e7                jalr    ra # 1e <.LBB0_1+0xa>
  26:   e20505d3                fmv.x.d a1,fa0
```

Hmm, the compiler used `fmv.x.d a1, fa0` to copy the integer value from FPU register `fa0` to the GPR `a0`.

From the assembly manual, we will learn that, `fmv` is special. In the sense that, it does not respect rounding mode. We can verify that from the encoding diagram provided by the manual:

![FMV encoding diagram](/fmv-diagram.drawio.svg)

Well, that didn't work. Let's just hand-craft some assembly code instead.
Let's see... There **has** to be a rounding instruction in RISC-V somewhere in the manual, isn't it?
It turns out, there is none. Our current best bet is to use `fcvt` to copy the float-point value to a GPR and then copy it back to the float-point register.

That... doesn't seem to be **the** solution, but nevertheless, **a** solution.

So let's implement the `rint`-family of functions like this:

```bash
rint:
  # copy from FPU register fa0 to GPR a0 with dynamic rounding mode
  FCVT.L.D a0,  fa0, dyn
  # copy from GPR a0 to FPU ft0 with dynamic rounding mode
  FCVT.D.L ft0,  a0, dyn
  # copy value from ft0 to fa0 while preserving sign bit (e.g. handles -0.0)
  FSGNJ.D  fa0, ft0, fa0
  RET

rintf:
  FCVT.W.S a0,  fa0, dyn
  FCVT.S.W ft0,  a0, dyn
  FSGNJ.S  fa0, ft0, fa0
  RET
```

## The Abyss

Okay, there is one more issue: we need to implement `rintl`, where the function signature looks like this:

```c
long double rintl(long double x);
```

Can we apply the previous knowledge here? *\*sigh\** I hope we could.

### The Story of `long double`

In order to understand why there is an issue. We need to first understand the type `long double`.

On `x86` machines, `long double` is defined as an extended precision floating point type, which is 80-bit long (for alignment reasons, it would be padded to 128-bit). On `arm64`, `long double` is the same as `double`, which is 64-bit long.

For RISC-V though, according to [RISC-V C ABI Specification](https://riscv.org/wp-content/uploads/2015/01/riscv-calling.pdf), `long double` on RISC-V is... **128-bit**, which is a quadruple-precision floating point type!

This pose a huge issue, where hardware `long double` support is only possible with RISC-V `Q` extension, which is outside of the RISC-V `riscv64gc` specification.

I have looked at GNU LibC (GLibc) implementation of this, and it seems like they are using the software only implementation (also does not respect rounding mode).

### To be continued...?

So we are facing an impasse here. We are still thinking of a way to resolve this. I guess this is the end of this "adventure"... for now.
