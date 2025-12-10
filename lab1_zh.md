# lab1_zh

6.830/6.814 实验1：SimpleDB

分配日期：2月24日（星期三）

截止日期：3月10日（星期三）晚上11:59（美国东部时间）

<!--
Bug更新： 我们有一个bugs.html来记录您或我们发现的SimpleDB错误。对错误/烦人问题的修复也会发布在那里。有些错误可能已经被发现，所以请查看该页面以获取实验代码的最新版本/补丁。
-->

在6.830的实验作业中，您将编写一个名为SimpleDB的基本数据库管理系统。对于本实验，您将专注于实现访问磁盘存储数据所需的核心模块；在未来的实验中，您将添加对各种查询处理操作符以及事务、锁定和并发查询的支持。

SimpleDB是用Java编写的。我们为您提供了一组大部分未实现的类和接口。您需要为这些类编写代码。我们将通过使用http://junit.sourceforge.net/编写的一组系统测试来运行您的代码进行评分。我们还提供了许多单元测试，虽然不会用于评分，但您可能会发现它们有助于验证您的代码是否正常工作。除了我们的测试外，我们还鼓励您开发自己的测试套件。

本文档的其余部分描述了SimpleDB的基本架构，提供了一些关于如何开始编码的建议，并讨论了如何提交您的实验。

我们强烈建议您尽早开始本实验。它需要您编写相当多的代码！

<!--

0. 查找错误，保持耐心，获得糖果！

SimpleDB是一个相对复杂的代码库。
很可能您会发现错误、不一致性以及糟糕、过时或不正确的文档等。

因此，我们要求您以冒险的心态来完成这个实验。如果某些内容不清楚甚至错误，请不要生气；而是要尝试自己弄清楚或给我们发一封友好的电子邮件。我们承诺会在报告错误和问题时发布错误修复、向HW仓库提交新提交等。

<p>……如果您在我们的代码中发现错误，我们会给您一块糖果（参见#bugs）！

-->
<!--您可以在bugs.html找到。-->

0. 环境设置

首先按照https://github.com/MIT-DB-Class/simple-db-hw-2021的说明从课程GitHub仓库下载实验1的代码。

这些说明是为Athena或任何其他基于Unix的平台（例如Linux、MacOS等）编写的。因为代码是用Java编写的，所以在Windows下也应该可以工作，尽管本文档中的说明可能不适用。

我们包含了#eclipse关于在Eclipse或IntelliJ中使用项目的说明。

1. 开始

SimpleDB使用http://ant.apache.org/来编译代码和运行测试。Ant类似于http://www.gnu.org/software/make/manual/，但构建文件是用XML编写的，并且更适合Java代码。大多数现代Linux发行版都包含Ant。在Athena下，它包含在sipb储物柜中，您可以通过在Athena提示符下键入add sipb来使用。请注意，在某些版本的Athena上，您还必须运行add -f java来为Java程序正确设置环境。有关更多详细信息，请参阅http://web.mit.edu/acs/www/languages.html#Java。

为了帮助您在开发过程中进行调试，除了用于评分的端到端测试外，我们还提供了一组单元测试。这些测试绝不是全面的，您不应仅仅依赖它们来验证项目的正确性（请运用6.170课程学到的技能！）。

要运行单元测试，请使用test构建目标：

$ cd [项目目录]
$ # 运行所有单元测试
$ ant test
$ # 运行特定的单元测试
$ ant runtest -Dtest=TupleTest


您应该看到类似于以下的输出：

构建输出...

test:
[junit] Running simpledb.CatalogTest
[junit] Testsuite: simpledb.CatalogTest
[junit] Tests run: 2, Failures: 0, Errors: 2, Time elapsed: 0.037 sec
[junit] Tests run: 2, Failures: 0, Errors: 2, Time elapsed: 0.037 sec

... 堆栈跟踪和错误报告 ...


上述输出表明在编译过程中发生了两个错误；这是因为我们给您的代码尚未正常工作。当您完成实验的各个部分时，您将逐步通过更多的单元测试。

如果您希望在编码时编写新的单元测试，应将它们添加到<tt>test/simpledb</tt>目录中。

<p>有关如何使用Ant的更多详细信息，请参阅http://ant.apache.org/manual/。http://ant.apache.org/manual/running.html部分提供了有关使用ant命令的详细信息。但是，下面的快速参考表应该足以完成实验。

命令 | 描述
--- | ---
ant | 构建默认目标（对于simpledb，这是dist）。
ant -projecthelp | 列出build.xml中的所有目标及其描述。
ant dist | 编译src中的代码并将其打包到dist/simpledb.jar中。
ant test | 编译并运行所有单元测试。
ant runtest -Dtest=测试名称 | 运行名为testname的单元测试。
ant systemtest | 编译并运行所有系统测试。
ant runsystest -Dtest=测试名称 | 编译并运行名为testname的系统测试。

如果您在Windows系统下并且不想从命令行运行ant测试，也可以从Eclipse中运行它们。右键单击build.xml，在targets选项卡中，您可以看到"runtest"、"runsystest"等。例如，选择runtest等同于从命令行运行"ant runtest"。诸如"-Dtest=testname"的参数可以在"Main"选项卡的"Arguments"文本框中指定。请注意，您也可以通过从build.xml复制、修改目标和参数并重命名为（例如）runtest_build.xml来创建运行测试的快捷方式。

1.1. 运行端到端测试

我们还提供了一组端到端测试，这些测试最终将用于评分。这些测试是作为JUnit测试构建的，位于<tt>test/simpledb/systemtest</tt>目录中。要运行所有系统测试，请使用systemtest构建目标：

$ ant systemtest

... 构建输出 ...

    [junit] Testcase: testSmall took 0.017 sec
    [junit]     Caused an ERROR
    [junit] expected to find the following tuples:
    [junit]     19128
    [junit] 
    [junit] java.lang.AssertionError: expected to find the following tuples:
    [junit]     19128
    [junit] 
    [junit]     at simpledb.systemtest.SystemTestUtil.matchTuples(SystemTestUtil.java:122)
    [junit]     at simpledb.systemtest.SystemTestUtil.matchTuples(SystemTestUtil.java:83)
    [junit]     at simpledb.systemtest.SystemTestUtil.matchTuples(SystemTestUtil.java:75)
    [junit]     at simpledb.systemtest.ScanTest.validateScan(ScanTest.java:30)
    [junit]     at simpledb.systemtest.ScanTest.testSmall(ScanTest.java:40)

... 更多错误信息 ...


<p>这表明测试失败，并显示了检测到错误的堆栈跟踪。要进行调试，首先阅读发生错误的源代码。当测试通过时，您将看到类似以下的内容：

$ ant systemtest

... 构建输出 ...

    [junit] Testsuite: simpledb.systemtest.ScanTest
    [junit] Tests run: 3, Failures: 0, Errors: 0, Time elapsed: 7.278 sec
    [junit] Tests run: 3, Failures: 0, Errors: 0, Time elapsed: 7.278 sec
    [junit] 
    [junit] Testcase: testSmall took 0.937 sec
    [junit] Testcase: testLarge took 5.276 sec
    [junit] Testcase: testRandom took 1.049 sec

BUILD SUCCESSFUL
Total time: 52 seconds


1.1.1 创建虚拟表

您可能希望创建自己的测试和自己的数据表来测试您自己实现的SimpleDB。您可以创建任何<tt>.txt</tt>文件，并使用以下命令将其转换为SimpleDB的HeapFile格式的<tt>.dat</tt>文件：

$ java -jar dist/simpledb.jar convert 文件.txt N


其中<tt>file.txt</tt>是文件名，<tt>N</tt>是文件中的列数。请注意，<tt>file.txt</tt>必须采用以下格式：

int1,int2,...,intN
int1,int2,...,intN
int1,int2,...,intN
int1,int2,...,intN


...其中每个intN都是一个非负整数。

要查看表的内容，请使用print命令：

$ java -jar dist/simpledb.jar print 文件.dat N


其中<tt>file.dat</tt>是使用<tt>convert</tt>命令创建的表的名称，<tt>N</tt>是文件中的列数。

<a name="eclipse"></a>

1.2. 使用IDE

IDE（集成开发环境）是图形化的软件开发环境，可以帮助您管理较大的项目。我们提供了设置http://www.eclipse.org和https://www.jetbrains.com/idea/的说明。我们为Eclipse提供的说明是使用带Java 1.7的Eclipse for Java Developers（非企业版）生成的。对于IntelliJ，我们使用的是Ultimate版，您可以通过mit.edu帐户https://www.jetbrains.com/community/education/#students获得教育许可。我们强烈建议您为这个项目设置并学习其中一个IDE。

准备代码库

运行以下命令为IDE生成项目文件：

ant eclipse


在Eclipse中设置实验

• 安装Eclipse后，启动它，第一个屏幕会要求您选择工作空间的位置（我们将此目录称为$W）。选择包含simple-db-hw仓库的目录。

• 在Eclipse中，选择File->New->Project->Java->Java Project，然后点击Next。

• 输入"simple-db-hw"作为项目名称。

• 在输入项目名称的同一屏幕上，选择"Create project from existing source"，并浏览到$W/simple-db-hw。

• 点击Finish，您应该能够在左侧屏幕的Project Explorer选项卡中看到"simple-db-hw"作为一个新项目。打开此项目会显示上面讨论的目录结构——实现代码可以在"src"中找到，单元测试和系统测试在"test"中找到。

注意： 本课程假定您使用的是Oracle官方发布的Java版本。这在MacOS X和大多数Windows Eclipse安装中是默认的；但许多Linux发行版默认使用替代的Java运行时（如OpenJDK）。请从http://www.oracle.com/technetwork/java/javase/downloads/index.html下载最新的Java8更新，并使用该Java版本。如果不切换，在以后的实验中的某些性能测试中可能会看到虚假的测试失败。

运行单个单元测试和系统测试

要运行单元测试或系统测试（两者都是JUnit测试，可以以相同的方式初始化），请转到屏幕左侧的Package Explorer选项卡。在"simple-db-hw"项目下，打开"test"目录。单元测试位于"simpledb"包中，系统测试位于"simpledb.systemtests"包中。要运行其中一个测试，选择测试（它们都叫做*Test.java——不要选择TestUtil.java或SystemTestUtil.java），右键单击它，选择"Run As"，然后选择"JUnit Test"。这将弹出一个JUnit选项卡，显示JUnit测试套件中各个测试的状态，并显示异常和其他错误，帮助您调试问题。

运行Ant构建目标

如果要运行诸如"ant test"或"ant systemtest"之类的命令，请右键单击Package Explorer中的build.xml。选择"Run As"，然后选择"Ant Build..."（注意：选择带有省略号（...）的选项，否则不会显示要运行的构建目标集）。然后，在下一个屏幕的"Targets"选项卡中，勾选要运行的目标（可能是"dist"和"test"或"systemtest"中的一个）。这将运行构建目标，并在Eclipse的控制台窗口中显示结果。

在IntelliJ中设置实验

IntelliJ是一个更现代的Java IDE，据某些说法更受欢迎且更直观。要使用IntelliJ，首先安装并打开应用程序。与Eclipse类似，在项目下，选择Open并导航到您的项目根目录。双击.project文件（您可能需要配置操作系统以显示隐藏文件才能看到它），然后点击"open as project"。IntelliJ具有Ant的工具窗口支持，您可能需要根据https://www.jetbrains.com/help/idea/ant.html的说明进行设置，但这对于开发不是必需的。您可以在https://www.jetbrains.com/help/idea/discover-intellij-idea.html找到IntelliJ功能的详细演练。

1.3. 实现提示

在开始编写代码之前，我们强烈鼓励您阅读整篇文档，以了解SimpleDB的高级设计。

<p>

您需要填写任何未实现的代码部分。我们将明确指出我们认为您应该编写代码的地方。您可能需要添加私有方法和/或辅助类。您可以更改API，但请确保我们的#grading测试仍然运行，并确保在您的文档中提及、解释和证明您的决策。

<p>

除了本实验需要填写的方法外，类接口还包含许多方法，您无需实现，直到后续实验。这些将按类指示：
// 对于lab1不是必需的。
public class Insert implements DbIterator {


或按方法指示：
public boolean deleteTuple(Tuple t)throws DbException{
// 这里有一些代码
// 对于lab1不是必需的
return false;
}


您提交的代码应该无需修改这些方法即可编译。

<p>

我们建议您按照本文档中的练习来指导您的实现，但您可能会发现不同的顺序对您来说更有意义。

以下是您可能进行SimpleDB实现的一种大致方式：

*

• 实现管理元组的类，即Tuple、TupleDesc。我们已经为您实现了Field、IntField、StringField和Type。因为您只需要支持整数和（固定长度）字符串字段以及固定长度元组，这些都很简单。

• 实现Catalog（这应该非常简单）。

• 实现BufferPool构造函数和getPage()方法。

• 实现访问方法，HeapPage和HeapFile以及相关的ID类。这些文件的大部分已经为您编写。

• 实现运算符SeqScan。

• 此时，您应该能够通过ScanTest系统测试，这是本实验的目标。

*

下面的第2节更详细地指导您完成这些实现步骤以及每个步骤对应的单元测试。

1.4. 事务、锁定和恢复

当您查看我们提供的接口时，您会看到许多关于锁定、事务和恢复的引用。在本实验中，您不需要支持这些功能，但您应该在代码的接口中保留这些参数，因为在未来的实验中您将实现事务和锁定。我们为您提供的测试代码生成一个假的事务ID，该ID传递给其运行的查询的操作符；您应该将此事务ID传递给其他操作符和缓冲池。

2. SimpleDB架构和实现指南

SimpleDB由以下部分组成：

• 表示字段、元组和元组模式的类；

• 将谓词和条件应用于元组的类；

• 一个或多个访问方法（例如堆文件），在磁盘上存储关系并提供迭代这些关系元组的方式；

• 处理元组的操作符类集合（例如，选择、连接、插入、删除等）；

• 缓存内存中活动元组和页面并处理并发控制和事务的缓冲池（本实验您无需担心这两者）；以及，

• 存储可用表及其模式信息的目录。

SimpleDB不包含许多您可能认为是"数据库"一部分的内容。特别是，SimpleDB没有：

• （在本实验中）SQL前端或解析器，允许您直接向SimpleDB键入查询。相反，查询是通过将一组操作符链接在一起形成手动构建的查询计划来构建的（参见#query_walkthrough）。我们将在以后的实验中提供一个简单的解析器。

• 视图。

• 除整数和固定长度字符串以外的数据类型。

• （在本实验中）查询优化器。

• （在本实验中）索引。

<p>

在本节的其余部分，我们将描述本实验中您需要实现的SimpleDB的主要组件。您应该使用此讨论中的练习来指导您的实现。本文档绝不是SimpleDB的完整规范；您需要就如何设计和实现系统的各个部分做出决策。请注意，对于实验1，您不需要实现除顺序扫描外的任何操作符（例如选择、连接、投影）。您将在未来的实验中添加对其他操作符的支持。

<p>

2.1. Database类

Database类提供对静态对象集合的访问，这些对象是数据库的全局状态。特别是，这包括访问目录（数据库中所有表的列表）、缓冲池（当前驻留在内存中的数据库文件页面的集合）和日志文件的方法。本实验中您无需担心日志文件。我们已经为您实现了Database类。您应该查看此文件，因为您将需要访问这些对象。

2.2. 字段和元组

<p>SimpleDB中的元组非常基础。它们由Field对象的集合组成，Tuple中的每个字段一个。Field是一个接口，不同的数据类型（例如整数、字符串）实现它。Tuple对象由底层的访问方法（例如堆文件或B树）创建，如下一节所述。元组还有一个类型（或模式），称为_元组描述符_，由TupleDesc对象表示。该对象由Type对象的集合组成，元组中的每个字段一个，每个描述相应字段的类型。

练习1

实现以下文件中的骨架方法：
*

• src/java/simpledb/storage/TupleDesc.java

• src/java/simpledb/storage/Tuple.java

*

此时，您的代码应通过单元测试TupleTest和TupleDescTest。此时，modifyRecordId()应该失败，因为您尚未实现它。

2.3. Catalog

目录（SimpleDB中的Catalog类）由数据库中当前存在的表及其模式的列表组成。您需要支持添加新表以及获取特定表信息的能力。与每个表关联的是一个TupleDesc对象，允许操作符确定表中字段的类型和数量。

全局目录是单个Catalog实例，为整个SimpleDB进程分配。可以通过方法Database.getCatalog()检索全局目录，全局缓冲池也是如此（使用Database.getBufferPool()）。

练习2

实现以下文件中的骨架方法：
*

• src/java/simpledb/common/Catalog.java

*

此时，您的代码应通过CatalogTest中的单元测试。

2.4. BufferPool

<p>缓冲池（SimpleDB中的BufferPool类）负责缓存最近从磁盘读取到内存中的页面。所有操作符都通过缓冲池从磁盘上的各种文件读取和写入页面。它由固定数量的页面组成，由BufferPool构造函数的numPages参数定义。在以后的实验中，您将实现一个驱逐策略。对于本实验，您只需要实现构造函数和SeqScan操作符使用的BufferPool.getPage()方法。BufferPool应存储最多numPages个页面。对于本实验，如果对不同页面的请求超过numPages个，那么您可以抛出DbException，而不是实现驱逐策略。在未来的实验中，您将需要实现一个驱逐策略。

Database类提供了一个静态方法Database.getBufferPool()，该方法返回对整个SimpleDB进程的单个BufferPool实例的引用。

练习3

在以下文件中实现getPage()方法：

*

• src/java/simpledb/storage/BufferPool.java

*

我们没有为BufferPool提供单元测试。您实现的功能将在下面的HeapFile实现中进行测试。您应使用DbFile.readPage方法来访问DbFile的页面。

<!--
当缓冲池中的页面超过这个数量时，在加载下一个页面之前，应驱逐一个页面。驱逐策略的选择由您决定；不需要做复杂的事情。
-->

<!--
<p>

注意，BufferPool要求您实现一个flush_all_pages()方法。在实际的缓冲池实现中，您永远不会需要这个方法。但是，我们需要这个方法用于测试目的。您绝对不应在代码中的任何地方调用此方法。
-->

2.5. HeapFile访问方法

访问方法提供了一种以特定方式从磁盘读取或写入数据的方式。常见的访问方法包括堆文件（未排序的元组文件）和B树；对于本作业，您将只实现堆文件访问方法，并且我们已经为您编写了部分代码。

<p>

一个HeapFile对象被组织成一组页面，每个页面由固定数量的字节组成，用于存储元组（由常量BufferPool.DEFAULT_PAGE_SIZE定义），包括一个头部。在SimpleDB中，数据库中的每个表都有一个HeapFile对象。HeapFile中的每个页面被组织为一组槽位，每个槽位可以容纳一个元组（SimpleDB中给定表的所有元组大小相同）。除了这些槽位，每个页面还有一个头部，由每个元组槽位一位的位图组成。如果对应于特定元组的位是1，则表示该元组有效；如果是0，则表示该元组无效（例如，已被删除或从未初始化。）HeapFile对象的页面类型为HeapPage，它实现了Page接口。页面存储在缓冲池中，但由HeapFile类读取和写入。

<p>

SimpleDB在磁盘上存储堆文件的方式与在内存中存储的方式大致相同。每个文件由在磁盘上连续排列的页面数据组成。每个页面由一个或多个表示头部的字节组成，然后是_页面大小_字节的实际页面内容。每个元组需要_tuple size_ * 8位用于其内容，以及1位用于头部。因此，可以放入单个页面的元组数量为：

<p>

`
_每页元组数_ = floor((_页面大小_  8) / (_元组大小_  8 + 1))
`

<p>

其中_元组大小_是页面中元组的大小（以字节为单位）。这里的想法是每个元组在头部中需要额外的一位存储。我们计算一个页面中的位数（通过将页面大小乘以8），然后将这个数量除以一个元组中的位数（包括这个额外的头部位）得到每页的元组数。floor操作向下取整到最接近的整数元组数（我们不想在页面上存储部分元组！）

<p>

一旦我们知道每页的元组数，存储头部所需的字节数就是：
<p>

`
头部字节数 = ceiling(每页元组数/8)
`

<p>

ceiling操作向上取整到最接近的整数字节数（我们永远不会存储少于一个完整的字节头部信息。）

<p>

每个字节的低位（最低有效位）表示文件中较早的槽位的状态。因此，第一个字节的最低位表示页面中的第一个槽位是否在使用中。第一个字节的次低位表示页面中的第二个槽位是否在使用中，依此类推。另外，请注意最后一个字节的高位可能不对应于实际在文件中的槽位，因为槽位数量可能不是8的倍数。还要注意，所有Java虚拟机都是http://en.wikipedia.org/wiki/Endianness。

<p>

练习4

实现以下文件中的骨架方法：
*

• src/java/simpledb/storage/HeapPageId.java

• src/java/simpledb/storage/RecordId.java

• src/java/simpledb/storage/HeapPage.java

*

尽管在实验1中您不会直接使用它们，但我们要求您在HeapPage中实现<tt>getNumEmptySlots()</tt>和<tt>isSlotUsed()</tt>。这需要在页面头部中操作位。您可能会发现查看HeapPage中提供的其他方法或<tt>src/simpledb/HeapFileEncoder.java</tt>中的方法来了解页面的布局是有帮助的。

您还需要实现一个遍历页面中元组的迭代器，这可能涉及辅助类或数据结构。

此时，您的代码应通过HeapPageIdTest、RecordIDTest和HeapPageReadTest中的单元测试。

<p> 

在您实现<tt>HeapPage</tt>之后，您将在本实验中编写<tt>HeapFile</tt>的方法来计算文件中的页数并从文件中读取页面。然后，您将能够从存储在磁盘上的文件中获取元组。

练习5

实现以下文件中的骨架方法：

*

• src/java/simpledb/storage/HeapFile.java

*

要从磁盘读取页面，您首先需要计算文件中的正确偏移量。提示：您将需要随机访问文件以便在任意偏移量处读写页面。从磁盘读取页面时，您不应调用BufferPool方法。

<p> 
您还需要实现HeapFile.iterator()方法，该方法应迭代HeapFile中每个页面的元组。迭代器必须使用BufferPool.getPage()方法来访问HeapFile中的页面。此方法将页面加载到缓冲池中，并最终（在以后的实验中）用于实现基于锁的并发控制和恢复。不要在open()调用时将整个表加载到内存中——对于非常大的表，这会导致内存不足错误。

<p>

此时，您的代码应通过HeapFileReadTest中的单元测试。

2.6. 操作符

操作符负责查询计划的实际执行。它们实现关系代数操作。在SimpleDB中，操作符基于迭代器；每个操作符都实现DbIterator接口。

<p>

通过将较低级别的操作符传递到较高级别操作符的构造函数中，操作符被连接成一个计划，即通过"将它们链接在一起"。计划叶子处的特殊访问方法操作符负责从磁盘读取数据（因此下面没有任何操作符）。

<p>

在计划的顶部，与SimpleDB交互的程序只需在根操作符上调用getNext；然后此操作符在其子操作符上调用getNext，依此类推，直到调用这些叶子操作符。它们从磁盘获取元组并将其向上传递（作为getNext的返回参数）；元组以这种方式向上传播计划，直到它们在根节点输出或被计划中的另一个操作符组合或拒绝。

<p>

<!--
对于实现INSERT和DELETE查询的计划，最顶层的操作符是一个特殊的Insert或Delete操作符，它修改磁盘上的页面。这些操作符返回一个包含受影响元组计数的元组给用户级程序。

<p>
-->

对于本实验，您只需要实现一个SimpleDB操作符。

练习6。

实现以下文件中的骨架方法：

*

• src/java/simpledb/execution/SeqScan.java

*
此操作符按顺序扫描构造函数中tableid指定的表的所有页面中的所有元组。此操作符应通过DbFile.iterator()方法访问元组。

<p>此时，您应该能够完成ScanTest系统测试。干得好！

您将在后续实验中填写其他操作符。

<a name="query_walkthrough"></a>

2.7. 一个简单的查询

本节的目的是说明这些各种组件如何连接在一起以处理简单查询。

假设您有一个数据文件"some_data_file.txt"，内容如下：

1,1,1
2,2,2
3,4,4


<p>
您可以将其转换为SimpleDB可以查询的二进制文件，如下所示：
<p>
java -jar dist/simpledb.jar convert some_data_file.txt 3
<p>
这里，参数"3"告诉conver输入有3列。
<p>
以下代码实现了对此文件的简单选择查询。此代码等效于SQL语句`SELECT * FROM some_data_file`。

```
package simpledb;
import java.io.*;

public class test {

    public static void main(String[] argv) {

        // 构造一个3列表模式
        Type types[] = new Type[]{ Type.INT_TYPE, Type.INT_TYPE, Type.INT_TYPE };
        String names[] = new String[]{ "field0", "field1", "field2" };
        TupleDesc descriptor = new TupleDesc(types, names);

        // 创建表，将其与some_data_file.dat关联
        // 并告诉目录此表的模式。
        HeapFile table1 = new HeapFile(new File("some_data_file.dat"), descriptor);
        Database.getCatalog().addTable(table1, "test");

        // 构造查询：我们使用一个简单的SeqScan，它通过其迭代器提供元组。
        TransactionId tid = new TransactionId();
        SeqScan f = new SeqScan(tid, table1.getId());

        try {
            // 并运行它
            f.open();
            while (f.hasNext()) {
                Tuple tup = f.next();
                System.out.println(tup);
            }
            f.close();
            Database.getBufferPool().transactionComplete(tid);
        } catch (Exception e) {
            System.out.println ("Exception : " + e);
        }
    }

}


我们创建的表有三个整数字段。为了表达这一点，我们创建一个`TupleDesc`对象并向其传递一个`Type`对象数组，以及可选的`String`字段名数组。创建此`TupleDesc`后，我们初始化一个`HeapFile`对象，表示存储在`some_data_file.dat`中的表。创建表后，我们将其添加到目录中。如果这是一个已经在运行的数据库服务器，我们将已经加载了此目录信息。我们需要显式加载它以使此代码自包含。

初始化数据库系统后，我们创建一个查询计划。我们的计划仅包含扫描磁盘元组的`SeqScan`操作符。通常，这些操作符使用对适当表（在`SeqScan`的情况下）或子操作符（在例如Filter的情况下）的引用进行实例化。然后，测试程序在`SeqScan`操作符上重复调用`hasNext`和`next`。当元组从`SeqScan`输出时，它们被打印到命令行上。

我们强烈建议您尝试将其作为一个有趣的端到端测试，这将帮助您获得为simpledb编写自己的测试程序的经验。您应在src/java/simpledb目录中创建文件"test.java"，其中包含上述代码，并且应在代码上方添加一些"import"语句，并将`some_data_file.dat`文件放在顶级目录中。然后运行：


ant
java -classpath dist/simpledb.jar simpledb.test


注意`ant`会编译`test.java`并生成包含它的新jar文件。

## 3. 后勤

您必须提交您的代码（见下文）以及一份简短（最多2页）的说明文档，描述您的方法。此文档应：

• 描述您做出的任何设计决策。对于实验1，这些可能很少。

• 讨论并证明您对API所做的任何更改。

• 描述代码中任何缺失或不完整的部分。

• 描述您在实验上花费的时间，以及是否有任何特别困难或令人困惑的地方。


### 3.1. 合作

本实验单人应该可以完成，但如果您更愿意与合作伙伴一起工作，也可以。不允许更大的团队。请在您的个人文档中清楚说明您与谁（如果有的话）合作。

### 3.2. 提交您的作业

<!--
要提交您的代码，请创建一个<tt>6.830-lab1.tar.gz</tt>压缩包（解压后创建<tt>6.830-lab1/src/simpledb</tt>目录，其中包含您的代码）并在[6.830 Stellar Site](https://stellar.mit.edu/S/course/6/sp13/6.830/index.html)上提交。您可以使用`ant handin`目标生成压缩包。
-->

我们将使用gradescope自动评分所有编程作业。你们应该都被邀请加入课程实例；如果没有，请查看piazza获取邀请码。如果仍然有问题，请告诉我们，我们可以帮助您设置。您可以在截止日期前多次提交代码；我们将使用gradescope确定的最新版本。将文档放在名为lab1-writeup.txt的文件中，与您的提交一起。您还需要显式添加您创建的任何其他文件，例如新的*.java文件。

向gradescope提交的最简单方式是使用包含您代码的`.zip`文件。在Linux/MacOS上，您可以通过运行以下命令来实现：

bash
$ zip -r submission.zip src/ lab1-writeup.txt
```

3.3. 提交错误

请将（友好的！）错误报告提交至mailto:6.830-staff@mit.edu。提交时，请尝试包括：

• 错误描述。

• 一个.java文件，我们可以将其放入test/simpledb目录，编译并运行。

• 一个.txt文件，其中包含重现错误的数据。我们应该能够使用HeapFileEncoder将其转换为.dat文件。

如果您是第一个报告代码中特定错误的人，我们将给您一块糖果！

<!--最新的错误报告/修复可以在bugs.html找到。-->

<a name="grading"></a>

3.4 评分

<p>您成绩的75%将基于您的代码是否通过我们将运行的系统测试套件。这些测试将是我们提供的测试的超集。在提交代码之前，您应确保它从<tt>ant test</tt>和<tt>ant systemtest</tt>中都不产生错误（通过所有测试）。

重要： 在测试之前，gradescope将用我们的版本替换您的<tt>build.xml</tt>和<tt>test</tt>目录的全部内容。这意味着您不能更改<tt>.dat</tt>文件的格式！您还应小心更改我们的API。您应测试您的代码是否能够编译未修改的测试。

提交后，您应立即从gradescope获得反馈和失败测试的错误输出（如果有）。给出的分数将是您作业自动评分部分的成绩。您成绩的另外25%将基于您的文档质量以及我们对您代码的主观评估。这部分也将在我们完成评分后发布在gradescope上。

我们设计这个作业时非常有趣，希望您也喜欢修改它！
