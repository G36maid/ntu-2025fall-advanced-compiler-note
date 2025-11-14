<!-- Slide number: 1 -->
# 淺談編譯器最佳化
Jim Huang (黃敬群) <jserv.tw@gmail.com>
台灣國立成功大學資訊工程系
Oct 17, 2025

### Notes:

<!-- Slide number: 2 -->

![](../course-materials/GoogleShape82p17.jpg)
# 經典的編譯流程
取自《Compilers: Principles, Techniques, and Tools (2/e)》編譯器無所不在，是資訊科技的基石
涵蓋分詞(tokenizer)、語法/語意分析、中間程式碼產生、中間表示式(IR)、與硬體無關的最佳化、針對標的架構的程式碼產生器，和一系列最佳化。

### Notes:

<!-- Slide number: 3 -->

![](../course-materials/GoogleShape87p18.jpg)

### Notes:

<!-- Slide number: 4 -->

![](../course-materials/GoogleShape92p19.jpg)

### Notes:

<!-- Slide number: 5 -->

![](../course-materials/GoogleShape97p20.jpg)
從無到有自幹的shecc，不依賴任何組譯器或連結器，直接輸出32位元Arm和RISC-V處理器架構的機械碼，具備基本最佳化能力，可編譯自身(self-hosting)

https://github.com/sysprog21/shecc

### Notes:

<!-- Slide number: 6 -->
# shecc: 以女子天團命名的開放原始碼編譯器專案
從無到有建構C語言編譯器的前端和後端，顧及最佳化需求，引入額外的中端(middle-end)來銜接前端及後端
Bootstrapping分成三個階段
stage0: 用gcc/clang/MSVC編譯，產生可執行檔out/shecc，針對原生的處理器架構
stage1: 用stage0產生的程式編譯shecc原始程式碼，產生的執行檔out/shecc-stage1.elf是ArmV7或 RISC-V指令集
stage2: 用stage1產生的out/shecc-stage1.elf編譯自己

![](../course-materials/GoogleShape104p21.jpg)

![](../course-materials/GoogleShape107p21.jpg)
Source: https://www.niusnews.com/=P30pi013

### Notes:

<!-- Slide number: 7 -->
# 編譯器最佳化和其陷阱

### Notes:

<!-- Slide number: 8 -->
# 編譯器最佳化的陷阱:
學習程式語言，若停留在語法，卻沒有深入編譯器和相對應的執行環境，往往事倍功半

![](../course-materials/GoogleShape118p23.jpg)
相關討論

### Notes:

<!-- Slide number: 9 -->

![](../course-materials/GoogleShape126p24.jpg)
# 編譯器最佳化起於IR

LLVM IR(Intermediate Representation)
let (x, y) = (10, 20)可略過，直接呼叫let z = add(10, 20)，因此編譯器產生%z = call i32 @add(i32 10, i32 20)
對照產生的x86-64機械碼，10和20保存於EDI跟ESI暫存器，add與print函式則在0x5200及0x51a0地址

![](../course-materials/GoogleShape127p24.jpg)

![](../course-materials/GoogleShape128p24.jpg)

### Notes:

<!-- Slide number: 10 -->

![](../course-materials/GoogleShape135p25.jpg)
# 常見編譯器最佳化策略(1)
Constant Folding

Constant Propagation

Multiplication and Division Optimization

![](../course-materials/GoogleShape137p25.jpg)

![](../course-materials/GoogleShape138p25.jpg)

![](../course-materials/GoogleShape141p25.jpg)
無論x的值如何變化，f(x)的值必為222x + 282

### Notes:

<!-- Slide number: 11 -->
# 常見編譯器最佳化策略(2)
Function Inlining
因add函式指令很少，因此編譯器會直接將其展開為加法運算，於是sub函式的內容就變成x+(-y)，簡化後就是x—y

Strength reduction
降低迴圈中的運算強度，例如乘法改為加法、除法改為乘法，從而累積效益

![](../course-materials/GoogleShape149p26.jpg)

![](../course-materials/GoogleShape150p26.jpg)

![](../course-materials/GoogleShape151p26.jpg)

### Notes:

<!-- Slide number: 12 -->

![](../course-materials/GoogleShape159p27.jpg)
# 常見編譯器最佳化策略(3)
Canonicalize Induction Variables
若不滿足x*x<10000就跳離迴圈，而x*x<10000等同x<100

Loop Unrolling
降低CPU在branch prediction出錯的次數

![](../course-materials/GoogleShape160p27.jpg)

![](../course-materials/GoogleShape161p27.jpg)

### Notes:

<!-- Slide number: 13 -->

![](../course-materials/GoogleShape168p28.jpg)
# 常見編譯器最佳化策略(4)
Sum-Product Optimization
為減少這類有規律的大量運算，編譯器自動進行公式推導，像sum_to_x經編譯後會轉變為return x*(x-1)/2 + x，簡化後就會變成梯形面積公式x*(x+1)/2
儘管編譯器推導出的sum_to_x(x)=x*(x-1)/2+x尚有進步空間，但相比原本的迴圈，x(x+1)/2 + x 只需要2個加法、1個右移跟1個乘法，且不用顧慮x的值
square_sum_to_x做1²+2²+3²+…+x²，只用到加法、乘法跟右移，且同於sum_to_x，不隨x增長而提高運算量

![](../course-materials/GoogleShape169p28.jpg)

### Notes:

<!-- Slide number: 14 -->
# Who's afraid of a big bad optimizing compiler?

C語言如a = b;這樣敘述，依據語言標準，編譯器可認定涉及的變數在load或store的同一時刻，沒有其他執行緒在存取它們，因此允許多種最佳化，而目前越來越強大的現代編譯器所做的程式碼最佳化已超出許多人預料。
經過某些最佳化手法後，開發者不能再期望。雖針對Linux核心，但其實很多場景也適用於其他的並行(concurrent)程式設計議題，也包括使用中斷和signal相關程式碼
隨著編譯器越來越強大，就讓我們不禁開始擔心：「這些來自編譯器的最佳化有多危險？」
理解編譯器最佳化有時是為了逃出困境

你所不知道的C語言：編譯器和最佳化原理篇

### Notes:

<!-- Slide number: 15 -->
# Data-Flow Analysis
二道指令可交換位置(即Code Motion)的條件：彼此沒有相依性
資料流程分析的作用：偵測指令相依性
Reaching definition for a given instruction is another instruction, the target variable of which may reach the given instruction without an intervening assignment
Compute use-def chains and def-use chains

### Notes:

<!-- Slide number: 16 -->
# Code Motion and Pointer Aliasing
C語言中阻擋編譯器最佳化的一大原因
Pointer Aliasing: 二個不同的指標指到同一個記憶體位置
while (y < z) {
  int x = a + b;
  y += x;
}
int t1 = a + b;
while (y < z) {
  y += t1;
}

以下最佳化是否可行？
void foo(int *a, int *b, int *y, int z) {
  while (*y < z) {
    int x = *a + *b;
    *y += x;
  }
}
編譯器不能把*a + *b移到迴圈外，因為不確定是否會遇到
foo(&num, &num, &num, 5566);
如此a, b, y互為別名，*y的值改變會使*a , *b改變，故不能提出到迴圈外

### Notes:

<!-- Slide number: 17 -->

![](../course-materials/GoogleShape200p32.jpg)

### Notes:

<!-- Slide number: 18 -->

![](../course-materials/GoogleShape205p33.jpg)

### Notes:

<!-- Slide number: 19 -->

![](../course-materials/GoogleShape210p34.jpg)

### Notes:

<!-- Slide number: 20 -->
# Reachability

除了起始節點，若一個基本區塊在最佳化中沒有前任節點連接，則此BB稱為不可到達(unreachable)，我們可移除該BB來簡化CFG，如C語言關鍵字break, continue, return後的程式碼以及if, for, while中條件恆為0的情況。
移除冗餘程式碼前/後

![](../course-materials/GoogleShape217p35.jpg)

### Notes:

<!-- Slide number: 21 -->
# Domination

若從起始點到節點n都必經節點d，則稱節點d為節點n的支配節點(dominator)，記作d dom n;而其中最接近節點n的支配節點m又稱為直接支配節點(immediate dominator)，記作m idom n
將m作為親代節點、n作為子節點， 我們可建立支配樹(dominator tree)，記錄從起始點到某個BB節點必經的路徑
結合reachability分析，若節點n不可達則其在支配樹中的所有子節點將同樣不可到達，這時我們可快速安全地從CFG移除這些BB節點

![](../course-materials/GoogleShape224p36.jpg)

### Notes:

<!-- Slide number: 22 -->

![](../course-materials/GoogleShape229p37.jpg)

### Notes:

<!-- Slide number: 23 -->

![](../course-materials/GoogleShape234p38.jpg)

### Notes:

<!-- Slide number: 24 -->

![](../course-materials/GoogleShape239p39.jpg)

![](../course-materials/GoogleShape240p39.jpg)
Source: https://youtu.be/9pIuSc_a45U

### Notes:

<!-- Slide number: 25 -->

![](../course-materials/GoogleShape246p40.jpg)

### Notes:

<!-- Slide number: 26 -->

![](../course-materials/GoogleShape251p41.jpg)

### Notes:

<!-- Slide number: 27 -->

![](../course-materials/GoogleShape256p42.jpg)

### Notes:

<!-- Slide number: 28 -->

![](../course-materials/GoogleShape261p43.jpg)

### Notes:

<!-- Slide number: 29 -->

![](../course-materials/GoogleShape266p44.jpg)

### Notes:

<!-- Slide number: 30 -->

![](../course-materials/GoogleShape271p45.jpg)

### Notes:

<!-- Slide number: 31 -->

![](../course-materials/GoogleShape276p46.jpg)

### Notes:

<!-- Slide number: 32 -->

![](../course-materials/GoogleShape281p47.jpg)

### Notes:

<!-- Slide number: 33 -->

![](../course-materials/GoogleShape286p48.jpg)

### Notes:

<!-- Slide number: 34 -->

![](../course-materials/GoogleShape291p49.jpg)

### Notes:

<!-- Slide number: 35 -->

![](../course-materials/GoogleShape296p50.jpg)

### Notes:

<!-- Slide number: 36 -->

![](../course-materials/GoogleShape301p51.jpg)

### Notes:

<!-- Slide number: 37 -->
# GCC中端: Gimple & Tree SSA Optimizer
Gimple衍生自Generic Gimplify
被限制每個運算只能有兩個運算元(3-Address IR)
 t1 = A op B  ( op is operator like +-*/ … etc )
被限制語法只能有某些控制流程
Gimple可被化簡成SSA Form
可用 -fdump-tree-<type>-<option> 來觀看其結構

Tree SSA Optimizer
500+ Passes
Loop, Scalar optimization, alias analysis, …etc
Inter-procedural analysis, Inter-procedural optimization

### Notes:

<!-- Slide number: 38 -->
# GCC後端: Register Transfer Language (RTL)
b = a - 56
(set (reg:SI 60 [b])
     (plus:SI (reg:SI 61 [a])
              (const_int -56 [0xffffffc8])))
LISP風格的表示法
RTL uses virtual register (無限多個暫存器)
Register Allocation (Virtual Register → Hard Register)
Instruction scheduling (Pipeline scheduling)
GCC Built-in Operation and Arch-defined Operation
Peephole optimization

### Notes:

<!-- Slide number: 39 -->
# GCC後端: RTL Pattern Match Engine
(set (reg:SI 60 [b])
     (plus:SI (reg:SI 61 [a])
              (const_int -56 [0xffffffc8])))
MIPS.md
(define_insn "*addsi3"
  [(set (match_operand:GPR 0 "register_operand" "=d,d")
        (plus:GPR (match_operand:GPR 1 "register_operand" "d,d")
                  (match_operand:GPR 2 "arith_operand" "d,Q")))]
  "!TARGET_MIPS16"
  "@
    addu\t%0,%1,%2
    addiu\t%0,%1,%2"
  [(set_attr "type" "arith")
   (set_attr "mode" "si")])

d代表是Register
Q代表是整數
指令的屬性(用於管線排程)
b = a - 56
addiu   $2, $3, -56
指令限制(非MIPS16才可用)

### Notes:

<!-- Slide number: 40 -->
# Code Generation
4  + 3 * 2
finish
start
(LL parsing)

expression

addi $t0, $zero, 4
addi $t1, $zero, 3
addi $t2, $zero, 2
mul  $t1, $t1, $t2
add  $t0, $t0, $t1

simpleExpression

evaluate

term      +       term

factor   *   factor
factor
evaluate
evaluate

literal
literal
literal
evaluate
evaluate
evaluate

### Notes:

<!-- Slide number: 41 -->
# Register Allocation Problem

addi $t0, $zero, 4
addi $t1, $zero, 3
addi $t2, $zero, 2
mul  $t1, $t1, $t2
add  $t0, $t0, $t1

destination
evaluate

Immediate result
evaluate
evaluate

### Notes:

<!-- Slide number: 42 -->
# GCC後端 : 暫存器配置
暫存器配置其實是個著色問題 (NP-Complete)
給一個無向圖，相鄰的二個vertex不能著同一種顏色
每個vertex代表的是一個變數, 每個 edge 代表的是兩個變數的生存時間有重疊
GCC的暫存器配置，其實比著色問題更複雜
在有些平台，某些指令只能搭配某組的暫存器(惡名昭彰的x86，缺乏指令正交性)
其實有些變數是常數，或是 memory form,或是他是參數或回傳值，所以只能配置到某些特定暫存器
Old GCC RA: Reload Pass
GCC把pseudo-registers根據指令的限制對應到硬體暫存器的過程
其實是個複雜到極點的程式，所以到最後就變得令人費解
所以後來就重寫一個
IRA (Integrated Register Allocator) : 整合 Coalescing, Live range splitting及挑選較好的Hard Register的做法

### Notes:

<!-- Slide number: 43 -->
# GCC後端 :管線排程
以經典課本的五級Pipeline 為例子
lw $8, a
IF
ID
EX
ME
IF
ID
EX
ME
WB
WB
lw $9, b

mul $10,$8,$8

add $11,$9,$10

lw $12,c

add $13,$12,$0

lw $8, a

lw $9, b

lw $12,c

mul $10,$8,$8

add $13,$12,$0

### Notes:

<!-- Slide number: 44 -->
# GCC後端: Peephole optimization
以管窺天最佳化法
掃過整個IR，看附近2 ~ n個IR有沒有最佳化的機會
感覺起來就像是用奧步
可是若奧步有用，那就得要好好使用
有時，這個最佳化很好用，尤其效能評比
;;  sub  rd, rn, #1
;;  cmn  rd, #1 (equivalent to cmp rd, #-1)
;;  bne  dest

;;  subs rd, rn, #1
;;  bcs  dest   ((unsigned)rn >= 1)

;; This is a common looping idiom (while (n--))

### Notes:

<!-- Slide number: 45 -->
# 在專案中拆分多個.c檔案
為什麼要分很多檔案?
因為要讓事情變得簡單
為什麼我們在編譯程式的時候可以下make -j24?
因為編譯器在Compile 每個 .c 的時候視為獨立個體，彼此之間沒有相依性
用Compiler的術語稱之為Compilation Unit
Static Global Variable vs Non-Static Global Variable
Static: 只有這個 Compilation Unit 可以看到這個變數
Non-Static: 所有Compilation Unit 可以看到這個變數
Q: 怎麼使用別的別的Compilation Unit內的變數
A: 使用extern關鍵字，如extern int i;

### Notes:

<!-- Slide number: 46 -->
# 深入淺出Compilation Unit
Compilation Unit
Compilation Unit產生的限制
關於沒有人使用的Static Global Variable
編譯器可砍掉它來節省空間
關於沒有人使用的 Non-Static Global Variable
編譯器不能砍掉它
因為無法判定別的Compilation Unit是否用到它
Note:  若確定沒有別的檔案會使用到這個變數或函式，請宣告成static

Internal Data
Visible Data
External Data Reference
Internal Function
Visible Function
External Func Reference
Internal Function

### Notes:

<!-- Slide number: 47 -->
# Compilation Unit
盡可能使用區域變數
int main() {
  int i;
  for (i = 0 ; i < 10; i++)
    printf ("Hello %d\n", i);
}
static int i;
int main() {
  for (i = 0 ; i < 10; i++)
    printf ("Hello %d\n", i);
}
main:
        mov     r3, #0
        stmfd   sp!, {r4, lr}
        movw    r4, #:lower16:.LANCHOR0
        movt    r4, #:upper16:.LANCHOR0
        mov     r1, r3
        str     r3, [r4, #0]
.L2:
        ldr     r0, .L4
        bl      printf
        ldr     r1, [r4, #0]
        add     r1, r1, #1
        str     r1, [r4, #0]
        cmp     r1, #9
        ble     .L2
        ldmfd   sp!, {r4, pc}
.L4:
        .word   .LC0
main:
        stmfd   sp!, {r4, lr}
        mov     r4, #0
.L2:
        mov     r1, r4
        ldr     r0, .L4
        add     r4, r4, #1
        bl      printf
        cmp     r4, #10
        bne     .L2
        ldmfd   sp!, {r4, pc}
.L4:
        .word   .LC0

11

### Notes:

<!-- Slide number: 48 -->
# GCC 內建函式 (Built-in Function)
GCC 為了進行最佳化，會辨認標準函式(因為語意有標準規範)，若符合條件，就會以處理器最適合的方式來計算
strcpy → x86 字串處理指令
printf → Puts, Putc
memcpy → Block Transfer Instruction

這對開發嵌入式系統(或bare-metal)的人常造成困擾
因為開發嵌入式系統的人喜歡開發自己殘廢的printf (X)
因為開發嵌入式系統需要客製化的printf  (O)
Note: 使用 -fno-builtin-XXX or -fno-builtin 來關閉這個功能

### Notes:

<!-- Slide number: 49 -->
# IPO (Inter-Procedural Optimization)
Procedure / Function: 結構化程式的主軸, 增加可用性，減少維護成本
副程式呼叫的執行成本
準備參數: 把變數移到到特定 Register 或 push 到 Stack
呼叫副程式，儲存和回復Register Context
現代化程式: 有許多很小的副程式
int IsOdd(int num) {
  return (num & 0x1);
}
最簡單的 IPO: Function inlining
簡單來說: 把整個副程式複製一份到caller端的程式碼去

### Notes:

<!-- Slide number: 50 -->
# IPO (Inter-Procedural Optimization)
如何 Inline 一個 外部函式?
基本上 Compiler 無法做到
因為 Compiler 在編譯程式的時候不會去看別的 Compilation Unit 的內容，不知道內容自然就沒辦法 inline 一個看不見的函式
解法: Link Time Optimization (LTO)
或叫: Whole-Program Optimization (WPO)
關鍵 Link Time Code Generation
IR 1
Source 1
Compiler
Linker
Full IR
Compiler

IR 2
Source 2
Compiler

Optimized Program
IR 3
Source 3
Compiler

### Notes:

<!-- Slide number: 51 -->
# 開發最佳化編譯器的背景知識
程式語言規格書
程式碼解析器：計算理論、離散數學
目標架構機械碼產生器：計算機結構
編譯器最佳化：離散數學、圖論、資料結構、演算法
執行時期函式庫和工具：作業系統、計算機結構、資訊安全

Check the lecture materials:
https://www.cs.cmu.edu/afs/cs/academic/class/15745-s19/www/lectures/

### Notes:

<!-- Slide number: 52 -->
# 超級電腦在許多領域改變世界
用於需要大量運算的工作，例如天氣預報、地球模擬、運算化學、分子模型、天體物理、密碼分析、機械模擬、基因定序、高頻交易等等。

![](../course-materials/GoogleShape617p67.jpg)

### Notes:

<!-- Slide number: 53 -->
# 富岳(Fugaku)超級電腦
富士通與日本理化學研究所共同開發，作為「京」(2011年TOP500榜首)的後繼，於2014年開始研發，「富岳」是富士山的別稱
全球首度奪冠的Arm架構超級電腦，採用富士通48+4核A64FX整合處理器晶片，不同於過往超級電腦採用的x86架構。富岳共有7,630,848個節點，尖峰效能可達442 Pflop/s (Peta=1015; Tera=1012)
SONY PlayStation 5可達10.3 Tflop/s; 台灣衫二號超級電腦達9 Pflop/s，TOP500排名第28名(2020年11月)
2020年6月23日，富岳以415 Pflops計算能力成為TOP500榜首，同年11月17日和2021年6月28日蟬聯榜首，浮點數運算能力是第二名的3倍 延伸閱讀
由於A64FX以Arm指令集(v8.2A)為基礎，執行在富岳的軟體就需要針對Arm和Linux進行最佳化，但顯然不是每套軟體都做好準備

### Notes:

<!-- Slide number: 54 -->
# 超級電腦裡頭出現我寫的程式，嚇死寶寶！
Spack是針對超級電腦需求而開發的套件管理系統
SSE2NEON被Spack收錄並活躍更新
但我起初不懂誰要用

![](../course-materials/GoogleShape630p69.jpg)

### Notes:

<!-- Slide number: 55 -->
雖然連簡介都看不懂，但裡頭的詞彙太嚇人:
Oxford nanopore: 第三代基因定序/奈米孔定序技術
Pacific Biosciences(簡稱PacBio): 單一分子即時定序
Long noisy reads assembly: 拼接基因組的手段

# 直到我看到軟體清單
bwa: Burrow-Wheeler Aligner for pairwise alignment between DNA sequences
ngmlr: a long-read mapper designed to align PacBio or Oxford Nanopore to a reference genome with a focus on reads that span structural variations.
wtdbg2: a fuzzy Bruijn graph approach to long noisy reads assembly
smartdenovo: a de novo assembler for PacBio and Oxford Nanopore (ONT) data.
fermikit: De novo assembly based variant calling pipeline for Illumina short reads
sortmerna: a program tool for filtering, mapping and OTU-picking NGS reads in metatranscriptomic and metagenomic data"
fermi: a WGS de novo assembler based on the FMD-index for large genomes.
enzo: adaptive mesh-refinement simulation code
racon: Ultrafast consensus module for raw de novo genome assembly of long uncorrected reads
openmx: software package for nano-scale material simulations based on density functional theories (DFT), norm-conserving pseudopotentials, and pseudo-atomic localized basis functions.
詞彙提示
基因組de novo測序，通過reads拼接獲得Contigs後，往往還需要構建454 Paired-end庫或Illumina Mate-pair庫，以獲得一定大小片段(如 3Kb, 8Kb, 10Kb, 20Kb)兩端的序列
二代測序技術: next generation sequencing (NGS)，又稱為高通量測序技術

### Notes:

<!-- Slide number: 56 -->
# 這些軟體原已針對x86最佳化，為何還要支援Arm架構呢？
Arm的網站刊載Genomics: Optimizing the BWA aligner for Arm Servers，針對AWS自行研發的第二代Arm架構的Graviton 2處理器進行生物資訊運算，得到的結論:
time saving of 14%-27% over the fastest x86_64 instance timings.
Saved 50% of the cost
NVIDIA網站刊載Spotlight: Petrobras Speeds Up Linear Solvers for Reservoir Simulation Using NVIDIA Grace CPU，針對Arm架構的Grace中央處理器
油藏模擬隸屬油藏工程，使用電腦模型來預測流體通過多孔介質的流動。單井鑽探成本高達1億美元，高效能運算有助於降低資源勘探的不確定性並提升生產的成功率
相較於x86中央處理器，Petrobras達成4.5倍的運算速度、4.3倍的能源效率及1.5倍的延展性
基因定序和油田探勘運算成本居高不下，於是成本效益高的Arm架構是關鍵，Linux無庸置疑是首選作業系統

### Notes:

<!-- Slide number: 57 -->

![](../course-materials/GoogleShape649p72.jpg)
Source: https://community.arm.com/developer/tools-software/hpc/b/hpc-blog/posts/optimizing-genomics-and-the-bwa-aligner-for-arm-servers

### Notes:

<!-- Slide number: 58 -->

![](../course-materials/GoogleShape656p73.jpg)
Source: 
https://developer.nvidia.com/blog/spotlight-petrobras-accelerates-linear-solvers-for-reservoir-simulation-using-nvidia-grace-cpu

### Notes:

<!-- Slide number: 59 -->
# 無心插柳：SSE2NEON已用於超級電腦
生物資訊運算高度依賴處理器，許多演算法開發時即考慮到高平行度，Arm伺服器的優勢在於功耗相當的狀況下，能夠用更多處理器核(core)數進行運算
不只富岳，TOP500也有其他超級電腦採用Arm架構
2000年代初期x86和x86-64取得優勢之前，TOP500超級電腦大多由各種RISC處理器系列構成，包括SPARC, MIPS, PA-RISC, Alpha等架構，Arm的加入引來新的衝擊

### Notes: