# Binius: highly efficient proofs over binary fields

这篇文章主要面向大致熟悉 2019 年密码学的读者，尤其是 SNARK 和 STARK。如果你不是，我建议你先阅读这些文章。特别感谢 Justin Drake、Jim Posen、Benjamin Diamond 和 Radi Cojbasic 的反馈和评论。

> This post is primarily intended for readers roughly familiar with 2019-era cryptography, especially SNARKs and STARKs. If you are not, I recommend reading those articles first. Special thanks to Justin Drake, Jim Posen, Benjamin Diamond and Radi Cojbasic for feedback and review.

在过去的两年中，STARK已经成为一种关键且不可替代的技术，可以有效地生成对**复杂语句的易验证加密证明**（例如，证明以太坊区块是有效的）。其中一个关键原因是域的大小：为了确保安全性，基于椭圆曲线的 SNARK 需要工作在 256 位整数上，而 STARK 可以使用更小的域，并获得更高的效率：首先是 Goldilocks field（64 位整数），然后是 Mersenne31 和 BabyBear（均为 31 位）。由于这些效率的提高，使用Goldilocks的Plonky2在证明多种计算方面比其前辈快数百倍。

> Over the past two years, [STARKs](https://vitalik.eth.limo/general/2018/07/21/starks_part_3.html) have become a crucial and irreplaceable technology for efficiently making [easy-to-verify cryptographic proofs of very complicated statements](https://vitalik.eth.limo/general/2021/01/26/snarks.html) (eg. proving that an Ethereum block is valid). A key reason why is *small field sizes*: whereas elliptic curve-based SNARKs require you to work over 256-bit integers in order to be secure enough, STARKs let you use much smaller field sizes, which are more efficient: first [the Goldilocks field](https://polygon.technology/blog/plonky2-a-deep-dive) (64-bit integers), and then [Mersenne31 and BabyBear](https://blog.icme.io/small-fields-for-zero-knowledge/) (both 31-bit). Thanks to these efficiency gains, Plonky2, which uses Goldilocks, is [hundreds of times faster](https://polygon.technology/blog/introducing-plonky2) at proving many kinds of computation than its predecessors.

自然而然的：在这种域越来越小的趋势下，我们能否通过直接操作 0 和 1 来构建一个更快的证明系统？这正是 Binius 试图做的事情，Binius 使用了许多数学技巧，使其与三年前的 SNARK 和 STARK 截然不同。这篇文章介绍了为什么小域（Small fields）使证明生成更有效率，为什么二进制字段具有独特的强大功能，以及 Binius 用来使二进制字段上的证明如此有效的技巧。

> A natural question to ask is: can we take this trend to its logical conclusion, building proof systems that run even faster by operating directly over zeroes and ones? This is exactly what [Binius](https://eprint.iacr.org/2023/1784.pdf) is trying to do, using a number of mathematical tricks that make it *very* different from the [SNARKs](https://vitalik.eth.limo/general/2019/09/22/plonk.html) and [STARKs](https://vitalik.eth.limo/general/2018/07/21/starks_part_3.html) of three years ago. This post goes through the reasons why small fields make proof generation more efficient, why binary fields are uniquely powerful, and the tricks that Binius uses to make proofs over binary fields work so effectively.

![Overview](https://vitalik.eth.limo/images/binius/binius.drawio.png)

## Table of contents
- [ ] 生成目录 `Markdown All in One: Create Table of Contents`

## Recap: finite fields

加密证明系统的关键任务之一是对大量数据进行操作，同时让生成的结果足够的小（存储意义上的足够小）。如果把一个大程序的语句“压缩”成一个包含几个数字的数学方程式，但这些数字和原来的程序一样大，毫无疑问，这样的操作是没有意义的。

> One of the key tasks of a cryptographic proving system is to operate over huge amounts of data, while keeping the numbers small. If you can compress a statement about a large program into a mathematical equation involving a few numbers, but those numbers are as big as the original program, you have not gained anything.

在进行复杂算术运算的同时确保数字维持在一个很小的范围内，密码学家通常使用模运算。简单来说， 我们选择一些素数$p$。$\%$ 运算符表示“取余数”：$15 \% 7=1, 53 \% 10 = 3$，依此类推（请注意，取模的结果始终是非负的，例如$-1 \% 10 = 9$）。

> To do complicated arithmetic while keeping numbers small, cryptographers generally use **modular arithmetic**. We pick some prime "modulus" $p$. The $\%$ operator means "take the remainder of" $15 \% 7=1, 53 \% 10 = 3$, etc (note that the answer is always non-negative, so for example $-1 \% 10 = 9$).

为了在保持小数字的同时进行复杂的算术运算，密码学家通常使用模算术（取模运算）。我们选择一些素数p。% 运算符表示“取余数”：15 % 7=1,53 % 10=3，依此类推（请注意，取模的结果始终是非负的，例如 −1 % 10=9）。

![Overview](https://vitalik.eth.limo/images/binius/clock.png)

*时钟就是一个模运算的例子，例如，9：00 之后的四个小时是什么时间？但在实际应用中，我们不只进行模加减运算，我们还可以进行进行乘法、除法和指数运算。*

> You've probably already seen modular arithmetic, in the context of adding and subtracting time (eg. what time is four hours after 9:00?). But here, we don't just add and subtract modulo some number, we also multiply, divide and take exponents.

我们重新定义如下的计算：

> We redefine:

$$
\begin{align*}
x + y =>& (x + y) \% p \\
x \times y =>& (x \times y) \% p \\
x^y =>& (x^y) \% p \\
x - y =>& (x - y) \% p \\
x/y =>& (x \times y^{p-2}) \% p
\end{align*}
$$

上面的规则都是自洽的， 例如，我们取 $p=7$，则：
- $5 + 3 = 1$ (because $8 \% 7 = 1$)
- $1 - 3 = 5$ (because $-2 \% 7 = 5$)
- $2 \times 5 = 3$ (because $10 \% 7 = 3$)
- $3/5 = 2$ (because $(3 \times 5^5) \% 7 = 9375 \%7 = 2$)

这种结构的一个更一般的术语是有限域。有限域是一种数学结构，它遵循上述的算术法则，但是可能的计算结果是有限的，因此每一种可能的输出都可以用固定的长度来表示。

> A more general term for this kind of structure is a **finite field**. A [finite field](https://en.wikipedia.org/wiki/Finite_field) is a mathematical structure that obeys the usual laws of arithmetic, but where there's a limited number of possible values, and so each value can be represented in a fixed size.
>
> 进一步解释：有限域是一种数学结构，其包含了有限个元素，域中的元素能够进行加减乘除的运算，且计算后的结果仍是属于这个有限域，因此有限域中的每个元素都可以用固定长度表示。有限域最常见的例子是 modulus $p$ 为素数。我们依然用 modulus = 7 举例，有限域 $F_7$ 包含的元素有 $\{0, 1, 2, 3, 4, 5, 6\}$, 这个域中的每一个元素都是整数，在这个域中任取两个元素进行计算后的结果仍然属于这个域。

模算术（或素数域）是最常见的有限域类型，但也有另一种类型：扩域。您之前可能已经看过一个扩域：复数。我们“想象”一个新元素，我们用 $i$ 表示 ，并声明它满足 $i^2=−1$。然后，您可以使用任意的实数和 $i$ 做线性组合 ，并用它做数学运算： $(3i+2)*(2i+4)=6i^2+12i+4i+8=16i+2$ 。我们同样可以对素数域进行扩域。当我们开始处理更小的字段时，素数域的扩域对于保持安全性变得越来越重要，而 Binius 使用的二进制域及其扩域具有非常好的实用性。

> Modular arithmetic (or **prime fields**) is the most common type of finite field, but there is also another type: **extension fields**. You've probably already seen an extension field before: the complex numbers. We "imagine" a new element, which we label $i$, and declare that it satisfies $i^2=−1$. You can then take any combination of regular numbers and $i$, and do math with it: $(3i+2)*(2i+4)=6i^2+12i+4i+8=16i+2$ . We can similarly take extensions of prime fields. As we start working over fields that are smaller, extensions of prime fields become increasingly important for preserving security, and binary fields (which Binius uses) depend on extensions entirely to have practical utility.

> 补充说明：

## Recap: arithmetization 

- [ ] TODO

> The way that SNARKs and STARKs prove things about computer programs is through **arithmetization**: you convert a statement about a program that you want to prove, into a mathematical equation involving polynomials. A valid solution to the equation corresponds to a valid execution of the program.

> To give a simple example, suppose that I computed the 100'th Fibonacci number, and I want to prove to you what it is. I create a polynomial $F$ that encodes Fibonacci numbers: so $F(0)=F(1)=1,F(2)=2,F(3)=3,F(4)=5$, and so on for 100 steps. The condition that I need to prove is that $F(x+2)=F(x)+F(x+1)$ across the range $x=\{0,1 \ldots 98\}$. I can convince you of this by giving you the quotient:

$$
H(x)=\frac{F(x+2)-F(x+1)-F(x)}{Z(x)}
$$

> Where $Z(x)=(x-0)*(x-1)*\ldots*(x-98)$. If I can provide valid $F$ and $H$ that satisfy this equation, then $F$ must satisfy $F(x+2)-F(x+1)-F(x)$ across that range. If I additionally verify that $F$ satisfies $F(0)=F(1)=1$ , then $F(100)$ must actually be the 100th Fibonacci number.

> If you want to prove something more complicated, then you replace the "simple" relation $F(x+2)=F(x)+F(x+1)$ with a more complicated equation, which basically says "$F(x+1)$ is the output of initializing a virtual machine with the state $F(x)$, and running one computational step". You can also replace the number 100 with a bigger number, eg. 100000000, to accommodate more steps.

> All SNARKs and STARKs are based on this idea of using a simple equation over polynomials (or sometimes vectors and matrices) to represent a large number of relationships between individual values. Not all involve checking equivalence between adjacent computational steps in the same way as above: [PLONK](https://vitalik.eth.limo/general/2019/09/22/plonk.html) does not, for example, and neither does R1CS. But many of the most efficient ones do, because enforcing the same check (or the same few checks) many times makes it easier to minimize overhead.

## Plonky2: from 256-bit SNARKs and STARKs to 64-bit... only STARKs

五年前，ZK证明系统被分为有两种类型: (基于椭圆曲线的) SNARK 和(基于 hash 的) STARK。准确地说，STARK 是 SNARK 的一种，但在实践中，通常用“ SNARK” 来指基于椭圆曲线的证明系统，用“ STARK”来指基于 hash 的证明系统。SNARK 生成的 proof size 很小，因此其可以被快速验证，并轻松的放到链上。STARK 的 proof size 很大，但是 STARK 不需要 trusted setups ，并具有抗量子的特性。

> Five years ago, a reasonable summary of the different types of zero knowledge proof was as follows. There are two types of proofs: (elliptic-curve-based) SNARKs and (hash-based) STARKs. Technically, STARKs are a type of SNARK, but in practice it's common to use "SNARK" to refer to only the elliptic-curve-based variety, and "STARK" to refer to hash-based constructions. SNARKs are small, and so you can verify them very quickly and fit them onchain easily. STARKs are big, but they don't require [trusted setups](https://vitalik.eth.limo/general/2022/03/14/trustedsetup.html), and they are quantum-resistant.

![Overview](https://vitalik.eth.limo/images/binius/starks.png)

*STARK 的工作原理是将数据当作多项式处理，计算该多项式在大量点上的评估，并使用这些数据的 Merkle 根作为“多项式承诺”*

> STARKs work by treating the data as a polynomial, computing evaluations of that polynomial across a large number of points, and using the Merkle root of that extended data as the "polynomial commitment"

这里一个关键的历史是，基于椭圆曲线的 SNARK 首先被广泛使用：直到 2018 年左右，FRI 出现，STARK 才变得足够高效，而那时 Zcash 已经运行了一年多。基于椭圆曲线的 SNARK 有一个关键限制：如果要使用 SNARK，则这些方程中的算术必须使用椭圆曲线上的点数来模一个整数
*(FIXME: 这句话和本人目前理解的椭圆曲线上的运算不太一致， 需要重新翻译)* 
来完成。这是一个很大的数字，通常接近 $2^{256}$ ：例如，对于 BN128 曲线，它是 $21888242871839275222246405745257275088548364400416034343698204186575808495617$。但被算数化的程序中往往用到较小数字的频率更多：考虑一个“真实”的程序，它使用的大部分东西都是 counter、for 循环中的索引、程序中的位置、表示 True 或 False 的 bit，以及其他几乎总是只有几位长度的东西。

> A key bit of history here is that elliptic curve-based SNARKs came into widespread use first: it took until roughly 2018 for STARKs to become efficient enough to use, thanks to [FRI](https://eccc.weizmann.ac.il/report/2017/134/), and by then [Zcash](https://z.cash/) had already been running for over a year. Elliptic curve-based SNARKs have a key limitation: if you want to use elliptic curve-based SNARKs, then the arithmetic in these equations must be done with integers modulo the number of points on the elliptic curve. This is a big number, usually near $2^{256}$: for example, for the bn128 curve, it's $21888242871839275222246405745257275088548364400416034343698204186575808495617$. But the actual computation is using small numbers: if you think about a "real" program in your favorite language, most of the stuff it's working with is counters, indices in for loops, positions in the program, individual bits representing True or False, and other things that will almost always be only a few digits long.

即使您的“原始”数据由“小”数字组成，证明过程也需要计算商、扩域、随机线性组合和其他数据转换，这会导致相同或更多数量的对象，平均而言，这些对象与域的全尺寸一样大。这是导致低效的关键，即：要证明 $n$ 个小值的计算，你必须对 n 个大得多的值进行更多的计算。起初，STARK 继承了使用 SNARK 的 256 位 Field 的习惯，因此也遭受了同样的低效现象。

> Even if your "original" data is made up of "small" numbers, the proving process requires computing quotients, extensions, random linear combinations, and other transformations of the data, which lead to an equal or larger number of objects that are, on average, as large as the full size of your field. This creates a key inefficiency: to prove a computation over $n$ small values, you have to do even more computation over $n$ much bigger values. At first, STARKs inherited the habit of using 256-bit fields from SNARKs, and so suffered the same inefficiency.

![](https://vitalik.eth.limo/images/binius/rs_example.png)

一些多项式求值的 Reed-Solomon 扩展。即使原始值很小，额外的值也会放大到字段的完整大小（在本例中为 $2^{31}-1$ ）。

> A Reed-Solomon extension of some polynomial evaluations. Even though the original values are small, the extra values all blow up to the full size of the field (in this case $2^{31} - 1$).

2022 年，Plonky2 发布。Plonky2 的主要创新是将算术模数计算为较小的素数： $2^{64}-2^{32}+1=18446744069414584321$  。现在，每次加法或乘法都可以在 CPU 上只需几条指令即可完成，并且将所有数据哈希在一起的速度比以前快 4 倍。但这有一个问题：这种方法是 STARK 独有的。如果您尝试使用具有如此小尺寸的椭圆曲线的 SNARK，则椭圆曲线会变得不安全。

> In 2022, Plonky2 was released. Plonky2's main innovation was doing arithmetic modulo a smaller prime: $2^{64}-2^{32}+1=18446744069414584321$. Now, each addition or multiplication can always be done in just a few instructions on a CPU, and hashing all of the data together is 4x faster than before. But this comes with a catch: this approach is STARK-only. If you try to use a SNARK, with an elliptic curve of such a small size, the elliptic curve becomes insecure.

> To continue to be safe, Plonky2 also needed to introduce *extension fields*. A key technique in checking arithmetic equations is "sampling at a random point": if you want to check if $H(x) * Z(x)$ actually equals $F(x+2) - F(x+1) - F(x)$, you can pick some random coordinate $r$, provide *polynomial commitment opening proofs* proving $H(r), Z(r), F(r), F(r+1)$ and $F(r+2)$, and then actually check if $H(r) * Z(r)$ equals $F(r+2) - F(r+1) - F(r)$. If the attacker can guess the coordinate ahead of time, the attacker can trick the proof system - hence why it must be random. But this also means that the coordinate must be sampled from a set large enough that the attacker cannot guess it by random chance. If the modulus is near $2^{256}$, this is clearly the case. But with a modulus of $2^{64} - 2^{32} + 1$, we're not quite there, and if we drop to $2^{31} - 1$, it's *definitely* not the case. Trying to fake a proof two billion times until one gets lucky is absolutely within the range of an attacker's capabilities.

>  To stop this, we sample $r$ from an extension field. For example, you can define $y$ where $y^3=5$, and take combinations of 1, $y$ and $y^2$. This increases the total number of coordinates back up to roughly $2^93$. The bulk of the polynomials computed by the prover don't go into this extension field; they just use integers modulo $2^{31} - 1$, and so you still get all the efficiencies from using the small field. But the random point check, and the FRI computation, does dive into this larger field, in order to get the needed security.

## From small primes to binary

计算机通过将较大的数字表示为 0 和 1 的序列来进行算术运算，并在这些位之上构建“电路”来计算加法和乘法等内容。计算机特别针对 16 位、32 位和 64 位整数进行计算进行了优化。模量像 $2^{64} - 2^{32} + 1$ 和 $2^{31} - 1$ 之所以选择它们，不仅是因为它们符合这些 Bound，还因为它们与这些 Bound 很好地对齐：你可以做乘法模 $2^{64} - 2^{32} + 1$ 通过执行常规的 32 位乘法，并在几个地方按位移位和复制输出；[这篇文章](https://xn--2-umb.com/22/goldilocks/)很好地解释了一些技巧。

> Computers do arithmetic by representing larger numbers as sequences of zeroes and ones, and building "circuits" on top of those bits to compute things like addition and multiplication. Computers are particularly optimized for doing computation with 16-bit, 32-bit and 64-bit integers. Moduluses like $2^{64} - 2^{32} + 1$ and $2^{31} - 1$ are chosen not just because they fit within those bounds, but also because they *align well* with those bounds: you can do multiplication modulo $2^{64} - 2^{32} + 1$ by doing regular 32-bit multiplication, and shift and copy the outputs bitwise in a few places; [this article](https://xn--2-umb.com/22/goldilocks/) explains some of the tricks well.
>
> 补充说明：

> What would be even better, however, is doing computation in binary directly. What if addition could be "just" XOR, with no need to worry about "carrying" the overflow from adding 1 + 1 in one bit position to the next bit position? What if multiplication could be more parallelizable in the same way? And these advantages would all come on top of being able to represent True/False values with just one bit.

> Capturing these advantages of doing binary computation directly is exactly what Binius is trying to do. A table from the [Binius team's zkSummit presentation](https://docs.google.com/presentation/d/1WuTiof1BiaL6vB50CSeb-hvi5H4j_oqUt19-sZTQEB4/edit#slide=id.g2c9c013854e_0_95) shows the efficiency gains:

![](https://vitalik.eth.limo/images/binius/zksummit_slides.png)

> Despite being roughly the same "size", a 32-bit binary field operation takes 5x less computational resources than an operation over the 31-bit Mersenne field.

## From univariate polynomials to hypercubes

假设我们被这个推理所说服，并希望用 0 和 1 做所有事情。我们如何实际承诺一个表示十亿位的多项式？

> Suppose that we are convinced by this reasoning, and want to do everything over bits (zeroes and ones). How do we actually commit to a polynomial representing a billion bits?

我们要面对两个问题：
1. 如果用多项式（polynomial）去表示很多值，那么这些值在多项式的评估（evaluations）时是可访问的：比如前文提到的斐波那契例子$F(0),\:F(1)\:...\:F(100)，而在更大规模的计算中，$F(x)$ 的索引会达到数百万。更大的索引需要我们用更大的域来表示。
2. 证明我们在Merkle树中承诺的任何值（就像所有的STARKs所做的那样）需要对它进行Reed-Solomon编码：将 $n$ 个值扩展到 $8n$ 个值，利用这种冗余来防止恶意的证明者通过伪造计算中间的一个值来进行欺诈。这也需要有一个足够大的域：为了将一百万的值扩展到八百万，你需要有八百万不同的点来评估多项式。

> Here, we face two practical problems:
> 1. For a polynomial to represent a lot of values, those values need to be accessible at evaluations of the polynomial: in our Fibonacci example above, $F(0),\:F(1)\:...\:F(100)$, and in a bigger computation, the indices would go into the millions. And the field that we use needs to contain numbers going up to that size.
> 2. Proving anything about a value that we're committing to in a Merkle tree (as all STARKs do) requires Reed-Solomon encoding it: extending $n$ values into eg. $8n$ values, using the redundancy to prevent a malicious prover from cheating by faking one value in the middle of the computation. This also requires having a large enough field: to extend a million values to 8 million, you need 8 million different points at which to evaluate the polynomial.

Binius 通过两种不同的方式来表示同一组数据，用以分别解决上述的两个问题。首先，基于椭圆曲线的 SNARKs ，2019年的 STARKs，Plonky2 和其他的证明系统通常处理单变量多项式 $F(x)$ 。Binus
则从 Spartan 协议中获取灵感，使用多变量多项式：$F(x_1,x_2\ldots x_k)$ 。事实上，我们用一个 Hypercube 来表示整个 computational trace ，其中每一个的 $x_i$ 的取值要么是 0 ，要么是 1 。句一个例子，如果我们想要表示一个斐波那契数列，同时我们用一个足够大的域去表示这些数字，我们把前 16 个数字进行可视化后得到

> A key idea in Binius is solving these two problems separately, and doing so by representing the same data in two different ways. First, the polynomial itself. Elliptic curve-based SNARKs, 2019-era STARKs, Plonky2 and other systems generally deal with polynomials over *one* variable: $F(x)$ . Binius, on the other hand, takes inspiration from the [Spartan](https://eprint.iacr.org/2019/550.pdf) protocol, and works with *multivariate* polynomials: $F(x_1,x_2\ldots x_k)$. In fact, we represent the entire computational trace on the "hypercube" of evaluations where each $x_i$ is either 0 or 1. For example, if we wanted to represent a sequence of Fibonacci numbers, and we were still using a field large enough to represent them, we might visualize the first sixteen of them as being something like this:

![](https://vitalik.eth.limo/images/binius/hypercube.png)

在这个例子中，“$F(0, 0, 0, 0)$ 为 1，$F(1, 0, 0, 0)$ 也为 1，$F(0, 1, 0, 0)$ 为 2，依此类推，直到 $F(1, 1, 1, 1) = 987$” 表示了一个超立方体中的各个点的取值。对于这样一组取值，存在唯一一个多线性多项式（每个变量的阶为1），可以生成这些值。因此，我们可以用这组取值来代表该多项式；从而无需计算系数。

> That is, $F(0, 0, 0, 0)$ would be 1, $F(1, 0, 0, 0)$ would also be 1, $F(0, 1, 0, 0)$ would be 2, and so forth, up until we get to $F(1, 1, 1, 1) = 987$. Given such a hypercube of evaluations, there is exactly one multilinear (degree-1 in each variable) polynomial that produces those evaluations. So we can think of that set of evaluations as representing the polynomial; we never actually need to bother computing the coefficients.
>
> 补充：这里也可以理解成一个 key-value pair（键值对），key 是这个 Hypercube 上的所有点，value 是斐波那契数列中的数字，并且如果读者去从右到左去阅读这些点，你会发现这些点和二进制表示的整数是一致的。
> | hypercube 上的点 | 用十进制整数表示的 Key/Index | Value |
> | ---- | ---- | ---- |
> |(0, 0, 0, 0)|0|1|
> |(1, 0, 0, 0)|1|1|
> |(0, 1, 0, 0)|2|2|
> |(1, 1, 0, 0)|3|3|
> |(0, 0, 1, 0)|4|5|
> |(1, 0, 1, 0)|5|8|
> |(0, 1, 1, 0)|6|13|
> |(1, 1, 1, 0)|7|21|
> |(0, 0, 0, 1)|8|34|
> |(1, 0, 0, 1)|9|55|
> |(0, 1, 0, 1)|10|89|
> |(1, 1, 0, 1)|11|144|
> |(0, 0, 1, 1)|12|233|
> |(1, 0, 1, 1)|13|377|
> |(0, 1, 1, 1)|14|610|
> |(1, 1, 1, 1)|15|987|

这个只是一个简化的例子：在实践中，使用 Hypercube 的真正目的是让我们能够处理单个比特。用“Binius-native” 方式来计算斐波那契数的话，会使用更高维的立方体，例如用每组16个比特来存储一个数字。

FIXME: 这需要一些巧妙的方法来实现这些bits的整数加法 ，

但使用Binius来实现这一点并不太困难。

> This example is of course just for illustration: in practice, the whole point of going to a hypercube is to let us work with individual bits. The "Binius-native" way to count Fibonacci numbers would be to use a higher-dimensional cube, using each set of eg. 16 bits to store a number. This requires some cleverness to implement integer addition on top of the bits, but with Binius it's not too difficult.


现在，我们来讨论纠删码。STARKs 的工作方式是：你取 $n$ 个值，通过 Reed-Solomon 编码将它们扩展到更多的值（通常是 $8n$，一般在 $2n$ 到 $32n$ 之间），然后从扩展的数据中随机选择一些 Merkle 分支并进行某种检查。一个超立方体在每个维度的长度为 2。因此，直接扩展它是不切实际的：从 16 个值中抽样 Merkle 分支的“空间”不足。那么我们该怎么办呢？我们假装超立方体是一个方块！

> Now, we get to the erasure coding. The way STARKs work is: you take $n$ values, Reed-Solomon extend them to a larger number of values (often $8n$, usually between $2n$ and $32n$), and then randomly select some Merkle branches from the extension and perform some kind of check on them. A hypercube has length 2 in each dimension. Hence, it's not practical to extend it directly: there's not enough "space" to sample Merkle branches from 16 values. So what do we do instead? We pretend the hypercube is a square!

## Simple Binius - an example

*See* *[here](https://github.com/ethereum/research/blob/master/binius/simple_binius.py) for a python implementation of this protocol.*

让我们用一个新的例子，为了方便起见，我们使用整数域（在实际的 Binius 实现中使用的是二进制域）。首先，我们获取要承诺的 Hypercube，并将其编码为方块：

> Let's go through an example, using regular integers as our field for convenience (in a real implementation this will be binary field elements). First, we take the hypercube we want to commit to, and encode it as a square:

![](https://vitalik.eth.limo/images/binius/basicbinius1.drawio.png)

现在我们使用里德所罗门编码去扩展这个方块，我们把每一行看成一个 3 阶多项式，并将行中的数字视为多项式在 $x = {0, 1, 2, 3}$ 上的取值，然后取这个多项式在 $x = {4, 5, 6, 7}$ 上的取值。

> Now, we Reed-Solomon extend the square. That is, we treat each row as being a degree-3 polynomial evaluated at $x = {0, 1, 2, 3}$, and evaluate the same polynomial at $x = {4, 5, 6, 7}$:
>
> 补充：以图中的第一行为例，让 $x = {0, 1, 2, 3}$ 作为输入，第一行中的四个值 $y = {3, 1, 4, 1}$ 作为输出。于是我们有点对 $(x=0, y=3), (x=1, y=1), (x=2, y=4), (x=1, y=1)$ ，计算其拉格朗日多项式，得到 $\frac{(x-1)(x-2)(x-3)}{-2} + \frac{x(x-2)(x-3)}{2} + 2x(x-1)(x-3) + \frac{x(x-1)(x-2)}{6}$ ，然后带入 $x = {4, 5, 6, 7}$ 到这个多项式中，得到 $y={-19, -67, -154, -291}$ 即图中右侧方块的第一行

![](https://vitalik.eth.limo/images/binius/basicbinius.drawio.png)

注意到数字的增长是非常快的，这也是我们总是在实际应用中采用有限域的原因。比如，我们采用 11 做为模，那么第一行的扩展会是 $[3, 10, 0, 6]$ 。

> Notice that the numbers blow up quickly! This is why in a real implementation, we always use a finite field for this, instead of regular integers: if we used integers modulo 11, for example, the extension of the first row would just be $[3, 10, 0, 6]$.

你可以使用该链接里的 [代码](https://github.com/ethereum/research/blob/master/binius/utils.py#L123) 来进行验证。

> If you want to play around with extending and verify the numbers here for yourself, you can use [my simple Reed-Solomon extension code here](https://github.com/ethereum/research/blob/master/binius/utils.py#L123).

接下来，从列的方向上来处理扩展后的方块，并且生成一个默克尔树，这个树的根节点就是我们的承诺

> Next, we treat this extension as columns, and make a Merkle tree of the columns. The root of the Merkle tree is our commitment.

![](https://vitalik.eth.limo/images/binius/binius_merkletree.drawio.png)

现在，我们假设 P 想要去证明这个多项式在某一个随机点上的取值 $r = (r_0, r_1, r_2, r_3)$，在  Binius 中，有一个细微的不同导致了其比其他的多项式承诺稍微弱一点，即 P 不应该在承诺默克尔根之前就知道或有能力猜到这个随机值 $r$ 。换句话说 $r$ 应该是一个依赖默克尔根的伪随机数。这使得该方案对
FIXME: "database lookup"
（举例：你给了我默克尔树的根，现在向我证明 $P(0, 0, 1, 0)$ ）毫无用处。但通常来说，我们使用的证明系统们通常不需要
FIXME: "database lookup"
这一特性，它们通常只需要检查多项式在一个随机点上的取值就好了，因此这个限制不会影响到我们要做的事情

> Now, let's suppose that the prover wants to prove an evaluation of this polynomial at some point $r = (r_0, r_1, r_2, r_3)$. There is one nuance in Binius that makes it somewhat weaker than other polynomial commitment schemes: the prover should not know, or be able to guess, $s$, until after they committed to the Merkle root (in other words, $r$ should be a pseudo-random value that depends on the Merkle root). This makes the scheme useless for "database lookup" (eg. "ok you gave me the Merkle root, now prove to me $P(0, 0, 1, 0)$ !"). But the actual zero-knowledge proof protocols that we use generally don't need "database lookup"; they simply need to check the polynomial at a random evaluation point. Hence, this restriction is okay for our purposes.

假设我们选取随机点 $r=\{1,2,3,4\}$ (多项式在该点的取值为 -137，可以用[这里的代码](https://github.com/ethereum/research/blob/master/binius/utils.py#L100)进行确认)。现在让我们来看一下 Proof 的生成过程，我们把 $r$ 分解成两个部份：第一个部份是 $\{1,2\}$ 表示对一个行中的列元素做线性组合，第二个部份 $\{3,4\}$ 表示对行做线性组合，

> Suppose we pick $r=\{1,2,3,4\}$ (the polynomial, at this point, evaluates to −137; you can confirm it [with this code](https://github.com/ethereum/research/blob/master/binius/utils.py#L100)). Now, we get into the process of actually making the proof. We split up $r$ into two parts: the first part $\{1,2\}$ representing a linear combination of *columns within a row*, and the second part $\{3,4\}$ representing a linear combination *of rows*. 

我们对列的计算其张量积

> We compute a "tensor product", both for the column part:

$$
\bigotimes_{i=0}^1(1-r_i,r_i)
$$

对于行的部份

> And for the row part:

$$
\bigotimes_{i=2}^3(1-r_i,r_i)
$$

上诉的计算意味着：取每一个集合中的元素相乘，并把所有的结果给写出来

> What this means is: a list of all possible products of one value from each set.

，在行的情形中，我们有：

> In the row case, we get:

$$
[(1-r_2)\times (1-r_3),r_2\times (1-r_3),(1-r_2)\times r_3,r_2\times r_3]
$$

> 补充：
> Binius 中张量积  $\bigotimes_{i=0}^{\nu-1}(1-r_i,r_i)$  的定义：
> 
> 域 $K^v$ 上的随机点表示为 $r := (r_0, r_1, \ldots, r_i, \ldots, r_{ν−1}) \in K^v, r_i \in K$, 
> 
>  $B^v$ 是 v-demotion-Hypercube 上所有的点（共计 $2^v$ 个）构成的集合。
> 
>  $b_k$ 表示 $B^v$ 中的元素， $b_k := (v_0, v_1, \ldots, v_i, \ldots, v_v), v_i \in \{0, 1\}$  。
>  
> $\widetilde{\textbf{eq}}$ 是计算 MLE 时用到的算式，在 MLE 中起到了一个选择器的效果，定义如下
>
> $$
> \begin{aligned}
> \widetilde{\textbf{eq}}(X_0,\ldots,X_{\nu-1},Y_0,\ldots,Y_{\nu-1}) &= \
> \prod_{i=0}^{\nu-1}X_i\cdot Y_i+(1-X_i)\cdot(1-Y_i)
> \end{aligned}
> $$
>
> 然后我们对 $B_v$ 中的每个点 $b_k$ ，计算 $\widetilde{\mathsf{eq}}(b_k,r)$ ，得到 $2^v$ 个结果
> 
> $$
> \begin{aligned}
> &\bigotimes_{i=0}^{\nu-1}(1-r_i,r_i) := {\left( \widetilde{\textbf{eq}}(b_k, r)\right) }\_{b_k\in\mathcal{B}\_\nu} \\
> =& \underbrace{ ( \widetilde{\textbf{eq}}(b_0, r), \widetilde{\textbf{eq}}(b_1, r), \ldots, \widetilde{\textbf{eq}}(b_{k}, r), \ldots) }\_\text{ count = $2^v$ }\\
> =& \underbrace{ ( \prod_{i=0}^{\nu-1}b_{0_i}\cdot r_i+(1-b_{0_i})\cdot(1-r_i), \prod_{i=0}^{\nu-1}b_{1_i}\cdot r_i+(1-b_{1_i})\cdot(1-r_i), \ldots, \prod_{i=0}^{\nu-1}b_{k_i}\cdot r_i+(1-b_{k_i})\cdot(1-r_i), \ldots ) }\_\text{ count = $2^v$ }\\
> =& \underbrace{ (1-r_0) \cdot\cdots\cdot (1-r_{\nu-1}), \ldots , r_0 \cdot\cdots\cdot r_{\nu-1}) }_\text{ count = $2^v$ }
> \end{aligned}
> $$

使用  $r={1,2,3,4}$  ，在行的情形下，我们取 $r_2 = 3$ and $r_3 = 4$ ：

> Using $r={1,2,3,4}$ (so $r_2 = 3$ and $r_3 = 4$):

$$
[(1-3)\times (1-4),3\times (1-4),(1-3)\times 4,3\times 4]=[6,-9,-8,12]
$$

现在 通过对现有的行计算其线性组合，我们得到了一个新行 $t'$ ：

> Now, we compute a new "row" $t'$, by taking this linear combination of the existing rows. That is, we take:

$$
\begin{align*}
[&3,1,4,1] \times 6 +\\
[&5,9,2,6] \times (-9) +\\
[&5,3,5,8] \times (-8) +\\
[&9,7,9,3] \times 12 =\\
[&41,-15,74,-76]
\end{align*}
$$

You can view what's going on here as a partial evaluation. If we were to multiply the full tensor product $\bigotimes_{i=0}^3(1-r_i,r_i)$ by the full vector of all values, you would get the evaluation $P(1,2,3,4)=-137$ . Here we're multiplying a *partial* tensor product that only uses half the evaluation coordinates, and we're reducing a grid of $N$ values to a row of $\sqrt{N}$ values. If you give this row to someone else, they can use the tensor product of the other half of the evaluation coordinates to complete the rest of the computation.

The prover provides the verifier with this new row, $t'$, as well as the Merkle proofs of some randomly sampled columns. This is $O(\sqrt{N})$ data. In our illustrative example, we'll have the prover provide just the last column; in real life, the prover would need to provide a few dozen columns to achieve adequate security.

Now, we take advantage of the linearity of Reed-Solomon codes. The key property that we use is: **taking a linear combination of a Reed-Solomon extension gives the same result as a Reed-Solomon extension of a linear combination**. This kind of "order independence" often happens when you have two operations that are both linear.

The verifier does exactly this. They compute the extension of $t'$, and they compute the same linear combination of columns that the prover computed before (but only to the columns provided by the prover), and verify that these two procedures give the same answer.

![](https://vitalik.eth.limo/images/binius/basicbinius2.drawio.png)


In this case, extending $t'$, and computing the same linear combination $([6,-9,-8,12])$ of the column, both give the same answer: $-10746$ . This proves that the Merkle root was constructed "in good faith" (or it at least "close enough"), and it matches $t'$ : at least the great majority of the columns are compatible with each other and with $t'$.

But the verifier still needs to check one more thing: actually check the evaluation of the polynomial at $\{r_0..r_3\}$ . So far, none of the verifier's steps actually depended on the value that the prover claimed. So here is how we do that check. We take the tensor product of what we labelled as the "column part" of the evaluation point:

$$
\bigotimes_{i=0}^1(1-r_i,r_i)
$$

In our example, where $r=\{1,2,3,4\}$ (so the half that chooses the column is $\{1, 2\}$ ), this equals:

$$
[(1-1)\times (1-2),1\times (1-2),(1-1)\times 2,1\times 2]=[0,-1,0,2]
$$

So now we take this linear combination of $t'$ :

$$
0\times 41+(-1)\times (-15)+0\times 74+2\times (-76)=-137
$$

Which exactly equals the answer you get if you evaluate the polynomial directly.

The above is pretty close to a complete description of the "simple" Binius protocol. This already has some interesting advantages: for example, because the data is split into rows and columns, you only need a field half the size. But this doesn't come close to realizing the full benefits of doing computation in binary. For this, we will need the full Binius protocol. But first, let's get a deeper understanding of binary fields.

## Binary fields