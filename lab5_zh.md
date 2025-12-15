# lab5_zh
6.830 实验5：B+树索引

分配日期：2021年4月21日（星期三）<br>
截止日期：2021年5月4日（星期二）晚上11:59（美国东部时间）

0. 介绍

在本实验中，您将实现一个B+树索引，用于高效查找和范围扫描。我们为您提供了实现树结构所需的所有底层代码。您将实现搜索、页面拆分、页面之间重新分布元组以及页面合并。

您可能会发现复习教科书第10.3-10.7节很有帮助，这些章节提供了关于B+树结构的详细信息以及搜索、插入和删除的伪代码。

如教科书所述并在课堂上讨论，B+树中的内部节点包含多个条目，每个条目由一个键值和左右子指针组成。相邻键共享一个子指针，因此包含m个键的内部节点有m+1个子指针。叶节点可以包含数据条目或指向其他数据库文件中数据条目的指针。为简单起见，我们将实现一个B+树，其中叶页面实际包含数据条目。相邻的叶页面通过左右兄弟指针链接在一起，因此范围扫描只需要通过根节点和内部节点进行一次初始搜索来找到第一个叶页面。后续的叶页面通过跟随右（或左）兄弟指针找到。

1. 开始

您应该从提交给实验4的代码开始（如果您没有提交实验4的代码，或者您的解决方案无法正常工作，请联系我们讨论选项）。此外，我们为本实验提供了额外的源文件和测试文件，这些文件不在您最初收到的原始代码分发中。

您需要将这些新文件添加到您的发布中并设置您的lab4分支。最简单的方法是切换到您的项目目录（可能称为simple-db-hw），设置分支，并从主GitHub仓库拉取：
$ cd simple-db-hw
$ git pull upstream master


2. 搜索

查看index/和BTreeFile.java。这是B+树实现的核心文件，也是您将在本实验中编写所有代码的地方。与HeapFile不同，BTreeFile由四种不同类型的页面组成。如您所料，树的节点有两种不同类型的页面：内部页面和叶页面。内部页面在BTreeInternalPage.java中实现，叶页面在BTreeLeafPage.java中实现。为方便起见，我们在BTreePage.java中创建了一个抽象类，其中包含叶页面和内部页面共有的代码。此外，头部页面在BTreeHeaderPage.java中实现，并跟踪文件中哪些页面正在使用。最后，每个BTreeFile的开头有一个页面，指向树的根页面和第一个头部页面。这个单例页面在BTreeRootPtrPage.java中实现。熟悉这些类的接口，特别是BTreePage、BTreeInternalPage和BTreeLeafPage。您将在B+树的实现中使用这些类。

您的第一个任务是实现BTreeFile.java中的findLeafPage()函数。此函数用于在给定特定键值的情况下找到适当的叶页面，并用于搜索和插入。例如，假设我们有一个带有两个叶页面的B+树（参见图1）。根节点是一个内部页面，其中包含一个条目，该条目包含一个键（本例中为6）和两个子指针。给定值1，此函数应返回第一个叶页面。同样，给定值8，此函数应返回第二个页面。不太明显的情况是，如果给定键值6。可能存在重复键，因此两个叶页面上都可能存在6。在这种情况下，函数应返回第一个（左侧）叶页面。

<p align="center"> <img width=500 src="simple_tree.png"><br> <i>图1：具有重复键的简单B+树</i> </p>

您的findLeafPage()函数应递归搜索内部节点，直到到达与提供的键值对应的叶页面。为了在每一步找到适当的子页面，您应迭代内部页面中的条目，并将条目值与提供的键值进行比较。BTreeInternalPage.iterator()使用BTreeEntry.java中定义的接口提供对内部页面中条目的访问。此迭代器允许您迭代内部页面中的键值，并访问每个键的左右子页面ID。递归的基本情况发生在传递的BTreePageId的pgcateg()等于BTreePageId.LEAF时，表示它是一个叶页面。在这种情况下，您只需从缓冲池获取页面并返回它。您不需要确认它实际包含提供的键值f。

您的findLeafPage()代码还必须处理提供的键值f为null的情况。如果提供的值为null，则每次递归最左侧的子节点，以找到最左侧的叶页面。找到最左侧的叶页面对于扫描整个文件很有用。一旦找到正确的叶页面，您应返回它。如上所述，您可以使用BTreePageId.java中的pgcateg()函数检查页面类型。您可以假设只有叶页面和内部页面会被传递给此函数。

我们建议调用我们提供的包装函数BTreeFile.getPage()，而不是直接调用BufferPool.getPage()获取每个内部页面和叶页面。它的工作方式与BufferPool.getPage()完全相同，但接受一个额外参数来跟踪脏页面列表。此函数对于接下来的两个练习很重要，您将在其中实际更新数据，因此需要跟踪脏页面。

您的findLeafPage()实现访问的每个内部（非叶）页面都应使用READ_ONLY权限获取，除了返回的叶页面，应使用作为函数参数提供的权限获取。这些权限级别在本实验中不重要，但对于代码在未来的实验中正确运行很重要。

*

练习1：BTreeFile.findLeafPage()

实现BTreeFile.findLeafPage()。

完成此练习后，您应能够通过BTreeFileReadTest.java中的所有单元测试和BTreeScanTest.java中的系统测试。

*

3. 插入

为了保持B+树元组的有序性和树的完整性，我们必须将元组插入到具有包含键范围的叶页面中。如上所述，findLeafPage()可用于找到我们应该插入元组的正确叶页面。但是，每个页面有有限数量的槽，我们需要能够插入元组，即使相应的叶页面已满。

如教科书所述，尝试将元组插入已满的叶页面应导致该页面拆分，以便元组均匀分布在两个新页面之间。每次叶页面拆分时，需要在父节点中添加一个对应于第二个页面中第一个元组的新条目。有时，内部节点也可能已满，无法接受新条目。在这种情况下，父节点应拆分并向其父节点添加新条目。这可能导致递归拆分，最终创建新的根节点。

在本练习中，您将实现BTreeFile.java中的splitLeafPage()和splitInternalPage()。如果要拆分的页面是根页面，您需要创建一个新的内部节点作为新的根页面，并更新BTreeRootPtrPage。否则，您需要使用READ_WRITE权限获取父页面，必要时递归拆分它，并添加新条目。您会发现函数getParentWithEmptySlots()对于处理这些不同情况非常有用。在splitLeafPage()中，您应将键"复制"到父页面，而在splitInternalPage()中，您应将键"推"到父页面。如果这令人困惑，请参见图2并复习教科书第10.5节。请记住根据需要更新新页面的父指针（为简单起见，我们在图中未显示父指针）。拆分内部节点时，您需要更新已移动的所有子节点的父指针。您可能会发现函数updateParentPointers()对此任务很有用。此外，请记住更新任何拆分的叶页面的兄弟指针。最后，返回应插入新元组或条目的页面，由提供的键字段指示。（提示：您不必担心提供的键实际上可能落在要拆分的元组/条目的确切中心。您应在拆分期间忽略键，仅使用它来确定要返回的两个页面中的哪一个。）

<p align="center"> <img width=500 src="splitting_leaf.png"><br> <img width=500 src="splitting_internal.png"><br> <i>图2：拆分页面</i> </p>

每当您创建新页面时，无论是由于拆分页面还是创建新的根页面，都调用getEmptyPage()获取新页面。此函数是一个抽象，将允许我们重用由于合并而删除的页面（在下一节中介绍）。

我们期望您将使用BTreeLeafPage.iterator()和BTreeInternalPage.iterator()与叶页面和内部页面交互，以迭代每个页面中的元组/条目。为方便起见，我们还为两种类型的页面提供了反向迭代器：BTreeLeafPage.reverseIterator()和BTreeInternalPage.reverseIterator()。这些反向迭代器对于将页面中的元组/条目子集移动到其右兄弟特别有用。

如上所述，内部页面迭代器使用BTreeEntry.java中定义的接口，该接口有一个键和两个子指针。它还有一个recordId，用于标识底层页面上键和子指针的位置。我们认为一次处理一个条目是与内部页面交互的自然方式，但请记住，底层页面实际上并不存储条目列表，而是存储有序的m个键和m+1个子指针列表。由于BTreeEntry只是一个接口，而不是实际存储在页面上的对象，更新BTreeEntry的字段不会修改底层页面。为了更改页面上的数据，您需要调用BTreeInternalPage.updateEntry()。此外，删除条目实际上只删除一个键和一个子指针，因此我们提供函数BTreeInternalPage.deleteKeyAndLeftChild()和BTreeInternalPage.deleteKeyAndRightChild()使其明确。条目的recordId用于查找要删除的键和子指针。插入条目也只插入一个键和一个子指针（除非它是第一个条目），因此BTreeInternalPage.insertEntry()检查提供的条目中的一个子指针是否与页面上的现有子指针重叠，并且在该位置插入条目将保持键的有序性。

在splitLeafPage()和splitInternalPage()中，您都需要使用任何新创建的页面以及由于新指针或新数据而修改的任何页面来更新dirtypages集合。这就是BTreeFile.getPage()将派上用场的地方。每次获取页面时，BTreeFile.getPage()将检查页面是否已存储在本地缓存（dirtypages）中，如果在那里找不到请求的页面，则从缓冲池获取它。BTreeFile.getPage()还将使用读写权限获取的页面添加到dirtypages缓存中，因为推测它们很快会被弄脏。这种方法的一个优点是，如果在单个元组插入或删除期间多次访问相同的页面，它可以防止更新丢失。

请注意，与HeapFile.insertTuple()的重大不同在于，BTreeFile.insertTuple()可能返回大量脏页面，特别是在任何内部页面拆分时。您可能记得之前的实验，返回脏页面集是为了防止缓冲池在脏页面刷新之前驱逐它们。

*

警告：由于B+树是一种复杂的数据结构，在修改之前理解每个合法B+树所需的属性是有帮助的。以下是一个非正式列表：

1. 如果父节点指向子节点，子节点必须指回相同的父节点。
2. 如果叶节点指向右兄弟，则右兄弟必须指回该叶节点作为左兄弟。
3. 第一个和最后一个叶必须分别指向null左兄弟和右兄弟。
4. Record Id必须与它们实际所在的页面匹配。
5. 具有非叶子节点的节点中的key必须大于左子节点中的任何键，并且小于右子节点中的任何键。
6. 具有叶子节点的节点中的key必须大于或等于左子节点中的任何键，并且小于或等于右子节点中的任何键。
7. 节点要么有所有非叶子节点，要么有所有叶子节点。
8. 非根节点不能少于半满。

我们在文件BTreeChecker.java中实现了所有这些属性的机械化检查。此方法也用于在systemtest/BTreeFileDeleteTest.java中测试您的B+树实现。随意添加对此函数的调用来帮助调试您的实现，就像我们在BTreeFileDeleteTest.java中所做的那样。

注意
1. 检查器方法应在树初始化后、开始前和完成完整的键插入或删除调用后始终通过，但不一定在内部方法中。
2. 树可能格式良好（因此通过checkRep()）但仍然不正确。例如，空树将始终通过checkRep()，但可能不总是正确的（如果您刚插入了一个元组，树不应为空）。

*

练习2：拆分页面

实现BTreeFile.splitLeafPage()和BTreeFile.splitInternalPage()。

完成此练习后，您应能够通过BTreeFileInsertTest.java中的单元测试。您还应能够通过systemtest/BTreeFileInsertTest.java中的系统测试。一些系统测试用例可能需要几秒钟才能完成。这些文件将测试您的代码是否正确插入元组和拆分页面，并处理重复元组。

*

4. 删除

为了保持树平衡且不浪费不必要的空间，B+树中的删除可能导致页面重新分布元组（图3）或最终合并（参见图4）。您可能会发现复习教科书第10.6节很有用。

<p align="center"> <img width=500 src="redist_leaf.png"><br> <img width=500 src="redist_internal.png"><br> <i>图3：重新分布页面</i> </p>

<p align="center"> <img width=500 src="merging_leaf.png"><br> <img width=500 src="merging_internal.png"><br> <i>图4：合并页面</i> </p>

如教科书所述，尝试从少于半满的叶页面删除元组应导致该页面从其兄弟之一窃取元组或与其兄弟之一合并。如果页面的兄弟之一有多余的元组，元组应均匀分布在两个页面之间，并且应相应更新父节点的条目（参见图3）。但是，如果兄弟也处于最小占用状态，则两个页面应合并，并从父节点删除条目（图4）。反过来，从父节点删除条目可能导致父节点少于半满。在这种情况下，父节点应从其兄弟窃取条目或与兄弟合并。这可能导致递归合并，甚至如果从根节点删除最后一个条目，则删除根节点。

在本练习中，您将实现BTreeFile.java中的stealFromLeafPage()、stealFromLeftInternalPage()、stealFromRightInternalPage()、mergeLeafPages()和mergeInternalPages()。在前三个函数中，您将实现代码，如果兄弟有多余的元组/条目，则均匀重新分布元组/条目。请记住更新父节点中相应的键字段（仔细查看图3中是如何完成的 - 键有效地通过父节点"旋转"）。在stealFromLeftInternalPage()/stealFromRightInternalPage()中，您还需要更新已移动子节点的父指针。您应能够重用函数updateParentPointers()来实现此目的。

在mergeLeafPages()和mergeInternalPages()中，您将实现代码来合并页面，有效地执行splitLeafPage()和splitInternalPage()的反向操作。您会发现函数deleteParentEntry()对于处理所有不同的递归情况非常有用。请务必在已删除的页面上调用setEmptyPage()以使它们可供重用。与之前的练习一样，我们建议使用BTreeFile.getPage()来封装获取页面和保持脏页面列表最新的过程。

*

练习3：重新分布页面

实现BTreeFile.stealFromLeafPage()、BTreeFile.stealFromLeftInternalPage()、BTreeFile.stealFromRightInternalPage()。

完成此练习后，您应能够通过BTreeFileDeleteTest.java中的一些单元测试（例如testStealFromLeftLeafPage和testStealFromRightLeafPage）。系统测试可能需要几秒钟才能完成，因为它们创建了一个大型B+树以充分测试系统。

练习4：合并页面

实现BTreeFile.mergeLeafPages()和BTreeFile.mergeInternalPages()。

现在您应能够通过BTreeFileDeleteTest.java中的所有单元测试和systemtest/BTreeFileDeleteTest.java中的系统测试。

*

5. 事务

您可能记得，B+树可以通过使用下一键锁定来防止幻影元组出现在两个连续的范围扫描之间。由于SimpleDB使用页面级严格两阶段锁定，如果B+树实现正确，防止幻影实际上免费获得。因此，此时您还应能够通过BTreeNextKeyLockingTest。

此外，如果您在B+树代码中正确实现了锁定，您应能够通过test/simpledb/BTreeDeadlockTest.java中的测试。

如果一切实现正确，您还应能够通过BTreeTest系统测试。我们预计许多人会发现BTreeTest困难，因此它不是必需的，但我们将向任何能够成功运行它的人提供额外学分。请注意，此测试可能需要长达一分钟才能完成。

6. 额外学分

*

奖励练习5：（10%额外学分）

创建并实现一个名为BTreeReverseScan的类，该类在给定可选的IndexPredicate的情况下反向扫描BTreeFile。

您可以使用BTreeScan作为起点，但您可能需要在BTreeFile中实现反向迭代器。您可能还需要实现BTreeFile.findLeafPage()的单独版本。我们提供了BTreeLeafPage和BTreeInternalPage上的反向迭代器，您可能会发现它们有用。您还应编写代码来测试您的实现是否正确工作。BTreeScanTest.java是一个寻找灵感的好地方。

*

7. 后勤

您必须提交您的代码（见下文）以及一份简短（最多1页）的说明文档，描述您的方法。此文档应：

• 描述您做出的任何设计决策，包括任何困难或意外的内容。

• 讨论并证明您在BTreeFile.java之外所做的任何更改。

• 本实验花了您多长时间？您有任何改进建议吗？

• 可选：如果您完成了额外学分练习，解释您的实现并向我们展示您已彻底测试了它。

7.1. 合作

本实验单人应该可以完成，但如果您更愿意与合作伙伴一起工作，也可以。不允许更大的团队。请在您的文档中清楚说明您与谁（如果有的话）合作。

7.2. 提交您的作业

我们将使用gradescope自动评分所有编程作业。你们应该都被邀请加入课程实例；如果没有，请告诉我们，我们可以帮助您设置。您可以在截止日期前多次提交代码；我们将使用gradescope确定的最新版本。将文档放在名为lab5-writeup.txt的文件中，与您的提交一起。您还需要显式添加您创建的任何其他文件，例如新的*.java文件。

向gradescope提交的最简单方式是使用包含您代码的.zip文件。在Linux/MacOS上，您可以通过运行以下命令来实现：
$ zip -r submission.zip src/ lab5-writeup.txt


<a name="bugs"></a>
7.3. 提交错误

SimpleDB是一个相对复杂的代码库。很可能您会发现错误、不一致性以及糟糕、过时或不正确的文档等。

因此，我们要求您以冒险的心态来完成这个实验。如果某些内容不清楚甚至错误，请不要生气；而是尝试自己弄清楚或给我们发送一封友好的电子邮件。

请将（友好的！）错误报告提交至<a href="mailto:6.830-staff@mit.edu">6.830-staff@mit.edu</a>。提交时，请尝试包括：

• 错误描述。

• 一个<tt>.java</tt>文件，我们可以将其放入test/simpledb目录，编译并运行。

• 一个<tt>.txt</tt>文件，其中包含重现错误的数据。我们应该能够使用HeapFileEncoder将其转换为<tt>.dat</tt>文件。

如果您觉得自己遇到了错误，也可以在Piazza的班级页面上发布。

7.4 评分

<p>您成绩的75%将基于您的代码是否通过我们将运行的系统测试套件。这些测试将是我们提供的测试的超集。在提交代码之前，您应确保它从<tt>ant test</tt>和<tt>ant systemtest</tt>中都不产生错误（通过所有测试）。

重要： 在测试之前，gradescope将用我们的版本替换您的<tt>build.xml</tt>、<tt>HeapFileEncoder.java</tt>和<tt>test</tt>目录的全部内容。这意味着您不能更改<tt>.dat</tt>文件的格式！您还应小心更改我们的API。您应测试您的代码是否能够编译未修改的测试。

提交后，您应立即从gradescope获得反馈和失败测试的错误输出（如果有）。给出的分数将是您作业自动评分部分的成绩。您成绩的另外25%将基于您的文档质量以及我们对您代码的主观评估。这部分也将在我们完成评分后发布在gradescope上。

我们设计这个作业时非常有趣，希望您也喜欢修改它！
