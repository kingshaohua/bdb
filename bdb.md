## Berkeley DB ##
Margo Seltzer and Keith Bostic  


康威定律说:"设计系统的组织，最终产生的设计等同于组织之内、之间的沟通结构。"。延伸一下，我们可以想象：初始由两个人设计的软件的架构，不仅仅反应了该组织间的结构，更反应了两个人之间的分歧与理念。我们两种的一个（Seltzer）的职业生涯都在从事文件系统和数据库管理系统。如果问她，她会说着两个东西从本质上讲其实是一回事。更进一步讲，操作系统和数据库管理系统本质上都是管理资源，并且提供抽象便捷的操作。它们的差异“仅仅”只是实现上的不同。另一个人（Bostic）则相信：基于工具的软件工程以及基于简单构件的组件架构。因为这样的系统在某些方面会比整体架构的系统要优秀的多--易理解性、可扩展性、可维护性、可测试性、灵活性。  

如果你结合两个观点，就不会惊讶于我们过去20年在Berkeley DB上的工作--一个软件库提供了快速，灵活，可靠并且可扩展的数据管理。Berkeley DB 提供了很多和传统系统一样的功能--人们期望的功能。比如，关系型数据库，但是，以不同的形式封装。比如，Berkeley DB访问数据很快，（key值访问和顺序访问），同样支持事务，崩溃恢复。但是，它是以库形式提供的，并且直接链接到应用程序中，而不是作为一个独立的应用服务。  

在本文中，我们会深入的剖析Berkeley DB（以下简称BDB），以及它的各个模块，每个模块都秉承了Unix的“do one thing well”的理念。内嵌了BDB应用可以直接使用这些组件，或者通过类似于get,put,delete数据项的操作来隐式的使用它们。我们关注于架构--我们如何开始，我们设计了什么，我们在哪结束，以及为什么。这些设计必须是（并且肯定是）可以适应和改变--随着时间的推移，重要的是维护原则以及一致性视角。我们也会简要的阐述一个长期软件项目的代码演变过程。BDB已经经过20年的持续开发，毫无疑问，这归功于一个优秀的设计。  

**1. In the Beginning**  
BDB可以追溯到Unix操作系统还是专属于AT&T的时代，数以百计的工具和库都有严格的许可限制。Margo Seltzer是California，Berkeley大学的毕业生， Keith Bostic是Berkeley计算机系统研究组的成员。那时候，keith真正致力于从Berkeley软件发行版上移除AT&T的软件。  

BDB项目开始的的目标是要替换 内存中hsearch hash package以及 磁盘上的dbm/ndbm hash package。使用一种新型改进的hash方法，可以应用于内存中和磁盘上。同时也是一个没有专属许可的自由发行版。Margo Seltzer写的这个hash库[SY91]是基于Litwin的《可扩展线性hash研究》。它使用一个很灵巧的方法使得能在固定时间内将hash值映射到数据页地址，同样可以处理大数据---数据项大于承载其的hash桶或者文件系统的页大小（通常是4到8个KB）。  

If hash tables were good, then Btrees and hash tables would be better（没看懂）。Mike Olson 同样是一名Berkeley大学的毕业生，之前写了很多BTree的实现，并且同意再写一个。我们中的三个将Margo的hash代码和Mike的转换成通用访问接口的API，应用通过此接口来操作数据库提供的hash或者BTree访问方法，继而来读取和修改数据。  

构建在这两种访问方式之上，Mike Olson 和 Margo Seltzer写了一篇论文 [SO92]，里面阐述了LIBTP，一种可以运行在应用程序地址空间的编程式事务型库。  

这个Hash和BTree库最终合入了4BSD的release版本中，并命名为Berkeley DB 1.85。技术上来讲，Btree访问方式是用B+link tree实现的，然而，在本文的接下来都会用BTree来指代。BDB 1.85的结构和API对于使用Linux或者基于BSD系统的人来说十分熟悉。  

BDB1.85库稳定了好几年，知道1996年，Netscape 签约了Margo Seltzer和Keith Bostic来构建出LIBTP论文中描述的完整事务系统。并且创建一个生产级别的软件版本。他们努力的结果就是BDB的第一个事务型版本，BDB2.0。  

BDB后来的历史就较为简单和传统了：BDB 2.0（1997）引入事务，DBD 3.0（1999）重新设计架构，添加了抽象层和间接的适应了不断增长的功能需求。BDB 4.0（2001）引入了备份和高可用性，以及Oracle BDB 5.0（2010）添加了SQL支持。  

在写这篇文章的时候，BDB已经是世界上使用最为广泛的数据库工具包了。数以百万计的拷贝运行在各种系统上，从路由，浏览器到邮箱和操作系统。经管已经过了20年，BDB的基于工具和面向对象的架构，使得它仍然可以继续改进以及自我重构来适应使用它的软件需求。  

**Design Lesson 1**  
对于任何一个复杂软件的测试和维护极为重要的是，软件在设计和构建时，就得设计成一系列模块的组合，并且定义好的API界限。这个界限可以（并且必须）根据需要改变，但是仍然需要这些界限。这些界限的存在，使得软件避免称为一堆不可维护的意大利面条。 Butler Lampson曾经说过，计算机科学中的所有问题可以通过另一种间接的层面解决。更为重要的是，当被问到面向对象意味着什么时，Lampson说，这意味着在一个API的背后你可以使用多种实现。BDB的设计和实现秉承了这一原则，在通用接口的之后运行多样的实现。提供面向对象的视角和感受，即使这些库是用C写的。  

**2. Architectural Overview**  
在本章节中，我们会回顾BDB库的架构，从LIBTP开始，讨论它的一些关键方面演进。  

![](./img/fig4.1.png)

图4.1，取自于 Seltzer 和 Olson的论文中，描述了原来的LIBTP架构，图4.2 描述了BDB2.0 的设计架构。  
![](./img/fig4.2.png)  

LIBTP的实现和BDB 2.0的设计最大的不同就是移除了process manager。LIBTP要求各个控制线程注册自己，并同步于那些单独的线程/进程，而不是提供一个子系统级的同步。正如第4节讨论的，原始的设计可能更适用于我们。  

![](./img/fig4.3.png)  

在设计和实际发布的db—2.0.6的架构（如图4.3）的不同之处在于，对于恢复模块的实现。图中灰色的部分就是恢复子系统。恢复模块包括了，驱动设备（图中的那个recovery方框）和一系列的redo,undo例程来恢复那些通过AM（access methond 访问接口）执行的操作。在架构图中就是那个标上“access method recovery routines”的圈。在BDB2.0中对于如何处理恢复是一致性设计，相对的LIBTB针对不同的访问接口使用不同的日志记录和恢复例程。这种通用的设计原则同样催生了不同模块间的丰富的接口。  

图4.4 描述了BDB-5.0.21的架构。该图所涉及到API列在表4.1中。尽管原来的架构仍然清晰可见，现在的架构添加了新的模块，分解了老的模块（比如，log变成了log和dbreg）,以及在模块间的API的数量上有很大的增加。  

经过十年的演讲，众多商用版本的发布，以及后续的数以百计的新特性，我们可以看到当前的架构比原来的复杂了好多。这边需要注意的是：首先，replication在系统中添加了一层，但是它处理的很干净，和原来的老代码一样，使用相同的接口和系统的其余模块交互。其次，log模块分裂成log和dbreg（数据库注册）。这在第8节会详细讨论。最后，我们将所有的内部模块调用放到命名空间中，并开头用下划线标识，这样应用就不会和我们的函数名冲突了。我们将会在 Design Lesson 6详细讨论。 

第四， 日志子系统的API目前是基于游标的（去除了log_get API,替换成了log_cursor API)。在历史上，BDB在任意时刻都不会有超过一个线程来读写日志，所以该库中有一个指向当前日志的指针。这不是一个很好的抽象，如果带上replication 就没法工作了。正如应用API使用游标支持迭代器，日志目前使用游标来迭代。第五，fileop模块内含的访问接口提供了事务性从保证了数据库的create,delete,rename操作。我们尝试多种方案来优雅的实现它（它仍没有我想象中那么干净），在对它重构了很长一段时间后，我们将它从模块中抽离出来。

**Design Lesson 2**
软件设计是几种能让你尝试解决问题之前对其完整思考的方法之一。有经验的程序员会采用一些不同的技巧来完成设计：有些人会写出一个版本，然后丢弃它，有些人会写一些使用手册或者设计文档，others fill out a code template where every requirement is identified and assigned to a specific function or comment. 比如，对于BDB，在写代码之前，我们创建了一系列的Unix-style的使用手册，关于访问接口和组件。无论使用哪种技术，在调试代码之前，很难考虑清楚程序的架构到底是什么样的，更不用说，当架构发生大的变更时，之前的调试工作就白费了。Software architecture requires a different mind set from debugging code, and the architecture you have when you begin debugging is usually the architecture you'll deliver in that release. 

[./img/fig4.4.png](./img/fig4.4.png)

[./img/table1.1.png](./img/table1.1.png)
[./img/table1.2.png](./img/table1.2.png)
[./img/table1.3.png](./img/table1.3.png)
[./img/table1.4.png](./img/table1.4.png)
[./img/table1.5.png](./img/table1.5.png)