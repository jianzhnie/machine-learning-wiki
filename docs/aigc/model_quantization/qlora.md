# QLORA: Efficient Finetuning of Quantized LLMs

我们提出了QLora，这是一种有效的微调方法，可以在单个48GB GPU上微调65B参数模型，同时保持完整的16位微调任务性能。QLora 使用一种低精度的存储数据类型（NF4）来压缩预训练的语言模型。通过冻结 LM 参数，将相对少量的可训练参数以 Low-Rank Adapters 的形式添加到模型中，LoRA 层是在训练期间更新的唯一参数，使得模型体量大幅压缩同时推理效果几乎没有受到影响。从QLora的名字可以看出，QLora实际上是Quantize+LoRA技术。

我们名为 Guanaco 的最佳模型家族在 Vicuna 基准测试中优于所有以前的公开发布模型，达到了 ChatGPT 的性能水平 99.3%，而在单个 GPU 上只需要 24 小时的微调。

QLora 引入了多种创新，旨在在不牺牲性能的情况下减少内存使用：

1. 4-bit NormalFloat：这是一种理论上针对正态分布数据的最优的量化数据类型，优于当前普遍使用的FP4与Int4。
2. Double Quantization：相比于当前的模型量化方法，更加节省显存空间。每个参数平均节省0.37bit，对于65B的LLaMA模型，大约能节省3GB显存空间。
3. Paged Optimizers：使用NVIDIA统一内存来避免在处理小批量的长序列时出现的梯度 Checkppints 内存峰值。
4. 增加 Adapter：4-bit NormalFloat与Double Quantization，节省了很多空间，但带来了性能损失，作者通过插入更多adapter来弥补这种性能损失。在LoRA中，一般会选择在query和value的全连接层处插入adapter。而QLora则在所有全连接层处都插入了adapter，增加了训练参数，弥补精度带来的性能损失。

我们使用 QLora 微调超过 1,000 个模型，对 8 个指令数据集、多个模型类型 (LLaMA, T5) 和模型尺度进行指令跟随和聊天机器人性能详细分析，这些尺度无法通过常规微调运行（例如 33B 和 65B 参数模型）。我们的结果表明，即使在使用比以前的 SoTA 更小的模型时，在小高质量数据集上的 QLora 微调也会导致最先进的结果。

## 背景技术

### Quantize

模型的大小通常由其参数的数量及其精度决定，常见的精度有全精度float32(FP32)、半精度float16(FP16)和bfloat16(BF16)。

- FP32：单精度浮点数，用8bit 表示指数，23bit 表示小数。使用此数据类型，可以表示各种浮点数，并且支持大多硬件。
- FP16：半精度浮点数，用5bit 表示指数，10bit 表示小数。FP16 表示整数范围较小，但是尾数精度较高。
- BF16：是对FP32单精度浮点数截断数据，用8bit 表示指数，7bit 表示小数。BF16 可表示的整数范围与FP32一样广泛。但只有新的硬件(A100\3090\4090等)才支持，V100/昇腾910等不支持

量化的本质实际是从一种数据类型舍入到另一种数据类型，通常包含量化和反量化两步，假如我们有两组数据类型A、B，A可以表示的数值为[0, 1, 2, 3, 4, 5]，B可以表示的数值为[0, 2, 4]。

- 现在有一个数据类型A表示的向量为[3, 1, 2, 3], 我们要做的便是：将数据范围从A标准化为B。
  - 找到向量[3, 1, 2, 3]的最大绝对值3
  - 向量[3, 1, 2, 3]除以最大值3：[3, 1, 2, 3]——>[1, 0.33, 0.66, 1.0]
  - 将向量[1, 0.33, 0.66, 1.0]与B的数据范围4相乘：[1, 0.33, 0.66, 1.0]——>[4.0, 1.33, 2.66, 4.0]
  - 将向量[4.0, 1.33, 2.66, 4.0]中的每个值用B中最接近的数值表示：[4.0, 1.33, 2.66, 4.0] ——> [4, 2, 2, 4]。

- 现在我们将 [4, 2, 2, 4] 反量化， 用B中最接近的数值表示A 。
  - [4, 2, 2, 4]除以4->[1.0, 0.5, 0.5, 1.0]
  - 乘以量化过程中找到的最大的绝对值：[1.0, 0.5, 0.5, 1.0] -> [3.0, 1.5, 1.5, 3.0]
  - 近似表示：[3.0, 1.5, 1.5, 3.0] -> [3, 2, 2, 3]


经过量化、反量化，原来的[3, 1, 2, 3]变成了[3, 2, 2, 3]，产生了量化误差。这样的误差在模型中传播、积累，最终会影响模型性能。提高量化精度的最佳方法是使用更多的量化参数。例如，我们有一个二维的张量：[[3, 1, 2, 3], [0, 1, 1, 0]]。如果分别为[3, 1, 2, 3]和[0, 1, 1, 0]设置独立的量化参数为各自的最大绝对值3和1，那么量化的精度会比为整个[[3, 1, 2, 3], [0, 2, 2, 0]]只设置一个量化参数3更高。

### LM.int8 量化

同样是Tim Dettmers，他在《LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale》]这篇文章中提出将vector-wise量化与混合精度分解结合的方法LLM.int8：

- vector-wise量化：为矩阵内积中独立的行、列向量设置各自的scaling constant。这种量化方式引入了更多的量化参数，量化的结果也更加精确。
- 混合精度分解：Transformer的 hidden state中存在一些绝对值很大的离群值，这种离群值对于数据分布的影响是重大的。混合精度分解将包含了离群值的几个维度从矩阵中分离出来，对其做高精度的矩阵乘法；其余部分进行量化。

### Block-wise k-bit Quantization

量化是将输入从包含更多信息的表示离散化到信息较少的表示的过程。它通常意味着采用具有更多位的数据类型并将其转换为更少的位，例如从 32 位浮点数到 8 位整数。为了确保使用低位数据类型的整个范围，通常通过输入元素的绝对最大值进行将输入数据归一化来重新缩放到目标数据类型范围内，这些元素通常结构化为张量。例如，将 32 位浮点 (FP32) 张量量化为范围为 [−127, 127] 的 Int8 张量：

这种方法的问题是，如果输入张量中出现大的数值（即异常值），则量化区间 （某些位组合）不能很好的利用， 在某些 bin 中量化数字很少或没有。为了防止异常值问题，一种常见的方法是将输入张量分块成独立量化的块，每个块都有自己的量化常数 c。这可以形式化如下：我们通过将输入张量展平并将线性段拆分为 n = (b × h)/B 块，将输入张量 X ∈ R （b×h ）分块成 n 个大小为 B 的连续块。我们使用等式 1 独立量化这些块，以创建量化张量和 n 个量化常数 ci。

### LoRA

低秩分解便是一种将高维稠密参数向量降维分解为稀疏的低维向量的方法。

LoRA的本质是在原模型的基础上插入若干新的参数，称之为adapter。在训练时，冻结原始模型的参数，只更新adapter的参数。对于不同的基座模型，adapter的参数量一般为几百万~几千万。

低秩自适应（LoRA）方法冻结了预训练的模型权重，并将可快速迭代的秩分解矩阵注入Transformer attention架构的Q、V中，从而大大减少了下游任务的可训练参数数量。

LoRA的优势很明显：

- 预训练模型参数可以共享并保持冻结，因此模型不容易发生灾难性遗忘。
- 秩分解矩阵的参数明显少于原始模型，根据不同的任务可以构建不同的小型LoRA模块，移植性很强。我们可以通过替换矩阵A和B来冻结共享模型并有效地切换任务，从而显著降低存储需求和任务切换开销。
- 当使用adapter时，因为我们不需要计算梯度或维护大多数参数的优化器状态，LoRA使显存开销下降。
- LoRA简单的线性设计允许我们在输出时将可训练矩阵与冻结权重合并即可，通过构造与完全微调的模型相比，LoRA不会引入推理延迟。
- LoRA与许多先前的方法正交，并且可以与其中的许多方法组合，例如 p-tuning。

LoRA 也有一些缺点：

- 参数空间小：LoRA中参与训练的参数量较少，解空间较小，效果相比全量微调有一定的差距。
- 微调大模型成本高：对于上百亿参数量的模型，LoRA微调的成本还是很高。
- 精度损失：LORA进行低秩分解时候可能会损失一些模型的表达能力和泛化能力。
- 量化精度损失：针对第二点，可以采用int8或int4量化，进一步对模型基座的参数进行压缩。但是又会引发精度损失的问题，降低模型性能。
- 参数调整：LORA微调方法可能会受到初始化和超参数的影响较大，需要进行适当的调整。

### PEFT

一个重要的讨论点是 LoRA 在训练期间的内存要求，无论是在使用的适配器的数量和大小方面。由于 LoRA 的内存占用非常小，我们可以使用更多的适配器来提高性能，而不会显着增加使用的总内存。如前所述，这对于恢复完整的 16 位精度性能至关重要。

## QLora 组件

QLora 通过两种技术实现了高保真 4 位微调——4-bit Normalfloat (NF4) 量化和双量化。此外，引入了 Paged Optimizer，以防止梯度检查点期间的内存峰值导致传统上对大型模型的单个机器进行微调的内存不足错误。

QLora 的量化过程如下：QLora有一个用于基本模型权重的低精度存储数据类型（NF4）和一个用于执行计算的计算数据类型（BF16）。QLora 将权重从存储数据类型反量化为计算数据类型 BFloat16以执行向前和向后传递， 但在传递过程中仅计算使用 BF16 的 LoRA 参数的权重梯度。权重仅在需要时解压缩，因此在训练和推理期间内存使用量保持较低。

### 4-bit NormalFloat（NF4）

4-bit NormalFloat（NF4） 是建立在分位数量化技术的基础之上的一种信息理论上最优的数据类型。可确保每个量化bin具有从输入张量分配的相同数量的值。分位数量化的工作原理是通过经验累积分布函数估计输入张量的分位数。

由于预训练的神经网络权值通常具有标准差为1的正态分布性质，因此我们可以通过缩放系数将所有的权值转换为固定期望值，从而使该分布完全适合我们的数据类型范围。一旦权重范围和数据类型范围匹配，我们就可以像往常一样进行量化。

分位数量化技术的主要思想便是将数值尽量落到均值为0，标准差为1的正态分布的固定期望值上。前面我们知道离群值对于模型量化的影响极其重要，而由于分位数估计算法的近似性质，精度量化对于离群值又有很大的误差。分位数量化技术使得每个量化分区中具有相等的期望值，相等的期望值可以避免昂贵的分位数估计和近似误差，使得精确的分位数估计在计算上可行。

分位数量化技术步骤如下：

- 估计N(0,1)分布的 2^k+1 个分位数，得到正态分布的k-bit位量化数据类型。
- 将其值归一化到[- 1,1]范围内
- 通过absmax去重新标准化[-1,1]，以获得k-bit的数据形式。

以NF4，即k=4为例，标准正态分布量化函数把[-1, 0]分成7份，然后生成[-1, ..., 0]共8个分位数, 把[0, 1]分成8份，然后生成[0, ..., 1]共9个分位数，两个合起来去掉一个0就生成全部的16个分位数了。

 创建了NF4数据类型的 2^4 =16 个值，并用零填充，以便在8位量化函数中使用（256个值，其中包括256-16个零）。该函数在bitsandbytes库中使用8位量化方法来“模拟”NF4。以下是更直观的解释：

> 我们的目标是找到等面积的量化区间，使得量化区间左右两侧的面积相等。这意味着我们不从正态分布的0和1量化区间开始，而是从一个偏移量量化区间开始。代码片段中称之为"offset"，其值为1-1/(215)。如果我们有一个非对称的数据类型，其中一侧的间隔等于每个量化区间周围的16个“半个”，而另一侧只有15个“半个”。因此，平均偏移量为(1-1/(215) + 1-1/(216))/2 = 0.9677083。

我们使用norm.ppf函数获取标准正态分布（N(0, 1)）的量化区间。然后，通过将这些量化区间的值除以绝对最大值来重新缩放它们。

### Double Quantization

Double Quantization是将额外的量化常数进行二次量化以减小内存开销的过程。例如每64个参数块共享一个32bit的量化常数， 这样的话相当于每一个参数的量化额外开销为32/64 = 0.5 bit。这个总体来说也是比较大的一个开销，所以为了进一步优化这个量化开销，我们对其进行二次量化(Double Quantization)，即把第一次32bit量化的输出作为第二次量化的输入，我们采用256的块大小对量化常数进行FP8量化，这样的话，我们可以把每个参数的量化额外开销降低到：8/64 + 32/(64 x 256) = 0.127 Bit

### Paged Optimizers

使用NVIDIA统一内存功能，该功能在CPU和GPU之间进行自动page对page传输，以便在GPU偶尔OOM的情况仍然下进行模型训练和微调。 可以理解成显存偶发OOM时，QLora会将优化器状态自动的驱逐到CPU RAM，当在优化器更新步骤中需要内存时，它们会被分页回GPU内存，从而保证训练正常训练下去。

Page Optimizer机制使得在GPU显存吃紧的时候可以把optimize转移到内存上，在需要更新optimizer状态时再加载回来，可以有效减少GPU显存的峰值占用，如想要达到在24gb上训练33B 参数模型这个机制是必须的。

## 实验结论

1.  NF4量化优于FP4和int4量化的性能，并且NF4+Double Quantization减少存储的同时，精度并未下降。
2. 单卡24G/48G训练33B/65B模型的时候，Paged optimziers 表现很好。
3. 使用默认的LoRA参数时，NF4训练达不到BF16的评估指标，QLora(4bit)需要LoRA应用到所有的transformer层才会有好的效果。
4. NF4+Double Quantization的方案finetune的模型精度可以媲美BFloat 16

## 总结

QLora方法有三个意义：

- QLora是第一种能够在单个消费者GPU上微调33B参数模型和在单个专业GPU上微调65B参数模型的方法，同时相比于full finetuning baseline并不会降低性能。这有助于大模型微调方法的普及，让更多的普通人和小团队参与到LLM研究中来
- 另一个是QLora为LLM在手机端微调训练带来了可能，作者估计在iPhone 12 Plus上, QLora可以在手机充电状态下每晚微调300万个token。
- QLora还可以帮助实现LLM的隐私保护，用户可以拥有和管理他们自己的数据和模型，同时LLM更容易部署。
