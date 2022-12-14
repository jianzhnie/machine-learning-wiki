## AutoML简述

机器学习的应用需要大量的人工干预，这些人工干预表现在：特征提取、模型选择、参数调节等机器学习的各个方面。AutoML 试图将这些与特征、模型、优化、评价有关的重要步骤进行自动化地学习，使得机器学习模型无需人工干预即可被应用.

机器学习在各种应用中的成功，导致对机器学习从业人员的需求不断增长，因此我们希望实现真正意义上的机器学习，让尽可能多的工作也能够被自动化完成，进一步降低机器学习的门槛，让没有该领域专业知识的人也可以使用机器学习来完成相关的工作。

AutoML应运而生。

从传统机器学习模型出发，AutoML从特征工程、模型构建、超参优化三方面实现自动化；并且也提出了end-to-end的解决方案。本专栏，贯彻AutoML的思想，将门槛降到最低，简略介绍原理，侧重介绍AutoML开源工具的使用方法。本篇文章主要对AutoML各个工具的优劣特性进行总结对比，关于AutoML各个工具的详情见专栏中细分文章。

**作者从机器学习和自动化两个角度给出了定义：**

1. 从机器学习角度讲，AutoML 可以看作是一个在给定数据和任务上学习和泛化能力非常强大的系统。但是它强调必须非常容易使用；

2. 从自动化角度讲，AutoML 则可以看作是设计一系列高级的控制系统去操作机器学习模型，使得模型可以自动化地学习到合适的参数和配置而无需人工干预。

对于机器学习的新用户而言，使用机器学习算法的一个主要的障碍就是算法的性能受许多的设计决策影响。随着深度学习的流行，工程师需要选择相应的神经网络架构，训练过程，正则化方法，超参数，等等，所有的这些都对算法的性能有很大的影响。于是深度学习工程师也被戏称为调参工程师。

自动机器学习（AutoML）的目标就是使用自动化的数据驱动方式来做出上述的决策。用户只要提供数据，自动机器学习系统自动的决定最佳的方案。领域专家不再需要苦恼于学习各种机器学习的算法。

自动机器学习不光包括大家熟知的算法选择，超参数优化，和神经网络架构搜索，还覆盖机器学习工作流的每一步：

![img](https://pic3.zhimg.com/80/v2-9a0242ae640da9712fa6fa0021c81ac6_1440w.jpg)

 AutoML的核心任务：

- Better performance
- No human assistance
- Lower computation budgets

## AutoML问题构成

AutoML的主要问题可以由三部分构成：特征工程、模型选择、算法选择。

## 特征工程

### **1.什么是特征工程**

**特征**是从现实世界的具体物体到用数值表示的抽象数值化变换。

**特征工程**是将原始数据转化为特征的过程，这些特征可以更好地向预测模型描述潜在问题，从而提高模型对未见数据的准确性。

**特征工程**通常包括三个工作：特征生成、特征选择、特征编码

- **特征选择** 在许多数据分析和建模项目中，数据科学家会收集到**成百上千个特征**。更糟糕的是，有时特征数目会大于样本数目。这种情况很普遍，但在大多数情况下，并**不是所有的变量都是与机器试图理解和建模的内容相关**的。所以数据科学家可以尝试设计一些有效的方法来**选择**那些重要的特征，并将它们合并到模型中，这叫做特征选择。
- **特征生成** 一般是在特征选择之前，它提取的对象是**原始数据**，目的就是自动地构建新的特征，**将原始数据转换为一组具有明显物理意义（比如 Gabor、几何特征、纹理特征）或者统计意义的特征**。
- **特征编码** 原始数据通常比较杂乱，可能会带有各种非数字特殊符号。**而实际上机器学习模型需要的数据是数字型的，因为只有数字类型才能进行计算。**因此，对于各种特殊的特征值，我们都需要对其进行相应的编码，也是**量化**的过程。

特征工程在机器学习中有着举足轻重的作用。在AutoML中，自动特征工程的目的是自动地发掘并构造相关的特征，使得模型可以有最优的表现。除此之外，还包含一些特定的特征增强方法，例如特征选择、特征降维、特征生成、以及特征编码等。这些步骤目前来说都没有达到自动化的阶段。

上述这些步骤也伴随着一定的参数搜索空间。第一种搜索空间是方法自带的，例如PCA自带降维参数需要调整。第二种是特征生成时会将搜索空间扩大。

### **2 .为什么需要自动特征工程**

在机器学习步骤中，特征工程会耗费数据科学家大量的人力去进行特征的提取和筛选，不仅耗费大量的时间，而且效率也不高。因此需要自动特征工程来将这些操作自动化，节省数据科学家的时间。

### 3.常见的特征工程工具总结和比较

本专栏调研了以下五种常用超参优化工具，并逐一撰写了报告发表在本专栏。

[特征工程工具总结(1)——Tsfresh](https://zhuanlan.zhihu.com/p/93310900)

[特征工程工具总结(2)——Trane](https://zhuanlan.zhihu.com/p/93311324)

[特征工程工具总结(3)——Categorical Encoding](https://zhuanlan.zhihu.com/p/93311921)

[特征工程工具总结(4)——boruta_py](https://zhuanlan.zhihu.com/p/93310121)

[特征工程工具总结(5)——Featuretools](https://zhuanlan.zhihu.com/p/89932693)

关于各个特征工程工具的细节可在以上链接中点击查阅。这里对这五种特征工程工具的功能、原理、优缺点等方面进行了评估了对比。

![img](https://pic1.zhimg.com/80/v2-acb8ec256ba7691e5304b4d1fcd2c500_1440w.jpg)

## 模型选择

模型选择包括两个步骤：选择一个模型，设定它的参数。相应地，AutoML的目的就是自动选择出一个最合适的模型，并且能够设定好它的最优参数。

## 算法选择

对于算法选择，AutoML的目的是自动地选择出一个优化算法，以便能够达到效率和精度的平衡。常用的优化方法有SGD、L-BFGS、GD等。使用哪个优化算法、对应优化算法的配置，也需要一组搜索空间。

## 从全局看

将以上三个关键步骤整合起来看，一个完整的AutoML过程可以分成这么两类：一类是将以上的三个步骤整合成一个完整的pipeline；另一类则是network architecture search，能够自动地学习到最优的网络结构。在学习的过程中，对以上三个问题都进行一些优化。

## 基本的优化策略

一旦搜索空间确定，我们便可以实用优化器(optimizer)进行优化。这里，AutoML主要回答三个问题： - 选择的优化器可以作用在哪个搜索空间上？ - 它需要什么样的反馈？ - 为了取得一个好的效果，它需要怎样的配置？

简单的优化搜索方式包括grid search和random search。其中grid search被广泛使用。

从样本中进行优化的方法主要包括启发式搜索、derivative-free优化、以及强化学习方法。梯度下降法是一种重要的优化策略。

## 评价策略

## 基本评价策略

在设计评价策略时，AutoML主要回答三个问题： - 这种策略能能够快速进行评价吗？ - 这种策略能够提供准确的评价吗？ - 这种策略需要怎样的反馈？

基本的评价策略包括： - 直接评价。直接在目标数据上进行评价。这是被使用最多的策略。 - 采样。当数据样本量非常大时，采样一些样本进行评价。 - Early stop。当遇到一些极端情况使得网络表现效果不好时，可以考虑进行early stop。 - 参数重用。将之前学习过的参数重复利用在新任务上。这在两种任务配置差不多时可用。 - 共轭评价。对于一些可量化的配置，可以用共轭评价法进行。

## 高级评价策略

高级评价策略主要包括两种：meta-learning和transfer learning。

- Meta-learning法。从先前的学习经验中提炼出基本的参数和结构配置。
- Transfer learning法。从先前的学习经验中提炼出可以重用的一些知识。

## 应用

- 使用Auto-sklearn进行模型选择。
- 使用强化学习进行neural architecture search。
- 使用ExploreKit进行自动特征构建。

## 展望

未来可能的研究方向：

- 提高AutoML的效率。
- 更明确的问题定义。
- 发展基本和高级的搜索策略。
- 找到更适合的应用。

我们对现有的自动机器学习开源框架展开了充分调研，它们一般作用于机器学习的单个或多个步骤。

自动机器学习框架能够处理项目中一部分步骤，包括模型构建、特征工程和超参数优化。因此，我们分析了一些成熟框架，它们宣称能优化其中某些任务组合。

我们挑选出一些能够容易应用到企业数据科学团队的函数库。在介绍每个框架时，我们都给出了它的建库目标、使用的统计方法和将其与新项目或现有项目整合时要考虑的主要因素。

![img](https://pic1.zhimg.com/80/v2-b426352d673406912235bed0361bc0f4_1440w.jpg)

## 超参数优化 Hyper-parameter Optimization

### 1.什么是超参数优化

学习器模型中一般有两类参数，一类是可以从数据中学习估计得到，还有一类参数时无法从数据中估计，只能靠人的经验进行设计指定，后者成为超参数。比如，支持向量机里面的C, Kernal, game；朴素贝叶斯里面的alpha等。

**超参数**是机器学习在学习之前预先设置好的参数，而非通过训练得到的参数，例如树的数量深度， 神经网络的学习率等，甚至在超参学习中神经网络的结构，包括层数，不同层的类型，层之间的连接方式等，都属于超参数的范畴。

手动修改调参既耗费大量的人力和时间，同时也难以寻找优化的方向，而对超参数选择进行优化既能节省大量人力和时间，又能让学习获得**更好的性能和效果**。因此出现了一系列的超参优化的工具来简化和改进超参选择和调整的过程。

超参数优化有很多方法：

最常见的类型是黑盒优化 （black-box function optimization）。所谓黑盒优化，就是将决策网络当作是一个黑盒来进行优化，仅关心输入和输出，而忽略其内部机制。决策网络通常是可以参数化的，这时候我们进行优化首先要考虑的是**收敛性。**

以下的几类方法都是属于黑盒优化：

- 网格搜索 （grid search）
  Grid search大家都应该比较熟悉，是一种通过遍历给定的参数组合来优化模型表现的方法。网格搜索的问题是很容易发生维度灾难，优点是很容易并行。
- 随机搜索 （random search）
  随机搜索是利用随机数求极小点而求得函数近似的最优解的方法。

![img](https://pic2.zhimg.com/80/v2-38d2ba328ff83b6bb0e40d0f02206891_1440w.jpg)


很多时候，随机搜索比网格搜索效果要更好，但是我们可以从上图看出，它们都不能保证找到最优解。

- [贝叶斯优化](https://en.wikipedia.org/wiki/Bayesian_optimization)
  贝叶斯优化是一种迭代的优化算法，包含两个主要的元素，**输入数据假设的模型**和一个**采集函数**用来来决定下一步要评估哪一个点。每一步迭代，都使用所有的观测数据fit模型，然后利用激活函数预测模型的概率分布，决定如何利用参数点，权衡是Explaoration还是Exploitation。相对于其它的黑盒优化算法，激活函数的计算量要少很多，这也是为什么贝叶斯优化被认为是更好的超参数调优的算法。

黑盒优化的一些工具：

- [hyperopt](https://github.com/hyperopt/hyperopt-sklearn)
  **hyperopt** 是一个Python库，可以用来寻找实数,离散值,条件维度等搜索空间的最佳值。
- [katib](https://github.com/kubeflow/katib)
  基于Kubernetes的超参数优化工具

由于优化目标具有不连续、不可导等数学性质，所以一些搜索和非梯度优化算法被用来求解该问题，包括我们上面提到的这些黑盒算法。此类算法通过采样和对采样的评价进行搜索，往往需要大量对采样的评价才能获得比较好的结果。然而，在自动机器学习任务中评价往往通过 k 折交叉验证获得，在大数据集的机器学习任务上，获得一个评价的时间代价巨大。这也影响了优化算法在自动机器学习问题上的效果。所以一些减少评价代价的方法被提出来，其中**多保真度优化（**multi-fidelity methods**）**就是其中的一种。这里的技术包括：基于学习曲线来决定是否要提前终止训练，探索-利用困境（exploration exploitation）的[多臂老虎机算法](https://en.wikipedia.org/wiki/Multi-armed_bandit) （Multi-armed bandit）等等。

另外还有一些研究是基于梯度下降的优化。

超参数优化面临许多挑战：

- 对于大规模的模型或者复杂的机器学习流水线而言，需要评估的空间规模非常大
- 配置空间很复杂
- 无法或者很难利用损失函数的梯度变化
- 训练集合的规模太小
- 很容易过拟合

## AutoML工具/项目资源列表

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020030110453929.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3h5czQzMDM4MV8x,size_16,color_FFFFFF,t_70)

### 十种超参优化工具的总结和比较

本专栏调研了以下十种常用超参优化工具，并逐一撰写了报告发表在本专栏。

[超参优化工具总结(1)——Skopt](https://zhuanlan.zhihu.com/p/93578702)

[超参优化工具总结(2)——Hyperopt](https://zhuanlan.zhihu.com/p/93583041)

[超参优化工具总结(3)——Simple(x)](https://zhuanlan.zhihu.com/p/93583733)

[超参优化工具总结(4)——Ray.tune](https://zhuanlan.zhihu.com/p/93584289)

[超参优化工具总结(5)——Chocolate](https://zhuanlan.zhihu.com/p/93589431)

[超参优化工具总结(6)——GpFlowOpt](https://zhuanlan.zhihu.com/p/93590655)

[超参优化工具总结(7)——FAR-HO](https://zhuanlan.zhihu.com/p/93591578)

[超参优化工具总结(8)——Xcessiv](https://zhuanlan.zhihu.com/p/93592370)

[超参优化工具总结(9)——HORD](https://zhuanlan.zhihu.com/p/93593103)

[超参优化工具总结(10)——ENAS](https://zhuanlan.zhihu.com/p/93594779)

[超参优化工具总结(11)——NNI](https://zhuanlan.zhihu.com/p/92812335)

关于各个超参优化工具的细节可在以上链接中点击查阅。这里从优化范围、基本配置、优化算法、各自的特点等五个方面对十一种工具进行了评估了对比。（注：表格增加了NNI的超参优化部分与其他超参优化工具的对比）

![img](https://pic2.zhimg.com/80/v2-e1f9d155191dee6e9c91ef03719c35e5_1440w.jpg)

并对10种超参优化工具评分如下：

![img](https://pic1.zhimg.com/80/v2-216562b71428d459e1f80708818fc490_1440w.jpg)评比规则：通用基础项（50%）+gihub评分（50%）+加分项（20%）

最终排名如下：

![img](https://pic1.zhimg.com/80/v2-4c7b53640873f2e03f2d286371b5c8a0_1440w.jpg)

## 四、AutoML开源框架分析

AutoML开源框架集成了特征工程模型构建以及超参优化，是一种全管道的AutoML工具。

本专栏调研了以下五个常用的AutoML开源框架，并逐一撰写了报告发表在本专栏。

[深度解析AutoML框架——MLBox：十行代码完成Titanic生存预测](https://zhuanlan.zhihu.com/p/85832939)

[深度解析AutoML框架——Autokeras：入门指南](https://zhuanlan.zhihu.com/p/85933783)

[深度解析AutoML框架——TPOT：一键生成ML代码，释放双手](https://zhuanlan.zhihu.com/p/85974357)

[深度解析AutoML框架——H2O：入门指南](https://zhuanlan.zhihu.com/p/87107981)

[深度解析AutoML框架——Auto_ml：初识AutoML的引路人](https://zhuanlan.zhihu.com/p/87196667)

关于各个开源框架的细节可在以上链接中点击查阅。这里从AutoML框架的特征工程、模型支持、超参优化、特性以及安装和上手难度五个方面最五个框架进行了评估了对比。

![img](https://pic1.zhimg.com/80/v2-a164d5efca1667eded185184d5b113b4_1440w.jpg)

可以看出，除了AutoKeras，其余框架均对特征工程有较强的支持，但支持的模型大多是传统机器模型。AutoKeras对神经网络支持较好，也因此对特征工程要求较低，它也是五种框架中唯一支持NAS的框架。TPOT支持显式地生成可执行代码，而MLBox甚至无法输出模型，只能输出对test的预测结果。H20是一款更加产品化的框架，有WebUI界面展示过程，对用户更加友好。auto_ml基础但全面，它更像是一个用于新手的教练机，没有特别的亮点但基本功能一应俱全。

------

在学术界，AutoML是一个活跃的新兴领域，大量关于AutoML的论文出现在机器学习会议与期刊中，同时也有很多开创性的开源项目（例如本文中列举的各类工具）受到大家的关注。在工业界，也有许多成熟的AutoML的产品，如专注特征工程的FeatureLab、包含NAS的Google's Cloud。这些产品都很大程度上帮助了机器学习从业人员将想法快速便捷的应用、落地。未来，AutoML能让机器学习自动化到什么程度、在算力需求上能否大众化、相关的优化理论能否加速整个学习过程？我们与您共同期待。
