# lab2_zh

6.830 实验2：SimpleDB操作符

分配日期：2021年3月9日（星期二）<br>
截止日期：2021年3月19日（星期五）晚上11:59（美国东部时间）

<!--
版本历史：

3/1/12：初始版本
-->

在这个实验作业中，您将为SimpleDB编写一组操作符来实现表修改（例如插入和删除记录）、选择、连接和聚合。这些操作符将建立在您在实验1编写的基础上，为您提供一个可以对多个表执行简单查询的数据库系统。

此外，我们在实验1中忽略了缓冲池管理的问题：当我们引用的页面数量超过数据库生命周期内内存所能容纳的数量时，我们还没有处理出现的问题。在实验2中，您将设计一个驱逐策略，从缓冲池中刷新陈旧的页面。

您在本实验中不需要实现事务或锁定。

本文档的其余部分提供了一些关于如何开始编码的建议，描述了一系列帮助您完成实验的练习，并讨论了如何提交您的代码。本实验需要您编写大量代码，因此我们鼓励您尽早开始！

<a name="starting"></a>

1. 开始

您应该从提交给实验1的代码开始（如果您没有提交实验1的代码，或者您的解决方案无法正常工作，请联系我们讨论选项）。此外，我们为本实验提供了额外的源文件和测试文件，这些文件不在您最初收到的代码分发中。

1.1. 获取实验2

您需要将这些新文件添加到您的发布中。最简单的方法是导航到您的项目目录（可能称为simple-db-hw）并从主GitHub仓库拉取：

$ cd simple-db-hw
$ git pull upstream master


IDE用户需要更新他们的项目依赖项以包含新的库jar文件。对于一个简单的解决方案，再次运行

ant eclipse


然后在Eclipse或IntelliJ中重新打开项目。如果您对项目设置进行了其他更改并且不想丢失它们，也可以手动添加依赖项。对于Eclipse，在包资源管理器下，右键单击项目名称（可能是<tt>simple-db-hw</tt>），然后选择属性。在左侧选择Java Build Path，在右侧单击Libraries选项卡。点击Add JARs...按钮，选择zql.jar和jline-0.9.94.jar，然后点击OK，接着点击OK。您的代码现在应该可以编译了。对于IntelliJ，转到File下的Project Structure，在Modules下选择<tt>simpledb</tt>项目，并导航到Dependencies选项卡。在窗格的底部，点击<tt>+</tt>图标将jar添加为编译时依赖项。

1.2. 实现提示

和以前一样，我们强烈鼓励您在编写代码之前阅读整篇文档，以了解SimpleDB的高级设计。

我们建议您按照本文档中的练习来指导您的实现，但您可能会发现不同的顺序对您来说更有意义。和以前一样，我们将通过查看您的代码并验证您是否通过了ant目标test和systemtest的测试来评分。请注意，代码只需要通过本实验中我们指出的测试，而不是所有的单元和系统测试。有关评分的完整讨论和您需要通过测试的列表，请参见第3.4节。

以下是您可能进行SimpleDB实现的一种大致方式；有关此大纲中步骤的更多详细信息，包括练习，将在下面的第2节给出。

• 实现操作符Filter和Join，并验证它们相应的测试是否正常工作。这些操作符的Javadoc注释包含了关于它们应该如何工作的详细信息。我们已经为您提供了Project和OrderBy的实现，这可以帮助您理解其他操作符的工作原理。

• 实现IntegerAggregator和StringAggregator。在这里，您将编写实际计算一系列输入元组中跨多个组的特定字段的聚合逻辑。由于SimpleDB只支持整数，计算平均值时使用整数除法。StringAggregator只需要支持COUNT聚合，因为其他操作对字符串没有意义。

• 实现Aggregate操作符。与其他操作符一样，聚合实现了OpIterator接口，以便可以将它们放置在SimpleDB查询计划中。注意，Aggregate操作符的输出是对每个next()调用整个组的聚合值，并且聚合构造函数接受聚合字段和分组字段。

• 在BufferPool中实现与元组插入、删除和页面驱逐相关的方法。此时您无需担心事务。

• 实现Insert和Delete操作符。像所有操作符一样，Insert和Delete实现了OpIterator，接受要插入或删除的元组流，并输出一个具有整数字段的单一元组，该字段指示插入或删除的元组数量。这些操作符需要调用BufferPool中实际修改磁盘页面的适当方法。检查插入和删除元组的测试是否正常工作。

请注意，SimpleDB没有实现任何类型的一致性或完整性检查，因此有可能将重复的记录插入文件中，并且没有办法强制执行主键或外键约束。

此时您应该能够通过ant目标systemtest中的测试，这是本实验的目标。

您还可以使用提供的SQL解析器针对您的数据库运行SQL查询！有关简要教程，请参见#parser。

最后，您可能会注意到本实验中的迭代器继承了Operator类，而不是实现OpIterator接口。由于<tt>next</tt>/<tt>hasNext</tt>的实现通常是重复、繁琐且容易出错，Operator通用地实现了这个逻辑，只需要您实现一个更简单的<tt>readNext</tt>。随意使用这种实现风格，或者如果您更喜欢，也可以只实现OpIterator接口。要实现OpIterator接口，请从迭代器类中移除extends Operator，并在其位置放上implements OpIterator。

2. SimpleDB架构和实现指南

2.1. Filter和Join

回想一下，SimpleDB的OpIterator类实现了关系代数的操作。现在您将实现两个操作符，使您能够执行比表扫描稍微有趣的查询。

• Filter：此操作符仅返回满足在其构造函数中指定的Predicate的元组。因此，它会过滤掉不匹配谓词的任何元组。

• Join：此操作符根据在其构造函数中传递的JoinPredicate连接其两个子节点的元组。我们只需要一个简单的嵌套循环连接，但您可以探索更有趣的连接实现。在实验报告中描述您的实现。

练习1。

实现以下文件中的骨架方法：

*

• src/java/simpledb/execution/Predicate.java

• src/java/simpledb/execution/JoinPredicate.java

• src/java/simpledb/execution/Filter.java

• src/java/simpledb/execution/Join.java

*

此时，您的代码应通过PredicateTest、JoinPredicateTest、FilterTest和JoinTest中的单元测试。此外，您应该能够通过FilterTest和JoinTest系统测试。

2.2. 聚合

一个额外的SimpleDB操作符实现了带有GROUP BY子句的基本SQL聚合。您应该实现五个SQL聚合（COUNT、SUM、AVG、MIN、MAX）并支持分组。您只需要支持对单个字段的聚合和按单个字段分组。

为了计算聚合，我们使用一个Aggregator接口，它将一个新元组合并到现有的聚合计算中。Aggregator在构造期间被告知它应该使用哪种操作进行聚合。随后，客户端代码应为子迭代器中的每个元组调用Aggregator.mergeTupleIntoGroup()。合并所有元组后，客户端可以检索聚合结果的OpIterator。结果中的每个元组都是形式为(groupValue, aggregateValue)的对，除非分组字段的值是Aggregator.NO_GROUPING，在这种情况下，结果是形式为(aggregateValue)的单个元组。

请注意，此实现需要与不同组数量成正比的线性空间。出于本实验的目的，您无需担心组数超过可用内存的情况。

练习2。

实现以下文件中的骨架方法：

*

• src/java/simpledb/execution/IntegerAggregator.java

• src/java/simpledb/execution/StringAggregator.java

• src/java/simpledb/execution/Aggregate.java

*

此时，您的代码应通过IntegerAggregatorTest、StringAggregatorTest和AggregateTest中的单元测试。此外，您应该能够通过AggregateTest系统测试。

2.3. HeapFile可变性

现在，我们将开始实现支持修改表的方法。我们从单个页面和文件的层面开始。有两组主要的操作：添加元组和删除元组。

删除元组： 要删除一个元组，您需要实现deleteTuple。元组包含RecordIDs，允许您找到它们所在的页面，因此这应该就像定位元组所属的页面并适当修改页面的标题一样简单。

添加元组： HeapFile.java中的insertTuple方法负责向堆文件添加元组。要向HeapFile添加新元组，您必须找到一个有空闲槽位的页面。如果HeapFile中没有这样的页面存在，则需要创建一个新页面并将其附加到磁盘上的物理文件中。您需要确保元组中的RecordID被正确更新。

练习3。

实现以下文件中的剩余骨架方法：

*

• src/java/simpledb/storage/HeapPage.java

• src/java/simpledb/storage/HeapFile.java<br>

（请注意，此时您不一定需要实现writePage）。

*

要实现HeapPage，您需要为诸如<tt>insertTuple()</tt>和<tt>deleteTuple()</tt>之类的方法修改标题位图。您可能会发现我们在实验1中要求您实现的<tt>getNumEmptySlots()</tt>和<tt>isSlotUsed()</tt>方法作为有用的抽象。请注意，提供了一个<tt>markSlotUsed</tt>方法作为抽象来修改页面标题中元组的填充或清除状态。

注意<tt>HeapFile.insertTuple()</tt>和<tt>HeapFile.deleteTuple()</tt>方法使用<tt>BufferPool.getPage()</tt>方法访问页面是很重要的；否则，您在下个实验中实现事务将无法正常工作。

在<tt>src/simpledb/BufferPool.java</tt>中实现以下骨架方法：

*

• insertTuple()

• deleteTuple()

*

这些方法应调用属于被修改表的HeapFile中的适当方法（这个额外的间接层级是为了未来支持其他类型的文件，比如索引）。

此时，您的代码应通过HeapPageWriteTest和HeapFileWriteTest以及BufferPoolWriteTest中的单元测试。

2.4. 插入和删除

既然您已经编写了所有HeapFile机制来添加和删除元组，您将实现Insert和Delete操作符。

对于实现insert和delete查询的计划，最顶层的操作符是一个特殊的Insert或Delete操作符，修改磁盘上的页面。这些操作符返回受影响的元组数量。这是通过返回一个具有一个整数字段的单一元组来实现的，该字段包含计数。

• Insert：此操作符将其从其子操作符读取的元组添加到其构造函数中指定的tableid中。它应使用BufferPool.insertTuple()方法来实现。

• Delete：此操作符从其构造函数中指定的tableid中删除其从其子操作符读取的元组。它应使用BufferPool.deleteTuple()方法来实现。

练习4。

实现以下文件中的骨架方法：

*

• src/java/simpledb/execution/Insert.java

• src/java/simpledb/execution/Delete.java

*

此时，您的代码应通过InsertTest中的单元测试。我们没有为Delete提供单元测试。此外，您应该能够通过InsertTest和DeleteTest系统测试。

2.5. 页面驱逐

在实验1中，我们没有正确观察由构造函数参数numPages定义的缓冲池中最大页面数的限制。现在，您将选择一个页面驱逐策略，并对任何读取或创建页面的先前代码进行检测以实现您的策略。

当缓冲池中有超过<tt>numPages</tt>个页面时，在加载下一页之前应从池中驱逐一个页面。驱逐策略的选择取决于您；没有必要做复杂的事情。在实验报告中描述您的策略。

注意BufferPool要求您实现一个flushAllPages()方法。在实际的缓冲池实现中，您永远不会需要这个方法。但是，我们需要这个方法用于测试目的。您永远不应从任何真正的代码中调用此方法。

由于我们实现ScanTest.cacheTest的方式，您需要确保您的flushPage和flushAllPages方法不从缓冲池中驱逐页面，以正确通过此测试。

flushAllPages应在BufferPool中的所有页面上调用flushPage，而flushPage应将任何脏页面写入磁盘并将其标记为非脏，同时将其保留在BufferPool中。

唯一应从缓冲池中删除页面的方法是evictPage，它应在任何驱逐的脏页面上调用flushPage。

练习5。

在以下文件中填写flushPage()方法和额外的辅助方法来实现页面驱逐：

*

• src/java/simpledb/storage/BufferPool.java

*

如果您在上面的<tt>HeapFile.java</tt>中没有实现writePage()，您也需要在这里实现它。最后，您还应实现discardPage()以从缓冲池中移除页面而不将其刷写到磁盘。我们在本实验中不会测试discardPage()，但在未来的实验室中它是必要的。

此时，您的代码应通过EvictionTest系统测试。

由于我们不会检查任何特定的驱逐策略，此测试通过创建一个具有16页的BufferPool（注意：虽然DEFAULT_PAGES是50，但我们正在用更少的页面初始化BufferPool！），扫描一个有许多超过16页的文件，并查看JVM的内存使用量是否增加了超过5 MB。如果您没有正确实现驱逐策略，您将不会驱逐足够的页面，并将超过大小限制，从而导致测试失败。

您现在已完成本实验。干得好！

<a name="query_walkthrough"></a>

2.6. 查询演练

以下代码实现了两个表之间的简单连接查询，每个表由三个整数列组成。（文件some_data_file1.dat和some_data_file2.dat是来自此文件的页面的二进制表示）。此代码相当于SQL语句：
SELECT *
FROM some_data_file1,
some_data_file2
WHERE some_data_file1.field1 = some_data_file2.field1
AND some_data_file1.id > 1


对于查询操作的更广泛示例，您可能会发现浏览连接、过滤器和聚合的单元测试有帮助。
package simpledb;

import java.io.*;

public class jointest {

    public static void main(String[] argv) {
        // 构造一个3列表的模式
        Type types[] = new Type[]{Type.INT_TYPE, Type.INT_TYPE, Type.INT_TYPE};
        String names[] = new String[]{"field0", "field1", "field2"};

        TupleDesc td = new TupleDesc(types, names);

        // 创建表，将其与数据文件关联
        // 并告诉目录有关表模式的信息。
        HeapFile table1 = new HeapFile(new File("some_data_file1.dat"), td);
        Database.getCatalog().addTable(table1, "t1");

        HeapFile table2 = new HeapFile(new File("some_data_file2.dat"), td);
        Database.getCatalog().addTable(table2, "t2");

        // 构造查询：我们使用两个SeqScans，它们通过迭代器将元组送入join
        TransactionId tid = new TransactionId();

        SeqScan ss1 = new SeqScan(tid, table1.getId(), "t1");
        SeqScan ss2 = new SeqScan(tid, table2.getId(), "t2");

        // 为where条件创建过滤器
        Filter sf1 = new Filter(
                new Predicate(0,
                        Predicate.Op.GREATER_THAN, new IntField(1)), ss1);

        JoinPredicate p = new JoinPredicate(1, Predicate.Op.EQUALS, 1);
        Join j = new Join(p, sf1, ss2);

        // 并运行它
        try {
            j.open();
            while (j.hasNext()) {
                Tuple tup = j.next();
                System.out.println(tup);
            }
            j.close();
            Database.getBufferPool().transactionComplete(tid);

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}


两个表都有三个整数字段。为了表达这一点，我们创建一个TupleDesc对象，并向其传递一个Type对象数组，指示字段类型，以及String对象数组，指示字段名称。一旦我们创建了这个TupleDesc，我们就初始化两个代表表的HeapFile对象。一旦我们创建了表，就将它们添加到Catalog中。（如果这是一个已经在运行的数据库服务器，我们将已经加载了此目录信息；我们只需要为此测试目的加载这个）。

初始化数据库系统后，我们创建一个查询计划。我们的计划由两个SeqScan操作符组成，它们扫描每个磁盘文件中的元组，连接到第一个HeapFile上的Filter操作符，再连接到根据JoinPredicate连接表中元组的Join操作符。通常，这些操作符使用对适当表（在SeqScan的情况下）或子操作符（在例如Join的情况下）的引用来实例化。然后，测试程序重复调用Join操作符上的next，后者依次从其子节点拉取元组。当元组从Join输出时，它们被打印到命令行上。

<a name="parser"></a>

2.7. 查询解析器

我们为您提供了一个SimpleDB的查询解析器，一旦您完成了本实验的练习，您就可以使用它来编写和运行针对数据库的SQL查询。

第一步是创建一些数据表和目录。假设您有一个文件data.txt，内容如下：

1,10
2,20
3,30
4,40
5,50
5,50


您可以使用convert命令将其转换为SimpleDB表（确保先输入<tt>ant</tt>！）：

java -jar dist/simpledb.jar convert data.txt 2 "int,int"


这将创建一个文件data.dat。除了表的原始数据外，两个额外的参数指定每条记录有两个字段，并且它们的类型是int和int。

接下来，创建一个目录文件catalog.txt，内容如下：

data (f1 int, f2 int)


这告诉SimpleDB有一个表data（存储在data.dat中），具有两个名为f1和f2的整数字段。

最后，调用解析器。您必须从命令行运行java（ant不能与交互式目标正常工作）。从simpledb/目录中输入：

java -jar dist/simpledb.jar parser catalog.txt


您应该看到类似以下的输出：

Added table : data with schema INT(f1), INT(f2),
SimpleDB>


最后，您可以运行一个查询：

SimpleDB> select d.f1, d.f2 from data d;
Started a new transaction tid = 1221852405823
ADDING TABLE d(data) TO tableMap
TABLE HAS  tupleDesc INT(d.f1), INT(d.f2),
1       10
2       20
3       30
4       40
5       50
5       50

6 rows.
----------------
0.16 seconds

SimpleDB>


解析器相对功能齐全（包括支持SELECT、INSERT、DELETE和事务），但确实存在一些问题，并且不一定报告完全有信息量的错误消息。以下是一些需要注意的限制：

• 您必须在每个字段名前加上其表名，即使字段名是唯一的（您可以如上例所示使用表名别名，但不能使用AS关键字。）

• WHERE子句中支持嵌套查询，但不支持FROM子句。

• 不支持算术表达式（例如，不能对两个字段求和。）

• 最多允许一个GROUP BY和一个聚合列。

• 不允许使用面向集合的操作符，如IN、UNION和EXCEPT。

• WHERE子句中只允许AND表达式。

• 不支持UPDATE表达式。

• 允许字符串操作符LIKE，但必须完整写出（也就是说，Postgres波浪号[~]简写不允许。）

3. 后勤

您必须提交您的代码（见下文）以及一份简短（最多2页）的说明文档，描述您的方法。此文档应：

• 描述您做出的任何设计决策，包括您选择的页面驱逐策略。如果您使用了嵌套循环连接以外的其他方法，请描述所选算法的权衡。

• 讨论并证明您对API所做的任何更改。

• 描述代码中任何缺失或不完整的部分。

• 描述您在实验上花费的时间，以及是否有任何特别困难或令人困惑的地方。

3.1. 合作

本实验单人应该可以完成，但如果您更愿意与合作伙伴一起工作，也可以。不允许更大的团队。请在您的个人文档中清楚说明您与谁（如果有的话）合作。

3.2. 提交您的作业

我们将使用gradescope自动评分所有编程作业。你们应该都被邀请加入课程实例；如果没有，请查看piazza获取邀请码。如果仍然有问题，请告诉我们，我们可以帮助您设置。您可以在截止日期前多次提交代码；我们将使用gradescope确定的最新版本。将文档放在名为lab2-writeup.txt的文件中，与您的提交一起。您还需要显式添加您创建的任何其他文件，例如新的*.java文件。

向gradescope提交的最简单方式是使用包含您代码的.zip文件。在Linux/MacOS上，您可以通过运行以下命令来实现：
$ zip -r submission.zip src/ lab2-writeup.txt


<a name="bugs"></a>

3.3. 提交错误

SimpleDB是一个相对复杂的代码库。很可能您会发现错误、不一致性以及糟糕、过时或不正确的文档等。

因此，我们要求您以冒险的心态来完成这个实验。如果某些内容不清楚甚至错误，请不要生气；而是尝试自己弄清楚或给我们发送一封友好的电子邮件。

请将（友好的！）错误报告提交至mailto:6.830-staff@mit.edu。提交时，请尝试包括：

• 错误描述。

• 一个<tt>.java</tt>文件，我们可以将其放入test/simpledb目录，编译并运行。

• 一个<tt>.txt</tt>文件，其中包含重现错误的数据。我们应该能够使用HeapFileEncoder将其转换为<tt>.dat</tt>文件。

如果您觉得自己遇到了错误，也可以在Piazza的班级页面上发布。

<a name="grading"></a>

3.4 评分

<p>您成绩的75%将基于您的代码是否通过我们将运行的系统测试套件。这些测试将是我们提供的测试的超集。在提交代码之前，您应确保它从<tt>ant test</tt>和<tt>ant systemtest</tt>中都不产生错误（通过所有测试）。

重要： 在测试之前，gradescope将用我们的版本替换您的<tt>build.xml</tt>、<tt>HeapFileEncoder.java</tt>和<tt>test</tt>目录的全部内容。这意味着您不能更改<tt>.dat</tt>文件的格式！您还应小心更改我们的API。您应测试您的代码是否能够编译未修改的测试。

提交后，您应立即从gradescope获得反馈和失败测试的错误输出（如果有）。给出的分数将是您作业自动评分部分的成绩。您成绩的另外25%将基于您的文档质量以及我们对您代码的主观评估。这部分也将在我们完成评分后发布在gradescope上。

我们设计这个作业时非常有趣，希望您也喜欢修改它！
