+++
title = "Developing a Clang-Based Safe Compiler"
date = "2024-12-22"
description = "Due to the use of aggressive optimizations by modern C/C++ compilers that exploit undefined behavior, there is a need for a safe compiler that does not perform such optimizations and prevents developers from using unsafe statements and expressions. Such a safe compiler based on GCC has been developed in ISP RAS, but some developers prefer Clang instead of GCC, which has mainly the same problems of exploiting undefined behavior. This paper examines the capabilities of Clang to perform safe compilation and describes the implementation of a safe compiler based on it. For the created safe compiler, the applicability in practice is shown and the impact on program performance is evaluated."

[taxonomies]
tags = ["compiler", "vulnerability", "undefined behavior", "clang", "llvm", "c", "c++"]
+++

P.D. Dunaev, A.A. Sinkevich, A.M. Granat, I.A. Batraeva, S.V. Mironov, N.U. Shugaley

Citation info and full text in Russian: <https://doi.org/10.15514/ISPRAS-2024-36(4)-3>

<details>
<summary>Updates from the original paper</summary>

- Added `-fkeep-div-by-zero`;
- Added `-finbounds-aliasing`;
- Updated `_FORTIFY_SOURCE` --- `-D_FORTIFY_SOURCE=3` is used for all safety classes;
- Updated Alpine Linux build results and added package bug fixes.
</details>

**Abstract**: Due to the use of aggressive optimizations by modern C/C++ compilers that exploit undefined behavior, there is a need for a safe compiler that does not perform such optimizations and prevents developers from using unsafe statements and expressions. Such a safe compiler based on GCC has been developed in ISP RAS, but some developers prefer Clang instead of GCC, which has mainly the same problems of exploiting undefined behavior. This paper examines the capabilities of Clang to perform safe compilation and describes the implementation of a safe compiler based on it. For the created safe compiler, the applicability in practice is shown and the impact on program performance is evaluated.

## 1. Introduction

Recently, Clang [^1] has gained great popularity as a C++ compiler. According to JetBrains statistics [^2], more than a third of respondents used the compiler in 2023, characterizing it as the second most popular C++ compiler. Clang has certain advantages over competing developments, including the most popular compiler --- GCC [^3]. Among these advantages is the Apache 2.0-based license [^4], which allows the compiler code to be used for projects under a larger number of licenses. A significant advantage of Clang is also that it is built on top of the LLVM compiler infrastructure [^5], the value of which lies in the fact that compiler tweaks can often focus entirely on transformations of the LLVM IR (intermediate representation), as used, for example, in [^6] [^7] [^8] [^9].

However, Clang, like GCC, has a significant drawback --- it performs optimizations that take advantage of undefined behavior. For example, the article [^10] lists a number of drawbacks, many of which apply to Clang. An example of an optimization performed by the compiler in question is the removal of checks of the form `if (1 << X == 0)`, where `X` is an integer. The result of performing this bitwise shift can be zero on some processor architectures, such as PowerPC. But the compiler considers this check redundant because it can prove that this statement is always false in the absence of undefined behavior. However, it is not possible to disable this optimization using options. So when compiling a project with Clang, you need additional protection against programmer and compiler introducing vulnerabilities into the output code.

The need to rid the code of vulnerabilities can be met by various methods, such as static and dynamic analysis, extensive testing, and the use of secure coding standards. The authors of the article [^11] show that there are situations where none of these methods is applicable to the problem of eliminating vulnerabilities created by the compiler. The authors propose an alternative solution --- a safe compiler with built-in functionality that prevents vulnerabilities introduced by aggressive optimizations and warns of vulnerabilities introduced by the programmer. The paper describes a safe compiler based on GCC [^12]. However, GCC cannot serve as a replacement for Clang because Clang is not fully compatible with GCC [^13], and due to a number of advantages described above, not all developers will be willing to abandon Clang. Thus, the problem of safe compilation via Clang becomes relevant.

## 2. The concept of a safe compiler

The paper [^11] describes the concept of a safe compiler. A safe compiler, to be considered as such, must satisfy the following requirements:

- The compiler cannot introduce vulnerabilities into the generated code during the execution of optimizations;
- The compiler must not remove code based on the assumption that there is no undefined behavior while not slowing down the output program significantly;
- The changes required to successfully compile the source code are minimal;
- The compiler does not provide the ability to partially disable options that control the fulfillment of the first two requirements.

Such a compiler can work as a replacement for the previously used one, but besides its main advantage --- vulnerability prevention --- the compiler will also have disadvantages in the form of slowing down the compiled application and the need to modify the source code, even if minimally, which in some cases may be difficult or impossible.

Detailed requirements for a safe compiler are given in the standard [^14]. There are three classes of requirements, each characterized by the strictness of the security mechanisms and the speed of the generated programs. The requirements of the standard can be met either by a compiler written from scratch, or by an existing compiler that has been modified or properly configured. Since the target audience of this paper is Clang users, the option of writing a new compiler is not considered; the possibilities of configuring Clang to meet the requirements of a safe compiler are discussed in Section 3, the work done to make Clang a safe compiler is described in Section 4, and the results of testing safe Clang on real applications are presented in Section 5.

## 3. Matching Clang's features to the requirements of a safe compiler

A class 3 safe compiler should only perform safe transformations of source and machine code. For example, the compiler may convert the condition `N + 1 > N` to `true` because it assumes that undefined behavior cannot occur (in this case --- integer overflow), and the third class safe compiler must guarantee protection against such transformations. The transformation restriction and the corresponding Clang options are summarized in Table 1.

In addition, a compiler of the third security class must include enhanced security mechanisms. The required mechanisms and their corresponding Clang options are listed in Table 2.

Also, one of the tasks of the third security class is to issue warnings during the build process in some cases of undefined behavior. The Clang options that include issuing such warnings are shown in Table 3.

<div class="table-caption">Table 1. Implementation of class 3 requirements for disabling unsafe transformations in Clang</div>

| **Paragraph of the standard** | **Requirement**                                                                                     | **Option**                        |
| ----------------------------- | --------------------------------------------------------------------------------------------------- | --------------------------------- |
| 5.2.1 а                       | Disable transformations related to integer overflow                                                 | `-fwrapv`                         |
| 5.2.1 б                       | Disable transformations related to the fact that values of pointers of different types may coincide | `-fno-strict-aliasing`            |
| 5.2.1 в                       | Disable transformations related to dereferencing of null pointers                                   | `-fno-delete-null-pointer-checks` |
| 5.2.1 г                       | Disable transformations related to division by zero and taking the remainder from division by 0     | ---                               |
| 5.2.1 д                       | Disable transformations related to bitwise shift argument values                                    | ---                               |

<div class="table-caption">Table 2. Implementation of class 3 requirements for enabling increased safety mechanisms in Clang</div>

| **Paragraph of the standard** | **Requirement**                                                                                                                                | **Option**                                                                                                                                                                                 |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 5.2.2 а                       | Protection against constant size buffer overflow when calling standard library functions                                                       | `-D_FORTIFY_SOURCE=2` with a caveat: if an error occurs, these functions may output redundant information, although the program should terminate immediately according to the requirements |
| 5.2.2 б                       | Stack integrity control mechanism                                                                                                              | `-fstack-protector-strong`                                                                                                                                                                 |
| 5.2.2 в                       | Mechanism of randomization of code placement in the address space                                                                              | `-fpic/-fPIC/-fPIE`                                                                                                                                                                        |
| 5.2.2 г,д                     | Prohibition of replacing calls to certain formatted output and memory manipulation functions with equivalent sequences of machine instructions | `-fno-builtin-*`, with some functions working with widechar strings having no built-in counterparts in Clang, so this requirement is automatically met for them                            |
| 5.2.6                         | Optional mechanisms (e.g. control flow integrity)                                                                                              | There is a solution in the form of `-fsanitize=cfi`, but it contradicts the logic of the third class, as it can slow down the program significantly                                        |

<div class="table-caption">Table 3. Implementation of class 3 requirements for enabling warnings about unsafe statements in Clang</div>

| **Paragraph of the standard** | **Requirement**                                                                                                                         | **Option**                                               |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| 5.2.3 а                       | Clobbering of a variable in automatic memory when calling `longjmp`                                                                     | ---                                                      |
| 5.2.3 б                       | Read or write to an array at an invalid index                                                                                           | `-Warray-bounds` and `-Warray-bounds-pointer-arithmetic` |
| 5.2.3 в                       | Division by zero or taking the remainder from division by 0                                                                             | `-Wdivision-by-zero`                                     |
| 5.2.3 г                       | Bitwise shift operation with the second argument less than zero or greater than or equal to the width of the type of the first argument | `-Wshift-count-negative` and `-Wshift-count-overflow`    |

The main goals of a class 2 safe compiler are to prevent vulnerabilities related to incorrect memory handling, and to prevent the use of undefined constructs --- the same ones a class 3 compiler warns against --- by stopping the compilation with an error. Clang implements only a few of these. A complete list of the requirements and the options by which they are implemented is given in Table 4.

<div class="table-caption">Table 4. Implementation of class 2 requirements in Clang</div>

| **Paragraph of the standard** | **Requirement**                                                                                                                    | **Option**                                |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| 5.3.1 а                       | Preservation of the side effects of memory writes                                                                                  | ---                                       |
| 5.3.1 б                       | Automatic initialization of variables with zeros                                                                                   | `-ftrivial-auto-var-init=zero`            |
| 5.3.1 доп. а                  | Prohibition of bitwise shift optimizations if the second argument can be negative or greater than or equal to the size of the type | ---                                       |
| 5.3.1 доп. б                  | Use of vector instructions that do not require data alignment                                                                      | ---                                       |
| 5.3.1 доп. в                  | Prohibition of pointer arithmetic optimizations based on object size information                                                   | ---                                       |
| 5.3.2 а                       | Compilation failure for class 3 warnings                                                                                           | `-Werror=*` instead of `-W*`, see Table 3 |
| 5.3.2 б                       | Prohibition of the use of the `gets` function                                                                                      | `-Werror=deprecated-declarations`         |

One of the main tasks of a safe compiler of the first class is the dynamic control of undefined constructs. In this case, the compiler is required to include machine code in the output file that prevents erroneous execution of undefined constructs during program runtime by crashing the program. This feature is implemented in Clang using the built-in UndefinedBehaviorSanitizer (UBSan) tool [^15], which is used by specifying various options of the form `-fsanitize=*` in the compilation command line arguments, where `*` is the identifier of the check for a type of undefined construct. UBSan performs most of the checks that a safe compiler should perform (see Table 5).

<div class="table-caption">Table 5. Implementation of class 1 requirements in Clang</div>

| **Paragraph of the standard** | **Identifier**            | **Fulfilled requirement: verifies that...**                                                                                     |
| ----------------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 5.4.3 а                       | `bool`                    | The value of `bool` type variables is `0` or `1`.                                                                               |
| 5.4.3 б                       | `float-cast-overflow`     | No overflow occurs when converting a `float` value to `int`. However, conversions between floating point types are not checked. |
| 5.4.3 в                       | `shift`                   | The right operand of the shift is non-negative and less than the width of the type.                                             |
| 5.4.3 г                       | `signed-integer-overflow` | Signed integer overflow does not occur.                                                                                         |
| 5.4.3 д                       | `alignment`               | Reads or writes are performed only via pointers whose address is aligned with the size of the operand.                          |
| 5.4.3 е                       | `null`                    | Null pointer is not used. However, indirect function call by a null pointer is not checked.                                     |
| 5.4.3 ж                       | `bounds`                  | Reads or writes are performed at indices that do not extend beyond the array.                                                   |
| 5.4.3 и                       | `pointer-overflow`        | There is no overflow of the pointer type.                                                                                       |
| 5.4.3 к                       | `function`                | In the case of an indirect call, the function signature matches the type of the pointer to it. Works for C++ and x86(-64) only. |
| 5.4.3 л                       | `return`                  | The function call ends with the `return` operator. Works only for C++.                                                          |
| 5.4.3 м                       | `builtin`                 | Correct parameters are passed to the built-in functions.                                                                        |
| 5.4.3 н                       | `unreachable`             | Execution does not reach code marked as unreachable.                                                                            |
| 5.4.3 п                       | `integer-divide-by-zero`  | There is no division by zero or taking the remainder from division by zero.                                                     |
| 5.4.3 р                       | `vla-bound`               | The array allocated in automatic memory has a positive size.                                                                    |

Another task of a safe compiler is to manage the distribution of automatic and static memory. As part of this task, the first class compiler must support the possibility of dynamic program layout, where functions are arranged in memory in a random order each time the program is launched. If full implementation of this mechanism is impossible (which may be hindered by the dynamic loader) or impractical (as in the case of compiling the operating system kernel), the distribution must be static --- unique for each compilation process. Clang does not support static randomized memory distribution, but can support dynamic distribution if the linker and dynamic loader have this feature implemented.

It should be noted that a safe compiler of the second class also borrows some of the requirements of the third class, and a compiler of the first class borrows all of the requirements of the second class.

Thus, there is no configuration of Clang that satisfies at least one of the safe compilation classes, but the compiler provides a number of features to prevent unsafe optimizations and the execution of unsafe constructs, which can be used in particular when developing a safe compiler based on it.

## 4. Safe compiler implementation

The implemented safe compiler was developed on the basis of Clang 16.0.6. The `-Safe3`, `-Safe2`, and `-Safe1` options were added to include the options of the corresponding safety classes available in Clang and developed in this work. To conveniently manage the functionality of the options, a domain-specific language based on TableGen [^16] was developed that describes the inclusion of the options in the following format:

```
def fno_strict_aliasing : Force<"-fno-strict-aliasing", 1, 3>;
def fassume_unaligned : Force<"-fassume-unaligned", 1, 2>;
```

The following options have been implemented for 3rd class:

1. `-fkeep-oversized-shifts`: prevents bitwise shift optimization in cases where the second argument of the shift operator is less than zero or greater than or equal to the type width. This option is implemented by replacing the generation of LLVM IR shift instructions with calls to new intrinsic functions. The `InstCombine` and `SCCP` passes are supplemented by optimizations of these calls, which are only performed if the compiler can prove that the second argument is non-negative and strictly less than type width. Otherwise, intrinsic function calls are expanded (replaced by the corresponding instructions) after all optimization passes, thus avoiding optimizations.
2. `-fkeep-div-by-zero`: prevents optimization of division and remainder operations in cases where the divisor can be zero. This option is implemented in the same way as `-fkeep-oversized-shifts`.
3. The headers containing fortified versions of the standard library functions, which are used in Alpine Linux and the SAFEC safe compiler, have been modified to support Clang, and more functions have been added. By using the header files included in the compiler, both the glibc and musl standard libraries are supported. Instead of `-D_FORTIFY_SOURCE=2`, support for `-D_FORTIFY_SOURCE=3` has been added, checking not only calls with constant-sized objects, but also with those for which a size expression can be constructed at compile time. Also, these headers have been used to implement immediate program termination on error in fortified functions, to prevent functions from being replaced by Clang built-ins, and to add a warning (an error in `-Safe2`) when the `gets` function is used.
4. Instead of warning about clobbering of a variable in automatic memory when `longjmp` is called, the `-fforce-volatile-before-setjmp` option was implemented to mark all local variables available at the time `setjmp` is called as `volatile`, preventing the compiler from placing these variables in registers and preventing them from being clobbered after `longjmp` is called. This option is implemented as a pass at the beginning of the optimization pipeline that handles LLVM IR. This pass detects vulnerable variables (allocations on the stack) and marks all instructions that use them as `volatile`.

The following options have been implemented for 2nd class:

1. To preserve the side effects of memory writes, the `-fpreserve-memory-writes` option has been created which prevents DSE (Dead Store Elimination) in multiple passes of the optimization pipeline. For example, this option preserves the clearing of memory containing sensitive data. In the `EarlyCSE` pass, which eliminates trivially redundant instructions, the option disables the deletion of consecutive writes to the same memory location without a read in between. In `InstCombine`, which performs merging and deleting of instructions, a similar optimization is disabled. In the `MemCpyOptimizer` pass, which optimizes memory manipulation instructions such as `memset` and `memcpy`, the merging of overlapping memory writes into a single `memset` is disabled. Finally, in the `DeadStoreElimination` pass, which does the main work of optimizing redundant writes, instead of deleting them, the `volatile` flag is set so that these instructions cannot be optimized by the next passes.
2. The `-fassume-unaligned` option adds a pass to the beginning of the optimization pipeline that removes alignment from `load` and `store` instructions, as well as from pointer arguments in function calls. This results in the generation of unaligned memory instructions instead of vector instructions expecting aligned memory, and prevents program crashes in cases where the used memory section was misaligned.
3. The `-finbounds-aliasing` option prevents optimizations where the compiler assumes that a pointer that goes beyond the boundaries of the object it points to cannot point to other objects. To accomplish this, `BasicAliasAnalysis` has been modified so that if there is a `getelementptr` instruction for which it fails to prove that the result does not extend beyond the boundaries of the object, `MayAlias` is returned as the result of the aliasing analysis of two objects. Also, in `InstCombineLoadStoreAlloca`, replacing of the GEP index with 0 in case any other value would be out of bounds is prevented. In `SelectionDAGAddressAnalysis`, the option prevents pointers from being assumed to be independent of each other if they are derived from different variables. In addition, in `ScheduleDAGInstrs`, the underlying object identification for a pointer returned by an instruction is disabled (except in the trivial case) to prevent optimization.

The following options have been implemented for 1st class:

1. Since the `float-cast-overflow` check in UndefinedBehaviorSanitizer checks for overflow only when converting from floating-point real types to integer types, the `-fsanitize=float-to-float-cast-overflow` option has been created to check conversions between real types. The implementation is based on an older version of the `float-cast-overflow` check, whose behavior was changed in Clang 9 because this type of overflow is defined by IEEE 754 [^17].
2. The `-fsanitize=null-call` option checks that a null pointer is not used in an indirect function call. This check is absent in `-fsanitize=null` in UBSan.
3. In Clang 16, the `-fsanitize=function` option, which checks the correspondence between the formal function type and the actual pointer type, only supports C++ and x86(-64) architecture, so improvements have been ported from Clang 17 to allow its use for C and other architectures. The main change is the use of type hashes instead of RTTI (Run-Time Type Information), which is only available for C++.
4. The built-in `-fsanitize=return` option checks for the presence of a return operation when exiting a function that has a return value, but only for C++, since in C, using the value returned by such a function is undefined behavior, not the absence of a return. To make the checking behavior uniform, the `-fsanitize=return-c` option has been implemented, which works the same way for C as for C++.
5. To support the unique distribution of a program's static memory at compile time, the `-frandom-func-reorder` and `-frandom-func-and-globals-reorder` options have been added to shuffle only functions or functions and global variables, respectively, in each compilation unit. These options include a pass at the end of the LLVM IR optimization pipeline that randomizes the order of functions and global variables based on the contents of the module and the number specified by the `-mllvm -rng-seed` option.
6. For automatic memory randomization, the `-floc-var-per` option is created, which shuffles local variables (more precisely, constant-sized allocations on the stack) in a random order. This functionality is implemented in the same pass as the previous two options. The `-fadd-loc-var` option is also supported, which specifies the number of local variables to add during shuffling. Without it, when using `-floc-var-per`, a small random number of variables will be added to increase the randomness of automatic memory.

## 5. Results

### 5.1. Correctness

The developed Clang-based safe compiler successfully passes all tests of a test suite created to verify the correctness of the SAFEC safe compiler [^11] and its conformance to the standard. This suite includes tests that check whether the required diagnostics are issued, tests that check whether dynamic checks are triggered at runtime, and tests that check the result of code generation.

### 5.2. Performance study

The performance of programs compiled with the safe compiler was evaluated using the same 5 tests as for SAFEC:

- playing a game of Go using GNU Go 3.8;
- transcoding files from WAV to MP3 using LAME 3.100;
- performing the fannkuch test from The Computer Language Benchmarks Game;
- transcoding files from YUV to MKV using x264 (x264-snapshot-20190407-2245-stable);
- text file compression using zlib 1.2.11.

<div class="table-caption">Table 6. Performance evaluation results</div>

| **Test** | **Baseline** | **`-⁠Safe3`** | **`-⁠Safe3` slowdown** | **`-⁠Safe2`** | **`-⁠Safe2` slowdown** | **`-⁠Safe1`** | **`-⁠Safe1` slowdown** |
| -------- | ------------ | ------------- | ---------------------- | ------------- | ---------------------- | ------------- | ---------------------- |
| GNU Go   | 3.93 s       | 4.03 s        | 2.54%                  | 4.26 s        | 8.12%                  | 6.66 s        | 69.04%                 |
| LAME     | 5.21 s       | 5.27 s        | 1.15%                  | 4.90 s        | -5.82%                 | 13.19 s       | 153.40%                |
| fannkuch | 2.24 s       | 2.10 s        | -6.25%                 | 2.08 s        | -7.14%                 | 2.72 s        | 21.43%                 |
| x264     | 1.69 s       | 1.81 s        | 7.42%                  | 1.79 s        | 5.20%                  | 6.32 s        | 274.74%                |
| zlib     | 1.54 s       | 1.63 s        | 5.88%                  | 1.62 s        | 5.87%                  | 2.40 s        | 55.60%                 |

Table 6 shows the results of measuring the test execution time on a computer with AMD Ryzen™ 5 4600H processor (x86-64 architecture) running Manjaro Linux 23.1.4. The Baseline column corresponds to the compiler run with the `-O2` option, and the `-Safe3`, `-Safe2`, `-Safe1` columns correspond to runs with the `-O2` optimization level and the corresponding safety class. Each value is the average of 5 runs, rounded to 0.01 s. The slowdown relative to the baseline time is also given for each safety level.

From the data presented, it can be seen that when using the 3rd or 2nd safety class, the slowdown does not exceed 10%, while programs compiled with the 2nd safety level sometimes appear to be faster. This may be due to the fact that in class 2, unlike in class 3, there is no prohibition to replace calls of memory manipulation functions from the standard library with equivalent sequences of machine instructions. When using the 1st class of protection, the slowdown ranges from 21% to 275%, which in the case of x264 exceeds the allowed 200% threshold.

The significant slowdown of x264 can be explained by the addition of a large number of pointer overflow checks by the `pointer-overflow` sanitizer --- without it, the slowdown is 165%. It was also found that the baseline version of x264 compiled with Clang was 64% faster than the one compiled with SAFEC 11.4.0, and with safety level 1 the runtimes were about the same, so when comparing safety level 1 of Clang with the base version of GCC the slowdown will be less than 200%.

In addition, the tests revealed that the `pointer-overflow` sanitizer in Clang, unlike GCC and SAFEC based on it, finds an overflow in GNU Go when adding an unsigned integer to a pointer. This is a known limitation of GCC [^18].

### 5.3. Building a Linux distribution

In addition to testing several applications individually with safe Clang, the applicability of the safe compiler was evaluated by building a Linux distribution. Alpine Linux 3.20.3 [^19], a lightweight and security-oriented distribution using musl, BusyBox, and OpenRC was chosen for this task. Since the distribution uses GCC for building from source by default, Clang was added as a dependency to the `build-base` build meta-package and `/usr/bin/gcc`, `/usr/bin/cc` and similar files were replaced with symbolic links to Clang. Safe compiler patches have also been added to the `clang16` and `llvm16` packages. As a result, every package from the `main` and `community` repositories has been built with all safety classes: from unsafe mode to class 1.

<div class="table-caption">Table 7. Alpine Linux package build results</div>

| **Class** | **Successfully built** | **Build or test errors** |
| --------- | ---------------------- | ------------------------ |
| Baseline  | 3810                   | 668                      |
| `-Safe3`  | 3792                   | Build: 13, test: 5       |
| `-Safe2`  | 3745                   | Build: 44, test: 3       |
| `-Safe1`  | 3340                   | Build/test: 403, ICE: 2  |

Table 7 summarizes for each safety class the number of packages that were successfully built and the number of packages that failed with a build or test error. The large number of packages not built by Clang in insecure mode is in many cases due to Clang's incompatibility with GCC, as some warnings are set as errors by default, and packages' use of options present only in GCC. It is also worth noting that almost half of all packages (3574 out of 8052) do not use the C/C++ compiler, or do not take into account the `CFLAGS`, `CXXFLAGS` or `CPPFLAGS` environment variables, through which the `-Safe` option is passed, so they are not considered in this paper.

At safety level 3, only 18 packages could not be built. Of these, 13 are incompatible with the `_FORTIFY_SOURCE` implementation in safe Clang, e.g. due to function declarations without specifying arguments, or due to trying to set `_FORTIFY_SOURCE` to 0 when `-Werror` is present. One package removes the `const` keyword with `#define const`, which prevents compilation of fortified function headers. The remaining packages failed the built-in tests, mostly due to triggering fortified functions. Since class 3 warnings become errors in safety class 2, 34 packages failed to compile at this level due to `-Warray-bounds-pointer-arithmetic`, 4 due to `-Warray-bounds`, and 6 due to `-Wshift-count-overflow`.

Bug fixes [^20] [^21] [^22] were proposed for 3 packages that failed the tests when built with safety class 3 or 2, of which 1 fix has already been accepted.

The 1st safety class includes sanitizers, which caused 400 packages to fail to build. Also, 2 packages failed due to `-fsanitize=function` incompatibility with WebAssembly. Another package failed due to `-Werror=shift-count-overflow`. In addition, while building the `community/cabextract` and `community/libu2f-server` packages, a bug was discovered in vanilla Clang 16.0.6 that caused the compiler to crash. This has been fixed in Clang 17.

In this work it was tested only if the packages of the distribution successfully build and pass the built-in tests, because complete testing of all programs would be too time-consuming. It was shown that 87.6% of packages built by unsafe Clang can be built by the safe version with safety class 1.

## 6. Conclusion

In this work, a safe compiler based on Clang has been implemented. The paper has considered the concept of a safe compiler in relation to the compiler under study. The components that implement the specified features have been highlighted, and the requirements that this compiler does not satisfy have been considered. The decision to develop a safe compiler based on Clang was justified. All the missing components were implemented and described. Using the created safe compiler, it was possible to build most of the packages of the Alpine Linux distribution. It was shown that the performance of the programs built with the safe compiler meets the requirements in almost all cases.

[^1]: Clang, Available at: <https://clang.llvm.org/>, accessed 22.12.2024.
[^2]: Jetbrains Developer Ecosystem: C++, Available at: <https://www.jetbrains.com/lp/devecosystem-2023/cpp/>, accessed 22.12.2024.
[^3]: GCC, Available at: <https://gcc.gnu.org/>, accessed 22.12.2024.
[^4]: LLVM Developer Policy, Available at: <https://llvm.org/docs/DeveloperPolicy.html>, accessed 22.12.2024.
[^5]: LLVM, Available at: <https://llvm.org/>, accessed 22.12.2024.
[^6]: Skvortsov L.V., Baev R.V., Dolgorukova K.Y., Sharygin E.Y. Developing an LLVM-based compiler for stack based TF16 processor architecture. Trudy ISP RAN/Proc. ISP RAS, vol. 33, issue 5, 2021, pp. 137-154 (in Russian). DOI: 10.15514/ISPRAS–2021–33(5)–8.
[^7]: Melnik D., Kurmangaleev S., Avetisyan A., Belevantsev A., Plotnikov D., Vardanyan M. Optimizing programs for given hardware architectures with static compilation: methods and tools. Trudy ISP RAN/Proc. ISP RAS, vol. 26, issue 1, 2014, pp. 343-356 (in Russian). DOI: 10.15514/ISPRAS-2014-26(1)-13.
[^8]: Ivannikov V., Kurmangaleev S., Belevantsev A., Nurmukhametov A., Savchenko V., Matevosyan H., Avetisyan A. Implementing Obfuscating Transformations in the LLVM Compiler Infrastructure. Trudy ISP RAN/Proc. ISP RAS, vol. 26, issue 1, 2014, pp. 327-342 (in Russian). DOI: 10.15514/ISPRAS-2014-26(1)-12.
[^9]: Gaissaryan S., Kurmangaleev S., Dolgorukova K., Savchenko V., Sargsyan S. Applying two-stage LLVM-based compilation approach to application deployment via cloud storage. Trudy ISP RAN/Proc. ISP RAS, vol. 26, issue 1, 2014, pp. 315-326 (in Russian). DOI: 10.15514/ISPRAS-2014-26(1)-11.
[^10]: Wang X., Chen H. et al. Undefined behavior: what happened to my code? In Proc. of the Asia-Pacific Workshop on Systems, 2012, pp. 1-7.
[^11]: Baev R.V., Skvortsov L.V., Kudryashov E.A., Buchatskiy R.A., Zhuykov R.A. Prevention of vulnerabilities arising from optimization of code with Undefined Behavior. Trudy ISP RAN/Proc. ISP RAS, vol. 33, issue 4, 2021. pp. 195-210 (in Russian). DOI: 10.15514/ISPRAS–2021–33(4)–14.
[^12]: Safe compiler SAFEC. Available at: <https://www.ispras.ru/technologies/safecomp/> (in Russian), accessed 22.12.2024.
[^13]: Clang: Language Compatibility, Available at: <https://clang.llvm.org/compatibility.html>, accessed 22.12.2024.
[^14]: GOST R 71206-2024 «Information protection. Secure software development. Safe C/C++ compiler. General requirements». Moscow, Russian Standardization Institute, 2024, 20 p. (in Russian)
[^15]: UndefinedBehaviorSanitizer, Available at: <https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html>, accessed 22.12.2024.
[^16]: TableGen Overview, Available at: <https://llvm.org/docs/TableGen/>, accessed 22.12.2024.
[^17]: IEEE 754-2019, Standard for Floating-Point Arithmetic, 2019. pp. 1-84. DOI: 10.1109/IEEESTD.2019.8766229.
[^18]: Missing pointer overflow detection with -fsanitize=pointer-overflow, Available at: <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=82079>, accessed 22.12.2024.
[^19]: Alpine Linux, Available at: <https://www.alpinelinux.org/>, accessed 22.12.2024.
[^20]: ngspice / Patches / #119 Fix buffer overflow in src/xspice/icm/digital/d_state/cfunc.mod, Available at: <https://sourceforge.net/p/ngspice/patches/119/>, accessed 22.12.2024.
[^21]: Fix test runner constructors order and `_FORTIFY_SOURCE` failure by ArtSin · Pull Request #729 · canonical/dqlite, Available at: <https://github.com/canonical/dqlite/pull/729>, accessed 22.12.2024.
[^22]: 69482 – teststr segfaults if built with -ftrivial-auto-var-init=zero, Available at: <https://bz.apache.org/bugzilla/show_bug.cgi?id=69482>, accessed 22.12.2024.