## cgroups 是什么
cgroups（Control Groups）最初叫 Process Container，由 Google 工程师（Paul Menage 和 Rohit Seth）于 2006 年提出，后来因为 Container 有多重含义容易引起误解，就在 2007 年更名为 Control Groups，并被整合进 Linux 内核。顾名思义就是把进程放到一个组里面统一加以控制。官方的定义如下:

cgroups 是 Linux 内核提供的一种机制，这种机制可以根据特定的行为，把一系列系统任务及其子任务整合（或分隔）到按资源划分等级的不同组内，从而为系统资源管理提供一个统一的框架。

通俗的来说，cgroups 可以限制、记录、隔离进程组所使用的物理资源（包括：CPU、memory、IO 等），为容器实现虚拟化提供了基本保证，是构建 Docker 等一系列虚拟化管理工具的基石。

对开发者来说，cgroups 有如下四个有趣的特点：

+ cgroups 的 API 以一个伪文件系统的方式实现，即用户可以通过文件操作实现 cgroups 的组织管理。
+ cgroups 的组织管理操作单元可以细粒度到线程级别，用户态代码也可以针对系统分配的资源创建和销毁 cgroups，从而实现资源再分配和管理。
+ 所有资源管理的功能都以“subsystem（子系统）”的方式实现，接口统一。
+ 子进程创建之初与其父进程处于同一个 cgroups 的控制组。

本质上来说，cgroups 是内核附加在程序上的一系列钩子（hooks），通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。

## cgroups 的作用

实现 cgroups 的主要目的是为不同用户层面的资源管理，提供一个统一化的接口。从单个进程的资源控制到操作系统层面的虚拟化。Cgroups 提供了以下四大功能。

+ 资源限制（Resource Limitation）：cgroups 可以对进程组使用的资源总额进行限制。如设定应用运行时使用内存的上限，一旦超过这个配额就发出 OOM（Out of Memory）。
+ 优先级分配（Prioritization）：通过分配的 CPU 时间片数量及硬盘 IO 带宽大小，实际上就相当于控制了进程运行的优先级。
+ 资源统计（Accounting）： cgroups 可以统计系统的资源使用量，如 CPU 使用时长、内存用量等等，这个功能非常适用于计费。
+ 进程控制（Control）：cgroups 可以对进程组执行挂起、恢复等操作。

过去有一段时间，内核开发者甚至把 namespace 也作为一个 cgroups 的 subsystem 加入进来，也就是说 cgroups 曾经甚至还包含了资源隔离的能力。但是资源隔离会给 cgroups 带来许多问题，如 PID 在循环出现的时候 cgroup 却出现了命名冲突、cgroup 创建后进入新的 namespace 导致脱离了控制等等，所以在 2011 年就被移除了。

## 术语表

+ task（任务）：cgroups 的术语中，task 就表示系统的一个进程。
+ cgroup（控制组）：cgroups 中的资源控制都以 cgroup 为单位实现。cgroup 表示按某种资源控制标准划分而成的任务组，包含一个或多个子系统。一个任务可以加入某个 cgroup，也可以从某个 cgroup 迁移到另外一个 cgroup。
+ subsystem（子系统）：cgroups 中的 subsystem 就是一个资源调度控制器（Resource Controller）。比如 CPU 子系统可以控制 CPU 时间分配，内存子系统可以限制 cgroup 内存使用量。
+ hierarchy（层级树）：hierarchy 由一系列 cgroup 以一个树状结构排列而成，每个 hierarchy 通过绑定对应的 subsystem 进行资源调度。hierarchy 中的 cgroup 节点可以包含零或多个子节点，子节点继承父节点的属性。整个系统可以有多个 hierarchy。

## 组织结构与基本规则

传统的 Unix 进程管理，实际上是先启动init进程作为根节点，再由init节点创建子进程作为子节点，而每个子节点由可以创建新的子节点，如此往复，形成一个树状结构。而 cgroups 也是类似的树状结构，子节点都从父节点继承属性。

它们最大的不同在于，系统中 cgroup 构成的 hierarchy 可以允许存在多个。如果进程模型是由init作为根节点构成的一棵树的话，那么 cgroups 的模型则是由多个 hierarchy 构成的森林。这样做的目的也很好理解，如果只有一个 hierarchy，那么所有的 task 都要受到绑定其上的 subsystem 的限制，会给那些不需要这些限制的 task 造成麻烦。

了解了 cgroups 的组织结构，我们再来了解 cgroup、task、subsystem 以及 hierarchy 四者间的相互关系及其基本规则。

+ 规则 1： 同一个 hierarchy 可以附加一个或多个 subsystem。如下图 1，cpu 和 memory 的 subsystem 附加到了一个 hierarchy。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308222040527.png)

+ 规则 2： 一个 subsystem 可以附加到多个 hierarchy，当且仅当这些 hierarchy 只有这唯一一个 subsystem。如下图 2，小圈中的数字表示 subsystem 附加的时间顺序，CPU subsystem 附加到 hierarchy A 的同时不能再附加到 hierarchy B，因为 hierarchy B 已经附加了 memory subsystem。如果 hierarchy B 与 hierarchy A 状态相同，没有附加过 memory subsystem，那么 CPU subsystem 同时附加到两个 hierarchy 是可以的。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308222041877.png)

+ 规则 3： 系统每次新建一个 hierarchy 时，该系统上的所有 task 默认构成了这个新建的 hierarchy 的初始化 cgroup，这个 cgroup 也称为 root cgroup。对于你创建的每个 hierarchy，task 只能存在于其中一个 cgroup 中，即一个 task 不能存在于同一个 hierarchy 的不同 cgroup 中，但是一个 task 可以存在在不同 hierarchy 中的多个 cgroup 中。如果操作时把一个 task 添加到同一个 hierarchy 中的另一个 cgroup 中，则会从第一个 cgroup 中移除。在下图 3 中可以看到，httpd进程已经加入到 hierarchy A 中的/cg1而不能加入同一个 hierarchy 中的/cg2，但是可以加入 hierarchy B 中的/cg3。实际上不允许加入同一个 hierarchy 中的其他 cgroup 野生为了防止出现矛盾，如 CPU subsystem 为/cg1分配了 30%，而为/cg2分配了 50%，此时如果httpd在这两个 cgroup 中，就会出现矛盾。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308222043262.png)

+ 规则 4： 进程（task）在 fork 自身时创建的子任务（child task）默认与原 task 在同一个 cgroup 中，但是 child task 允许被移动到不同的 cgroup 中。即 fork 完成后，父子进程间是完全独立的。如下图 4 中，小圈中的数字表示 task 出现的时间顺序，当httpd刚 fork 出另一个httpd时，在同一个 hierarchy 中的同一个 cgroup 中。但是随后如果 PID 为 4840 的httpd需要移动到其他 cgroup 也是可以的，因为父子任务间已经独立。总结起来就是：初始化时子任务与父任务在同一个 cgroup，但是这种关系随后可以改变。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308222044073.png)

## subsystem 简介

subsystem 实际上就是 cgroups 的资源控制系统，每种 subsystem 独立地控制一种资源，目前 Docker 使用如下八种 subsystem，还有一种net_cls subsystem 在内核中已经广泛实现，但是 Docker 尚未使用。他们的用途分别如下。

+ blkio： 这个 subsystem 可以为块设备设定输入 / 输出限制，比如物理驱动设备（包括磁盘、固态硬盘、USB 等）。
+ cpu： 这个 subsystem 使用调度程序控制 task 对 CPU 的使用。
+ cpuacct： 这个 subsystem 自动生成 cgroup 中 task 对 CPU 资源使用情况的报告。
+ cpuset： 这个 subsystem 可以为 cgroup 中的 task 分配独立的 CPU（此处针对多处理器系统）和内存。
+ devices 这个 subsystem 可以开启或关闭 cgroup 中 task 对设备的访问。
+ freezer 这个 subsystem 可以挂起或恢复 cgroup 中的 task。
+ memory 这个 subsystem 可以设定 cgroup 中 task 对内存使用量的限定，并且自动生成这些 task 对内存资源使用情况的报告。
+ perf_event_ _ 这个 subsystem 使用后使得 cgroup 中的 task 可以进行统一的性能测试。
+ *net_cls 这个 subsystem Docker 没有直接使用，它通过使用等级识别符 (classid) 标记网络数据包，从而允许 Linux 流量控制程序（TC：Traffic Controller）识别从具体 cgroup 中生成的数据包。

## cgroups 实现方式及工作原理简介

### cgroups 实现结构讲解

cgroups 的实现本质上是给系统进程挂上钩子（hooks），当 task 运行的过程中涉及到某个资源时就会触发钩子上所附带的 subsystem 进行检测，最终根据资源类别的不同使用对应的技术进行资源限制和优先级分配。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308232251732.png)

Linux 中管理 task 进程的数据结构为task_struct（包含所有进程管理的信息），其中与 cgroup 相关的字段主要有两个，一个是css_set *cgroups，表示指向css_set（包含进程相关的 cgroups 信息）的指针，一个 task 只对应一个css_set结构，但是一个css_set可以被多个 task 使用。另一个字段是list_head cg_list，是一个链表的头指针，这个链表包含了所有的链到同一个css_set的 task 进程（在图中使用的回环箭头，均表示可以通过该字段找到所有同类结构，获得信息）。

每个css_set结构中都包含了一个指向cgroup_subsys_state（包含进程与一个特定子系统相关的信息）的指针数组。cgroup_subsys_state则指向了cgroup结构（包含一个 cgroup 的所有信息），通过这种方式间接的把一个进程和 cgroup 联系了起来，如下图。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308232251590.png)

另一方面，cgroup结构体中有一个list_head css_sets字段，它是一个头指针，指向由cg_cgroup_link（包含 cgroup 与 task 之间多对多关系的信息，后文还会再解释）形成的链表。由此获得的每一个cg_cgroup_link都包含了一个指向css_set *cg字段，指向了每一个 task 的css_set。css_set结构中则包含tasks头指针，指向所有链到此css_set的 task 进程构成的链表。至此，我们就明白如何查看在同一个 cgroup 中的 task 有哪些了，如下图。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308242212548.png)

css_set中也有指向所有cg_cgroup_link构成链表的头指针，通过这种方式也能定位到所有的 cgroup，这种方式与图 1 中所示的方式得到的结果是相同的。

那么为什么要使用cg_cgroup_link结构体呢？因为 task 与 cgroup 之间是多对多的关系。熟悉数据库的读者很容易理解，在数据库中，如果两张表是多对多的关系，那么如果不加入第三张关系表，就必须为一个字段的不同添加许多行记录，导致大量冗余。通过从主表和副表各拿一个主键新建一张关系表，可以提高数据查询的灵活性和效率。

而一个 task 可能处于不同的 cgroup，只要这些 cgroup 在不同的 hierarchy 中，并且每个 hierarchy 挂载的子系统不同；另一方面，一个 cgroup 中可以有多个 task，这是显而易见的，但是这些 task 因为可能还存在在别的 cgroup 中，所以它们对应的css_set也不尽相同，所以一个 cgroup 也可以对应多个·css_set。

在系统运行之初，内核的主函数就会对root cgroups和css_set进行初始化，每次 task 进行 fork/exit 时，都会附加（attach）/ 分离（detach）对应的css_set。

综上所述，添加cg_cgroup_link主要是出于性能方面的考虑，一是节省了task_struct结构体占用的内存，二是提升了进程fork()/exit()的速度。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308242213158.png)