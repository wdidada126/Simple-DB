# lab3_zh
6.830 实验3：查询优化

分配日期：2021年3月17日（星期三）<br>
截止日期：2021年4月6日（星期二）

在这个实验中，您将在SimpleDB之上实现一个查询优化器。主要任务包括实现选择性估计框架和基于成本的优化器。您可以自由决定具体实现什么，但我们建议使用类似于课堂上讨论的Selinger基于成本的优化器（第9讲）。

本文档的其余部分描述了添加优化器支持涉及的内容，并提供了如何实现的基本概述。

与之前的实验一样，我们建议您尽早开始。

1. 开始

您应该从提交给实验2的代码开始（如果您没有提交实验2的代码，或者您的解决方案无法正常工作，请联系我们讨论选项）。

我们为您提供了额外的测试用例以及本实验的源代码文件，这些文件不在您最初收到的原始代码分发中。我们再次鼓励您在我们提供的测试之外，开发自己的测试套件。

您需要将这些新文件添加到您的发布中。最简单的方法是切换到您的项目目录（可能称为simple-db-hw）并从主GitHub仓库拉取：

$ cd simple-db-hw
$ git pull upstream master


1.1. 实现提示

我们建议您按照本文档中的练习来指导您的实现，但您可能会发现不同的顺序对您来说更有意义。和以前一样，我们将通过查看您的代码并验证您是否通过了ant目标<tt>test</tt>和<tt>systemtest</tt>的测试来评分。有关评分的完整讨论和您需要通过测试的列表，请参见第3.4节。

以下是您可能进行本实验的一种大致方式。下面第2节详细介绍了这些步骤。

• 实现<tt>TableStats</tt>类中的方法，使其能够使用直方图（为<tt>IntHistogram</tt>类提供了骨架）或您设计的其他形式的统计数据来估计过滤器的选择性和扫描的成本。

• 实现<tt>JoinOptimizer</tt>类中的方法，使其能够估计连接的成本和选择性。

• 编写<tt>JoinOptimizer</tt>中的<tt>orderJoins</tt>方法。该方法必须为一系列连接生成最优的顺序（可能使用Selinger算法），给定在前两步中计算的统计数据。

2. 优化器概述

回想一下，基于成本的优化器的主要思想是：

• 使用表的统计数据来估计不同查询计划的"成本"。通常，计划的成本与中间连接和选择的基数（产生的元组数）以及过滤器和连接谓词的选择性有关。

• 使用这些统计数据以最优方式排序连接和选择，并从多个备选方案中选择连接算法的最佳实现。

在本实验中，您将实现代码来执行这两个功能。

优化器将从<tt>simpledb/Parser.java</tt>中调用。在开始本实验之前，您可能希望回顾https://github.com/MIT-DB-Class/simple-db-hw-2021/blob/master/lab2.md#27-query-parser。简而言之，如果您有一个描述表的目录文件<tt>catalog.txt</tt>，您可以通过键入以下内容来运行解析器：

java -jar dist/simpledb.jar parser catalog.txt


当解析器被调用时，它将计算所有表的统计信息（使用您提供的统计代码）。当发出查询时，解析器会将查询转换为逻辑计划表示，然后调用您的查询优化器来生成最优计划。

2.1 整体优化器结构

在开始实现之前，您需要了解SimpleDB优化器的整体结构。解析器和优化器的SimpleDB模块的整体控制流如图1所示。

<p align="center">
<img width=400 src="controlflow.png"><br>
<i>图1：说明解析器中使用的类、方法和对象的图表</i>
</p>

底部的图例解释了符号；您将实现带有双边框的组件。类和方法的更多细节将在下文中解释（您可能希望参考此图），但基本操作如下：

1. <tt>Parser.java</tt>在初始化时构造一组表统计信息（存储在<tt>statsMap</tt>容器中）。然后它等待输入查询，并对该查询调用<tt>parseQuery</tt>方法。
2. <tt>parseQuery</tt>首先构造一个表示已解析查询的<tt>LogicalPlan</tt>。<tt>parseQuery</tt>然后调用它构造的<tt>LogicalPlan</tt>实例上的<tt>physicalPlan</tt>方法。<tt>physicalPlan</tt>方法返回一个<tt>DBIterator</tt>对象，可用于实际运行查询。

在接下来的练习中，您将实现帮助<tt>physicalPlan</tt>设计最优计划的方法。

2.2. 统计估计

准确估计计划成本相当棘手。在本实验中，我们将只关注连接序列和基表访问的成本。我们不会担心访问方法的选择（因为我们只有一个访问方法，表扫描）或其他操作符（如聚合）的成本。

在本实验中，您只需要考虑左深计划。有关您可能实现的额外"奖励"优化器功能的描述，包括处理丛生计划的方法，请参见第2.3节。

2.2.1 整体计划成本

我们将编写形式为p=t1 join t2 join ... tn的连接计划，这表示一个左深连接，其中t1是最左边的连接（树中最深的）。给定像p这样的计划，其成本可以表示为：

scancost(t1) + scancost(t2) + joincost(t1 join t2) +
scancost(t3) + joincost((t1 join t2) join t3) +
...


这里，scancost(t1)是扫描表t1的I/O成本，joincost(t1,t2)是连接t1到t2的CPU成本。为了使I/O和CPU成本具有可比性，通常使用一个常数缩放因子，例如：

cost(predicate application) = 1
cost(pageScan) = SCALING_FACTOR x cost(predicate application)


对于本实验，您可以忽略缓存的影响（例如，假设每次访问表都会产生完整的扫描成本）——这又是您可能作为可选奖励扩展添加到您的实验中的内容，在第2.3节中。因此，scancost(t1)就是t1 x SCALING_FACTOR中的页数。

2.2.2 连接成本

当使用嵌套循环连接时，回想一下两个表t1和t2（其中t1是外部表）之间的连接成本是：

joincost(t1 join t2) = scancost(t1) + ntups(t1) x scancost(t2) //IO成本
+ ntups(t1) x ntups(t2)  //CPU成本


这里，ntups(t1)是表t1中的元组数。

2.2.3 过滤器选择性

ntups可以通过扫描基表直接计算。对于有一个或多个选择谓词的表，估计ntups可能更棘手——这是过滤器选择性估计问题。以下是您可能使用的一种方法，基于计算表中值的直方图：

• 计算表中每个属性的最小值和（通过扫描一次）。

• 为表中的每个属性构造一个直方图。一种简单的方法是使用固定数量的桶NumB，每个桶表示直方图属性域中固定范围内的记录数。例如，如果字段f的范围从1到100，并且有10个桶，那么桶1可能包含记录数在1到10之间的计数，桶2包含记录数在11到20之间的计数，依此类推。

• 再次扫描表，选择所有元组的所有字段，并使用它们填充每个直方图中桶的计数。

• 要估计相等表达式f=const的选择性，计算包含值const的桶。假设桶的宽度（值范围）是w，高度（元组数）是h，表中的元组数是ntups。那么，假设值均匀分布在整个桶中，表达式的选择性大约是(h / w) / ntups，因为(h/w)表示具有值const的桶中的预期元组数。

• 要估计范围表达式f>const的选择性，计算const所在的桶b，宽度为w_b，高度为h_b。那么，b包含总元组的一部分<nobr>b_f = h_b / ntups</nobr>。假设元组均匀分布在整个b中，b中>的部分是<nobr>(b_right - const) / w_b</nobr>，其中b_right是b桶的右端点。因此，桶b为谓词贡献(b_f x b_part)选择性。此外，桶b+1...NumB-1贡献它们的全部选择性（可以使用类似于b_f的公式计算）。对所有桶的选择性贡献求和将得到表达式的整体选择性。图2说明了这个过程。

• 涉及小于的表达式的选择性可以类似于大于的情况执行，查看向下到0的桶。

<p align="center">
<img width=400 src="lab3-hist.png"><br>
<i>图2：说明您将在实验5中实现的直方图</i>
</p>

在接下来的两个练习中，您将编写代码来估计连接和过滤器的选择性。

*
练习1：IntHistogram.java

您需要实现某种方式来记录表统计信息以进行选择性估计。我们提供了一个骨架类<tt>IntHistogram</tt>来完成此操作。我们的意图是您使用上述基于桶的方法计算直方图，但只要它提供合理的选择性估计，您可以自由使用其他方法。

我们提供了一个类<tt>StringHistogram</tt>，它使用<tt>IntHistogram</tt>来计算字符串谓词的选择性。如果您想实现更好的估计器，可以修改<tt>StringHistogram</tt>，尽管您可能不需要这样做来完成本实验。

完成此练习后，您应该能够通过<tt>IntHistogramTest</tt>单元测试（如果您选择不实现基于直方图的选择性估计，则不需要通过此测试）。

*
练习2：TableStats.java

类<tt>TableStats</tt>包含计算表中元组和页数的方法，以及估计该表字段上谓词选择性的方法。我们创建的查询解析器为每个表创建<tt>TableStats</tt>的一个实例，并将这些结构传递到您的查询优化器中（您将在后续练习中需要）。

您应该在<tt>TableStats</tt>中填写以下方法和类：

• 实现<tt>TableStats</tt>构造函数：一旦您实现了跟踪统计信息（如直方图）的方法，您应该实现<tt>TableStats</tt>构造函数，添加代码扫描表（可能多次）以构建您需要的统计信息。

• 实现<tt>estimateSelectivity(int field, Predicate.Op op, Field constant)</tt>：使用您的统计信息（例如，根据字段类型使用<tt>IntHistogram</tt>或<tt>StringHistogram</tt>），估计表上谓词<tt>field op constant</tt>的选择性。

• 实现<tt>estimateScanCost()</tt>：此方法估计顺序扫描文件的成本，假设读取一页的成本是<tt>costPerPageIO</tt>。您可以假设没有寻道，并且没有页面在缓冲池中。此方法可以使用您在构造函数中计算的成本或大小。

• 实现<tt>estimateTableCardinality(double selectivityFactor)</tt>：此方法返回关系中元组的数量，给定应用了选择性为selectivityFactor的谓词。此方法可以使用您在构造函数中计算的成本或大小。

您可能希望修改<tt>TableStats.java</tt>的构造函数，例如，为选择性估计目的计算字段上的直方图，如上所述。

完成这些任务后，您应该能够通过<tt>TableStatsTest</tt>中的单元测试。

*

2.2.4 连接基数

最后，请注意上面计划<tt>p</tt>的成本包括形式为<tt>joincost((t1 join t2) join t3)</tt>的表达式。要评估此表达式，您需要某种方式估计<tt>t1 join t2</tt>的大小（<tt>ntups</tt>）。这个连接基数估计问题比过滤器选择性估计问题更难。在本实验中，您不需要为此做任何花哨的事情，尽管第2.4节中的可选练习之一包括一种基于直方图的连接选择性估计方法。

在实现简单解决方案时，您应记住以下几点：

• 对于相等连接，当其中一个属性是主键时，连接产生的元组数不能大于非主键属性的基数。

• 对于没有主键的相等连接，很难说明输出的大小——它可能是表基数的乘积（如果两个表的所有元组都具有相同的值）——或者可能是0。可以制定一个简单的启发式方法（例如，两个表中较大表的大小）。

• 对于范围扫描，同样很难准确说明大小。输出的大小应与输入的大小成比例。可以假设交叉积的固定部分由范围扫描发出（例如，30%）。通常，范围连接的成本应大于两个相同大小的表的非主键相等连接的成本。

*
练习3：连接成本估计

类<tt>JoinOptimizer.java</tt>包括用于排序和计算连接成本的所有方法。在此练习中，您将编写估计连接的选择性和成本的方法，具体来说：

• 实现<tt>estimateJoinCost(LogicalJoinNode j, int card1, int card2, double cost1, double cost2)</tt>：此方法估计连接j的成本，给定左输入的基数为card1，右输入的基数为card2，扫描左输入的成本为cost1，访问右输入的成本为card2。您可以假设连接是NL连接，并应用前面提到的公式。

• 实现<tt>estimateJoinCardinality(LogicalJoinNode j, int card1, int card2, boolean t1pkey, boolean t2pkey)</tt>：此方法估计连接j输出的元组数，给定左输入大小为card1，右输入大小为card2，标志t1pkey和t2pkey指示左侧和右侧（分别）字段是否唯一（主键）。

实现这些方法后，您应该能够通过<tt>JoinOptimizerTest.java</tt>中的单元测试<tt>estimateJoinCostTest</tt>和<tt>estimateJoinCardinality</tt>。

*

2.3 连接排序

现在您已经实现了估计成本的方法，您将实现Selinger优化器。对于这些方法，连接表示为一组连接节点（例如，两个表上的谓词），而不是如课堂上描述的一组要连接的关系。

将讲座中给出的算法转换为上面提到的连接节点列表形式，伪代码的大纲如下：

1. j = 连接节点集合
2. for (i in 1...|j|):
3.     for s in {j的长度为i的所有子集}
4.       bestPlan = {}
5.       for s' in {s的长度为d-1的所有子集}
6.            subplan = optjoin(s')
7.            plan = 将(s-s')连接到subplan的最佳方式
8.            if (cost(plan) < cost(bestPlan))
9.               bestPlan = plan
10.      optjoin(s) = bestPlan
11. return optjoin(j)


为了帮助您实现此算法，我们提供了几个类和方法来帮助您。首先，<tt>JoinOptimizer.java</tt>中的方法<tt>enumerateSubsets(List<T> v, int size)</tt>将返回<tt>v</tt>的大小为<tt>size</tt>的所有子集。此方法对于大集效率非常低；您可以通过实现更高效的枚举器来获得额外学分（提示：考虑使用原地生成算法和惰性迭代器（或流）接口来避免具体化整个幂集）。

其次，我们提供了方法：
private CostCard computeCostAndCardOfSubplan(Map<String, TableStats> stats,
Map<String, Double> filterSelectivities,
LogicalJoinNode joinToRemove,  
Set<LogicalJoinNode> joinSet,
double bestCostSoFar,
PlanCache pc)


给定连接的子集（<tt>joinSet</tt>）和要从此集中移除的连接（<tt>joinToRemove</tt>），此方法计算将<tt>joinToRemove</tt>连接到<tt>joinSet - {joinToRemove}</tt>的最佳方式。它在<tt>CostCard</tt>对象中返回此最佳方法，该对象包括成本、基数和最佳连接顺序（作为列表）。如果找不到任何计划（例如，因为没有可能的左深连接），或者如果所有计划的成本都大于<tt>bestCostSoFar</tt>参数，<tt>computeCostAndCardOfSubplan</tt>可能返回null。该方法使用先前连接的缓存<tt>pc</tt>（上面伪代码中的<tt>optjoin</tt>）来快速查找连接<tt>joinSet - {joinToRemove}</tt>的最快方式。其他参数（<tt>stats</tt>和<tt>filterSelectivities</tt>）传递到您必须作为练习4的一部分实现的<tt>orderJoins</tt>方法，并在下面解释。此方法基本上执行前面描述的伪代码的第6-8行。

第三，我们提供了方法：
private void printJoins(List<LogicalJoinNode> js,
PlanCache pc,
Map<String, TableStats> stats,
Map<String, Double> selectivities)


此方法可用于显示连接计划的图形表示（当通过优化器的"-explain"选项设置"explain"标志时，例如）。

第四，我们提供了一个类<tt>PlanCache</tt>，可用于缓存到目前为止在您的Selinger实现中连接考虑的连接的子集的最佳方式（需要此类的实例才能使用<tt>computeCostAndCardOfSubplan</tt>）。

*
练习4：连接排序

在<tt>JoinOptimizer.java</tt>中，实现方法：
List<LogicalJoinNode> orderJoins(Map<String, TableStats> stats,
Map<String, Double> filterSelectivities,  
boolean explain)


此方法应在<tt>joins</tt>类成员上操作，返回一个指定连接应执行顺序的新列表。此列表的第0项表示左深计划中最左边、最底部的连接。返回列表中的相邻连接应至少共享一个字段以确保计划是左深的。这里<tt>stats</tt>是一个对象，允许您查找查询<tt>FROM</tt>列表中出现的给定表名的<tt>TableStats</tt>。<tt>filterSelectivities</tt>允许您查找表上任何谓词的选择性；它保证在<tt>FROM</tt>列表中每个表名有一个条目。最后，<tt>explain</tt>指定您应输出连接顺序的表示以提供信息。

您可能希望使用上述帮助方法和类来帮助实现。大致上，您的实现应遵循上述伪代码，循环遍历子集大小、子集和子集的子计划，调用<tt>computeCostAndCardOfSubplan</tt>并构建一个存储执行每个子集连接的最小成本方式的<tt>PlanCache</tt>对象。

实现此方法后，您应该能够通过<tt>JoinOptimizerTest</tt>中的所有单元测试。您还应通过系统测试<tt>QueryTest</tt>。

2.4 额外学分

在本节中，我们描述了几个可选练习，您可以实现以获得额外学分。这些练习比前面的练习定义不明确，但让您有机会展示您对查询优化的掌握！请在报告中清楚标记您选择完成哪些练习，并简要解释您的实现并展示您的结果（基准数字、经验报告等）。

*
奖励练习。 每个这些奖励都值得最多5%的额外学分：

• 添加代码以执行更高级的连接基数估计。不是使用简单的启发式方法来估计连接基数，设计一种更复杂的算法。

• 一种选择是在每对表t1和t2中的每对属性a和b之间使用联合直方图。想法是创建a的桶，对于a的每个桶A，创建与A中的a值共同出现的b值的直方图。

• 另一种估计连接基数的方法是假设较小表中的每个值在较大表中都有匹配值。那么连接选择性的公式将是：1/(Max(num-distinct(t1, column1), num-distinct(t2, column2)))。这里，column1和column2是连接属性。然后连接的基数是t1和t2的基数乘以选择性。<br>

• 改进的子集迭代器。我们的<tt>enumerateSubsets</tt>实现效率非常低，因为它在每次调用时创建大量Java对象。在此奖励练习中，您将改进<tt>enumerateSubsets</tt>的性能，以便您的系统可以对具有20个或更多连接的计划进行查询优化（目前这样的计划需要几分钟或几小时来计算）。

• 考虑缓存成本模型。估计扫描和连接成本的方法不考虑缓冲池中的缓存。您应扩展成本模型以考虑缓存效应。这很棘手，因为由于迭代器模型，多个连接同时运行，因此可能难以预测每个连接使用我们在先前实验中实现的简单缓冲池可以访问多少内存。

• 改进的连接算法和算法选择。我们当前的成本估计和连接操作符选择算法（参见<tt>JoinOptimizer.java</tt>中的<tt>instantiateJoin()</tt>）只考虑嵌套循环连接。扩展这些方法以使用一种或多种额外的连接算法（例如，使用<tt>HashMap</tt>的某种形式的内存哈希）。

• 丛生计划。改进提供的<tt>orderJoins()</tt>和其他辅助方法以生成丛生连接。我们的查询计划生成和可视化算法完全能够处理丛生计划；例如，如果<tt>orderJoins()</tt>返回列表(t1 join t2; t3 join t4; t2 join t3)，这将对应于顶部具有(t2 join t3)节点的丛生计划。

*

您现在已经完成了本实验。
干得好！

3. 后勤

您必须提交您的代码（见下文）以及一份简短（最多2页）的说明文档，描述您的方法。此文档应：

• 描述您做出的任何设计决策，包括选择性估计、连接排序的方法，以及您选择实现的任何奖励练习以及如何实现它们（对于每个奖励练习，您可以提交最多1页的附加内容）。

• 讨论并证明您对API所做的任何更改。

• 描述代码中任何缺失或不完整的部分。

• 描述您在实验上花费的时间，以及是否有任何特别困难或令人困惑的地方。

• 描述您已完成的任何额外学分实现。

3.1. 合作

本实验单人应该可以完成，但如果您更愿意与合作伙伴一起工作，也可以。不允许更大的团队。请在您的文档中清楚说明您与谁（如果有的话）合作。

3.2. 提交您的作业

我们将使用gradescope自动评分所有编程作业。你们应该都被邀请加入课程实例；如果没有，请告诉我们，我们可以帮助您设置。您可以在截止日期前多次提交代码；我们将使用gradescope确定的最新版本。将文档放在名为lab3-writeup.txt的文件中，与您的提交一起。您还需要显式添加您创建的任何其他文件，例如新的*.java文件。

向gradescope提交的最简单方式是使用包含您代码的.zip文件。在Linux/MacOS上，您可以通过运行以下命令来实现：
$ zip -r submission.zip src/ lab3-writeup.txt


<a name="bugs"></a>
3.3. 提交错误

SimpleDB是一个相对复杂的代码库。很可能您会发现错误、不一致性以及糟糕、过时或不正确的文档等。

因此，我们要求您以冒险的心态来完成这个实验。如果某些内容不清楚甚至错误，请不要生气；而是要尝试自己弄清楚或给我们发送一封友好的电子邮件。

请将（友好的！）错误报告提交至<a href="mailto:6.830-staff@mit.edu">6.830-staff@mit.edu</a>。提交时，请尝试包括：

• 错误描述。

• 一个<tt>.java</tt>文件，我们可以将其放入test/simpledb目录，编译并运行。

• 一个<tt>.txt</tt>文件，其中包含重现错误的数据。我们应该能够使用HeapFileEncoder将其转换为<tt>.dat</tt>文件。

如果您觉得自己遇到了错误，也可以在Piazza的班级页面上发布。

3.4 评分

<p>您成绩的75%将基于您的代码是否通过我们将运行的系统测试套件。这些测试将是我们提供的测试的超集。在提交代码之前，您应确保它从<tt>ant test</tt>和<tt>ant systemtest</tt>中都不产生错误（通过所有测试）。

重要： 在测试之前，gradescope将用我们的版本替换您的<tt>build.xml</tt>、<tt>HeapFileEncoder.java</tt>和<tt>test</tt>目录的全部内容。这意味着您不能更改<tt>.dat</tt>文件的格式！您还应小心更改我们的API。您应测试您的代码是否能够编译未修改的测试。

提交后，您应立即从gradescope获得反馈和失败测试的错误输出（如果有）。给出的分数将是您作业自动评分部分的成绩。您成绩的另外25%将基于您的文档质量以及我们对您代码的主观评估。这部分也将在我们完成评分后发布在gradescope上。

我们设计这个作业时非常有趣，希望您也喜欢修改它！
