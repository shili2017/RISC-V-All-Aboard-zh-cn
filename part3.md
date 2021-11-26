# All Aboard 第3节：RISC-V 工具链中的链接器松弛（Linker Relaxation）

原文链接：https://www.sifive.com/blog/all-aboard-part-3-linker-relaxation-in-riscv-toolchain

作者：Palmer Dabbelt, 2017.08.28

翻译：Li Shi, 2021.11.25

上一节我们讨论了重定位与其在 RISC-V 工具链中的应用。本文中我们将进一步深入研究 RISC-V 链接器以及链接器松弛。链接器松弛是一个非常重要的概念，极大地影响了 RISC-V 指令集的设计。链接器松弛是一种在链接时优化程序的机制，且不同于在编译时发生的传统程序优化。本文将通过一个例子来解释链接器松弛的过程，演示链接器松弛可以如何显著提高程序的性能，介绍一种新的 RISC-V 重定位方式。我们不会展开讨论链接器松弛对 RISC-V 指令集的影响，相关内容会在后续文章中阐述。

我们将从一个简单的 C 测试程序开始，这个程序不与任何其他程序链接。这个程序不包括什么有意义的计算过程，我们的目的只是让这个程序足够简单，让我们可以理解相关的概念就行。既然这篇文章是关于链接器的，我会直接跳过汇编器及其之前的编译过程，直接假设我们已经生成了目标文件（译者注：即 .o 文件）。我们直接看下面的一些编译命令：

```
$ cat test.c
int func(int a) __attribute__((noinline));
int func(int a) {
  return a + 1;
}

int _start(int a) {
  return func(a);
}
$ riscv64-unknown-linux-gnu-gcc test.c -o test -O3
$ riscv64-unknown-linux-gnu-objdump -d -r test.o
test.o:     file format elf64-littleriscv
Disassembly of section .text:

0000000000000000 <func>:
   0:   2505                    addiw   a0,a0,1
   2:   8082                    ret

0000000000000004 <_start>:
   4:   00000317                auipc   ra,0x0
                        4: R_RISCV_CALL func
                        4: R_RISCV_RELAX        *ABS*
   8:   00030067                jr      ra
```

我们可以看到一个新的 RISC-V 重定位项：`R_RISCV_CALL` ，位于 `auipc` 和 `jalr` 指令之间（由于这是一个尾调用，可以简写为 `jr` 指令），指向跳转目标（此处跳转到 `func` 符号）。此处 `R_RISCV_CALL` 和后一行的 `R_RISCV_RELAX` 配对，即允许链接器对其进行松弛，这一优化就是本文讨论的重点。

为了理解松弛的概念，我们先回顾一下 RISC-V 指令集。在 RISC-V 指令集中有两条无条件控制转移指令： `jalr` （跳转到寄存器中存储的绝对地址）与 `jal` （跳转到 pc + 立即数偏移）。在这个例子中，目标文件中的 `auipc` + `jalr` 两条指令与单条 `jal` 指令之间的唯一区别是，`auipc` + `jalr` 两条指令可以寻址距当前 PC 的 32 位有符号偏移量，但 `jal` 只能寻址距当前 PC 的 21 位有符号偏移量（不过相比之下 `jal` 毕竟是少了一条指令，提升了速度，也是很合理的）。

由于编译器不知道 `_start` 和 `func` 之间的偏移量是否可以用 21 位有符号数的范围内，保险起见我们有必要生成两条指令，但是我们也不想白费力气，在不必要的情况下多运行一条指令，因此我们选择在链接器中优化这种情况。让我们先来看一下可执行文件，看看链接器松弛优化的结果：
```
$ riscv64-unknown-linux-gnu-objdump -d -r test
test:     file format elf64-littleriscv

Disassembly of section .text:

0000000000010078 <func>:
   10078:       2505                    addiw   a0,a0,1
   1007a:       8082                    ret

000000000001007c <_start>:
   1007c:       ffdff06f                j       10078 <func>
```

如上所示，链接器检测到了 `_start` 到 `func` 的函数调用可以用一条 `jal` 指令中的21位偏移量表示，并将两条指令转换为一条 `jal` 指令。

## 链接器优化在 RISC-V 中的实现

虽然链接器松弛的概念本身非常简单，但需要正确处理很多细节，才能确保链接器可以始终生成正确的符号地址。据我所知，RISC-V BFD 版本非常积极地采用了链接器松弛：在链接阶段之前， `.text` 中的符号地址几乎都是未知的。这也会产生一些有意思的副作用：

* 在所有可松弛的 section 中，`.align` 指令必须由链接器处理。
* 会生成两次调试信息：一次由编译器生成（输出在目标文件中），一次由链接器生成（输出在可执行文件中）。

这些内容比较复杂，我可能会今后单独撰文解释。

链接器松弛的实现其实相当复杂。相关代码在 `binutils-gdb/bfd/elfnn-riscv.c` 中的 `_bfd_riscv_relax_section` ，大致如下所示：
```
_bfd_riscv_relax_section:
  if section shouldn't be relaxed:
    return DONE
  for each relocation:
    if relocation is relaxable:
      store per-relocation function pointer
    read the symbol table
    obtain the symbol's address
    call the per-relocation function
```

本质上，这段代码就是在记录一些信息，然后调用重定位项的相关函数来进行松弛。松弛函数看起来都比较相似，因此我将围绕先前的例子中对 `R_RISCV_CALL` 重定位进行松弛的函数展开讨论：
```
_bfd_riscv_relax_call:
  compute a pessimistic address range
  if relocation doesn't fit into a UJ-type immediate:
    return DONE
  compute offsets for various short jumps
  if RVC is enabled and the relocation fits in a C.JAL:
    convert the jump to c.jal
  if relocation fits in an JAL:
    convert the jump to a jal
  if call target is near absolute address 0:
    convert the jump to a x0-offset JALR
  delete the JALR, as it's not necessary anymore
```

上述函数只会对 `R_RISCV_CALL` 重定位进行松弛，但该函数也遵循了大多数松弛函数实现的模式：
```
generic_relax_function:
  add some slack to the address, as all addresses can move
  for each possible instruction to shorten the relocation:
    if possible instruction can fit the target address:
      replace the relocation
      cleanup
      return DONE
  return DONE
```

下面的每一个松弛函数都对应了一种类型的重定位：

* `_bfd_riscv_relax_call` ： 对 `auipc` + `jalr` 指令组合，通过 `R_RISCV_CALL` 与 `R_RISCV_CALL_PLT` 重定位进行松弛
* `_bfd_riscv_relax_lui` ： 对 `lui` + `addi` （或类似指令），通过 `R_RISCV_HI20` + `R_RISCV_LO12_I` （或类似重定位）重定位进行松弛。第二项指令 / 重定位可以是任何符合 `R_RISCV_LO12_I` 或 `R_RISCV_LO12_S` 重定位的指令（如 `addi` 、 `lw` 、 `sd` 等）。
* `_bfd_riscv_relax_pc` ： 对 `auipc` + `addi` （或类似指令），通过 `R_RISCV_PCREL_HI20` + `R_RISCV_PCREL_LO12_I` （或类似重定位）重定位进行松弛。和 `lui` 的情况类似，第二项有不止一种重定位类型符合要求。
* `_bfd_riscv_relax_tls_le` ： 使用本地执行模型（local executable model）时，对线程本地存储（thread local storage）引用进行松弛。我们会在之后的文章中进一步讨论线程本地存储。
* `_bfd_riscv_relax_align` ： 对 text section 中的 `.align` 命令进行松弛。我们之后也会对这种情况进一步讨论，但需要注意的是，为了确保正确性， `R_RISCV_ALIGN` 重定位始终需要被松弛，且与其他松弛优化是否启用无关。

## 对全局指针（Global Pointer）的松弛

现在看起来似乎我们把事情搞得非常复杂，只是为了一点小小的收益：就为了链接时可以缩小一点指令数量，编译时我们需要假设 `.text` section 的符号地址一直是未知的。然而，事实证明，链接器松弛对于在真实的程序上获得良好性能非常重要。在此之前，我们先来看看 Dhrystone 基准测试。这个基准测试本身非常简单，但同时也会用大量时间来加载全局变量，因而也会很明显地受益于链接器松弛。

我们先来看看 Dhrystone 的源代码。虽然比之前的例子稍微复杂一些，但细看一下，代码其实非常简单。如下所示是一个 Dhrystone 源代码中的函数与其引用的各种全局变量的定义：

```c
/* Global Variables: */
Boolean         Bool_Glob;
char            Ch_1_Glob,
                Ch_2_Glob;

Proc_4 () /* without parameters */
/*******/
    /* executed once */
{
  Boolean Bool_Loc;

  Bool_Loc = Ch_1_Glob == 'A';
  Bool_Glob = Bool_Loc | Bool_Glob;
  Ch_2_Glob = 'B';
} /* Proc_4 */
```

如上所示，这个函数访问了三次全局变量，进行了简单的比较和逻辑运算。虽然这看起来没做什么有用的计算，但这就是很多 Dhrystone 基准测试函数的样子。由于 Dhrystone 几乎是唯一一个可以在任何地方运行的基准测试（相比之下，比如说， SPECInt 就没法在我的智能手表上运行），这个测试仍然被广泛用作处理器微架构的测试基准，我们也因此希望可以运行得更快。

为了理解这个例子中所使用的松弛优化，我们也许可以从工具链在优化之前生成的代码看起来，如下所示：
```
0000000040400826 <Proc_4>:
    40400826:   3fc00797                auipc   a5,0x3fc00
    4040082a:   f777c783                lbu     a5,-137(a5) # 8000079d <Ch_1_Glob>
    4040082e:   3fc00717                auipc   a4,0x3fc00
    40400832:   f7272703                lw      a4,-142(a4) # 800007a0 <Bool_Glob>
    40400836:   fbf78793                addi    a5,a5,-65
    4040083a:   0017b793                seqz    a5,a5
    4040083e:   8fd9                    or      a5,a5,a4
    40400840:   3fc00717                auipc   a4,0x3fc00
    40400844:   f6f72023                sw      a5,-160(a4) # 800007a0 <Bool_Glob>
    40400848:   3fc00797                auipc   a5,0x3fc00
    4040084c:   04200713                li      a4,66
    40400850:   f4e78a23                sb      a4,-172(a5) # 8000079c <Ch_2_Glob>
    40400854:   8082                    ret
```

这个函数有 13 条指令，其中 4 条是 `auipc` 指令。所有的 `auipc` 指令都用于为后续的内存访问计算全局变量的地址，并且所有这些生成的地址都在 12 位偏移量内。如果你认为，我们其实只需要一条 `auipc` 指令就够了，这个想法也许是对的，也许也不对：虽然我们可以只生成一条 `auipc` 指令（这其实涉及到一些我们尚未完成的 GCC 相关的工作），我们实际上可以进行更彻底的优化，不需要任何 `auipc` 指令！

你也许翻遍 RISC-V 指令集手册也找不到办法可以用一条指令加载 `Ch_1_Glob` （地址为 `0x8000079D` ），其实确实不行。但是，有个小技巧，这个技巧在**有很多寄存器，但是寻址模式较少**的指令集上很常见，通常会有一个专用的 ABI 寄存器，叫做全局指针（即 `gp` ，global pointer），专门用来存放 `.data` 段的地址。链接器可以充分利用这个寄存器，访问 `gp` 寄存器 12 位有符号偏移量内的全局变量。本质上而言，我们只是把 `lui` 指令执行的功能转移到了 `gp` 寄存器，优化了代码。

为了更深入地理解内在的工作原理，我们来看一下 GCC 的 RISC-V 默认链接器脚本的片段：

```c
/* We want the small data sections together, so single-instruction offsets
   can access them all, and initialized data all before uninitialized, so
   we can shorten the on-disk segment size.  */
.sdata          :
{
  __global_pointer$ = . + 0x800;
  *(.srodata.cst16) *(.srodata.cst8) *(.srodata.cst4) *(.srodata.cst2) *(.srodata .srodata.*)
  *(.sdata .sdata.* .gnu.linkonce.s.*)
}
_edata = .; PROVIDE (edata = .);
. = .;
__bss_start = .;
.sbss           :
{
  *(.dynsbss)
  *(.sbss .sbss.* .gnu.linkonce.sb.*)
  *(.scommon)
}
```

如上所示， `__global_pointer$` 符号被定义为指向从 `.sdata` section 开始的 `0x800` 个字节。此处 `0x800` 意思就是可以通过 `__global_pointer$` 访问 `.sdata` section 开始的，有符号 12 位偏移量内的符号。链接器假定，如果定义了 `__global_pointer$` 符号，那么 `gp` 寄存器就会存储这个符号的值，之后就可以用这个寄存器对 12 位偏移量范围内的符号进行松弛。编译器将 `gp` 寄存器视为常量，因此也不需要保存或恢复，也就是说 `gp` 通常只会被 `_start` （也就是 ELF 可执行文件的入口）的代码写入。下面这个例子来自于 RISC-V newlib 中的 `crt0.S` 文件：
```
.option push
.option norelax
1:auipc gp, %pcrel_hi(__global_pointer$)
  addi  gp, gp, %pcrel_lo(1b)
.option pop
```

需要注意的是，在设置 `gp` 的时候我们需要禁用松弛，否则链接器会将这两条指令松弛为 `mv gp, gp` 。

松弛过程的实际实现其实很简单，就是在 `_bfd_riscv_relax_lui` 和 `_bfd_riscv_relax_pc` 两个函数中实现。类似于所有其他类型的松弛，这个过程会进行边界检查，移除未使用的指令，然后将短偏移指令转换为不同的类型。我们可能会在以后的文章中更深入地研究各种链接器松弛的实现，但现在我们就先简单看一下松弛过程的输出：
```
00000000400003f0 <Proc_4>:
    400003f0:   8651c783                lbu     a5,-1947(gp) # 80001fbd <Ch_1_Glob>
    400003f4:   8681a703                lw      a4,-1944(gp) # 80001fc0 <Bool_Glob>
    400003f8:   fbf78793                addi    a5,a5,-65
    400003fc:   0017b793                seqz    a5,a5
    40000400:   00e7e7b3                or      a5,a5,a4
    40000404:   86f1a423                sw      a5,-1944(gp) # 80001fc0 <Bool_Glob>
    40000408:   04200713                li      a4,66
    4000040c:   86e18223                sb      a4,-1948(gp) # 80001fbc <Ch_2_Glob>
    40000410:   00008067                ret
```

## 12 位偏移量有时并不够用

需要明确的是：链接器松弛是针对通常情况的优化。对无法优化的情况来说，链接器仅仅是简单地生成两条指令并进行寻址。我们通过下一个例子来解释，链接器无法对某个符号进行松弛时会发生什么：
```
$ cat relax.c
long near;
long far[2];

long data(void) {
  return near | far;
}

int main() {
  return data();
}
$ riscv64-unknown-linux-gnu-gcc relax.c -O3 -o relax --save-temps
$ riscv64-unknown-linux-gnu-objdump -d relax.o
relax.o:     file format elf64-littleriscv

Disassembly of section .text:

0000000000000000 data:
   0:   000007b7                lui     a5,0x0
                        0: R_RISCV_HI20 near
                        0: R_RISCV_RELAX        *ABS*
   4:   0007b503                ld      a0,0(a5) # 0 data
                        4: R_RISCV_LO12_I       near
                        4: R_RISCV_RELAX        *ABS*
   8:   000007b7                lui     a5,0x0
                        8: R_RISCV_HI20 far
                        8: R_RISCV_RELAX        *ABS*
   c:   0007b783                ld      a5,0(a5) # 0 data
                        c: R_RISCV_LO12_I       far
                        c: R_RISCV_RELAX        *ABS*
  10:   8d5d                    or      a0,a0,a5
  12:   8082                    ret

Disassembly of section .text.startup:

0000000000000000 main:
   0:   1141                    addi    sp,sp,-16
   2:   e406                    sd      ra,8(sp)
   4:   00000097                auipc   ra,0x0
                        4: R_RISCV_CALL data
                        4: R_RISCV_RELAX        *ABS*
   8:   000080e7                jalr    ra
   c:   60a2                    ld      ra,8(sp)
   e:   2501                    sext.w  a0,a0
  10:   0141                    addi    sp,sp,16
  12:   8082                    ret
```

在这里我们可以看到三组重定位项：两组 `HI20` / `LO12` 重定位项以及一个 `CALL` 重定位项。在这个例子中， `CALL` 以及标注了 `near` 的 `HI20` / `LO12` 重定位项可以被链接器松弛，但是另一对标注了 `far` 的 `HI20` / `LO12` 重定位项就不行。链接器仍然可以正常工作，对标注了 `near` 的符号进行松弛并生成单条寻址指令，但是对标注了 `far` 的符号就无法进行松弛。
```
$ riscv64-unknown-linux-gnu-objdump -d -r relax
Disassembly of section .text:

0000000000010330 main:
   10330:       1141                    addi    sp,sp,-16
   10332:       e406                    sd      ra,8(sp)
   10334:       0b8000ef                jal     ra,103ec data
   10338:       60a2                    ld      ra,8(sp)
   1033a:       2501                    sext.w  a0,a0
   1033c:       0141                    addi    sp,sp,16
   1033e:       8082                    ret

00000000000103ec data:
   103ec:       8181b503                ld      a0,-2024(gp) # 12038 near
   103f0:       67e9                    lui     a5,0x1a
   103f2:       0407b783                ld      a5,64(a5) # 1a040 far
   103f6:       8d5d                    or      a0,a0,a5
   103f8:       8082                    ret
```

尽管有点多余，我还是想用下面的例子来进一步详细说明：

```
--- relax.o
+++ relax
-      4:   00000097                auipc   ra,0x0
-                           4: R_RISCV_CALL         data
-                           4: R_RISCV_RELAX        *ABS*
-      8:   000080e7                jalr    ra
+  10334:   0b8000ef                jal     ra,103ec data
```

此处我们可以看到 `R_RISCV_CALL` 需要进行重定位。该重定位被定义为，在相邻的 `auipc` / `jalr` 两条指令上进行，并引用一个有符号的 32 位 PC 相关的函数调用目标地址。在这个例子中，由于跳转目标位于当前 PC 的 21 位有符号偏移量内，我们可以对这两条指令进行松弛，生成单条 `jal` 指令。由于大多数代码都表现出了函数调用的局部性特征，我们可以发现几乎所有的 `R_RISCV_CALL` 重定位都是可以松弛的。

```
--- relax.o
+++ relax
-      0:   000007b7                lui     a5,0x0
-                          0: R_RISCV_HI20         near
-                          0: R_RISCV_RELAX        *ABS*
-      4:   0007b503                ld      a0,0(a5) # 0 data
-                          4: R_RISCV_LO12_I       near
-                          4: R_RISCV_RELAX        *ABS*
+  103ec:   8181b503                ld      a0,-2024(gp) # 12038 near
```

此处我们可以看到 `R_RISCV_HI20` / `R_RISCV_LO12_I` 也需要进行重定位。这里每一个重定位项都定义为对单个指令进行操作：`R_RISCV_HI20` 对 `lui` 指令的 20 位偏移量进行重定位，`R_RISCV_LO12` 对 I 型指令（在本例中即 `ld` 指令）的 12 位偏移量进行重定位。在这个例子中，由于最终的符号地址在 `gp` 的 12 位偏移量内，我们可以对这条 `ld` 指令进行重定位。

```
--- relax.o
+++ relax
-      8:   000007b7                lui     a5,0x0
-                           8: R_RISCV_HI20         far
-                           8: R_RISCV_RELAX        *ABS*
-      c:   0007b783                ld      a5,0(a5) # 0 data
-                           c: R_RISCV_LO12_I       far
-                           c: R_RISCV_RELAX        *ABS*
+  103f0:   67e9                    lui     a5,0x1a
+  103f2:   0407b783                ld      a5,64(a5) # 1a040 far
```

此处有和上面类似的 `R_RISCV_HI20` / `R_RISCV_LO12_I` ，但是由于不在 `gp` 的 12 位偏移量内，我们无法对其进行松弛，但我们仍然可以往指令立即数进行填充，生成正确的代码。不过，如果无论如何都无法正确进行重定位，无法生成正确的代码，链接器就会报错。

请继续关注，在未来的文章中还会有更多关于链接器松弛的内容。
