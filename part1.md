# All Aboard 第1节：RISC-V编译器的`-march`, `-mabi`和`-mtune`参数

原文链接：https://www.sifive.com/blog/all-aboard-part-1-compiler-args

作者：Palmer Dabbelt, 2017.08.14

翻译：Li Shi, 2021.11.24

在我们开始 RISC-V 的旅程之前，让我们先在起点停留一会儿：机器特定（machine-specific）的 GCC 命令行参数。这些参数都以 `-m` 开头，并且都是 RISC-V 指令集架构独有的。许多编译器参数的设定都和已有的设定保持一致，但是 RISC-V 的编译器仍有一些特殊之处，值得单独写一篇文章。本文讨论了 RISC-V 指令集最基本的编译器参数：`-march`, `-mabi`和`-mtune`。

在我们最终确定 SiFive 的软件接口之前，使用 GCC 的 RISC-V 版本的好处在于，我们可以直接迁移到现有的成熟完善的 C/C++ 编译器工具链。我们可以在 GNU 工具链（GCC 和 binutils）和 LLVM 工具链中使用相同的命令行接口，避免用户需要通过编译器的 `-Wa` 和 `-Wl` 参数直接将编译选项传递给汇编器或链接器。

为确保 RISC-V 编译器命令行接口在未来易于扩展，我们决定采用以下方案，用户使用三个参数来描述编译的 RISC-V 目标：
* `-march=ISA` 目标架构，即编译器可以使用哪些指令和寄存器。
* `-mabi=ABI` 目标 ABI，即函数调用约定（在哪些寄存器中传递哪些参数）和内存中数据的布局。
* `-mtune=CODENAME` 目标微架构，即告知 GCC 每条指令的性能，允许编译器执行特定于目标的优化。

## `-march` 参数
`-march` 参数由 RISC-V 用户级指令集手册定义。 `-march` 控制编译器生成指令的指令集。这个参数决定了编译出来的程序可以运行在什么样的 RISC-V 处理器上：任何包含了 `-march` 指定的指令集的 RISC-V 处理器都应该能够运行该程序。

具体而言，RISC-V 用户级指令集 2.2 版定义了编译器工具链当前支持的三种基本指令集：
* RV32I：具有 32 个 32 位通用整数寄存器的指令集。
* RV32E：RV32I 的嵌入式风格，只有 16 个整数寄存器。
* RV64I：RV32I 的 64 位风格，其中通用整数寄存器为 64 位宽。
 
除了这些基本指令集之外，还定义了一些指令集扩展。工具链已定义并支持的指令集扩展有：
* M：整数乘法和除法
* A：原子指令
* F：单精度浮点
* D：双精度浮点
* C：压缩指令

RISC-V 指令集扩展的字符串可以通过按照上面列出的顺序，将支持的扩展附加到基本指令集来定义。例如，具有 32 个 32 位整数寄存器和乘除法指令的 RISC-V 指令集可以表示为 `RV32IM` 。用户可以通过将小写的 ISA 字符串传递给 `-march` GCC 参数来控制 GCC 在生成汇编代码时使用的指令集：例如 `-march=rv32im` 。

在不支持某些运算的 RISC-V 系统中，可以使用模拟函数 (emulation routine）来提供缺少的功能。例如下面的C代码
```c
double dmul(double a, double b) {
  return a * b;*
}
```

使用 D 扩展编译时将直接编译为浮点乘法指令
```bash
$ riscv64-unknown-elf-gcc test.c -march=rv64imafdc -mabi=lp64d -o- -S -O3
dmul:
  fmul.d  fa0,fa0,fa1
  ret
```

未使用 D 扩展编译时将编译为模拟函数
```bash
$ riscv64-unknown-elf-gcc test.c -march=rv64i -mabi=lp64 -o- -S -O3
dmul:
  add     sp,sp,-16
  sd      ra,8(sp)
  call    __muldf3
  ld      ra,8(sp)
  add     sp,sp,16
  jr      ra
```

M 和 F 扩展也实现了类似的模拟函数。在撰写本文时，A 扩展还没有对应的模拟函数的实现，这些函数的实现被 Linux 上游拒绝。在未来可能仍有变化，单目前来说，我们计划在 RISC-V 平台规范中，强制把 A 扩展的实现作为支持 Linux 运行的一部分。

## `-mabi` 参数
`-mabi` 参数指定了整数与浮点 ABI（译者注： ABI 即 Application Binary Interface，应用程序二进制接口）。类似于 `-march` 参数指定了代码可以运行在什么样的硬件上， `-mabi` 参数指定了代码可以链接到什么样的软件。我们在整数 ABI （ `ilp32` 或 `lp64` ）使用标准命名规范，并可以附加一个字母来选择 ABI 使用的浮点寄存器 （ `ilp32` vs `ilp32f` vs `ilp32d` ）。为了可以链接目标文件（译者注：即 .o 文件），它们必须遵循相同的 ABI。

RISC-V 定义了两个整数 ABI 和三个浮点 ABI，可以由一个 ABI 字符串描述。整数 ABI 遵循标准 ABI 命名方案：
* `ilp32`： `int`, `long` 以及指针都是 32 位长的。 `long` 是 64 位类型，`char` 是 8位类型，`short` 是 16 位类型。
* `lp64`： `long` 以及指针是 64 位长的，但`int` 仍是32 位长的。其他类型与 ilp32 相同。

浮点 ABI 是 RISC-V 独有的：
* “”（空字符串）：寄存器中不传递浮点参数。
* `f`： 32 位和位宽更小的浮点参数在寄存器中传递。此 ABI 需要 F 扩展，因为如果没有 F 扩展也就没有浮点寄存器。
* `d`： 64 位和位宽更小的浮点参数在寄存器中传递。此 ABI 需要 D 扩展。

和指令集字符串一样，ABI 字符串连接在一起并通过 `-mabi` 参数传递给 GCC。为了解释为什么 ISA 和 ABI 应该分成两个单独的参数，我们可以看一些 `-march` / `-mabi` 的组合：
* `-march=rv32imafdc -mabi=ilp32d`： 可以生成硬件浮点指令并将浮点参数传递到寄存器中，类似于 ARM GCC 中的 `-mfloat-abi=hard` 参数。
* `-march=rv32imac -mabi=ilp32`： 不能生成浮点指令，也不能在寄存器中传递浮点参数，类似于 ARM GCC 中的 `-mfloat-abi=soft` 参数。
* `-march=rv32imafdc -mabi=ilp32`： 可以生成硬件浮点指令，但不会在寄存器中传递浮点参数，类似于 ARM GCC 中的 `-mfloat-abi=softfp` 参数。该组合通常在硬件浮点系统上，与软浮点程序交互时使用。
* `-march=rv32imac -mabi=ilp32d`：非法，因为 ABI 要求在寄存器中传递浮点参数，但指令集没有定义浮点寄存器，无法传递这些浮点参数。

我们来看一个更具体的例子，在这个简单的 C 函数中，输入两个双精度浮点参数，并返回它们的乘积。为了更明显地看出参数的位置，我们将颠倒函数调用和执行乘法操作的参数顺序：
```c
double dmul(double a, double b) {
  return b * a;
}
```

第一种情况是最简单的：如果 ABI 和 ISA 都不包含浮点硬件相关的内容，那么 C 编译器就不能编译出任何硬件浮点指令。在这种情况下，模拟函数参与计算，参数在整数寄存器中传递。双精度参数在 32 位整数寄存器中传递，参数的顺序被交换，`ra` 保存在栈上（被调用者保存，即 callee-saved），然后调用模拟函数，最后从栈上恢复数据，并返回运算结果（根据 `__muldf3` 函数，结果存储在`a0,a1` 寄存器中）。
```bash
$ riscv64-unknown-elf-gcc test.c -march=rv32imac -mabi=ilp32 -o- -S -O3
dmul:
  mv      a4,a2
  mv      a5,a3
  add     sp,sp,-16
  mv      a2,a0
  mv      a3,a1
  mv      a0,a4
  mv      a1,a5
  sw      ra,12(sp)
  call    __muldf3
  lw      ra,12(sp)
  add     sp,sp,16
  jr      ra
```

第二种情况与第一种情况完全相反：硬件完善地支持浮点指令与浮点寄存器。在这种情况下，我们可以生成一条 `fmul.d` 指令来执行浮点运算，当寄存器正确分配时，它会处理颠倒的输入参数并产生返回值。
```bash
$ riscv64-unknown-elf-gcc test.c -march=rv32imafdc -mabi=ilp32d -o- -S -O3
dmul:
  fmul.d  fa0,fa1,fa0
  ret
```

最后一种情况揭示了为什么 RISC-V 编译器需要分离 `-march` 和  `-mabi` 参数：用户可能希望生成的代码可以与**为不包含特定扩展的系统编译的程序**链接，同时仍然可以利用额外的、特定扩展的指令。对于把旧代码集成到新系统而言，这是一个很常见的问题，因此我们设计了这样的编译器参数和 multilib 路径，可以顺利完成这样的工作。

这种情况下生成的代码本质上是上面两种情况的混合体：参数在 `ilp32` ABI指定的寄存器中传递（注意和 `ilp32d` ABI的区别， `ilp32d` 允许在浮点寄存器中传递这些参数），但是一旦进入函数主体，编译器就可以自由地使用`rv32imafdc` 指令集的全部指令来做实际的运算。因此，编译器将双精度浮点数参数放入内存（ `rv32` 中唯一可以构造一个双精度浮点数的方法），随后在函数中将参数 load 到浮点寄存器中，执行浮点计算，将浮点寄存器中的 结果 store 到堆栈，并将结果 load 到符合 ABI 的返回值寄存器（ `a0` 和 `a1` ）。虽然这比充分利用 D 扩展寄存器生成的代码效率低，但也比没有 D 扩展指令的情况下计算浮点乘法高效得多。
```bash
$ riscv64-unknown-elf-gcc test.c -march=rv32imafdc -mabi=ilp32 -o- -S -O3
dmul:
  add     sp,sp,-16
  sw      a0,8(sp)
  sw      a1,12(sp)
  fld     fa5,8(sp)
  sw      a2,8(sp)
  sw      a3,12(sp)
  fld     fa4,8(sp)
  fmul.d  fa5,fa5,fa4
  fsd     fa5,8(sp)
  lw      a0,8(sp)
  lw      a1,12(sp)
  add     sp,sp,16
  jr      ra
```

最后一种 ABI/ISA 组合很简单理解：这种组合是非法的。如果编译器无法生成访问浮点寄存器的指令，那么就根本无法通过浮点寄存器传递参数。由于这是编译器输入命令本身的错误，编译器直接报错退出。
```bash
$ riscv64-unknown-elf-gcc test.c -march=rv32imac -mabi=ilp32d -o- -S -O3
cc1: error: requested ABI requires -march to subsume the ‘D’ extension
```

## `-mtune` 参数
选择 RISC-V 目标所涉及的最后一个编译器参数也是最简单的。如果 `-march` 参数设置错误，可能会导致系统无法执行代码；如果 `-mabi` 参数设置错误，可能会导致目标文件彼此不兼容，无法被链接。相比之下， `-mtune` 参数只会改变生成代码运行时的性能。就目前而言，我们确实没有任何用于 RISC-V 系统的模型。除非您向 GCC 软件版本添加了新的微架构调整参数，否则您不应该使用这个参数以达成任何目的。
