# lab6_zh
6.830 实验6：回滚与恢复

分配日期：2021年5月3日（星期一）<br>
截止日期：2021年5月19日（星期三）晚上11:59（美国东部标准时间）

0. 介绍

在本实验中，您将实现基于日志的回滚（用于中止）和基于日志的崩溃恢复。我们为您提供了定义日志格式的代码，并在事务期间的适当时机将记录追加到日志文件。您将使用日志文件的内容实现回滚和恢复。

我们提供的日志记录代码生成用于物理整页撤销和重做的记录。当页面首次读入时，我们的代码会记住页面的原始内容作为前像。当事务更新页面时，相应的日志记录包含该记住的前像以及修改后页面的内容作为后像。您将使用前像在中止期间回滚，并在恢复期间撤销失败事务，使用后像在恢复期间重做成功事务。

我们能够做到整页物理UNDO（而ARIES必须做逻辑UNDO），因为我们进行页面级锁定，并且没有索引可能在UNDO时具有与最初写入日志时不同的结构。页面级锁定简化事务的原因是，如果事务修改了页面，它必须拥有对该页面的独占锁，这意味着没有其他事务同时修改它，因此我们可以通过覆盖整个页面来撤销对它的更改。

您的BufferPool已经通过删除脏页面实现中止，并假装通过在提交时才强制脏页面写入磁盘来实现原子提交。日志记录允许更灵活的缓冲管理（STEAL和NO-FORCE），我们的测试代码在特定点调用BufferPool.flushAllPages()以利用这种灵活性。

1. 开始

您应该从提交给实验5的代码开始（如果您没有提交实验5的代码，或者您的解决方案无法正常工作，请联系我们讨论选项。）

您需要修改一些现有的源代码并添加一些新文件。以下是需要做的：

• 首先切换到您的项目目录（可能称为simple-db-hw）并从主GitHub仓库拉取：

$ cd simple-db-hw
$ git pull upstream master


• 现在对您的现有代码进行以下更改：

    1. 在BufferPool.flushPage()中对writePage(p)的调用之前插入以下行，其中p是对正在写入的页面的引用：

        // 向日志追加更新记录，包含前像和后像
        TransactionId dirtier = p.isDirty();
        if (dirtier != null){
          Database.getLogFile().logWrite(dirtier, p.getBeforeImage(), p);
          Database.getLogFile().force();
        }
    
    这导致日志系统向日志写入更新。我们强制日志以确保日志记录在页面写入磁盘之前就在磁盘上。

    2. 您的BufferPool.transactionComplete()为每个已提交事务弄脏的页面调用flushPage()。对于每个这样的页面，在刷新页面后添加对p.setBeforeImage()的调用：

    // 使用当前页面内容作为下一个修改此页面的事务的前像
    p.setBeforeImage();
    
    更新提交后，页面的前像需要更新，以便后续中止的事务回滚到此页面的已提交版本。（注意：我们不能只在flushPage()中调用setBeforeImage()，因为即使事务没有提交，也可能调用flushPage()。我们的测试用例实际上就是这样做的！如果您通过调用flushPages()实现transactionComplete()，您可能需要向flushPages()传递一个额外参数，告诉它刷新是否是为了提交事务。但是，在这种情况下，我们强烈建议您直接重写transactionComplete()以使用flushPage()。）

• 进行这些更改后，进行干净构建（从命令行执行ant clean; ant，或在Eclipse的"Project"菜单中选择"Clean"）。

• 此时，您的代码应通过LogTest系统测试的前三个子测试，但无法通过其余测试：

% ant runsystest -Dtest=LogTest
...
[junit] Running simpledb.systemtest.LogTest
[junit] Testsuite: simpledb.systemtest.LogTest
[junit] Tests run: 10, Failures: 0, Errors: 7, Time elapsed: 0.42 sec
[junit] Tests run: 10, Failures: 0, Errors: 7, Time elapsed: 0.42 sec
[junit]
[junit] Testcase: PatchTest took 0.057 sec
[junit] Testcase: TestFlushAll took 0.022 sec
[junit] Testcase: TestCommitCrash took 0.018 sec
[junit] Testcase: TestAbort took 0.03 sec
[junit]     Caused an ERROR
[junit] LogTest: tuple present but shouldn't be
...


• 如果您没有从ant runsystest -Dtest=LogTest看到上述输出，说明拉取新文件时出现了问题，或者您所做的更改与现有代码不兼容。您应该在继续之前找出并解决问题；如有必要，请向我们寻求帮助。

2. 回滚

阅读LogFile.java中的注释以了解日志文件格式的描述。您应在LogFile.java中看到一组函数，如logCommit()，用于生成每种日志记录并将其追加到日志中。

您的第一个任务是实现LogFile.java中的rollback()函数。此函数在事务中止时调用，在事务释放其锁之前。其工作是撤销事务可能对数据库所做的任何更改。

您的rollback()应读取日志文件，找到与中止事务关联的所有更新记录，从每个记录中提取前像，并将前像写入表文件。使用raf.seek()在日志文件中移动，使用raf.readInt()等检查它。使用readPageData()读取每个前像和后像。您可以使用映射tidToFirstLogRecord（从事务ID映射到堆文件中的偏移量）来确定在日志文件中开始读取特定事务的位置。您需要确保从缓冲池中丢弃任何将其前像写回表文件的页面。

在开发代码时，您可能会发现Logfile.print()方法对于显示日志的当前内容很有用。

*

练习1：LogFile.rollback()

实现LogFile.rollback()。

完成此练习后，您应能够通过LogTest系统测试的TestAbort和TestAbortCommitInterleaved子测试。

*

3. 恢复

如果数据库崩溃然后重新启动，LogFile.recover()将在任何新事务开始之前被调用。您的实现应：

1. 读取最后一个检查点（如果有的话）。

2. 从检查点（如果没有检查点，则从日志文件开头）向前扫描，构建失败事务集合。在此过程中重做更新。您可以安全地从检查点开始重做，因为LogFile.logCheckpoint()将所有脏缓冲刷新到磁盘。

3. 撤销失败事务的更新。

*

练习2：LogFile.recover()

实现LogFile.recover()。

完成此练习后，您应能够通过所有LogTest系统测试。

*

4. 后勤

您必须提交您的代码（见下文）以及一份简短（最多1页）的说明文档，描述您的方法。此文档应：

• 描述您做出的任何设计决策，包括任何困难或意外的内容。

• 讨论并证明您在LogFile.java之外所做的任何更改。

4.1. 合作

本实验单人应该可以完成，但如果您更愿意与合作伙伴一起工作，也可以。不允许更大的团队。请在您的文档中清楚说明您与谁（如果有的话）合作。

4.2. 提交您的作业

我们将使用gradescope自动评分所有编程作业。你们应该都被邀请加入课程实例；如果没有，请告诉我们，我们可以帮助您设置。您可以在截止日期前多次提交代码；我们将使用gradescope确定的最新版本。将文档放在名为lab6-writeup.txt的文件中，与您的提交一起。您还需要显式添加您创建的任何其他文件，例如新的*.java文件。

向gradescope提交的最简单方式是使用包含您代码的.zip文件。在Linux/MacOS上，您可以通过运行以下命令来实现：
$ zip -r submission.zip src/ lab6-writeup.txt


<a name="bugs"></a>
4.3. 提交错误

SimpleDB是一个相对复杂的代码库。很可能您会发现错误、不一致性以及糟糕、过时或不正确的文档等。

因此，我们要求您以冒险的心态来完成这个实验。如果某些内容不清楚甚至错误，请不要生气；而是尝试自己弄清楚或给我们发送一封友好的电子邮件。请将（友好的！）错误报告提交至<a href="mailto:6.830-staff@mit.edu">6.830-staff@mit.edu</a>。提交时，请尝试包括：

• 错误描述。

• 一个<tt>.java</tt>文件，我们可以将其放入src/simpledb/test目录，编译并运行。

• 一个<tt>.txt</tt>文件，其中包含重现错误的数据。我们应该能够使用PageEncoder将其转换为<tt>.dat</tt>文件。

如果您觉得自己遇到了错误，也可以在Piazza的班级页面上发布。

<a name="grading"></a>
4.4 评分

<p>您成绩的75%将基于您的代码是否通过我们将运行的系统测试套件。这些测试将是我们提供的测试的超集。在提交代码之前，您应确保它从<tt>ant test</tt>和<tt>ant systemtest</tt>中都不产生错误（通过所有测试）。

重要： 在测试之前，gradescope将用我们的版本替换您的<tt>build.xml</tt>、<tt>HeapFileEncoder.java</tt>和<tt>test</tt>目录的全部内容。这意味着您不能更改<tt>.dat</tt>文件的格式！您还应小心更改我们的API。您应测试您的代码是否能够编译未修改的测试。

提交后，您应立即从gradescope获得反馈和失败测试的错误输出（如果有）。给出的分数将是您作业自动评分部分的成绩。您成绩的另外25%将基于您的文档质量以及我们对您代码的主观评估。这部分也将在我们完成评分后发布在gradescope上。

我们设计这个作业时非常有趣，希望您也喜欢修改它！
