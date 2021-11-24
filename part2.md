# All Aboard 第2节：ELF 工具链中的重定位

原文链接：https://www.sifive.com/blog/all-aboard-part-2-relocations

作者：Palmer Dabbelt, 2017.08.21

翻译：Li Shi, 2021.11.24

我们探索 RISC-V 工具链的第一站，将概述 ELF 重定位以及 RISC-V 工具链中与之相关的内容。我们将在后面一节中讨论链接器松弛（linker relaxation）及其对性能的影响，但在本节中不涉及相关内容。为了避免误会，本节中的示例程序已精心构造为无法进行链接器放松的。此外，我们将只讨论静态链接的可执行文件使用的重定位，暂时不讨论位置无关的可执行文件，也不讨论线性局部存储——类似于链接器松弛，这些进阶内容都需要单独写一整篇文章来解释。在之后的文章中也会进一步涉及到重定位的相关内容。

## C 程序中的重定位示例

重定位是**由于大多数工具链中编译器和链接器相分离**而存在的概念。虽然本文的细节仅适用于基于 ELF 的 RISC-V 工具链（即 GCC + binutils 或 LLVM），但重定位的一般概念存在于更广泛的编译器（如 Hotspot）中。重定位的目的是为了在编译器和链接器之间传递信息，让我们首先来看一个简单的程序是如何编译的。如以下 C 代码：
```c
long global_symbol[2];

int main() {
  return global_symbol[0] != 0;
}
```

尽管一条 GCC 命令就可以为这个简单的程序生成一个二进制文件，但实际上 GCC 内部脚本依次运行了预处理器、编译器、汇编器、以及链接器。GCC `—save-temps` 参数允许用户查看所有这些中间文件，用户可以更好地理解编译器工具链的内部实现。
```
$ riscv64-unknown-linux-gnu-gcc relocation.c -o relocation -O3 —save-temps
```

GCC 脚本运行的每一步都会生成一个文件：
* `relocation.i`： 预处理源代码，处理了代码中的预处理器指令（如 `#include` 或 `#ifdef` ）。
* `relocation.s`： 编译器的输出，汇编代码（RISC-V汇编格式的文本文件）。
* `relocation.o`： 汇编器的输出，未链接的目标文件（一个 ELF 文件，但不是可执行的 ELF）。
* `relocation`： 链接器的输出，已链接的可执行文件（一个可执行的 ELF 文件）。

第一步是运行预处理器。由于这是一个没有预处理器宏的简单程序，预处理器没做什么事：它所做的只是发出一些指令，以便在之后生成调试信息时使用：
```
$ cat relocation.i
# 1 "relocation.c"
# 1 "built-in"
# 1 "command-line"
# 31 "command-line"
# 1 "_scratch_palmer_work_upstream_riscv-gnu-toolchain_build_install_sysroot_usr_include/stdc-predef.h" 1 3 4
# 32 "command-line" 2
# 1 "relocation.c"
long global_symbol;

int main() {
  return global_symbol != 0;
}
```

预处理的输出马上输入到编译器，编译器生成目标的汇编代码。此时，我们开始理解为什么我们需要重新定位。生成的文件是包含 RISC-V 汇编代码的纯文本文件，很容易阅读，如下所示：
```
$ cat relocation.s
main:
  lui   a5,%hi(global_symbol)
  ld    a0,%lo(global_symbol)(a5)
  snez  a0,a0
  ret
```

如果之前没有看过 RISC-V GCC 的汇编代码输出，那么这段代码可能读起来有点奇怪。这里似乎有两种没有在 RISC-V 指令集手册中出现过的寻址模式，其实实际上也没有什么好办法在硬件中实现： `%hi(global_symbol)` 和 `%lo(global_symbol)(a5)` 。

这两种寻址模式的存在是为了允许编译器寻址全局符号（global symbols）。寻址全局符号的根本问题是编译器需要生成访问这些全局符号的汇编指令，但这些全局符号的实际地址直到链接时才能生成，目前还不知道它们的地址。在这个例子中，编译器在 `lui` 指令中还不知道 `global_symbol` 的地址。

重定位解决了这一问题。当编译器无法生成指令中的部分位时，只需填充任意位即可，并生成一个重定位条目。这一重定位条目指向这些位，并包含足够的信息供链接器填充这些位。

我们通过例子来解释这一概念，回到刚刚简单的程序。工具链的下一步是是汇编器，从前面的步骤中接收汇编文件并生成尚未链接的 ELF 目标文件。可以使用 objdump 工具看这些目标文件的内容，如下所示：
```
$ riscv64-unknown-linux-gnu-objdump -d -t -r relocation.o

relocation.o:     file format elf64-littleriscv

SYMBOL TABLE:
0000000000000000 l    df *ABS*  0000000000000000 relocation.c
0000000000000000 l    d  .text  0000000000000000 .text
0000000000000000 l    d  .data  0000000000000000 .data
0000000000000000 l    d  .bss   0000000000000000 .bss
0000000000000000 l    d  .text.startup  0000000000000000 .text.startup
0000000000000000 l    d  .comment       0000000000000000 .comment
0000000000000000 g     F .text.startup  000000000000000e main
0000000000000010       O *COM*  0000000000000008 global_symbol

Disassembly of section .text.startup:

0000000000000000 main:
  0:   000007b7                lui     a5,0x0
                       0: R_RISCV_HI20         global_symbol
                       0: R_RISCV_RELAX        *ABS*
  4:   0007b503                ld      a0,0(a5) # 0 main
                       4: R_RISCV_LO12_I       global_symbol
                       4: R_RISCV_RELAX        *ABS*
  8:   00a03533                snez    a0,a0
  c:   8082                    ret
```

现在我们第一次可以明显看到重定位（仅在将 `-r` 参数传递给 objdump 时才会显示）。在这里，我们可以看到两对、四行 RISC-V 特有的重定位项：第一对是 `lui` 指令下的 `R_RISCV_HI20`+`R_RISCV_RELAX`，第二对是 `ld` 指令下的 `R-RISCV_LO12_I`+`R_RISCV_RELAX`。`R_RISCV_RELAX` 重定位项仅是为了表明对前一个重定位项进行链接器松弛是合法的。我们在本文中不讨论链接器松弛，先暂时跳过这些。

另外两个重定位项与 RISC-V ISA 中的寻址模式对应： `R_RISCV_HI20` 与 U 型立即数对应， `R_RISCV_LO12_I` 与 I 型立即数对应。一般而言，每个带有立即数的寻址模式都会至少有一个重定位方式来填充该立即数，如果某种指令格式需要链接更复杂的符号形式，可能会有不止一种重定位方式（如 PIC 或 TLS 重定位，译者注：PIC 即 Position Independent Code，位置无关代码，TLS 即 Thread Local Storage，线程本地存储）。

在我们深入研究重定位之前，让我们快速检查一下，正确进行重定位后，工具链是如何运行的。工具链的下一步是运行链接器，填充汇编器生成的重定位项，输出 ELF 可执行文件。该程序现在链接了所有 glibc 的代码，会变得非常大，因此我们只看一下 ELF 文件中相关的片段：
```
$ riscv64-unknown-linux-gnu-objdump -d -t -r relocation
relocation:     file format elf64-littleriscv

SYMBOL TABLE:
0000000000012038 g     O .bss 0000000000000010              global_symbol
...

Disassembly of section .text:

0000000000010330 main:
 10330:       67c9                    lui     a5,0x12
 10332:       0387b503                ld      a0,56(a5) # 12038 global_symbol
 10336:       00a03533                snez    a0,a0
 1033a:       8082                    ret
```

如上所示，符号表现在已经有了 `global_symbol` 的实际地址，重定位相关的指令填充了实际地址，并且重定位项已不再需要，且已从 ELF 文件中删除——注意这只是一种特殊情况，我们暂时只考虑静态链接符号。与之相对的是动态链接，重定位动态符号的过程会被推迟到加载阶段进行。

## `relocation truncated to fit` 错误信息

现在我们已经大致了解了重定位的概念，接下来我们讨论一下大多数人唯一碰到的关于重定位的问题：链接时出现的错误信息 `relocation truncated to fit`。如果不了解重定位，很难解释发生了什么，但是了解重定位之后，这个错误也没有那么棘手。

为了解释该错误信息，我们将从一个非常简单的程序开始。在这个例子中，我们不希望 C 库中的任何内容出现在我们的错误信息中，因此我们会定义 `_start` 而不是 `main` 函数，然后通过给 GCC 传递 `-nostdlib -nostartfiles` 参数，避免任何标准库对象出现在例子中——这个程序实际上运行不了，但有助于解释到底会发生什么。用 `-Wl,-Ttext-segment,0x80000000` 参数来移动 text section 将会触发该错误，如下所示：
```
$ cat reloc_fail.c
long global_symbol;
int _start() {
  return global_symbol;
}
$ riscv64-unknown-linux-gnu-gcc reloc_fail.c -o reloc_fail -O3 -nostartfiles -nostdlib —save-temps  -Wl,-Ttext-segment,0x80000000
reloc_fail.o: In function `_start’:
reloc_fail.c:(.text+0x0): relocation truncated to fit: R_RISCV_HI20 against symbol `global_symbol’ defined in COMMON section in reloc_fail.o
_scratch_palmer_work/20170725-binutils-2.29_install_bin_.._lib_gcc_riscv64-unknown-linux-gnu/7.1.1_.._.._.._.._riscv64-unknown-linux-gnu_bin_ld: final link failed: Symbol needs debug section which does not exist
collect2: error: ld returned 1 exit status
```

乍一看，好像是一个超级可怕的错误信息：有各种对临时对象的引用；有关各种符号、 section 与重定位；以及关于 debug section 的奇怪信息。通常到这个时候，人们就会选择放弃，去找一个工具链专家来帮忙解决问题，但是理解重定位之后，我们就可以明白发生了什么。

首先，让我们只关注错误信息的重要部分，忽略所有实际上不相关的细节。我们真正关注的错误信息是：
```
reloc_fail.c:(.text+0x0): relocation truncated to fit: R_RISCV_HI20 against symbol `global_symbol’
```

这说明编译器尝试用 `global_symbol` 符号的地址填充 `R_RISCV_HI20` 重定位项，但是链接器无法将该符号的完整地址放入这个重定位项指定的位。这个错误信息 `truncated to fit` 有点奇怪：链接器实际的意思是，为了填充指令中的立即数，重定位项中的地址必须要被截断，但由于这里发生了一个错误，链接器并没有真正截断任何地址。

为了真正深入研究错误的原因，我们需要首先检查链接器的输入。在这种例子中，链接器的输入是汇编器生成的目标文件。像之前的例子一样，由于编译器需要引用一个地址未知的全局符号，我们需要对其进行重定位。
```
$ riscv64-unknown-linux-gnu-objdump -d -r reloc_fail.o
reloc_fail.o:     file format elf64-littleriscv

Disassembly of section .text:

0000000000000000 <_start>:
  0:   000007b7                lui     a5,0x0*
                       0: R_RISCV_HI20 global_symbol
                       0: R_RISCV_RELAX        `ABS`
  4:   0007a503                lw      a0,0(a5) # 0 <_start>
                       4: R_RISCV_LO12_I       global_symbol
                       4: R_RISCV_RELAX        `ABS`
  8:   8082                    ret
```

由于无法进行链接，我们无法看到链接器的输出。我不喜欢动手做算术，那么我就只修改一下链接器的代码，在进行重定位时忽略范围检查，如下所示：
```
$ git diff
diff --git a_bfd_elfnn-riscv.c b_bfd_elfnn-riscv.c
index 3c04507623c3..f8a97411de35 100644
--- a_bfd_elfnn-riscv.c
+++ b_bfd_elfnn-riscv.c
@@ -1492,8 +1492,6 @@ perform_relocation (const reloc_howto_type *howto,
     case R_RISCV_GOT_HI20:
     case R_RISCV_TLS_GOT_HI20:
     case R_RISCV_TLS_GD_HI20:
-      if (ARCH_SIZE > 32 && !VALID_UTYPE_IMM (RISCV_CONST_HIGH_PART (value)))
-       return bfd_reloc_overflow;
       value = ENCODE_UTYPE_IMM (RISCV_CONST_HIGH_PART (value));
       break;
```

我们用这个 patch，可以让链接器生成我们可以检查的错误的目标文件，如下所示：
```
$ riscv64-unknown-linux-gnu-objdump -d -t reloc_fail
reloc_fail:     file format elf64-littleriscv

SYMBOL TABLE:
00000000800000b0 l    d  .text  0000000000000000 .text
00000000800010c0 l    d  .bss   0000000000000000 .bss
0000000000000000 l    d  .comment       0000000000000000 .comment
0000000000000000 l    df *ABS`  0000000000000000 reloc_fail.c
00000000800018ba g       .text  0000000000000000 __global_pointer$
00000000800010c0 g     O .bss   0000000000000008 global_symbol
00000000800000b0 g     F .text  000000000000000a _start
00000000800010ba g       .bss   0000000000000000 __bss_start
00000000800010ba g       .bss   0000000000000000 _edata
00000000800010c8 g       .bss   0000000000000000 _end

Disassembly of section .text:

00000000800000b0 <_start>:
    800000b0:   800017b7                lui     a5,0x80001
    800000b4:   0c07a503                lw      a0,192(a5) # ffffffff800010c0 <__global_pointer$+0xfffffffefffff806>
    800000b8:   8082                    ret
```

我们可以发现，`lw` 指令中 `global_symbol` 的值（译者注： `ffffffff800010c0` ）与符号表中列出的 `global_symbol` 的地址（译者注： `00000000800010c0` ）不一致，这正是 `relocation truncated to fit` 错误信息背后的原因。对于这一例子中的 `R_RISCV_HI20`+`R_RISCV_LO12_I` 重定位项，可以访问的最大绝对地址是 `0x7FFFFFFF` ——这是 RISC-V 指令集中 U 型立即数的限制，任何大于该值的绝对地址都会在 RV64 架构上发生溢出。

虽然每一个指令集架构在链接阶段都会进行一部分重定位，但 RISC-V 比任何其他架构都会更激进地利用链接器软件的重定位功能，因此此类问题在 RISC-V 中可能会比其他架构更频繁地出现。由于重定位可能会导致其他工具链设计上的问题，我们将会进一步深入讨论重定位。
