*tables* 模块提供NFD的主要数据结构（ *data structures* ）。

**转发信息库** （FIB，*Forwarding Information Base* ）（第3.1节）用于将兴趣包转发到匹配数据（ *matching Data* ）的潜在源（ *potential source(s)* ）。它与IP的FIB表类似，只是它允许列出传出的 *face* 是一个 *face* 列表而不是单个 *face* 。

**网络区域表** （ *The Network Region Table* ，第3.2节）包含用于移动性支持（ *mobility support* ）的生产者区域名称的列表。

**内容存储** （CS，*Content Store* ，第3.3节）是一个对数据包（ *Data packet* ）的缓存。为了满足将来请求相同数据的兴趣，将到达数据包尽可能长地放置在此缓存中。

**未决兴趣表** （PIT，*Pending Interest Table* ，第3.4节）跟踪（ *track* ）向上游内容源转发的兴趣（ *Interest* ），构造一条反向路劲，以便可以将数据向下游发送到其请求者。它还包含用于 **环路检测** 和 **测量目的** 的最近满足的兴趣（ *recently satisfied Interests* ）。

**Dead Nonce List** （第3.5节）作为未决兴趣表的补充，以进行循环检测。

**策略选择表** （*The Strategy Choice Table* ，第3.6节）包含为每个名称空间（ *namespace* ）选择的转发策略（第5节）。

转发策略使用 **测量表**（*The Measurements Table* ，第3.7节）来存储有关名称前缀的测量信息。

**FIB** 、**PIT** 、 **策略选择表** 和 **测量表** 在其索引结构上有很多共性。为了提高性能并减少内存使用，我们设计了 **名称树** （*Name Tree* 第3.8节）结构，用于在这四个表之间共享一个通用索引。

### 3.1 转发信息库（FIB）

转发信息库（FIB）用于将兴趣包转发到匹配数据的潜在源[9]。对于需要转发的每个兴趣，将在FIB上执行最长的前缀匹配（ *longest prefix match* ）查找，并且存储在找到的FIB条目中的传出 *face* 列表是转发（ *forwarding* ）的重要参考。

第3.1.1节概述了FIB的结构（ *structure* ）、语义（ *semantic* ）和算法（ *algorithm* ）。第3.1.2节介绍了NFD其余的部分是如何如何使用FIB的。FIB算法的实现将在3.8节中讨论。

#### 3.1.1 结构和语义

![图5  FIB及其相关的条目](assets/1583212520757.png)

<center>图5  FIB及其相关的条目</center>

- **FIB条目和下一跳记录**

  FIB条目（`nfd::fib::Entry`）包含 **名称前缀** （ *Name prefix* ）和 **下一跳记录** （ *Nexthop record* ）的非空集合。FIB表中有某个前缀的FIB条目意味着，如果收到以该名称前缀开头的兴趣包，则可以通过此FIB条目中的下一跳记录所提供的 *face* 来找到匹配数据的潜在来源。

  每个下一跳记录（`nfd::fib::NextHop`）包含面向潜在内容源的传出 *face* 及其路由成本（ *cost* ）。 一个FIB条目最多可以包含一个朝向同一出口（ *outgoing* ） *face* 的下一跳记录。在FIB条目中，下一跳记录按升序排序。路由成本是下一跳记录之间的相对成本，具体值为多少无关紧要。

  与RIB（第7.3.1节）不同，FIB条目之间没有继承（ *inheritance* ）。FIB条目中的下一跳记录列表是此FIB条目仅有的“有效”下一跳列表。

- **FIB**

  FIB（`nfd::Fib`）是FIB条目的集合，按名称前缀索引。支持通常的插入、删除及完全匹配（ *exact match* ）操作。FIB条目可以在前向迭代器中以未指定的顺序进行迭代。

  最长前缀匹配算法（`Fib::findLongestPrefixMatch`）找到FIB条目，该条目应用于指导兴趣的转发。 它以名称作为输入参数。此名称应该是兴趣（ *Interest* ）中的名称字段。返回值是FIB条目，其名称前缀为：

  - （1）输入参数参数的前缀
  - （2）在满足条件（1）的结果中的的最长者，如果没有FIB条目满足条件（1），则返回NULL。

#### 3.1.2 使用（ *Usage* ）

**FIB仅通过使用FIB管理协议来更新** ，该协议在NFD方面由FIB管理器（ *FIB manager* ）操作（第6.5节）。通常， `FIB manager` 从`RIB Management`（第7节）获取命令，`RIB Management`既接收手动配置（ *configured manually* ）或由应用程序注册的静态路由，也接收来自路由协议的动态路由。由于大多数FIB条目最终都来自动态路由，因此，如果网络具有少量的公告前缀，则FIB预计将包含少量条目。

FIB预计会相对稳定。FIB更新由RIB更新触发，而RIB更新又是由手动配置、应用程序启动或关闭以及路由更新引起的。在稳定的网络中，这些事件很少发生。但是，每个RIB更新都会导致大量FIB更新，因为一个RIB条目的更改可能会由于继承而影响其后代。

最长前缀匹配算法用于传入兴趣管道（ *incoming Interest pipeline* ）中的转发（第4.2.1节）。对于每个传入兴趣最多调用一次。

`cleanupOnFaceRemoval`函数（在`daemon/table/cleanup.hpp`中声明）是一个在 *face* 移除时用于FIB-PIT清理的函数。当一个 *face* 销毁（ *destroyed* ）后会调用此函数。它访问所有FIB和PIT条目，并删除FIB下一跳记录，PIT入记录和PIT出记录，这些记录指向已删除的 *face* 。丢失最后一条下一跳记录的FIB条目也将被删除。不同的是，丢失所有其入记录和出记录的PIT条目被保存在表$^2$中。FIB和PIT的清除在同一函数中执行，因为两个表都存储在`NameTree`中（第3.8节），因此此函数可以在一次`NameTree`枚举中清除两个表。

> $^2$ This is pending discussion in [https://redmine.named-data.net/issues/3685#note-7](https://redmine.named-data.net/issues/3685#note-7).

### 3.2 网络区域表（Network Region Table）

网络区域表（`nfd::NetworkRegionTable`）用于移动性支持（第4.2.4节）。 它包含一组无序的生产者区域名称，这些名称取自NFD配置文件（第6.7.2节）。如果兴趣的转发提示（ *forwarding hint* ）中的任何委托名称（ *delegation name* ）是此表中任何区域名称的前缀，则表示兴趣已到达生产者区域，应根据其名称而不是委托名称进行转发。

### 3.3 内容存储（CS）

内容存储（CS）是用作数据包的缓存。转发管道（ *Forwarding pipeline* ，第4节）将到达的数据包放在CS中，这样就可以满足将来请求相同数据的兴趣，而无需进一步转发。

CS提供了插入数据包，查找与兴趣匹配的缓存数据以及枚举缓存数据的过程。第3.3.1节描述了这些过程的语义及其用法。

CS在（`nfd::cs::Cs`）类中实现。该实现由两部分组成：查找表（ *lookup table* ）和缓存替换策略（ *cache replacement policy* ）。查找表（第3.3.2节）是CS条目的基于名称的索引，其中存储了缓存的数据包。缓存替换策略（第3.3.3节）负责将CS保持在容量限制之下。它单独维护一个清理索引，以便确定在CS满（ *full* ）时删除哪个条目。 NFD提供了多种缓存替换策略，包括优先级FIFO策略和LRU策略，可以在NFD配置文件中选择它们。

#### 3.3.1 语义和使用（Semantic and Usage）

- **插入（Insertion）**

  不管是在 *incoming Data pipleline* （第4.3.1节）还是在 *Data unsolicited pipleline* （第4.3.2节）中，对于任意的数据包，只要 *forwarding* 可确保其完全符合要处理的数据包的条件，该数据包就会被插入CS（`Cs::insert`）中 。（例如，数据包不违反基于名称的范围控制[10]）。

  在存储数据包之前，先评估准入策略（ *admission policy* ）。本地应用程序可以通过附加到数据包的NDNLPv2 [5] `CachePolicy`字段来提示接纳策略，这些提示被认为是建议性的。

  通过准入策略后，数据包（ *Data packet* ）将被存储下来，同时保存该数据包过时（ *stale* ）的时间点（ *time point* ），并且不会去满足带有`MustBeFresh Selector`的兴趣（ *Interest* ）。

  CS配置有容量限制，此时将对其进行检查。如果此新数据包的插入导致CS超过容量限制，则缓存替换策略将驱逐过多的条目以使CS处于容量限制之下。

- **查找（Lookup）**

  CS提供了一个`异步`查找API（`Cs::find`）。 *incoming Interest pipleline* （第4.2.1节）使用传入兴趣（ *Interetst* ）来调用此API。搜索算法会将与兴趣（ *Interest* ）最匹配的数据包（ *Data packet* ）提供给 *ContentStore hit pipleline* （第4.2.3节），或者如果没有匹配项，则通知 *ContentStore miss pipleline* （第4.2.4节）。

- **枚举和条目抽象（Enumeration and Entry Abstraction）**

  可以通过正向迭代器（ *forward iterators* ）枚举 *Content Store* 。此功能未在NFD中直接使用，但在仿真环境中可能很有用。

  为了保持稳定的枚举接口，且仍允许替换CS实现，将迭代器取消引用（ *dereferenced* ）为`nfd::cs::Entry`类型，这是CS条目的抽象。这种类型的公共API允许调用者获取数据包，无论它是否是自发的，以及何时该数据包会过时。内容存储实现可以定义自己的具体类型，并在枚举期间转换为抽象类型。

#### 3.3.2 查询表（Lookup Table）

*Table* 是一个有序的容器，用于存储具体的条目（`nfd::cs::EntryImpl`，条目抽象类型的子类）。该容器按具有隐式摘要（ *implicit digest* ）的数据名称排序。$^3$

> $^3$ 隐式摘要计算会占用大量CPU。该表进行了优化，可以在大多数情况下避免隐式摘要计算，同时仍保证正确的排序顺序。

查找完全使用表来完成。NFD优化了查找过程（`Cs::find*`），以最大程度地减少预期情况下访问的条目数。在最坏的情况下，查找将访问所有以兴趣名称为前缀的条目。

尽管查找API是异步的，但当前实现会同步进行查找。

该表使用`std::set`作为基础容器，因为它在基准测试中表现良好。先前的CS实现使用 *skip list*，但其性能比std :: set差，这可能是由于算法复杂性和代码质量所致。

#### 3.3.3 缓存替换策略（Cache Replacement Policy）

缓存替换策略将CS保持在其容量限制内。主要容量限制是根据缓存的数据包的数量来衡量的。选择此度量标准而不是选择数据分组大小，因为表索引的内存开销对于小数据包可能很重要，因此数据包大小度量标准将不准确。可以在NFD配置文件`table.cs\_max\_packets`项中配置此容量限制，也可以在模拟环境中通过`Cs::setLimit`API进行配置。 允许运行时更改容量限制。

NFD通过实现`nfd::cs::Policy`的多个子类提供多种策略实现 可以在NFD配置文件`table.cs\_policy`项中配置有效策略，也可以在模拟环境中通过`Cs::setPolicy`API进行配置。此配置只能在初始化期间应用，不允许在运行时更改。

- **策略类API（Policy class API）**

  `nfd::cs::Policy`是所有缓存替换策略实现的基类。

  容量限制存储在`Policy`类上，可以通过`Policy::setLimit`公共方法进行更改。策略实现必须提供基类中纯虚函数`evictEntries`的实现，以便基类`Policy::setLimit`公共方法通过驱逐足够的条目使CS重新回到新的容量限制下，来处理容量限制的降低。除了主要容量限制（称为“硬限制-*hard limit* ”）之外，策略还可以提供其他容量限制（例如证书的单独限制或基于数据包大小的限制）。

  为了使策略维持容量限制，它需要知道哪些数据包已添加到缓存及其访问模式。 因此，策略类公开了四个公共方法来接收这些信息：

  - 插入新条目后，将调用`Policy::afterInsert`；
  - 在同一数据刷新现有条目后，将调用`Policy::afterRefresh`；
  - `Policy::beforeErase`将在通过管理命令删除条目之前被调用（当前未使用）；
  - 当通过查找找到某个条目并将其用于转发之前，将调用`Policy::beforeUse`。

  这些公共方法调用相应的纯虚函数：`doAfterInsert`、`doAfterRefresh`、`doBeforeErase`和`doBeforeUse`。 这些纯虚函数以及`evictEntries`应该在子类中重写（ *overriden* ）。

  根据通过上述API收到的信息，策略会维护一个内部清除索引，该索引用于确定在CS超过容量限制时应清除哪个数据包。在每个策略实施中都定义了此内部清理索引的结构。它应该通过迭代器（`nfd::cs::iterator`）引用CS条目（存储在表中）。 当策略决定逐出一个条目时，它应该发出beforeEvict信号来通知CS从表中删除该条目，然后删除该策略的内部清除索引中的相应项目。 请注意，对于通过beforeEvict信号驱逐的条目，不会调用beforeErase。

- 

