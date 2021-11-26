# All Aboard 第4节：RISC-V 代码模型

原文链接：https://www.sifive.com/blog/all-aboard-part-4-risc-v-code-models

作者：Palmer Dabbelt, 2017.09.11

翻译：Li Shi, 2021.11.26

RISC-V 指令集的设计简单且模块化。为了实现这些设计目标，RISC-V 尽可能简化了实现复杂指令集时开销最大的部分：寻址模式。寻址模式在小型设计（由于指令解码的开销）和大型设计（由于隐式依赖）中的开销都很大。RISC-V 只有三种寻址模式：
* 相对于 PC 寻址（ `auipc` 、 `jal` 和 `br*` 指令）
* 寄存器-偏移量寻址（ `jalr` 、 `addi` 以及所有访存指令）
* 绝对地址寻址（ `lui` 指令，不过也可以说这是 `x0` -偏移量寻址）

这些寻址模式经过了精心选择取舍，可以在最简化的硬件上高效地生成代码。我们通过现代化的工具链，在软件中优化寻址，以实现这种简化的寻址模式设计——这与传统的指令集形成了鲜明对比，后者在硬件中实现了过多复杂的寻址模式。研究表明，RISC-V 的设计是合理的：我们能够在基准测试中得到类似的代码体积，且解码规则更简单，空闲的编码空间更大。

硬件复杂性的降低都以软件复杂性的提升为代价。本文介绍了 RISC-V 中的另一种软件复杂性：代码模型的概念。类似于重定位和松弛，代码模型并不是 RISC-V 指令集独有的——事实上，RISC-V 工具链的代码模型数量比大多数流行的指令集要少，这一点主要是因为我们更依赖软件优化而非复杂的寻址模式，我们的寻址模式也因而更加灵活。

## 什么是代码模型

程序的地址空间一般都非常大，但是大多数程序的符号不会用到整个可用的地址空间（有些程序会用堆来填充可用的地址空间）。指令集可以充分利用这一局部性特征，在硬件中只实现简单的寻址模式，并依赖软件来提供更丰富的寻址模式。代码模型决定了我们要使用哪种软件寻址模式，因此也就决定了链接器的约束条件。注意，软件寻址模式决定了程序员如何在软件中访问内存地址，而硬件寻址模式决定了硬件如何处理指令中的地址位。

由于我们分离了编译器和链接器，代码模型这一概念是有必要的。具体来说，在生成一个未链接的目标文件时，编译器不知道任何符号的绝对地址，但是编译器仍然必须知道使用什么样的寻址模式，其原因在于某些寻址模式可能需要临时寄存器（scratch registers）进行操作。编译器无法生成实际的寻址代码，但是取而代之的是，编译器会生成寻址模板（即重定位），根据编译器生成的这些信息，当链接器确定每个符号的地址之后，链接器就可以填充这些指令中的地址。代码模型确定了这些寻址模板，从而确定如何生成重定位信息。

我们用一个例子来解释这些概念，如下 C 程序所示：
```c
long global_symbol[2];

int main() {
  return global_symbol[0] != 0;
}
```

尽管一条 GCC 命令就可以为这个简单的程序生成一个二进制文件，但实际上 GCC 内部脚本依次运行了预处理器、编译器、汇编器、以及链接器。GCC `—save-temps` 参数允许用户查看所有这些中间文件，用户可以更好地理解编译器工具链的内部实现。
```
$ riscv64-unknown-linux-gnu-gcc cmodel.c -o cmodel -O3 --save-temps
```

GCC 脚本运行的每一步都会生成一个文件：
* `cmodel.i`： 预处理源代码，处理了代码中的预处理器指令（如 `#include` 或 `#ifdef` ）。
* `cmodel.s`： 编译器的输出，汇编代码（RISC-V汇编格式的文本文件）。
* `cmodel.o`： 汇编器的输出，未链接的目标文件（一个 ELF 文件，但不是可执行的 ELF）。
* `cmodel`： 链接器的输出，已链接的可执行文件（一个可执行的 ELF 文件）。

为了理解为什么我们需要代码模型，我们需要更深入地研究一下工具链的流程。由于这是一个没有预处理器宏的简单程序，预处理器没做什么事：它所做的只是发出一些指令，以便在之后生成调试信息时使用：
```
$ cat cmodel.i
# 1 "cmodel.c"
# 1 "built-in"
# 1 "command-line"
# 31 "command-line"
# 1 "/scratch/palmer/work/upstream/riscv-gnu-toolchain/build/install/sysroot/usr/include/stdc-predef.h" 1 3 4
# 32 "command-line" 2
# 1 "cmodel.c"
long global_symbol;

int main() {
  return global_symbol != 0;
}
```

预处理的输出作为编译器的输入，编译器生成目标的汇编代码。生成的文件是包含 RISC-V 汇编代码的纯文本文件，很容易阅读，如下所示：
```
$ cat cmodel.s
main:
  lui   a5,%hi(global_symbol)
  ld    a0,%lo(global_symbol)(a5)
  snez  a0,a0
  ret
```

在生成的汇编中， `lui` 和 `ld` 两条指令会对 `global_symbol` 寻址，这里我们需要对 `global_symbol` 可以放置的地址进行限制：必须可以通过一个 32 位有符号常量进行寻址（注意不是某个寄存器的值或是 PC 的值加上一个 32 位偏移量，而是一个 32 位的绝对地址）。需要注意的是，这里对符号地址的限制与架构中的指针位宽无关，也就是说，指针可能是 64 位长，但是所有的全局符号仍然需要可以通过 32 位绝对地址寻址。

编译器生成汇编后，GCC 脚本调用汇编器生成目标文件。这个文件是一个 ELF 二进制文件，可以用 Binutils 中的各种工具读取。我们可以用 `objdump` 工具来显示符号表（symbol table），输出 text 段的反汇编，并显示汇编器生成的重定位信息：
```
$ riscv64-unknown-linux-gnu-objdump -d -t -r cmodel.o

cmodel.o:     file format elf64-littleriscv

SYMBOL TABLE:
0000000000000000 l    df *ABS*  0000000000000000 cmodel.c
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
                        0: R_RISCV_HI20 global_symbol
                        0: R_RISCV_RELAX        *ABS*
   4:   0007b503                ld      a0,0(a5) # 0 main
                        4: R_RISCV_LO12_I       global_symbol
                        4: R_RISCV_RELAX        *ABS*
   8:   00a03533                snez    a0,a0
   c:   8082                    ret
```

在生成的目标文件中，我们仍然不知道任何全局符号的实际地址。这里其实工具链各个阶段所做的工作有一些重合：将文本形式的指令转换成二进制代码应该是汇编器的工作，但是我们发现这里一些全局符号的地址仍然是未知的（比如 `lui` 指令中的立即数），汇编器也就没办法真正去填充这些地址。为了让链接器在生成最终的可执行文件时填充这些地址，汇编器会生成一张重定位表（relocation table）。表中的重定位项定义了链接器应该如何填充这些地址。 text section 中出现的任何重定位类型都是指令集特定的，比如 RISC-V 关于重定义的定义可以参考 [ELF psABI 文档](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/blob/master/riscv-elf.adoc)。

汇编器完成工作后，GCC 脚本将运行链接器，生成 ELF 格式的可执行文件。由于最终的可执行文件中包含大量的 C 库代码，我们就只看其中的相关片段：
```
$ riscv64-unknown-linux-gnu-objdump -d -t -r cmodel
cmodel:     file format elf64-littleriscv

SYMBOL TABLE:
0000000000012038 g     O .bss    0000000000000010              global_symbol
...

Disassembly of section .text:

0000000000010330 main:
 10330:       67c9                    lui     a5,0x12
 10332:       0387b503                ld      a0,56(a5) # 12038 global_symbol
 10336:       00a03533                snez    a0,a0
 1033a:       8082                    ret
```

这里有几点需要注意：

* 符号表中存储了具有实际绝对地址的符号，这也是链接器真正所做的工作。
* text section 的代码中正确引用了全局符号的地址，而不像先前仅仅用 0 填充。
* 非必要的全局符号的重定位项都已被移除。注意最终生成的可执行文件中出于需要进行动态链接等目的，仍然可能存在一些重定位信息，但是在这个简单的例子中不存在。

到现在为止，这个例子一直在使用 RISC-V 的默认代码模型 `medlow` 。为了更具体地解释什么是代码模型，我们可以将其与其他代码模型进行对比，如 `medany` 。我们可以通过下面的例子来观察两种代码模型的差异：
```
0000000000000000 main:
   0:   00000797                auipc   a5,0x0
                        0: R_RISCV_PCREL_HI20   global_symbol
                        0: R_RISCV_RELAX        *ABS*
   4:   0007b503                ld      a0,0(a5) # 0 main
                        4: R_RISCV_PCREL_LO12_I .LA0
                        4: R_RISCV_RELAX        *ABS*
   8:   00a03533                snez    a0,a0
   c:   8082                    ret
```

我们发现， `medany` 代码模型会通过 `auipc` / `ld` 两条指令来引用全局符号，在这种情况下，代码可以链接到任意地址；相比之下， `medlow` 通过 `lui` / `ld` 两条指令来引用全局符号，在这种情况下，代码只能在地址 `0x0` 附近进行链接。相同之处在于，两种代码模型在引用符号时都使用 32 位有符号偏移量，因此生成的代码都只能在 2 GiB 范围内链接。

## `-mcmodel=medlow` 是什么？

`medlow` 即 medium-low 代码模型，程序及其静态定义的符号必须位于 2 GiB 地址范围内，并且必须位于绝对地址 -2 GiB 和 +2 GiB 之间。该模型使用 `lui` / `addi` 指令对全局符号进行寻址，编译器生成 `R_RISCV_HI20` / `R_RISCV_LO12_I` 重定位。下面是使用 `medlow` 代码模型生成代码的一个例子：
```
$ cat cmodel.c
long global_symbol[2];

int main() {
  return global_symbol[0] != 0;
}

$ riscv64-unknown-linux-gnu-gcc cmodel.c -o cmodel -O3 --save-temps -mcmodel=medlow

$ cat cmodel.s
main:
        lui     a5,%hi(global_symbol)
        ld      a0,%lo(global_symbol)(a5)
        snez    a0,a0
        ret

$ riscv64-unknown-linux-gnu-objdump -d -r cmodel.o
cmodel.o:     file format elf64-littleriscv

Disassembly of section .text.startup:

0000000000000000 main:
   0:   000007b7                lui     a5,0x0
                        0: R_RISCV_HI20 global_symbol
                        0: R_RISCV_RELAX        *ABS*
   4:   0007b503                ld      a0,0(a5) # 0 main
                        4: R_RISCV_LO12_I       global_symbol
                        4: R_RISCV_RELAX        *ABS*
   8:   00a03533                snez    a0,a0
   c:   8082                    ret

$ riscv64-unknown-linux-gnu-objdump -d -r cmodel
Disassembly of section .text:

0000000000010330 main:
   10330:       67c9                    lui     a5,0x12
   10332:       0387b503                ld      a0,56(a5) # 12038 global_symbol
   10336:       00a03533                snez    a0,a0
   1033a:       8082                    ret
```

## `-mcmodel=medany` 是什么？

`medany` 即 medium-any 代码模型，程序及其静态定义的符号必须位于 2 GiB 地址范围内。该模型使用 `auipc` / `addi` 指令对全局符号进行寻址，编译器生成 `R_RISCV_PCREL_HI20` / `R_RISCV_PCREL_LO12_I` 重定位。下面是使用 `medany` 代码模型生成代码的一个例子（为了和先前 `medlow` 的例子可以更清晰地显示出区别，此处加上了 `-mexplicit-relocs` 参数）：
```
$ cat cmodel.c
long global_symbol[2];

int main() {
  return global_symbol[0] != 0;
}

$ riscv64-unknown-linux-gnu-gcc cmodel.c -o cmodel -O3 --save-temps -mcmodel=medany -mexplicit-relocs

$ cat cmodel.s
main:
        .LA0: auipc     a5,%pcrel_hi(global_symbol)
        ld      a0,%pcrel_lo(.LA0)(a5)
        snez    a0,a0
        ret

$ riscv64-unknown-linux-gnu-objdump -d -r cmodel.o
cmodel.o:     file format elf64-littleriscv

SYMBOL TABLE:
0000000000000000 l    df *ABS*  0000000000000000 cmodel.c
0000000000000000 l    d  .text  0000000000000000 .text
0000000000000000 l    d  .data  0000000000000000 .data
0000000000000000 l    d  .bss   0000000000000000 .bss
0000000000000000 l    d  .text.startup  0000000000000000 .text.startup
0000000000000000 l       .text.startup  0000000000000000 .LA0
0000000000000000 l    d  .comment       0000000000000000 .comment
0000000000000000 g     F .text.startup  000000000000000e main
0000000000000010       O *COM*  0000000000000008 global_symbol

Disassembly of section .text.startup:

0000000000000000 main:
   0:   00000797                auipc   a5,0x0
                        0: R_RISCV_PCREL_HI20   global_symbol
                        0: R_RISCV_RELAX        *ABS*
   4:   0007b503                ld      a0,0(a5) # 0 main
                        4: R_RISCV_PCREL_LO12_I .LA0
                        4: R_RISCV_RELAX        *ABS*
   8:   00a03533                snez    a0,a0
   c:   8082                    ret

$ riscv64-unknown-linux-gnu-objdump -d -r cmodel.o
Disassembly of section .text:

0000000000010330 main:
   10330:       00002797                auipc   a5,0x2
   10334:       d087b503                ld      a0,-760(a5) # 12038 global_symbol
   10338:       00a03533                snez    a0,a0
   1033c:       8082                    ret
        ...
```

需要注意的是，使用 `-mcmodel=medany` 时会默认使用 `-mno-explicit-relocs` 参数，会对性能产生影响。我们会在之后的文章中讨论这一性能上的差异。

## 代码模型与 ABI 之间的区别

人们经常分不清代码模型和 ABI 之间的区别。其不同之处在于， ABI 确定函数之间的接口，而代码模型确定如何在函数内生成代码。具体来说：两种 RISC-V 代码模型都限制使用 32 位偏移量对符号进行寻址，但在 RV64I 的系统上，地址的指针还是 64 位长的。

也就是说，用 `-mcmodel=medany` 编译的函数与用 `-mcmodel=medlow` 编译的函数可以互相调用。为了方便链接最终的可执行文件，函数需要同时满足这两种模式下符号寻址的约束，不过一般来说这种约束都可以满足。由于代码模型不影响内存布局或参数在函数之间传递的方式，因此程序员其实通常来说也无需关心具体的代码模型。

相比之下，两段使用了不同 ABI 的代码就无法被链接。假设我们现在有一个函数，接受一个 `double` 类型的参数，为 `lp64d` 编译的函数会认为这个参数会存储在浮点寄存器中。但是如果有另一个为 `lp64` 编译的函数需要调用这个函数，就会把参数放在整数寄存器中，也就无法正确链接了。

## 代码模型和链接器松弛

到目前为止，我们还没有讨论代码模型会如何影响链接器松弛，但其实很简单：只需要使用 RISC-V 版本的工具链就可以正确编译并链接代码了。

链接器松弛实际上是一个很重要的优化，很大程度上影响了 RISC-V 指令集的设计：链接器松弛使得 RISC-V 可以放弃一部分复杂的寻址模式，且不会显著影响许多用户代码的性能。RISC-V 可以使用以下寻址模式：
* 从 0（或从 `__global_pointer$` ）开始的 7 位偏移量内的符号：2 个字节。
* 从 0（或从 `__global_pointer$` ）开始的 12 位偏移量内的符号：4 个字节。
* 从 0 开始的 17 位偏移量内的符号：6 个字节。
* 从 0 开始的 32 位偏移量内的符号：8 个字节。对 RV32I 而言就是整个地址空间。

这四种模式都可以通过同一种代码模型 + 八种指令格式（U、I、S、CI、CR、CIW、CL 和 CS）的同一种硬件寻址模式（寄存器 + 偏移量）来实现，也可以将其视作一种可以由硬件支持的、可变长度的地址编码（具体可以参考 "Compressed Macro-Op Fusion" 一文）。由于这种压缩都是由链接器实现的，我们只需要一种代码模型即可。与之相对的是，对 ARMv8 GCC 而言，则需要对生成的每一个地址都要选择不同的代码模型。

实现可变长度寻址通常是复杂指令集处理器的特性，在硬件中实现多种寻址模式，并在汇编阶段尽可能压缩寻址指令。相比之下，RISC-V的做法是采用了可融合的多条指令序列进行寻址，并允许链接器对其进行松弛，其优势在于在代码体积接近的情况下，也可以实现很简单的设计。
