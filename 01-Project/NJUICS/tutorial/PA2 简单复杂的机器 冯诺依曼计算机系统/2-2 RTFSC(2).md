# RTFM

我们在上一小节中已经在概念上介绍了一条指令具体如何执行, 其中有的概念甚至显而易见得难以展开. 但当我们决定往TRM中添加各种高效指令的同时, 也意味着我们无法回避繁琐的细节.

首先你需要了解指令确切的行为, 为此, 你需要阅读生存手册中指令集相关的章节. 具体地, 无论你选择何种ISA, 相应手册中一般都会有以下内容, 尝试RTFM并寻找这些内容的位置:

-   每一条指令具体行为的描述
-   指令opcode的编码表格

特别地, 由于x86的指令集的复杂性, 我们为选择x86的同学提供了[一个简单的阅读教程](https://nju-projectn.github.io/ics-pa-gitbook/ics2022/i386-intro.md).

>[!cloud] RISC - 与CISC平行的另一个世界
>你是否觉得x86指令集的格式特别复杂? 这其实是CISC的一个特性, 不惜使用复杂的指令格式, 牺牲硬件的开发成本, 也要使得一条指令可以多做事情, 从而提高代码的密度, 减小程序的大小. 随着时代的发展, 架构师发现CISC中复杂的控制逻辑不利于提高处理器的性能, 于是RISC应运而生. RISC的宗旨就是简单, 指令少, 指令长度固定, 指令格式统一, 这和KISS法则有异曲同工之妙. [这里](http://cs.stanford.edu/people/eroberts/courses/soco/projects/risc/risccisc)有一篇对比RISC和CISC的小短文.
>
>另外值得推荐的是[这篇文章](http://blog.sciencenet.cn/blog-414166-763326.html), 里面讲述了一个从RISC世界诞生, 到与CISC世界融为一体的故事, 体会一下RISC的诞生对计算机体系结构发展的里程碑意义. 

如果你非常幸运地选择了riscv32, 你会发现目前只需要阅读很少部分的手册内容就可以了: 在PA中, riscv32的客户程序只会由RV32I和RV32M两类指令组成. 这得益于RISC-V指令集的设计理念 - 模块化.

>[!cloud] RISC-V - 一款设计精巧的指令集
>RISC-V是一款非常年轻的指令集 - 第一版RISC-V是在2011年5月由UC Berkeley的研究团队提出的, 至今已经风靡全球. 开放性是RISC-V的最大卖点, 就连ARM和MIPS也都为之震撼, 甚至还因竞争关系而互撕... [这篇文章](http://blog.sciencenet.cn/blog-414166-1089206.html)叙述了RISC-V的理念以及成长的一些历史.
>
>当然, 这些和处于教学领域的PA其实没太大关系. 关键是
>
>-   RISC-V真的很简单.
>-   简单之余, 还有非常多对程序运行深思熟虑的考量. 如果你阅读 RISC-V 的手册, 你就会发现里面阐述了非常多设计的推敲和取舍. 另外 David Patterson 教授 (因推广 RISC 而获得 2018 年的图灵奖, 可谓体系结构领域的一代宗师) 还为 RISC-V 的推广编写了一本入门书籍 [The RISC-V Reader](http://www.riscvbook.com/), 书中从系统的角度叙述了 RISC-V 大量的设计原则, 并与现有指令集进行对比, 非常值得一读. 中科院计算所的三位研究生为这本书编写了[中文翻译的版本](http://riscvbook.com/chinese/RISC-V-Reader-Chinese-v2p1.pdf) (其中一位也算是你们的直系师兄了), 不过由于这本书并没有跟进 RISC-V 官方手册的最新内容, 而且书中有不少笔误 (欢迎到[相应的github repo](https://github.com/Lingrui98/RISC-V-book) 中提 issue), 我们还是建议你阅读 RISC-V 的官方手册.

# RTFSC(2)

理解了上一小节的YEMU如何执行指令之后, 你就会对模拟器的框架有一个基本的认识了. NEMU要模拟一个真实的ISA, 因此代码要比YEMU复杂得多, 但其中蕴含的基本原理是和YEMU相同的. 下面我们来介绍NEMU的框架代码如何实现指令的执行.

在RTFSC的过程中, 你会遇到用于抽象ISA差异的大部分API, 因此我们建议你先阅读[这个页面](https://nju-projectn.github.io/ics-pa-gitbook/ics2022/nemu-isa-api.html)来对这些API的功能进行基本的了解, 将来在代码中遇到它们的时候可以进行查阅.

我们在PA1中提到:

> `cpu_exec()` 又会调用 `execute()` , 后者模拟了CPU的工作方式: 不断执行指令. 具体地, 代码将在一个for循环中不断调用 `exec_once()` 函数, 这个函数的功能就是我们在上一小节中介绍的内容: 让CPU执行当前PC指向的一条指令, 然后更新PC.

具体地,  `exec_once()` 接受一个 `Decode` 类型的结构体指针 `s` , 这个结构体用于存放在执行一条指令过程中所需的信息, 包括指令的PC, 下一条指令的PC等. 还有一些信息是ISA相关的, NEMU用一个结构类型 `ISADecodeInfo` 来对这些信息进行抽象, 具体的定义在 `nemu/src/isa/$ISA/include/isa-def.h` 中.  `exec_once()` 会先把当前的PC保存到 `s` 的成员 `pc` 和 `snpc` 中, 其中 `s->pc` 就是当前指令的PC, 而 `s->snpc` 则是下一条指令的PC, 这里的 `snpc` 是"static next PC"的意思.

然后代码会调用 `isa_exec_once()` 函数 (在 `nemu/src/isa/$ISA/inst.c` 中定义), 这是因为执行指令的具体过程是和 ISA 相关的, 在这里我们先不深究 `isa_exec_once()` 的细节. 但可以说明的是, 它会随着取指的过程修改 `s->snpc` 的值, 使得从 `isa_exec_once()` 返回后 `s->snpc` 正好为下一条指令的 PC. 接下来代码将会通过 `s->dnpc` 来更新 PC, 这里的 `dnpc` 是"dynamic next PC"的意思. 关于 `snpc` 和 `dnpc` 的区别, 我们会在下文进行说明.

忽略 `exec_once()` 中剩下与 trace 相关的代码, 我们就返回到 `execute()` 中. 代码会对一个用于记录客户指令的计数器加 1, 然后进行一些 trace 和 difftest 相关的操作 (此时先忽略), 然后检查 NEMU 的状态是否为 `NEMU_RUNNING`, 若是, 则继续执行下一条指令, 否则则退出执行指令的循环.

事实上, `exec_once()`函数覆盖了指令周期的所有阶段: 取指, 译码, 执行, 更新PC, 接下来我们来看看NEMU是如何实现指令周期的每一个阶段的.

## 取指(instruction fetch, IF)

`isa_exec_once()` 做的第一件事情就是取指令. 在 NEMU 中, 有一个函数 `inst_fetch()` (在 `nemu/include/cpu/ifetch.h` 中定义) 专门负责取指令的工作. `inst_fetch()` 最终会根据参数 `len` 来调用 `vaddr_ifetch()` (在 `nemu/src/memory/vaddr.c` 中定义), 而目前 `vaddr_ifetch()` 又会通过 `paddr_read()` 来访问物理内存中的内容. 因此, 取指操作的本质只不过就是一次内存的访问而已.

`isa_exec_once()` 在调用 `inst_fetch()` 的时候传入了 `s->snpc` 的地址, 因此 `inst_fetch()` 最后还会根据 `len` 来更新 `s->snpc`, 从而让 `s->snpc` 指向下一条指令.

## 译码(instruction decode, ID)

接下来代码会进入 `decode_exec()` 函数, 它首先进行的是译码相关的操作. 译码的目的是得到指令的操作和操作对象, 这主要是通过查看指令的 `opcode` 来决定的. 不同 ISA 的 `opcode` 会出现在指令的不同位置, 我们只需要根据指令的编码格式, 从取出的指令中识别出相应的 `opcode` 即可.

和 YEMU 相比, NEMU 使用一种抽象层次更高的译码方式: 模式匹配, NEMU 可以通过一个模式字符串来指定指令中 `opcode`, 例如在 riscv32 中有如下模式:

```c
INSTPAT_START();
INSTPAT("??????? ????? ????? ??? ????? 01101 11", lui, U, R(dest) = imm);
// ...
INSTPAT_END();
```

其中`INSTPAT`(意思是instruction pattern)是一个宏(在`nemu/include/cpu/decode.h`中定义), 它用于定义一条模式匹配规则. 其格式如下:

```
INSTPAT(模式字符串, 指令名称, 指令类型, 指令执行操作);
```

`模式字符串`中只允许出现4种字符:

-   `0` 表示相应的位只能匹配 `0`
-   `1` 表示相应的位只能匹配 `1`
-   `?` 表示相应的位可以匹配 `0` 或 `1`
-   空格是分隔符, 只用于提升模式字符串的可读性, 不参与匹配

`指令名称`在代码中仅当注释使用, 不参与宏展开; `指令类型`用于后续译码过程; 而`指令执行操作`则是通过C代码来模拟指令执行的真正行为.

此外, `nemu/include/cpu/decode.h` 中还定义了宏 `INSTPAT_START` 和 `INSTPAT_END`. `INSTPAT` 又使用了另外两个宏 `INSTPAT_INST` 和 `INSTPAT_MATCH`, 它们在 `nemu/src/isa/$ISA/inst.c` 中定义. 对上述代码进行宏展开并简单整理代码之后, 最后将会得到:

```c
{ const void ** __instpat_end = &&__instpat_end_;
do {
  uint32_t key, mask, shift;
  pattern_decode("??????? ????? ????? ??? ????? 01101 11", 38, &key, &mask, &shift);
  if (((s->isa.inst.val >> shift) & mask) == key) {
    {
      decode_operand(s, &dest, &src1, &src2, &imm, TYPE_U);
      R(dest) = imm;
    }
    goto *(__instpat_end);
  }
} while (0);
// ...
__instpat_end_: ; }
```

上述代码中的 `&&__instpat_end_` 使用了 GCC 提供的[标签地址](https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html)扩展功能, `goto` 语句将会跳转到最后的 `__instpat_end_` 标签. 此外, `pattern_decode()` 函数在 `nemu/include/cpu/decode.h` 中定义, 它用于将模式字符串转换成 3 个整型变量.

`pattern_decode()` 函数将模式字符串中的 `0` 和 `1` 抽取到整型变量 `key` 中, `mask` 表示 `key` 的掩码, 而 `shift` 则表示 `opcode` 距离最低位的比特数量, 用于帮助编译器进行优化. 具体地, 上述例子中:

```c
key   = 0x37;
mask  = 0x7f;
shift = 0;
```

考虑PA1中介绍的内建客户程序中的如下指令:

```
0x800002b7   lui t0,0x80000
```

NEMU 取指令的时候会把指令记录到 `s->isa.inst.val` 中, 此时指令满足上述宏展开的 `if` 语句, 表示匹配到 `lui` 指令的编码, 因此将会进行进一步的译码操作.

刚才我们只知道了指令的具体操作 (比如 `lui` 是读入一个立即数到寄存器的高位), 但我们还是不知道操作对象 (比如立即数是多少, 读入到哪个寄存器). 为了解决这个问题, 代码需要进行进一步的译码工作, 这是通过调用 `decode_operand()` 函数来完成的. 这个函数将会根据传入的指令类型 `type` 来进行操作数的译码, 译码结果将记录到函数参数 `dest`, `src1`, `src2` 和 `imm` 中, 它们分别代表目的操作数, 两个源操作数和立即数.

我们会发现, 类似寄存器和立即数这些操作数, 其实是非常常见的操作数类型. 为了进一步实现操作数译码和指令译码的解耦, 我们对这些操作数的译码进行了抽象封装:

-   框架代码定义了 `src1R()` 和 `src2R()` 两个辅助宏, 用于寄存器的读取结果记录到相应的操作数变量中
-   框架代码还定义了 `immI` 等辅助宏, 用于从指令中抽取出立即数

有了这些辅助宏, 我们就可以用它们来方便地编写 `decode_operand()` 了, 例如 riscv 中 I-型指令的译码过程可以通过如下代码实现:

```c
case TYPE_I: src1R(); immI(); break;
```

另外补充几点说明:

-   `decode_operand` 中用到了宏 `BITS` 和 `SEXT`, 它们均在 `nemu/include/macro.h` 中定义, 分别用于位抽取和符号扩展
-   `decode_operand` 会首先统一对目标操作数进行寄存器操作数的译码, 即调用 `*dest = rd`, 不同的指令类型可以视情况使用 `dest`
-   在模式匹配过程的最后有一条 `inv` 的规则, 表示"若前面所有的模式匹配规则都无法成功匹配, 则将该指令视为非法指令

>[!cloud] x86的变长指令
>由于CISC指令变长的特性, x86指令长度和指令形式需要一边取指一边译码来确定, 而不像RISC指令集那样可以泾渭分明地处理取指和译码阶段, 因此你会在x86的译码过程中看到`inst_fetch()`的操作.

>[!sq] 立即数背后的故事
>框架代码通过`inst_fetch()`函数进行取指, 别看这里就这么一行代码, 其实背后隐藏着针对[字节序](http://en.wikipedia.org/wiki/Endianness)的慎重考虑. 大部分同学的主机都是x86小端机, 当你使用高级语言或者汇编语言写了一个32位常数`0x1234`的时候, 在生成的二进制代码中, 这个常数对应的字节序列如下(假设这个常数在内存中的起始地址是x):
>
>```
x   x+1  x+2  x+3
+----+----+----+----+
| 34 | 12 | 00 | 00 |
+----+----+----+----+
>```
>
>而大多数PC机都是小端架构(我们相信没有同学会使用IBM大型机来做PA), 当NEMU运行的时候,
>
>```
imm = inst_fetch(pc, 4);
>```
>
>这行代码会将`34 12 00 00`这个字节序列原封不动地从内存读入`imm`变量中, 主机的CPU会按照小端方式来解释这一字节序列, 于是会得到`0x1234`, 符合我们的预期结果.
>
>Motorola 68k系列的处理器都是大端架构的. 现在问题来了, 考虑以下两种情况:
>
>-   假设我们需要将NEMU运行在Motorola 68k的机器上(把NEMU的源代码编译成Motorola 68k的机器码)
>-   假设我们需要把Motorola 68k作为一个新的ISA加入到NEMU中
>
>在这两种情况下, 你需要注意些什么问题? 为什么会产生这些问题? 怎么解决它们?
>
>事实上不仅仅是立即数的访问, 长度大于1字节的内存访问都需要考虑类似的问题. 我们在这里把问题统一抛出来, 以后就不再单独讨论了.

>[!sq] 立即数背后的故事(2)
>mips32和riscv32的指令长度只有32位, 因此它们不能像x86那样, 把C代码中的32位常数直接编码到一条指令中. 思考一下, mips32和riscv32应该如何解决这个问题?

>[!idea]+ 我要被宏定义绕晕了, 怎么办?
>为了理解一个宏的语义, 你可能会尝试手动对它进行宏展开, 但你可能会碰到如下困难:
>
>-   宏嵌套的次数越多, 理解越困难
>-   一些拼接宏会影响编辑器的代码跳转功能
>
>事实上, 为了进行宏展开, 你并不需要手动去进行操作, 因为肯定有工具能做这件事: 我们只需要让GCC把编译预处理的结果输出出来, 就可以看到宏展开的结果了. 有了宏展开的结果, 你就可以快速理解展开之后的语义, 然后反过来理解相应的宏是如何一步步被展开的了.
>
>当然, 最方便的做法是让GCC编译NEMU的时候顺便输出预处理的结果, 如果你对Makefile的组织有一定的认识, 这件事当然也难不倒你了.

## 执行(execute, EX)

译码阶段结束之后, 代码将会执行模式匹配规则中指定的`指令执行操作`, 这部分操作会用到译码的结果, 并通过C代码来模拟指令执行的真正行为. 例如对于`lui`指令, 由于译码阶段已经把U-型立即数记录到操作数`imm`中了, 我们只需要通过`R(dest) = imm`将这个立即数存放到目标寄存器中, 这样就完成了指令的执行.

指令执行的阶段结束之后, `decode_exec()`函数将会返回`0`, 并一路返回到`exec_once()`函数中. 不过目前代码并没有使用这个返回值, 因此可以忽略它.

## 更新PC

最后是更新PC. 更新PC的操作非常简单, 只需要把`s->dnpc`赋值给`cpu.pc`即可. 我们之前提到了`snpc`和`dnpc`, 现在来说明一下它们的区别.

>[!idea] 静态指令和动态指令
>在程序分析领域中, 静态指令是指程序代码中的指令, 动态指令是指程序运行过程中的指令. 例如对于以下指令序列
>
>```
100: jmp 102
101: add
102: xor
>```
>
>`jmp`指令的下一条静态指令是`add`指令, 而下一条动态指令则是`xor`指令.

有了静态指令和动态指令这两个概念之后, 我们就可以说明 `snpc` 和 `dnpc` 的区别了: `snpc` 是指代码中的下一条指令, 而 `dnpc` 是指程序运行过程中的下一条指令. 对于顺序执行的指令, 它们的 `snpc` 和 `dnpc` 是一样的; 但对于跳转指令, `snpc` 和 `dnpc` 就会有所不同, `dnpc` 应该指向跳转目标的指令. 显然, 我们应该使用 `s->dnpc` 来更新 PC, 并且在指令执行的过程中正确地维护 `s->dnpc`.

---

上文已经把一条指令在NEMU中执行的流程进行了大概的介绍, 但还有少量的细节没有完全覆盖(例如x86的指令组译码表), 这些细节就交给你来去尝试理解啦. 不过为了特别照顾选择x86的同学, 我们还是准备了[一个例子](https://nju-projectn.github.io/ics-pa-gitbook/ics2022/exec.md)来RTFSC.

>[!notice] 驾驭项目, 而不是被项目驾驭
>你和一个项目的关系会经历4个阶段:
>
>1.  被驾驭: 你对它一无所知
>2.  一知半解: 你对其中的主要模块和功能有了基本的了解
>3.  驾轻就熟: 你对整个项目的细节都了如指掌
>4.  为你所用: 你可以随心所欲地在项目中添加你认为有用的功能
>
>在PA中, 达到第二个阶段的主要手段是阅读讲义和代码, 达到第三个阶段的主要手段是独立完成实验内容和独立调试. 至于要达到第四个阶段, 就要靠你的主观能动性了: 代码还有哪里做得不够好? 怎么样才算是够好? 应该怎么做才能达到这个目标?
>
>你毕业后到了工业界或学术界, 就会发现真实的项目也都是这样:
>
>1.  刚接触一个新项目, 不知道如何下手
>2.  RTFM, RTFSC, 大致明白项目组织结构和基本的工作流程
>3.  运行项目的时候发现有非预期行为(可能是配置错误或环境错误, 可能是和已有项目对接出错, 也可能是项目自身的bug), 然后调试. 在调试过程中, 对这些模块的理解会逐渐变得清晰.
>4.  哪天需要你在项目中添加一个新功能, 你会发现自己其实可以胜任.
>
>这说明了: <font color="#ff0000">如果你一遇到bug就找大神帮你调试, 你失去的机会和能力会比你想象的多得多.</font>

## 结构化程序设计

我们刚才介绍了译码过程中的一些辅助用的函数和宏, 它们的引入都是为了实现代码的解偶, 提升可维护性. 如果指令集越复杂, 指令之间的共性特征就越多, 以x86为例:

-   对于同一条指令的不同形式, 它们的执行阶段是相同的. 例如`add_I2E`和`add_E2G`等, 它们的执行阶段都是把两个操作数相加, 把结果存入目的操作数.
-   对于不同指令的同一种形式, 它们的译码阶段是相同的. 例如`add_I2E`和`sub_I2E`等, 它们的译码阶段都是识别出一个立即数和一个`E`操作数.
-   对于同一条指令同一种形式的不同操作数宽度, 它们的译码阶段和执行阶段都是非常类似的. 例如`add_I2E_b`, `add_I2E_w`和`add_I2E_l`, 它们都是识别出一个立即数和一个`E`操作数, 然后把相加的结果存入`E`操作数.

这意味着, 如果独立实现每条指令不同形式不同操作数宽度的译码和执行过程, 将会引入大量重复的代码. 需要修改的时候, 所有相关代码都要分别修改, 遗漏了某一处就会造成bug, 工程维护的难度急速上升.

>[!cloud] 来体会一下
>过去有同学通过如下代码实现`isa_reg_str2val()`函数:
>
>```c
>if (strcmp(s, "$0") == 0)
>  return cpu.gpr[0]._64;
>else if (strcmp(s, "ra") == 0)
>  return cpu.gpr[1]._64;
>else if (strcmp(s, "sp") == 0)
>  return cpu.gpr[2]._64;
>else if (strcmp(s, "gp") == 0)
>  return cpu.gpr[3]._64;
>else if (strcmp(s, "tp") == 0)
>  return cpu.gpr[4]._64;
>else if (strcmp(s, "t0") == 0)
>  return cpu.gpr[5]._64;
>else if (strcmp(s, "t1") == 0)
>  return cpu.gpr[6]._64;
>else if (strcmp(s, "s2") == 0)
>  return cpu.gpr[7]._64;
>else if (strcmp(s, "s0") == 0)
>  return cpu.gpr[8]._64;
>else if (strcmp(s, "s1") == 0)
>  return cpu.gpr[9]._64;
>else if (strcmp(s, "a0") == 0)
>  return cpu.gpr[10]._64;
>else if (strcmp(s, "a1") == 0)
>  return cpu.gpr[11]._64;
>else if (strcmp(s, "a2") == 0)
>  return cpu.gpr[12]._64;
>else if (strcmp(s, "a3") == 0)
>  return cpu.gpr[13]._64;
>else if (strcmp(s, "a4") == 0)
>  return cpu.gpr[14]._64;
>else if (strcmp(s, "a5") == 0)
>  return cpu.gpr[15]._64;
>else if (strcmp(s, "a6") == 0)
>  return cpu.gpr[16]._64;
>else if (strcmp(s, "a7") == 0)
>  return cpu.gpr[17]._64;
>else if (strcmp(s, "s2") == 0)
>  return cpu.gpr[18]._64;
>else if (strcmp(s, "s3") == 0)
>  return cpu.gpr[19]._64;
>else if (strcmp(s, "s4") == 0)
>  return cpu.gpr[20]._64;
>else if (strcmp(s, "s5") == 0)
>  return cpu.gpr[21]._64;
>else if (strcmp(s, "s6") == 0)
>  return cpu.gpr[22]._64;
>else if (strcmp(s, "s7") == 0)
>  return cpu.gpr[23]._64;
>else if (strcmp(s, "s8") == 0)
>  return cpu.gpr[24]._64;
>else if (strcmp(s, "s8") == 0)
>  return cpu.gpr[25]._64;
>else if (strcmp(s, "s10") == 0)
>  return cpu.gpr[26]._64;
>else if (strcmp(s, "t2") == 0)
>  return cpu.gpr[27]._64;
>else if (strcmp(s, "t3") == 0)
>  return cpu.gpr[28]._64;
>else if (strcmp(s, "t4") == 0)
>  return cpu.gpr[29]._64;
>else if (strcmp(s, "t5") == 0)
>  return cpu.gpr[30]._64;
>else if (strcmp(s, "t5") == 0)
>  return cpu.gpr[31]._64;
>```
>
>你应该能想象到这位同学是如何编写上述代码的. 现在问题来了, 你能快速检查上述代码是否正确吗?
>
>更多地, 如果你的项目中有很多这样的代码, 你还愿意仔细地读一读它们吗?

>[!notice] Copy-Paste - 一种糟糕的编程习惯
>事实上, 第一版PA发布的时候, 框架代码就恰恰是引导大家独立实现每一条指令的译码和执行过程. 大家在实现指令的时候, 都是把已有的代码复制好几份, 然后进行一些微小的改动(例如把`<<`改成`>>`). 当你发现这些代码有bug的时候, 噩梦才刚刚开始. 也许花了好几天你又调出一个bug的时候, 才会想起这个bug你好像之前在哪里调过. 你也知道代码里面还有类似的bug, 但你已经分辨不出哪些代码是什么时候从哪个地方复制过来的了. 由于当年的框架代码没有足够重视编程风格, 导致学生深深地陷入调试的泥淖中, 这也算是PA的一段黑历史了.
>
>这种糟糕的编程习惯叫Copy-Paste, 经过上面的分析, 相信你也已经领略到它的可怕了. 事实上, [周源源教授](https://cseweb.ucsd.edu/~yyzhou/)的团队在2004年就设计了一款工具CP-Miner, 来自动检测操作系统代码中由于Copy-Paste造成的bug. 这个工具还让周源源教授收获了一篇[系统方向顶级会议OSDI的论文](http://pages.cs.wisc.edu/~shanlu/paper/OSDI04-CPMiner.pdf), 这也是她当时所在学校UIUC史上的第一篇系统方向的顶级会议论文.
>
>后来周源源教授发现, 相比于操作系统, 应用程序的源代码中Copy-Paste的现象更加普遍. 于是她们团队把CP-Miner的技术应用到应用程序的源代码中, 并创办了PatternInsight公司. 很多IT公司纷纷购买PatternInsight的产品, 并要求提供相应的定制服务, 甚至PatternInsight公司最后还被VMWare收购了.
>
>这个故事折射出, 大公司中程序员的编程习惯也许不比你好多少, 他们也会写出Copy-Paste这种难以维护的代码. 但反过来说, 重视编码风格这些企业看中的能力, 你从现在就可以开始培养.

一种好的做法是把译码, 执行和操作数宽度的相关代码分离开来, 实现解耦, 也就是在程序设计课上提到的结构化程序设计. 在框架代码中, 实现译码和执行之间的解耦的是通过 `INSTPAT` 定义的模式匹配规则, 这样我们就可以分别编写译码和执行的内容, 然后来进行组合了: 这样的设计可以很容易实现执行行为相同但译码方式不同的多条指令. 对于x86, 实现操作数宽度和译码, 执行这两者之间的解耦的是 `ISADecodeInfo` 结构体中的 `width` 成员, 它们记录了操作数宽度, 译码和执行的过程中会根据它们进行不同的操作, 通过同一份译码和执行的代码实现不同操作数宽度的功能.

>[!edit] RTFSC理解指令执行的过程
>这一小节的细节非常多, 你可能需要多次阅读讲义和代码才能理解每一处细节. 根据往届学长学姐的反馈, 一种有效的理解方法是通过做笔记的方式来整理这些细节. 事实上, 配合GDB食用效果更佳.
>
>为了避免你长时间对代码的理解没有任何进展, 我们就增加一道必答题吧:
>
>> 请整理一条指令在NEMU中的执行过程.
>
>除了 `nemu/src/device` 和 `nemu/src/isa/$ISA/system` 之外, NEMU的其它代码你都已经有能力理解了. 因此不要觉得讲义中没有提到的文件就不需要看, 尝试尽可能地理解每一处细节吧! 在你遇到bug的时候, 这些细节就会成为帮助你调试的线索.

# 运行第一个C程序

说了这么多, 现在到了动手实践的时候了. 首先克隆一个新的子项目`am-kernels`(你可能已经在PA1中克隆这个子项目了), 里面包含了一些测试程序:

```bash
cd ics2022
bash init.sh am-kernels
```

你在PA2的第一个任务, 就是实现若干条指令, 使得第一个简单的C程序可以在NEMU中运行起来. 这个简单的C程序是 `am-kernels/tests/cpu-tests/tests/dummy.c` , 它什么都不做就直接返回了.

>[!edit] 准备交叉编译环境
>如果你选择的ISA不是x86, 你还需要准备相应的gcc和binutils, 才能正确地进行编译.
>
>-   mips32
> 	   -   `apt-get install g++-mips-linux-gnu binutils-mips-linux-gnu`
>-   riscv32(64)
> 	   -   `apt-get install g++-riscv64-linux-gnu binutils-riscv64-linux-gnu`

在 `am-kernels/tests/cpu-tests/` 目录下键入

```bash
make ARCH=$ISA-nemu ALL=dummy run
```

编译`dummy`程序, 并启动NEMU运行它.

>[!idea] 修复riscv32编译错误
>如果你选择的是riscv32, 并在编译`dummy`程序时报告了如下错误:
>
>```
>/usr/riscv64-linux-gnu/include/bits/wordsize.h:28:3: error: #error "rv32i-based targets are not supported"
>```
>
>则需要使用sudo权限修改以下文件:
>
>```diff
>--- /usr/riscv64-linux-gnu/include/bits/wordsize.h
>+++ /usr/riscv64-linux-gnu/include/bits/wordsize.h
>@@ -25,5 +25,5 @@
>  #if __riscv_xlen == 64
>  # define __WORDSIZE_TIME64_COMPAT32 1
>  #else
>-# error "rv32i-based targets are not supported"
>+# define __WORDSIZE_TIME64_COMPAT32 0
>  #endif
>```
>
>如果报告的是如下错误:
>
>```
>/usr/riscv64-linux-gnu/include/gnu/stubs.h:8:11: fatal error: gnu/stubs-ilp32.h: No such file or directory
>```
>
>则需要使用sudo权限修改以下文件:
>
>```diff
>--- /usr/riscv64-linux-gnu/include/gnu/stubs.h
>+++ /usr/riscv64-linux-gnu/include/gnu/stubs.h
>@@ -5,5 +5,5 @@
>  #include <bits/wordsize.h>
>
>  #if __WORDSIZE == 32 && defined __riscv_float_abi_soft
>-# include <gnu/stubs-ilp32.h>
>+//# include <gnu/stubs-ilp32.h>
>  #endif
>```

事实上, 并不是每一个程序都可以在NEMU中运行, `abstract-machine`子项目专门用于编译出能在NEMU中运行的程序, 我们在下一小节中会再来介绍它.

在NEMU中运行`dummy`程序, 你会发现NEMU输出以下信息(以riscv32为例):

```
invalid opcode(PC = 0x80000000):
        13 04 00 00 17 91 00 00 ...
        00000413 00009117...
There are two cases which will trigger this unexpected exception:
1. The instruction at PC = 0x80000000 is not implemented.
2. Something is implemented incorrectly.
Find this PC(0x80000000) in the disassembling result to distinguish which case it is.

If it is the first case, see
       _                         __  __                         _ 
      (_)                       |  \/  |                       | |
  _ __ _ ___  ___ ________   __ | \  / | __ _ _ __  _   _  __ _| |
 | '__| / __|/ __|______\ \ / / | |\/| |/ _` | '_ \| | | |/ _` | |
 | |  | \__ \ (__        \ V /  | |  | | (_| | | | | |_| | (_| | |
 |_|  |_|___/\___|        \_/   |_|  |_|\__,_|_| |_|\__,_|\__,_|_|

for more details.

If it is the second case, remember:
* The machine is always right!
* Every line of untested code is always wrong!
```

这是因为你还没有实现`0x00000413`的指令, 因此, 你需要开始在NEMU中添加指令了.

>[!sq] 为什么执行了未实现指令会出现上述报错信息
>RTFSC, 理解执行未实现指令的时候, NEMU具体会怎么做.

要实现哪些指令才能让 `dummy` 在NEMU中运行起来呢? 答案就在其反汇编结果( `am-kernels/tests/cpu-tests/build/dummy-$ISA-nemu.txt` )中: 你只需实现那些目前还没实现的指令就可以了. 框架代码引入的模式匹配规则, 对在NEMU中实现客户指令提供了很大的便利, 为了实现一条新指令, 你只需要在 `nemu/src/isa/$ISA/inst.c` 中添加正确的模式匹配规则即可.

>[!idea] 交叉编译工具链
>如果你选择的 ISA 不是 x86, 在查看客户程序的二进制信息 (如 `objdump`, `readelf` 等)时, 需要使用相应的交叉编译版本, 如 `mips-linux-gnu-objdump`, `riscv64-linux-gnu-readelf` 等. 特别地, 如果你选择的 ISA 是 riscv32, 也可以使用 riscv64为前缀的交叉编译工具链.

这里要再次强调, <font color="#ff0000">你务必通过RTFM来查阅指令的功能, 不能想当然. 手册中给出了指令功能的完整描述(包括做什么事, 怎么做的, 有什么影响), 一定要仔细阅读其中的每一个单词, 对指令功能理解错误和遗漏都会给以后的调试带来巨大的麻烦.</font>

>[!idea] 再提供一些x86的提示吧
>-   `call`: `call`指令有很多形式, 不过在PA中只会用到其中的几种, 现在只需要实现`CALL rel32`的形式就可以了. 至于跳转地址, 框架代码里面已经有不少提示了, 也就算作是RTFSC的一个练习吧.
>-   `push`: 现在只需要实现`PUSH r32`和`PUSH imm32`的形式就可以了
>-   `sub`: 在实现`sub`指令之前, 你首先需实现EFLAGS寄存器. 你只需要在寄存器结构体中添加EFLAGS寄存器即可. EFLAGS是一个32位寄存器, 但在NEMU中, 我们只会用到EFLAGS中以下的5个位: `CF`, `ZF`, `SF`, `IF`, `OF`, 其它位的功能可暂不实现. 关于EFLAGS中每一位的含义, 请查阅i386手册. 实现了EFLAGS寄存器之后, 你就可以实现`sub`指令了
>-   `xor`, `ret`: RTFM吧

>[!edit]+ 运行第一个客户程序
>在NEMU中实现上文提到的指令, 具体细节请务必参考手册. 实现成功后, 在NEMU中运行客户程序`dummy`, 你将会看到`HIT GOOD TRAP`的信息. 如果你没有看到这一信息, 说明你的指令实现不正确, 你可以使用PA1中实现的简易调试器帮助你调试.

# 运行更多的程序

未测试代码永远是错的, 你需要更多的测试用例来测试你的NEMU. 我们在`am-kernels/tests/cpu-tests/`目录下准备了一些简单的测试用例. 在该目录下执行

```bash
make ARCH=$ISA-nemu ALL=xxx run
```

其中`xxx`为测试用例的名称(不包含`.c`后缀).

上述`make run`的命令最终会启动NEMU, 并运行相应的客户程序. 如果你需要使用GDB来调试NEMU运行客户程序的情况, 可以执行以下命令:

```bash
make ARCH=$ISA-nemu ALL=xxx gdb
```

>[!edit] 实现更多的指令
>你需要实现更多的指令, 以通过上述测试用例.
>
>你可以自由选择按照什么顺序来实现指令. 经过 PA1 的训练之后, 你应该不会实现所有指令之后才进行测试了. 要养成尽早做测试的好习惯, 一般原则都是"实现尽可能少的指令来进行下一次的测试". 你不需要实现所有指令的所有形式, 只需要通过这些测试即可. 如果将来仍然遇到了未实现的指令, 就到时候再实现它们.
>
>框架代码已经实现了部分指令, 但可能未编写相应的模式匹配规则. 此外, 部分函数的功能也并没有完全实现好 (框架代码中已经插入了 `TODO()` 作为提示), 你还需要编写相应的功能.
>
>由于 `string` 和 `hello-str` 还需要实现额外的内容才能运行 (具体在后续小节介绍), 目前可以先使用其它测试用例进行测试.

^71dbc4

>[!notice] 不要以为只需要在TODO处写代码
>过去经常有同学认为, "我只需要在出现 `TODO` 的地方写代码就可以了, 如果一个功能在框架代码中没有相应的 `TODO`, 它就是超出必做内容的范围, 我不需要实现."
>
><font color=" #ff0000 ">在 PA 中, 这种想法是错误的</font>. 如果你 RTFSC, 你会发现 `TODO()` 只是个宏, 展开之后会调用 `panic()`. 因此框架代码中的 `TODO` 更多地是在 NEMU 运行的时候给出可读性更好的结果 (如 xxx 未实现), 而不是让 NEMU 触发让你畏惧的段错误.
>
>你毕业后进入公司/课题组, 不会再有讲义具体地告诉你应该做什么, 总有一天你需要在脱离讲义的情况下完成任务. 我们希望你现在就放弃"讲义和框架代码会把我应该做的一切细节清楚地告诉我"的幻想, 为自己承担起"理解整个系统工作原理"的责任, 而不是成为框架代码的奴仆. 因此, 当你疑惑一个功能是否需要实现时, 你不应该通过框架代码中是否有 `TODO` 来进行判断, 而是应该根据你对代码的理解和当下的需求来做决定.

>[!must]  x86指令相关的注意事项
>-   `push imm8` 指令行为补充. `push imm8` 指令需要对立即数进行符号扩展, 这一点在 i386 手册中并没有明确说明. 在 [IA-32手册](http://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-manual-325462.pdf)中关于 `push` 指令有如下说明:
>> If the source operand is an immediate and its size is less than the operand size, a sign-extended value is pushed on the stack.
>
>-   字符串操作指令. 如 `movsb` 等, 这些指令需要用到段寄存器 `DS`, `ES` 以及 EFLAGS 寄存器中的 `DF` 标志. 在 PA 中无需实现这些寄存器, RTFM 时认为这些寄存器的值恒为 `0` 来理解指令的语义即可.
>-   `endbr32` 指令. 具体见[这里](https://nju-projectn.github.io/ics-pa-gitbook/ics2022/2.2.html#%E5%8E%BB%E9%99%A4endbr32%E6%8C%87%E4%BB%A4)

>[!sq] mips32的分支延迟槽
>为了提升处理器的性能, mips使用了一种叫[分支延迟槽](https://en.wikipedia.org/wiki/Delay_slot)的技术. 采用这种技术之后, 程序的执行顺序会发生一些改变: 我们把紧跟在跳转指令(包括有条件和无条件)之后的静态指令称为延迟槽, 那么程序在执行完跳转指令后, 会先执行延迟槽中的指令, 再执行位于跳转目标的指令. 例如
>
>```
>100: beq 200
>101: add
>102: xor
>...
>200: sub
>201: j   102
>202: slt
>```
>
>若 `beq` 指令的执行结果为跳转, 则相应的动态指令流为 `100 -> 101 -> 200`; 若 `beq` 指令的执行结果为不跳转, 则相应的动态指令流为 `100 -> 101 -> 102`; 而对于 `j` 指令, 相应的动态指令流为 `201 -> 202 -> 102`.
>
>你一定会对这种反直觉的技术如何提升处理器性能而感到疑惑. 不过这需要你先了解一些微结构的知识, 例如[处理器流水线](http://en.wikipedia.org/wiki/Classic_RISC_pipeline), 但这已经超出了 ICS 的课程范围了, 所以我们也不详细解释了, 感兴趣的话可以 STFW.
>
>但我们可以知道, 延迟槽技术需要软硬件协同才能正确工作: mips手册中描述了这一约定, 处理器设计者按照这一约定设计处理器, 而编译器开发者则会让编译器负责在延迟槽中放置一条有意义的指令, 使得无论是否跳转, 按照这一约定的执行顺序都能得到正确的执行结果.
>
>如果你是编译器开发者, 你将会如何寻找合适的指令放到延迟槽中呢?

>[!cloud] mips32-NEMU的分支延迟槽
>既然 mips 有这样的约定, 而编译器也已经遵循这一约定, 那么对于 mips32 编译器生成的程序, 我们也应该遵循这一约定来解释其语义. 这意味着, mips32-NEMU 作为一个模拟的 mips32 CPU, 也需要实现分支延迟槽技术, 才能正确地支撑 mips32 程序的运行.
>
>事实上, gcc 为 mips32 程序的生成提供了一个 `-fno-delayed-branch` 的编译选项, 让 mips32 程序中的延迟槽中都放置 `nop` 指令. 这样以后, 执行跳转指令之后, 接下来就可以直接执行跳转目标的指令了, 因为延迟槽中都是 `nop` 指令, 就算不执行它, 也不会影响程序的正确性.
>
>我们已经在编译 mips32 程序的命令中添加了这一编译选项, 于是我们在实现 mips32-NEMU 的时候就可以进行简化, 无需实现分支延迟槽了.
>
>对PA来说, 去掉延迟槽还有其它的好处, 我们会在后续内容中进行讨论.

>[!sq] 指令名对照
>AT&T格式反汇编结果中的少量指令, 与手册中列出的指令名称不符, 如x86的`cltd`, mips32和riscv32则有不少伪指令(pseudo instruction). 除了STFW之外, 你有办法在手册中找到对应的指令吗? 如果有的话, 为什么这个办法是有效的呢?

>[!abstract] 温馨提示
>PA2阶段1到此结束.