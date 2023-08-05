## 1.1 Contributions

本文的主要贡献如下：

+ 我们确定并模拟了现代模糊测试器中常见的构建模块；
+ 我们提出了LibAFL，一个全新的用Rust从头开始编写的开源模糊测试框架；
+ 我们实现了最先进的构建模块和技术；
+ 基于这些构建模块，我们评估了先前工作中提出的15种技术以及一系列新的组合方法。
+ 我们提供了一个案例研究，重新实现了一个差异模糊测试器并使用自定义反馈；
+ 我们的通用模糊测试器胜过所有现成的模糊测试器。

# 3 ENTITIES IN MODERN FUZZING

在支持我们框架的设计时，我们首先确定了一组9个基本实体，这些实体通常存在于大多数现代模糊器中。在本节中，我们介绍这些实体并提供一些示例，使用最先进的模糊器。

+ `Input`- 形式上，程序的输入，或者一般情况下系统的输入，是来自外部源的数据，影响其行为。在我们的抽象模糊器模型中，我们将输入定义为程序输入（或其一部分）的内部表示。在最简单的情况下，程序的输入是一个单字节数组。像AFL这样的模糊器直接存储和操作这个字节数组，在执行时将结果传递给目标。然而，有些情况下，字节数组不是输入的理想表示，例如当目标程序期望一系列系统调用的序列时[70]。在这种情况下，模糊器的内部表示与程序使用它的方式不同。另一个例子是类似Aschermann等人的NAUTILUS的语法模糊器[4]的输入。在这里，模糊器将输入内部表示为抽象语法树，这是一种可以轻松操作的数据结构，并根据语法维持其有效性。由于目标程序期望输入为字节数组，因此在执行之前，树被序列化为字节序列。其他模糊器也可以使用其他输入表示，例如编码为整数的令牌序列[63]，或者编程语言的中间表示[33]。

+ `Corpus`- 语料库是存储输入及其相关元数据的地方。不同类型的存储方式会影响模糊器的功能。例如，完全存储在内存中的语料库可以使模糊器运行更快，但在模糊大型目标时可能很快耗尽可用内存；而存储在磁盘上的语料库允许用户检查模糊器的状态，但会引入磁盘操作的瓶颈。大多数主流的模糊器[14, 47, 76]将语料库存储在磁盘上，但这种选择会影响并行模糊的可扩展性，并需要标准库来执行文件IO操作。在我们的模型中，一个模糊器至少需要两个单独的语料库：一个用于存储有趣的测试用例（3），这些测试用例作为模糊器进化算法的组成部分，另一个用于存储解决方案，即满足模糊器目标的测试用例（例如，程序崩溃）。

+ `Scheduler`- 调度器（Scheduler）是一个与语料库紧密相关的组件。它决定了模糊器如何选择下一个要模糊的测试用例，通常是从语料库中选择一个条目。调度器可以采用不同的策略。简单的调度器可能实现简单的策略，如先进先出（FIFO）或随机选择。更复杂的调度器可能使用基于模糊器内省统计的概率算法，或将其他调度器应用于语料库的子集，就像AFL在计算“优选”最小集时所做的那样。还有一些调度器旨在减轻由于过于敏感的反馈而导致语料库膨胀的问题，而其他调度器则优先考虑具有有趣属性的测试用例。调度器的选择可以显著影响模糊过程的效果和效率。

+ `Stage`- 阶段（Stage）是一个组件，定义了对来自语料库的单个测试用例要执行的操作。通常情况下，调度器会选择一个测试用例，然后模糊器会对该输入执行每个阶段的操作。阶段是一个非常广泛的实体，在现有的模糊器中，它通常是调用一个或多个变异器对输入进行变异的组件（例如，AFL中的随机破坏阶段），或者是一个分析阶段，例如在白盒模糊器中进行污点跟踪以收集信息。许多模糊器采用的另一个广为人知的阶段是最小化阶段，AFL引入了这个阶段，它可以减小从语料库中获取的测试用例的大小，同时保持触发的覆盖点。阶段的选择可以显著影响模糊过程的效果和效率。

+ `Observer`- 观察器（Observer）是提供来自目标程序单次执行信息的实体。为了对一个输入的执行进行推理，模糊器执行该输入，然后查看观察器的状态。在执行后，观察器状态的快照等效于执行本身，从模糊器状态的影响上看是相同的。按照这种方式定义观察器在分布式模糊器中特别有用，因为它可以将观察器状态跨多个节点发送。这样可以避免在模糊非常缓慢的目标时，需要重新执行相同的输入。一个示例观察器是覆盖率映射，常用于AFL或HonggFuzz等常见的覆盖率导向模糊器。映射在执行过程中被填充以报告已执行的边。这些信息在运行之间不保留，它是程序动态属性的观察。其他模糊器，如Ijon或FuzzFactory，使用不同形式的观察器，但始终依赖于映射来跟踪除代码覆盖率之外的其他指标。

+ `Executor`- 执行器（Executor）是负责在模糊器给定输入的情况下执行目标系统的组件。在不同的模糊器中，这个实体的实现可能会有很大的变化。例如，对于像LibFuzzer这样的内存中的模糊器，执行是对测试用例函数的调用，而对于像Nyx [64]这样的基于虚拟化技术的模糊器，则需要每次运行时从快照重新启动整个操作系统。在我们的模型中，执行器不仅定义了如何执行目标，还包括与单次运行相关的所有易失性操作。因此，执行器负责告知程序在运行中使用模糊器想要的输入，例如通过写入内存位置或将其作为参数传递给测试用例函数。执行器还维护与每次执行相关联的一组观察器。

+ `Feedback`- 反馈（Feedback）是将被测程序的执行结果分类为有趣或无趣的实体。通常，这个信息用于决定是否将相应的输入添加到语料库中。大多数情况下，反馈的概念与观察器（Observer）紧密相关，但两者是不同的概念。事实上，反馈通常处理一个或多个观察器报告的信息，以决定执行是否有趣。虽然"有趣"的概念是抽象的，但通常与新颖性搜索相关（即，有趣的输入是那些在控制流图中达到先前未见过的边的输入）。在另一个示例中[58]，观察器可以用于报告所有内存分配的大小，而最大化反馈可以用于最大化这些值，以发现在内存消耗方面的病态输入。识别有趣的输入的过程也具有模糊中的第二个重要目标：找到满足特定目标的解决方案，例如，被测程序中的可观察崩溃。这种类型的反馈，即目标（Objectives），充当了一个预期模糊运行结果的标准，例如，像HonggFuzz中一组具有唯一堆栈跟踪的崩溃测试用例，或者像AFLGo [13]中触发沿指定路径崩溃的输入。

+ `Mutator`- 变异器（Mutator）是一个将一个或多个输入转换为新的派生输入的实体。变异器可以由其他变异器组成，它们通常与特定的输入类型相关联。在传统的模糊器中，变异器由许多bit-level变异组成，如位翻转或块交换。变异器还可以了解输入格式并改变输入的内部表示，例如，在语法模糊器中交换抽象语法树中的节点。在创建自定义模糊器时，变异器通常是更频繁更改的部分。

+ `Generator`- 生成器（Generator）是一个用于从头开始生成新的输入的组件。例如，可以使用随机生成器来生成随机输入。在反馈驱动的模糊中，生成器不太常见，但也有一些显著的例外情况采用了生成器。例如，Nautilus [4]使用基于语法的生成器来创建初始语料库，并使用子树生成器作为其语法变异器的一种变异。

# 4 FRAMEWORK ARCHITECTURE

LibAFL的目标是通过基于可重用组件的模块化设计，以及对最先进技术的可靠、快速和可扩展的实现，提供构建新一代模糊测试器所需的基本模块。为了实现这一目标，我们决定在设计阶段将框架的设计限制在我们使用的实际编程语言Rust中，并利用其特性。在本节中，我们介绍并讨论LibAFL作为一个系统以及其各个组件的设计。

## 4.1 Principles and High-level Design

LibAFL框架围绕三个关键原则设计：

+ 可扩展性：允许用户在不影响其他部分的情况下替换解释第3节中实体的不同实现。这使得可以轻松地组合正交技术，并且也便于设计和开发新的组件；

+ 可移植性：大多数现有的模糊测试器都是特定于操作系统的，运行在*nix或Microsoft Windows下。为了避免这个问题，我们选择了以系统无关的方式设计核心库。此外，为了最大程度的可移植性，我们还实现了LibAFL的一个子集，包括所有核心组件，没有依赖于任何标准库，因此允许用户为嵌入式系统和内核等裸金属目标编写模糊测试器；

+ 可扩展性：设计选择不能妨碍模糊测试器在多个核心和/或多台机器上扩展的能力。因此，我们设计了一种基于事件的接口，以实现并促进模糊测试器之间的通信；

正如我们之前讨论的，目前没有一个现有的模糊测试框架是完全可扩展的。有些框架在不同操作系统上可移植，例如LibFuzzer [46]，但没有一个可以在没有标准库的系统上编译。最后但并非最不重要的一点是，现有模糊测试器的一个已知弱点是可扩展性。AFL及其许多衍生版本的设计都是基于磁盘IO通信和昂贵的系统调用，比如fork(2) [75]。这导致当模糊测试器在多个核心上扩展时性能变得糟糕[23]。其他更可扩展的解决方案，如HonggFuzz，仍然基于系统调用来控制目标并在所有并行线程之间维护共享状态，导致锁争用。另一方面，LibFuzzer实现了更高的可扩展性，因为不同的节点在模糊测试时不能进行通信，语料库在一定时间间隔后合并，然后重新启动模糊测试器。

为了创建一个遵循上述三个目标的模糊测试框架，我们将系统设计围绕三个核心库：

+ LibAFL Core是主要的库，包含模糊测试的组件及其实现。这个库的大部分依赖只有Rust的core+alloc，因此可以在没有任何标准库的情况下运行；
+ LibAFL Targets包含目标程序中的代码，比如用于覆盖跟踪的运行时库；
+ LibAFL CC提供编写LibAFL编译器包装器的功能，通过为用户提供一组对插桩有用的编译器扩展。

除了这三个核心库外，LibAFL还包含了几个Instrumentation Backends，提供API来将LibAFL与不同的执行引擎（如QEMU用户模式和Frida）连接起来。按照我们的命名惯例，所有这些库都是用来创建模糊测试器的工具包，称为Fuzzer Frontends。一些现成的前端已经包含在框架的另一个库LibAFL Sugar中，该库提供高级的粘合API，可以在几行代码中快速设置前端。我们还提供了Python绑定到Sugar crate，用于快速原型开发而无需重新编译。

例如，清单1展示了一个简单的模糊测试器，它将一个类似LibFuzzer的测试组件与一个使用通用bit-level突变的模糊测试器桥接，并使用LibAFL Sugar的高级API在进程中执行目标程序。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308032007234.png)

截至撰写本文时，整个LibAFL框架，包括测试，共有53,000行Rust代码和15,400行C/C++代码。

## 4.2 The Core Library

图1展示了核心库的架构，显示了组件之间的链接。大多数组件与我们在第3节中讨论的实体是一对一的映射，另外还有三个额外的宏组件：State、Fuzzer和Events Manager。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308032009109.png)

每个组件都映射到一个Rust泛型特征，使其能够与任何其他正交组件结合使用。这种架构配置是LibAFL中建议的前端标准架构，但也可以定义自定义架构。LibAFL中已实现的另一种替代架构是一个没有执行器的管道，其中没有传统的模糊测试循环，而模糊测试器是一个服务，可以从中请求输入。这样，LibAFL可以嵌入到仿真循环中，用于在Panda等仿真器的钩子中使用。

### 4.2.1 Zero-cost Abstractions.

可扩展性带来了引入抽象的代价，通常会在性能方面付出代价。由于速度是模糊测试中的重要指标，我们设计了一种允许灵活抽象而在运行时几乎不付出显著代价的设计。

从一开始，在开发的早期阶段，我们通过微基准测试避免使用传统的面向对象模式，而是选择了使用泛型特征。通过这种方式，我们利用了Rust编程语言的设计，使得编译器能够进行强大的优化。在LibAFL中，每个泛型特征将其他相关组件作为泛型参数传入。然后，通过组合来定义子组件。这样，我们在编译时支付了链接但独立的实体组合的成本，比如执行器和不同类型的输入。作为第二种设计模式，受Haskell的启发，我们使用类似于hlist [ 56 ]的编译时列表来指定多个对象，例如可组合变异器中的观察者集合。这些列表具有匹配功能，可以检索存储在数据结构中的单个对象。例如，反馈可以通过名称或类型访问有用于确定执行的有趣性的观察者。通过利用Rust提供的强大编译时功能，我们的代码对编译器优化友好。

### 4.2.2 The State

状态(State)是所有非易失性数据的存放地。进化算法的所有数据都必须包含在状态(State)中，还包括执行次数、伪随机数生成器的状态以及语料库(包括主要的和解决方案的语料库)。

```rust
pub fn new<F, O>(
    rand: R,
    corpus: C,
    solutions: SC,
    feedback: &mut F,
    objective: &mut O,
) -> Result<Self, Error>
where
    F: Feedback<Self>,
    O: Feedback<Self>,
{
    let mut state = Self {
        rand,
        executions: 0,
        start_time: Duration::from_millis(0),
        metadata: SerdeAnyMap::default(),
        named_metadata: NamedSerdeAnyMap::default(),
        corpus,
        solutions,
        max_size: DEFAULT_MAX_SIZE,
        #[cfg(feature = "introspection")]
        introspection_monitor: ClientPerfMonitor::new(),
        #[cfg(feature = "std")]
        remaining_initial_files: None,
        phantom: PhantomData,
    };
    feedback.init_state(&mut state)?;
    objective.init_state(&mut state)?;
    Ok(state)
}
```

由于某些类型的反馈(例如在覆盖引导模糊测试中观察到的覆盖范围)也需要维护状态，因此我们引入了反馈状态(FeedbackState)组件，它与状态(State)和反馈(Feedbacks)相关联。反馈状态(FeedbackState)的实例包含在状态(State)中，并在模糊过程开始时生成。

拥有一个存放模糊器数据的地方的主要目的是利用Rust的序列化功能。将状态(State)进行序列化和反序列化，允许任何基于LibAFL的模糊器停止运行，并在稍后从完全相同的内部状态重新启动。对于进程内模糊测试，这种新颖的方法允许LibAFL通过在崩溃处理程序中重新加载序列化状态而从崩溃中恢复实例，而无需重新执行整个语料库，如先前的解决方案所需。

```rust

impl<I, C, R, SC> Serialize for StdState<I, C, R, SC>
where
    C: Serialize + for<'a> Deserialize<'a>,
    SC: Serialize + for<'a> Deserialize<'a>,
    R: Serialize + for<'a> Deserialize<'a>,


impl<'de, I, C, R, SC> Deserialize<'de> for StdState<I, C, R, SC>
where
    C: Serialize + for<'a> Deserialize<'a>,
    SC: Serialize + for<'a> Deserialize<'a>,
    R: Serialize + for<'a> Deserialize<'a>,
```

### 4.2.3 The Fuzzer

模糊器(Fuzzer)是定义模糊器功能的操作的接收者。它包含了反馈(Feedbacks)、目标(Objectives)和调度器(Scheduler)等独立的操作

```rust
pub fn new(scheduler: CS, feedback: F, objective: OF) -> Self {
    Self {
        scheduler,
        feedback,
        objective,
        phantom: PhantomData,
    }
}
```

这些操作可能会改变模糊器的状态。为了遵循Rust的借用规则，这些阶段与模糊器是分开的，因为它们可能调用一些同时改变模糊器和状态的操作。

此外，模糊器还提供了如何处理单个测试用例以及如何评估新输入的定义。默认情况下，它的标准实现是基于反馈的模糊测试，其中FuzzOne是负责处理单个测试用例的操作，它调用调度器以获取要模糊的测试用例，并在测试用例上调用每个阶段。

```rust
fn fuzz_one(
    &mut self,
    stages: &mut ST,
    executor: &mut E,
    state: &mut EM::State,
    manager: &mut EM
) -> Result<CorpusId, Error>;
```

InputEvaluation是默认的输入评估操作，它执行目标程序并使用反馈判断输入是否有趣或是否是解决方案。

```rust
fn on_evaluation<OT>(
    &mut self,
    _state: &mut Self::State,
    _input: &<Self::State as UsesInput>::Input,
    _observers: &OT
) -> Result<(), Error>
    where OT: ObserversTuple<Self::State> { ... }
```

实现自定义架构的Fuzzer和State实体可以在LibAFL中重新创建类似于Vuzzer [62]的概念，将输入生成与即时评估解耦。

### 4.2.4 The Events Manager

事件管理器(Events Manager)是一个用于生成和处理事件的接口，可以用于实现并行模糊器的多节点同步，或仅用于日志记录。事件管理器的设计旨在最大限度地提高可伸缩性。实际上，如果我们假设通信通道具有线性可伸缩性（例如共享内存消息传递[67]），则管理器不会引入任何额外的瓶颈，因为每个模糊器在完全分离的数据上工作，并且待处理事件的过程会延迟到模糊器循环中特定的协调点，即在模糊器从调度器请求新的测试用例之前触发。

我们的框架包含了丰富的事件集合。例如，组件可以在一个模糊器将新的测试用例添加到其语料库时收到通知，接收包含输入的序列化版本以及被反馈认为有趣的观察者集合的事件。

### 4.2.5 The Metadata System

模糊算法通常需要考虑与给定测试用例相关联的信息或模糊器的整体状态。因此，LibAFL必须提供一种扩展语料库中测试用例和状态维护的数据的方式。一种简单但有效的解决方案是通过组合重新定义新类型，但在这种情况下，开发人员需要了解所有使用的算法所需的每个元数据。因此，为了提供这种功能并保持简单和性能，我们为State和Testcase组件设计了专用的元数据系统。特别地，在LibAFL中，任何实现了SerdeAny trait的结构体都可以用作元数据。我们创建了SerdeAny trait，它允许对trait对象进行序列化，而无需使用标准库。此trait需要具有序列化功能和静态生命周期，因为实例必须能够转换为trait对象。

```rust
pub trait HasMetadata {
    // Required methods
    fn metadata_map(&self) -> &SerdeAnyMap;
    fn metadata_map_mut(&mut self) -> &mut SerdeAnyMap;

    // Provided methods
    fn add_metadata<M>(&mut self, meta: M)
       where M: SerdeAny { ... }
    fn has_metadata<M>(&self) -> bool
       where M: SerdeAny { ... }
    fn metadata<M>(&self) -> Result<&M, Error>
       where M: SerdeAny { ... }
    fn metadata_mut<M>(&mut self) -> Result<&mut M, Error>
       where M: SerdeAny { ... }
}
```

LibAFL提供了可序列化的映射，可以存储可以转换为SerdeAny trait对象的任何实例。Testcase和State都持有此类型的映射，作为元数据的可扩展容器。通过这种方式，不同但相关的组件可以通过操作相同的元数据进行合作，同时完全忽略其他组件对其自己元数据的操作。然而，这是LibAFL中唯一引入小型运行时开销的模式，这是由于映射查找（目前实现为哈希映射）。

### 4.2.6 Composable Feedbacks

模糊器可能需要结合多个反馈来评估给定输入的“有趣程度”或支持不同的目标。在LibAFL中，为了避免需要从头开始创建新的聚合反馈，可以使用逻辑运算符组合反馈。例如，模糊器可能不想保存每个崩溃的输入，而是执行某种崩溃去重。在LibAFL中，可以通过使用将崩溃输入视为有趣的反馈和将输入视为有趣的反馈（当它触发之前未观察到的新堆栈跟踪时）来实现此目标。在这种情况下，可以通过逻辑AND组合这两个反馈来实现崩溃去重的目标。

```rust
macro_rules! feedback_and {
    ( $last:expr ) => { ... };
    ( $head:expr, $($tail:expr), +) => { ... };
}
```

### 4.2.7 The Monitor

LibAFL-based模糊器中的最后一个组件是监视器。它是从触发的事件中收集的统计信息，并将其显示给用户的组件。虽然此组件不是工作模糊器所必需的，但缺乏人工审查会降低模糊活动的效果。监视器允许开发人员报告和显示自定义统计信息，并实现各种报告接口，例如在终端中打印状态屏幕，或将数据转发到Grafana Web界面与statsd进行统计分析。

```rust
pub trait Monitor {
    // Required methods
    fn client_stats_mut(&mut self) -> &mut Vec<ClientStats>;
    fn client_stats(&self) -> &[ClientStats];
    fn start_time(&mut self) -> Duration;
    fn display(&mut self, event_msg: String, sender_id: ClientId);

    // Provided methods
    fn corpus_size(&self) -> u64 { ... }
    fn objective_size(&self) -> u64 { ... }
    fn total_execs(&mut self) -> u64 { ... }
    fn execs_per_sec(&mut self) -> f64 { ... }
    fn execs_per_sec_pretty(&mut self) -> String { ... }
    fn client_stats_mut_for(&mut self, client_id: ClientId) -> &mut ClientStats { ... }
}
```

## 4.3 Instrumentation Backends

LibAFL可以轻松地插入到任何仪器后端中，例如二进制翻译器或简单的编译器仪器传递。默认情况下，我们提供了附加库，将LibAFL与一些流行的仪器后端进行了绑定：LLVM [42]，SanitizerCoverage [44]，QEMU用户模式 [9]和Frida [1]。

LibAFL Targets中的运行时可以链接到任何SanitizerCoverage目标，通过使用编译器标志为模糊器添加覆盖和比较跟踪。SanCov支持允许用户创建与非C/C++ SanCov启用目标兼容的前端，例如用于Python的Atheris [40]和用于Rust的cargo-fuzz [8]。

LibAFL CC提供了一组LLVM传递，用于扩展Clang和其他基于LLVM的编译器，以跟踪边缘覆盖率、上下文敏感和K上下文敏感[7]边缘覆盖率、N-gram覆盖率[71]、覆盖率计数[73]、与CmpLog [6，27]和字典tokens进行比较以及使用autotokens的增强版本的AFL++的dict2file传递，该传递从诸如strcmp等有趣的函数中提取tokens。

LibAFL QEMU桥接了QEMU用户模式，并且在不久的将来将桥接到完整系统，以使用具有挂钩功能的新型模拟器API对目标的执行进行编程控制。在与QEMU的此接口周围，库公开了诸如执行器和助手的结构，以安装用于常见模糊任务（例如边缘覆盖跟踪、客户机快照还原和仅限二进制的ASan [26]）的默认挂钩。

LibAFL Frida提供了类似于QEMU桥接的功能，但具有动态二进制插桩（DBI）的特性，没有明确的宿主机-客户机分离。它包含了一个仅限二进制的ASan，并且与QEMU用户模式不同，它可以在除Linux之外的各种操作系统上工作，例如Windows、macOS和Android。

LibAFL还提供了用于共轭执行的插桩功能，通过LibAFL Concolic及其与SymCC [60]和SymQEMU [61]的Rust桥接。我们的API允许用户在Rust中编写自定义约束收集过滤器。在目标运行时，约束以易于处理的格式报告回到基于LibAFL的模糊器。然后可以在mutator中使用这些约束，例如调用求解器生成输入，或者用于模糊测试，类似于Borzacchiello等人提出的方法[15]。

除了这些稳定的后端，LibAFL还已经部分支持TinyInst [28]来在Windows和macOS上对二进制文件进行插装，以及Nyx [64]用于超级监视器级别的快照模糊测试。

# 5 APPLICATIONS AND EXPERIMENTS

在本节中，我们讨论LibAFL中实现的一些技术及其与前面部分所介绍的实体的关系。虽然LibAFL已经实现了许多技术，但在本节的第一部分中，我们重点关注了模糊化中许多已发表文献的焦点问题：roadblocks bypassing（例如，[6、17、57、62]），sructure-aware fuzzing（例如，[4、10、25、59]），corpus scheduling（例如，[18、21、72、73]）和energy assignment（例如，[12-14、43]）。

在撰写本文时，LibAFL集成的一系列技术列在表1中

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308042239034.png)

同时提供了这些技术是否需要在模糊器端实现附加组件或在目标端进行附加工具代码的信息。在讨论这些技术的实现方式之后，我们通过三个不同的实验集来评估它们的性能：

(1) 我们以代码覆盖率和漏洞检测的角度测量LibAFL中已实现和可供使用的多种方法的性能；

(2) 我们展示了如何结合在我们的框架中实现的正交方法来构建新的、以前未经评估的模糊器，并测量它们的性能；

(3) 最后，为了展示我们的框架在传统环境中的效率，我们将基于LibAFL的新型通用bit-level模糊器与AFL++和HonggFuzz等其他最先进的模糊器在FuzzBench [54]上进行对比和评估；

我们在一台配备Intel® Xeon® Platinum 8260 CPU（主频2.40 GHz）的x86_64机器上运行前两个实验集。数据集是FuzzBench套件的一个子集，选取了具有多样特性的程序。我们每次运行实验24小时，并重复每个实验五次以减轻模糊化过程中的随机性影响。

对于与其他模糊器的比较，我们在Google提供的FuzzBench套件上运行了完整的实验。每次运行持续23小时，重复20次。基准套件为每个基准提供了初始语料库，我们使用了默认设置。我们讨论的整体排名是基于由FuzzBench计算的平均标准化分数，该分数表示在给定基准上达到的最高中位代码或漏洞覆盖率的百分比。

最后，在本节的最后部分，我们展示了一个案例研究，说明了如何使用LibAFL轻松实现与第一部分展示的传统设置不同的模糊器，即一种差异化模糊器，它可以基于VM状态而不是代码覆盖率的不同反馈方式来发现以太坊VM中的逻辑漏洞。

最后，我们讨论了其他已实现的方法及其之间的关系，但出于时间和简洁性的考虑，未对其进行评估。

## 5.1 Bypassing Roadblocks

在模糊测试中，一个重要的研究领域是绕过难以解决的约束来增加代码覆盖率。例如，多字节比较对于使用通用bit-level fuzzing来说是一个严重的问题，因为随机的通用变异无法绕过这些比较，由于解空间巨大且盲目猜测不可行。LibAFL提供了几种现成的技术，用于实现能够克服这些障碍的模糊器。

在模糊测试领域，出现在文献中的第一个技术是LibFuzzer于2016年提出的value-profile [45]。该技术通过最大化指令两个操作数之间的匹配位数来解决比较指令。在LibAFL中，这是通过使用一个映射观察器和一个反馈来实现的，该反馈最大化映射的条目，并且在发现至少一个新的最大值时将输入视为有趣。这种类型的反馈，`MaxMapFeedback`，内置于我们的框架中，并且可以与基本的边覆盖率轻松组合使用。

LibAFL提供的第二种技术是AFL++的cmplog。这个解决方案基于RedQueen [6]和Weizz [25]采用的方法，可以通过查找和替换输入状态值来绕过比较。它通过对比较指令和任何具有两个指针作为参数的函数进行插桩，并在运行时记录相关值来工作。这个记录操作在每个测试用例的语料库中只进行一次，在我们的框架中，这是通过第二个执行器和一个处理cmplog映射的观察器来实现的。执行器在管道的开始阶段调用，并将记录的值存储为元数据。随后，一个自定义的变异器在输入中匹配模式，并在特定的变异器中用比较的另一个操作数进行替换。

第三种技术autotokens也受到了AFL++的启发，并且只能通过在目标上使用LTO（链接时优化）的方式进行插桩。在LibAFL CC中，这种插桩也可用于常规编译。该步骤从比较指令和具有立即值的函数中提取令牌，并将它们编码在二进制文件的一个部分中。然后，基于LibAFL的模糊器可以检索这些令牌，将它们添加到State的元数据字典中，并在变异器中使用这些令牌而无需任何开销。在这个实验中，我们将autotokens作为基准。事实上，由于它不引入任何开销，因此没有理由不在模糊器中使用它。

另一方面，cmplog和value-profile需要额外的插桩，并且value-profile可能会通过增加敏感性[71]来增加更多的输入，使语料库膨胀。因此，我们现在评估四种不同的选择。包括plain（具有autotokens的基准）、value_profile、cmplog以及结合了上述两种技术的value_profile_cmplog。这种组合在以前的研究中从未得到过评估，并且展示了我们解决方案的组合性允许尝试不同组件组合的可能性，并简化了复杂模糊器的开发。

表2显示了来自FuzzBench的5个基准测试的覆盖率增长图。总体而言，cmplog是表现最好的（95.94），紧随其后的是value_profile_cmplog（95.03），然后是plain（94.65）。相反，value_profile表现较差（90.13）。这很有趣，因为它表明autotokens本身能够解决许多路障，而无需额外的开销，从而使得plain在libpcap基准测试中表现出色，libpcap是一个具有许多输入状态比较的基准测试。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308042258847.png)

这证实了大多数路障是输入状态比较，并且value_profile的解决能力并不能对其引入的额外敏感性提供足够的回报（由于可能由于太多相似测试用例而导致的语料库的内部浪费）。然而，我们认为两种技术的结合在特定目标上可能有有趣的应用，这可以在将来进行研究。

## 5.2 Structure-aware Fuzzing

通用变异器可以通过生成无效的输入有效地对解析器进行压力测试，但也很重要的是模糊测试更深入的路径，超越解析例程，以发现这些代码区域中的错误。解决这个问题的常见方法是让模糊器了解输入格式。尽管从规范生成测试用例是最古老的模糊测试体现之一，但最近的文献中还探索了将现代基于反馈的模糊测试与结构感知变异器相结合的方法。此外，其他方法还提出了利用学习启发式方法来近似结构感知模糊测试，而无需提供任何用户提供的规范，从而在没有已知输入格式的目标上工作，并减少了人工工作的量。

LibAFL提供了几种处理结构化输入的技术，利用了其他组件对输入类型的灵活性。Nautilus [4]是一种基于语法的覆盖率引导模糊器，通过子树生成和从语料库中的其他输入进行替换等变异来演变语法树的语料库。在我们的框架中，我们为Nautilus实现了一种输入类型，即可以从头开始创建测试用例的生成器，以及一组使用生成器创建子树的变异器。其他所有实体保持不变，与语法模糊器完全兼容。例如，`ScheduledMutator`（用于调度其他变异器作为变异的trait）可以无缝地与最多8个堆叠变异一起使用。

```rust
pub trait ScheduledMutator<I, MT, S>: ComposedByMutations<I, MT, S> + Mutator<I, S>
where
    MT: MutatorsTuple<I, S>,
{
    // Required methods
    fn iterations(&self, state: &mut S, input: &I) -> u64;
    fn schedule(&self, state: &mut S, input: &I) -> MutationId;

    // Provided method
    fn scheduled_mutate(
        &mut self,
        state: &mut S,
        input: &mut I,
        stage_idx: i32
    ) -> Result<MutationResult, Error> { ... }
}
```

我们框架中的另一种技术是对Gramatron [68]的重新实现，Gramatron是一种基于语法的模糊器，它利用语法到自动机的转换来实现快速变异。在LibAFL中，我们提供了语法预处理工具，并且相关结构与Nautilus使用的结构类似，只是底层实现不同。

作为执行语法学习的例子，LibAFL实现了Grimoire [10]，这是一种使用导致覆盖率新颖性的输入部分作为标记来构建广义的“类树状”输入并执行类似语法的变异的模糊器。它还采用了基于标记提取的令牌级模糊化方法 [63]，这种方法是基于词法分析器的标记提取。虽然原始解决方案是特定于JavaScript的，但我们的实现是通用的，可以应用于任何编程语言。然后，经典的bit-level变异被应用于编码后的输入，并在目标执行之前解码。

为了评估这三种方法，我们决定使用未发现缺陷的数量，而不是代码覆盖率。实际上，这种类型的模糊器的有效性对代码覆盖率的依赖性较小，特别是与生成无效输入并通过探索错误路径来扩大覆盖率的变体相比。

在这个实验中，由于Grimoire和标记级模糊化方法都需要一些初始种子，我们使用Nautilus为这两个模糊器生成了4096个初始输入，以避免由于FuzzBench提供的种子语料库的质量而引起的偏差[36]。表3显示了每种变体在3个流行的编译器上找到的错误数量。语法感知方法仍然优于其他方法，在3个基准测试中Nautilus的性能大幅提升。但令人惊讶的是，标记级方法在PHP上与Nautilus的性能非常相似，考虑到它们从相似的种子输入开始。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308042229111.png)

鉴于这个实验的有希望的结果，我们决定通过将其与LibAFL提供的其他正交技术结合起来，进一步改进最佳表现者Nautilus。例如，我们将其与MOpt变异调度器 [48] 结合起来，创建一个新的未经评估的变体。到目前为止，MOpt只用于通过使用粒子群优化算法在学习阶段基于观察到的有效性分配概率来调度bit-level变异。我们重复了实验，并将这个新的变体与3个语法基准进行了比较，每个模糊器运行5次，持续时间为24小时。结果如表4所示，显示了当Nautilus (蓝色)与MOpt (绿色) 结合在一起时，在mruby上表现良好，但在php上表现较差。总的来说，这证实了MOpt的高度目标依赖性特性，这一点在bit-level fuzzing中也被观察到 [27]。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308042231423.png)

## 5.3 Corpus Scheduling

在选择下一个要使用的测试用例时，有许多不同的研究重点。最简单的解决方案要么依赖于随机选择，要么依赖于FIFO队列。LibAFL提供了这两种解决方案，以及受其他最先进的模糊器启发的其他调度器。第一种调度器来自AFL，在每个队列循环中从语料库中选择一组“喜爱的”种子。种子是基于执行速度和输入长度选择的，同时保留最大的覆盖率。在LibAFL中，我们实现了这种方法的通用版本，称为`MinimizerScheduler`，它根据给定映射反馈的条目计算minset，但权重是可定制的。为了简单起见，在下面的实验中，我们使用了AFL使用的传统加权策略。

第二个调度器使用AFL++提出的一种基于概率抽样的最近改进。其思想是将语料库中的每个测试用例映射到一个概率和一个在计算分数方面更“有希望”的相邻测试用例。分数是通过使用各种指标（包括执行时间和覆盖图大小）计算的，并且也用于计算概率。在选择过程中，调度器从语料库中随机选择一个测试用例，并生成一个介于0和1之间的随机数。如果这个数字小于与测试用例关联的概率，则选择该测试用例，否则选择更有希望的邻居。

第三个调度器来自TortoiseFuzz [73]，通过使用三个安全影响度量来优先考虑输入：具有块和函数粒度的内存操作，以及循环回边计数。跟踪这些度量需要自定义的插装，LibAFL CC中使用LLVM实现，并使用新的相关观察器。调度器根据较高得分优先考虑输入，并根据输入长度和执行时间来解决冲突。

表5显示了基于上述调度器的四个模糊器的结果：accouting是使用TortoiseFuzz插装和具有函数级粒度的调度器，minimizer使用AFL的算法，rand是随机选择的基线，weighted使用AFL++的概率调度器。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308042218151.png)

在未覆盖覆盖率的平均归一化分数方面，weighted获得最佳结果（得分为98.91），紧随其后的是minimizer（98.71）、accounting（98.03），rand排名最后（97.50）。所有解决方案之间的小差异是有趣的，表明尽管文献中对这个问题给予了很多关注，但简单的随机方法仍然能够取得不错的结果，并且完全适用于快速目标的实际模糊测试活动。我们也预计accounting会优于minimizer，但实际情况并非如此。然而，与TortoiseFuzz论文中使用AFL的原始评估相比，LibAFL通过在进程内执行目标实现了更高的吞吐量，从而减少了这些调度技术的差异和影响。虽然在快速目标上差异微不足道，但我们认为在慢速目标上，预先决定要模糊的测试用例可以对模糊测试活动产生很大的影响，因此调度问题至关重要。

## 5.4 Energy Assignment

能量分配旨在回答一个问题：在语料库中的单个输入需要进行多少次变异才能生成新的测试用例。这个问题通常被称为功率调度问题（power scheduling problem），由Böhme等人在2016年的文献中首次提出。

最简单的解决方案是使用一个恒定的值，而最常用的简单方法是为每个种子分配一个在给定区间内的随机值。LibAFL也提供了这种简单算法，名为"plain"，在使用默认变异阶段时，范围为1到128。

虽然许多解决方案调整这个特定的参数，通常是为了特定领域的模糊测试工具 [13, 59]，但AFLFast的开创性工作仍然是对于通用模糊测试问题的最完整的覆盖。AFLFast提出了六种不同的算法：exploit、explore、coe、fast、lin和quad。具体来说，exploit根据一些指标（如输入的执行时间、覆盖密度和测试用例的创建时间）分配能量。explore分配较低的能量，将exploit的能量除以一个常数。coe是指数方案，将高频边触发的输入的能量分配为0，直到它们变为低频。fast是coe的扩展，在这个方案中，它不是将能量分配为0，而是将能量分配为与访问的高频边的数量成反比。lin根据测试用例被选择为模糊次数的线性关系分配能量，而quad则是二次关系。

```rust
pub enum PowerSchedule {
    EXPLORE,
    EXPLOIT,
    FAST,
    COE,
    LIN,
    QUAD,
}
```

在LibAFL中，我们设计了一个基于元数据的功率调度接口，并在此基础上实现了上述六种算法。我们的实现是基于AFL++中优化后的版本，该版本是在AFLFast论文之后的多年开发的，从未在文献中进行过评估。在这六种算法中，最有效的三种是explore（AFL的默认算法）以及coe和fast（AFL++的默认算法）。因此，我们将重点测试这三种算法，并将它们与基于plain算法（从未被评估过）的基线进行比较。表6显示了代码覆盖率方面的结果。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308051109670.png)

总体而言，考虑到所有基准测试的平均归一化得分，explore是表现最好的（99.72），其次是fast（99.43），然后是plain（98.12）和coe（97.08）。这些结果确认了AFL++版本中功率调度的趋势，其中fast和explore是最好的表现者，fast现在是AFL++中的默认调度。

LibAFL的实现再次强调了我们对语料库调度问题的相同观察。对于快速目标的快速模糊测试器减少了通过使用更复杂调度获得的收益，因为较少有用的执行对吞吐量的影响有限。coe的得分比随机基线还差，可以考虑是特定于目标。实际上，尽管该算法在某些目标上表现不佳（尤其是在bloaty基准测试上），但在其他目标上（例如一些mbedtls的运行中）表现最好。

## 5.5 A Generic Bit-level Fuzzer

LibAFL的主要目标是成为一个万能工具，用于构建定制的模糊测试器，但我们也努力提供一些优秀的默认实现，以供开箱即用的通用bit-level模糊测试。

在本节中，我们展示了一个用于使用通用变异器对LibFuzzer测试框架进行模糊测试的LibAFL前端。我们将这个模糊测试器与用于每天对数千个开源项目进行模糊测试的Google OSS-Fuzz的最新模糊测试器进行了对比，包括AFL++、HonggFuzz和LibFuzzer（启用了Entropic选项以获得更好的性能）。

与LibFuzzer和前面实验中使用的任何其他模糊测试器一样，我们的模糊测试器使用内部进程执行器来运行目标测试框架，而AFL++和HonggFuzz则通过IPC形式（如管道）控制目标测试框架。我们的模糊测试器还采用了前面实验中涵盖的一些改进措施：cmplog、加权语料库调度器、explore能量分配方案和MOpt变异器。

对于这个特定的实验，我们向FuzzBench服务提交了一个请求，要求对22个不同的基准测试进行四个模糊测试器的测试，并评估达到的代码覆盖率。每个模糊测试器安排运行时间为23小时，每个实验都重复了20次以减小随机性的影响。每个基准测试的结果报告在表7中。根据所有目标中覆盖的代码的平均归一化得分，总体结果是LibAFL明显优于所有其他模糊测试器，得分为98.61，其次是HonggFuzz（96.65）、AFL++（96.32）和Entropic（94.22）。这个结果尤其重要，因为其他模糊测试器是根据多次FuzzBench运行的结果逐步改进的[54]，而LibAFL是独立开发的，与这个基准测试套件无关。

通过独立查看结果，我们可以看到LibAFL在3个基准测试上表现出色，分别是harbfuzz、openthread和sqlite3。值得注意的是，LibAFL是唯一一个能够在mbedtls上通过解除模糊测试器的饱和状态在几次运行中取得覆盖突破，并且在openthread上保持一致的模糊覆盖率。

另一方面，我们的模糊测试器在libpng上明显表现不佳，与AFL++和HonggFuzz相比，覆盖率几乎相同，与Entropic一样。这两个模糊测试器缺少的覆盖率可以通过它们使用的执行模型来解释，即内部进程与其他两个模糊测试器使用的外部进程执行器。后者速度较慢，但在处理超时时更可靠。我们的方法的优势在于，通过在前端改变几行代码，可以轻松克服此类限制，在这种情况下，将InProcessExecutor改为ForkserverExecutor即可。

```rust
pub type InProcessExecutor<'a, H, OT, S> = GenericInProcessExecutor<H, &'a mut H, OT, S>;

pub struct ForkserverExecutor<OT, S, SP>
where
    SP: ShMemProvider,
{ /* private fields */ }
```

总的来说，这些结果表明LibAFL是一个成熟的框架，能够成为现代通用模糊测试器的支柱，可以与最先进的解决方案竞争。我们预见基于LibAFL的高度可定制但带有良好默认设置的通用实现，如AFL++，将得到发展和推广。

## 5.6 Differential Fuzzing

在前面的部分中，我们介绍了基于覆盖率引导模糊测试的变种。然而，LibAFL并不局限于代码覆盖率及其衍生物（例如上下文和ngrams [71]），而且还可以与其他类型的反馈一起使用。在本节中，我们将讨论一个前端的开发，受到NeoDiff [50]的启发，用于对比两个以太坊虚拟机的差异性模糊测试。

简而言之，NeoDiff最初是用Python编写的，它比较了两个给定相同输入的虚拟机执行的结果。为了实现这一目标，它使用了状态哈希（state hash），这是寄存器值、内存以及每个指令处执行轨迹的栈的概率采样的哈希值。作为进化语料库的反馈，它使用了类型哈希（type hash），即每个指令的操作码和栈上前两个项目的类型的哈希值。任何生成具有新类型哈希的轨迹的输入都会被添加到语料库中。

在LibAFL中，NeoDiff可以通过利用差异执行器组件来实现，这是一个充当两个底层执行器的代理的结构。内部执行器的类型是CommandExecutor，这是一种简单的执行器类型，它根据命令行启动一个新进程，并将其观察者链接到这些命令的stdout上，因为以太坊虚拟机在执行输入字节码后会在JSON格式中打印其轨迹。

```rust
pub struct CommandExecutor<OT, S, T> { /* private fields */ }
```

这些观察者由自定义反馈（TypeHashFeedback）处理，TypeHashFeedback会解码轨迹，计算类型哈希，并将其与链接的反馈状态中到目前为止观察到的其他哈希进行比较。

```rust
pub trait HasFeedback: UsesState
where
    Self::State: HasClientPerfMonitor,
{
    type Feedback: Feedback<Self::State>;

    // Required methods
    fn feedback(&self) -> &Self::Feedback;
    fn feedback_mut(&mut self) -> &mut Self::Feedback;
}
```

差异反馈（differential feedback）用作目标，负责比较两个观察者。

```rust
pub struct DiffFeedback<F, I, O1, O2, S>
where
    F: FnMut(&O1, &O2) -> DiffResult,
{ /* private fields */ }
```

差异反馈使用状态哈希（state hash）来比较链接到两个执行器的观察者，并且如果不同，则只有在具有新的类型哈希用于去重时才将输入视为解决方案。

```rust
pub trait HashSetState<T> {
    // Required methods
    fn with_hash_set(hash_set: HashSet<T>) -> Self;
    fn update_hash_set(&mut self, value: T) -> Result<bool, Error>;
}
```

总体而言，我们在LibAFL中从头开始重新实现NeoDiff共花费了2个工作日的时间，并由900行Rust代码组成。我们还决定与原始的NeoDiff实现进行比较，采用了原始论文中使用的相同度量标准，即具有唯一类型哈希的差异输入的数量。我们将这两个模糊器都运行了12个小时，测试了原始论文中测试的相同的go-ethereum和openethereum版本。在图2中，我们报告了这两个模糊器随时间的发现情况，显示我们的实现在这个度量标准下明显优于原始实现。我们认为，在这个实验中，LibAFL内置的bit-level变异器对此起到了主要作用。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308051131556.png)

在唯一差异指令方面，原始论文报告了有6个指令导致了找到的差异。在我们的实验中，我们通过NeoDiff在12小时内只找到了5个指令，而通过我们的实现找到了15个指令。

具体而言，这些差异操作码是（以十六进制表示）：3、31、38、3b、3c、3f、44、45、46、52、5a、f1、f2、f4、fa。原始NeoDiff和我们的NeoDiff实现都找到的操作码在粗体中显示，其他操作码只有LibAFL版本发现。

我们的发现是NeoDiff 6个操作码的超集，表明LibAFL的性能远远优于Python实现。