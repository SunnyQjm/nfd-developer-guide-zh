在NFD转发过程中，转发策略（ *forwarding strategy* ）智能地做出决策，决定是否，何时以及在何处转发 `Interest`  。转发策略（ *forwarding strategy* ）与转发管道（ *forwarding pipelines* ，第4节）一起构成了NFD中的数据包处理逻辑。在转发管道（ *forwarding pipeline* ）处理过程中，当需要做出有关 `Interest` 转发的决定时，会触发转发策略（ *forwarding strategy* ）中对应的方法进行转发决策。此外，该策略可以接收有关其转发决策结果的通知，例如，当一个被转发的 `Interest` 被 *satisfied* 、超时或  `Nack`  时。

我们在NDN应用程序方面的经验表明，不同的应用程序需要不同的转发行为。例如，文件检索应用程序希望从具有最高带宽的内容源中检索内容，音频聊天应用程序希望具有最小的延迟，而数据集同步库（例如 *ChronoSync* ）希望对所有可用的 *Face* 多播 `Interest` ，以便 `Interest` 能到达其对等方（ *peers* ）。对不同转发行为的需求促使我们在NFD中采用多种转发策略。

尽管NFD中有多种策略，但对某个具体的 `Interest` 的转发必须由单一策略决定。NFD实现了基于每个命名空间（ *per-namespace* ）的策略选择。运营商可以为名称前缀配置特定策略，该名称前缀下的兴趣将由该策略处理。此配置记录在策略选择表（第3.6节）中，并由转发管道（ *forwarding pipelines* ）进行查询。

对于目前的实现，转发策略的选择是通过本地配置的。同一 `Interest` 可以通过不同路由节点上的完全不同的策略来处理。自2016年1月起，我们将讨论此方法的替代方法。一个值得注意的想法是路由注释（ *routing annotations* ），其中路由公告带有有关首选策略的指示。

### 5.1 Strategy API

从概念上讲，策略是为抽象机（策略API）编写的程序。这种抽象机器的“语言”包含标准的算术和逻辑运算，以及与NFD其余部分的交互。此抽象机的状态存储在NFD表中。

每个NFD策略都作为 `nfd::fw::Strategy` 类的子类实现，该类为实现的策略与NFD其余部分之间的交互提供API。该API是策略可以访问NFD组件的唯一方法，因此策略API中的可用功能决定了NFD策略可以或不能执行的操作。

本节将分三部分别讲述策略中的触发器（ *Triggers* ）、操作（ *Actions* ）和存储（ *Storage* ）。NFD的其它组件（ *目前应该只有 forwarding pipelines* ）通过调用其中一种触发器（ *triggers* ，第5.1.1节）来调用策略。而在策略内部，是通过执行某个操作（ *actions* ，第5.1.2节）来做出转发决策的。 策略还被允许在某些表条目存储有关的信息（ *Storage* ，第5.1.3节）。

#### 5.1.1 触发器（Triggers）

> 下面插入的代码片段来自：[`NFD/daemon/fw/strategy.hpp`](https://gitea.qjm253.cn/PKUSZ-future-network-lab/MIR/src/branch/master/daemon/fw/strategy.hpp) 和 [`NFD/daemon/fw/strategy.cpp`](https://gitea.qjm253.cn/PKUSZ-future-network-lab/MIR/src/branch/master/daemon/fw/strategy.cpp)

触发器是策略程序的入口点。触发器被声明为 `nfd::fw::Strategy` 类的虚拟方法，并且希望被子类覆盖以实现不同子类特定的行为。

##### After Receive Interest Trigger

```cpp
/** \brief trigger after Interest is received
   *
   *  The Interest:
   *  - does not violate Scope
   *  - is not looped
   *  - cannot be satisfied by ContentStore
   *  - is under a namespace managed by this strategy
   *
   *  The PIT entry is set to expire after InterestLifetime has elapsed at each downstream.
   *
   *  The strategy should decide whether and where to forward this Interest.
   *  - If the strategy decides to forward this Interest,
   *    invoke \c sendInterest for each upstream, either now or shortly after via a scheduler event,
   *    but before PIT entry expires.
   *    Optionally, the strategy can invoke \c setExpiryTimer to adjust how long it would wait for a response.
   *  - If the strategy has already forwarded this Interest previously and decides to continue waiting,
   *    do nothing.
   *    Optionally, the strategy can invoke \c setExpiryTimer to adjust how long it would wait for a response.
   *  - If the strategy concludes that this Interest cannot be satisfied,
   *    invoke \c rejectPendingInterest to erase the PIT entry.
   *
   *  \warning The strategy must not retain shared_ptr<pit::Entry>, otherwise undefined behavior
   *           may occur. However, the strategy is allowed to store weak_ptr<pit::Entry>.
   */
virtual void
    afterReceiveInterest(const FaceEndpoint& ingress, const Interest& interest,
                         const shared_ptr<pit::Entry>& pitEntry) = 0;
```

此触发器声明为 `Strategy::afterReceiveInterest` 方法。此方法是纯虚拟方法，因此必须被子类覆盖。

当NFD收到一个 `Interest` ，会传递给 *Incoming Interest* 管道处理，经过必要的检查之后如果这个 `Interest` 还需要被转发，则 *Incoming Interest* 管道会触发 `Interest` 所关联策略的 `Strategy::afterReceiveInterest` 触发器（以收到的 `Interest`、入口 *Face* 和对应的PIT条目作为参数）。这样符合条件的 `Interest` 需要满足以下几条限定：

- `Interest` 不违反 `/localhost`  *scope* 限制；
- `Interest` 不是循环的（ *loop* ）；
- `Interest` 没有命中缓存；
- `Interest` 位于此策略管理的名称空间下。

当本触发器被触发后，策略应决定是否以及在何处转发此 `Interest` 。大多数策略都需要读取FIB条目来做出此决定，这可以通过调用 `Strategy::lookupFib` 访问器函数来获得。如果该策略决定转发此 `Interest` ，则应至少调用一次 *send interest* 操作；如果该策略得出结论认为不能转发此 `Interest` ，则应调用 `Strategy::setExpiryTimer` 操作并将该定时器设置为立即过期 $^8$ ，以便相关PIT条目最终可以被移除。

> $^8$ **警告** ：尽管允许策略通过计时器延迟调用 *send interest* 操作，但在特殊情况下这种转发可能永远不会发生。例如，如果在等待此类计时器的过程中，NFD管理员在兴趣的名称空间上更新策略，则该计时器事件将被取消，并且新策略可能直到PIT条目中的所有记录都到期后才决定转发该兴趣。

##### After Content Store Hit Trigger

```cpp
/** \brief trigger after a Data is matched in CS
*
*  In the base class this method sends \p data to \p ingress
*/
virtual void
afterContentStoreHit(const shared_ptr<pit::Entry>& pitEntry,
                   const FaceEndpoint& ingress, const Data& data);
```

此触发器声明为 `Strategy::afterContentStoreHit` 方法。基类提供了默认实现，该实现将匹配的数据发送到下游。

```cpp
NFD_LOG_DEBUG("afterContentStoreHit pitEntry=" << pitEntry->getName()
                << " in=" << ingress << " data=" << data.getName());

this->sendData(pitEntry, data, ingress);
```

##### After Receive Data Trigger

```cpp
/** \brief trigger after Data is received
*
*  This trigger is invoked when an incoming Data satisfies exactly one PIT entry,
*  and gives the strategy full control over Data forwarding.
*
*  When this trigger is invoked:
*  - The Data has been verified to satisfy the PIT entry.
*  - The PIT entry expiry timer is set to now
*
*  Within this function:
*  - A strategy should return Data to downstream nodes via \c sendData or \c sendDataToAll.
*  - A strategy can modify the Data as long as it still satisfies the PIT entry, such as
*    adding or removing congestion marks.
*  - A strategy can delay Data forwarding by prolonging the PIT entry lifetime via \c setExpiryTimer,
*    and forward Data before the PIT entry is erased.
*  - A strategy can collect measurements about the upstream.
*  - A strategy can collect responses from additional upstream nodes by prolonging the PIT entry
*    lifetime via \c setExpiryTimer every time a Data is received. Note that only one Data should
*    be returned to each downstream node.
*
*  In the base class this method invokes \c beforeSatisfyInterest trigger and then returns
*  the Data to downstream faces via \c sendDataToAll.
*/
virtual void
afterReceiveData(const shared_ptr<pit::Entry>& pitEntry,
               const FaceEndpoint& ingress, const Data& data);
```

此触发器声明为 `Strategy::afterReceiveData` 方法。当传入的数据恰好满足一个PIT条目时，将调用此触发器，并为该策略提供对数据转发的完全控制权。调用此触发器时，已验证数据满足PIT条目，并且PIT条目到期计时器设置为立即触发。基类提供了默认实现，它调用 `Strategy::beforeSatisfyInterest` 触发器，然后将 *Data* 返回到所有下游 *Face* 。

```cpp
NFD_LOG_DEBUG("afterReceiveData pitEntry=" << pitEntry->getName()
                << " in=" << ingress << " data=" << data.getName());

this->beforeSatisfyInterest(pitEntry, ingress, data);

this->sendDataToAll(pitEntry, ingress, data);
```

在此触发器内：

- 策略应通过 `Strategy::sendData` 或 `Strategy::sendDataToAll` 方法将数据返回到下游节点；
- 策略可以对 `Data` 做一些修改，只要该 `Data` 仍能满足PIT条目，例如添加或删除拥塞标记（ *`Data` 的拥塞标记是在 LpPacket 里面添加一个 LpHeaderField 来实现的，所以添加或删除拥塞标记并不影响 `Data` 本身* ）；
- 策略可以通过 `Strategy::setExpiryTimer` 延长PIT条目的生存时间，从而延迟 `Data` 转发，并在删除PIT条目之前转发 `Data` ；
- 策略可以在此触发器内收集有关上游的度量（ *measurements about the upstream* ，例如可以通过收到 `Data` 包的时间以及PIT条目的 *out-record* 中记录的兴趣包转发出去的时间计算RTT）；
- 策略可以通过延长每次接收到的 `Data` 的PIT条目的寿命，从其他上游节点收集响应（ *即可以期望等待从多个上游节点收到 `Data` 然后再做决策将哪个 `Data` 转发到下游* ）。但请注意，每个下游节点仅应返回一个 `Data` 。

##### Before Satisfy Interest Trigger

```cpp
/** \brief trigger before PIT entry is satisfied
*
*  This trigger is invoked when an incoming Data satisfies more than one PIT entry.
*  The strategy can collect measurements information, but cannot manipulate Data forwarding.
*  When an incoming Data satisfies only one PIT entry, \c afterReceiveData is invoked instead
*  and given full control over Data forwarding. If a strategy does not override \c afterReceiveData,
*  the default implementation invokes \c beforeSatisfyInterest.
*
*  Normally, PIT entries would be erased after receiving the first matching Data.
*  If the strategy wishes to collect responses from additional upstream nodes,
*  it should invoke \c setExpiryTimer within this function to prolong the PIT entry lifetime.
*  If a Data arrives from another upstream during the extended PIT entry lifetime, this trigger will be invoked again.
*  At that time, this function must invoke \c setExpiryTimer again to continue collecting more responses.
*
*  In this base class this method does nothing.
*
*  \warning The strategy must not retain shared_ptr<pit::Entry>, otherwise undefined behavior
*           may occur. However, the strategy is allowed to store weak_ptr<pit::Entry>.
*/
virtual void
beforeSatisfyInterest(const shared_ptr<pit::Entry>& pitEntry,
                    const FaceEndpoint& ingress, const Data& data);
```

此触发器声明为 `Strategy::beforeSatisfyInterest` 方法。基类提供了一个不执行任何操作的默认实现。如果需要通过此触发器调用策略（例如，记录未决兴趣的数据平面测量结果），则子类可以覆盖此方法。

当PIT条目被满足，在将 `Data` （如果有）发送到下游之前， *Incoming Data* 管道（第4.3.1节）将调用此触发器（以PIT条目， `Data` 及其传入 *Face* 作为参数）。 PIT条目可以表示未决 `Interest` 或最近满足的 `Interest` 。

##### Before Expire Interest Trigger

=> 此触发器在目前较新版本的NFD代码中已经移除了（v0.6.6 和 v0.7.0 两个版本的代码里面都没有）

##### After Receive Nack Trigger

```cpp
/** \brief trigger after Nack is received
*
*  This trigger is invoked when an incoming Nack is received in response to
*  an forwarded Interest.
*  The Nack has been confirmed to be a response to the last Interest forwarded
*  to that upstream, i.e. the PIT out-record exists and has a matching Nonce.
*  The NackHeader has been recorded in the PIT out-record.
*
*  If the PIT entry is not yet satisfied, its expiry timer remains unchanged.
*  Otherwise, the PIT entry normally would expire immediately after this function returns.
*
*  If the strategy wishes to collect responses from additional upstream nodes,
*  it should invoke \c setExpiryTimer within this function to retain the PIT entry.
*  If a Nack arrives from another upstream during the extended PIT entry lifetime, this trigger will be invoked again.
*  At that time, this function must invoke \c setExpiryTimer again to continue collecting more responses.
*
*  In the base class this method does nothing.
*
*  \warning The strategy must not retain shared_ptr<pit::Entry>, otherwise undefined behavior
*           may occur. However, the strategy is allowed to store weak_ptr<pit::Entry>.
*/
virtual void
afterReceiveNack(const FaceEndpoint& ingress, const lp::Nack& nack,
               const shared_ptr<pit::Entry>& pitEntry);
```

此触发器声明为 `Strategy::afterReceiveNack` 方法。基类提供了一个不执行任何操作的默认实现，这意味着所有传入的 *Nacks* 将被删除，并且不会传递给下游。如果需要为此触发器调用策略，则子类可以重写此方法。

当NFD收到一个 `Nack` ，会传递给 *Incoming Nack* 管道（第4.4.1节）处理，经过必要的检查后， *Incoming Nack* 管道将触发此触发器（以传入 *Face* 和PIT条目作为参数）。这样符合条件的 `Nack` 需要满足以下几条限定：

- `Nack` 是作为一个已转发的 `Interest` 的响应被接收的；
- `Nack` 已经被确认是对转发给上游的最后一个兴趣的响应，即在对应PIT条目中存在一个 *out-record* 且和收到的 `Nack` 具有匹配的 *Nonce* ；
- PIT条目位于此策略管理的名称空间下 $^9$；
-  *NackHeader* 已被记录在对应PIT条目 *out-record* 的 *Nacked* 字段中。

> $^9$ 注意：`Nack` 对应的 `Interest` 不一定是由同一个策略转发的。如果在转发 `Interest` 后更改了有效策略，然后收到了对应的 `Nack` ，则会触发新的有效策略，而不是先前转发 `Interest` 的策略。

当 *After Receive Nack Trigger* 被触发后，该策略通常可以执行以下操作之一：

- 通过调用 *send Interest* 操作将其转发到相同或不同的上游来重试兴趣（ *Retry the Interest* ）。大多数策略都需要一个FIB条目来找出潜在的上游，这可以通过调用 `Strategy::lookupFib` 访问器函数获得；
- 通过调用 *send Nack* 操作将 `Nack` 反回到下游，放弃对该 `Interest` 的重传尝试；
- 不对这个 `Nack` 做任何处理。如果 `Nack` 对应的 `Interest` 转发给了多个上游，且某些（但不是全部）上游回复了 `Nack` ，则该策略可能要等待来自更多上游的 `Data` 或 `Nack` 。在这种情况下，该策略无需将 `Nack` 记录在其自己的 `StrategyInfo` 中，因为 *NackHeader* 已经存储在PIT *out-record* 的 *Nacked* 字段中了。

#### 5.1.2 操作（Actions）

> 下面插入的代码片段来自：[`NFD/daemon/fw/strategy.hpp`](https://gitea.qjm253.cn/PKUSZ-future-network-lab/MIR/src/branch/master/daemon/fw/strategy.hpp) 和 [`NFD/daemon/fw/strategy.cpp`](https://gitea.qjm253.cn/PKUSZ-future-network-lab/MIR/src/branch/master/daemon/fw/strategy.cpp)

操作（ *Action* ）是转发策略（ *forwarding strategy* ）做出的决策。操作（ *Action* ）被实现为 `nfd::fw::Strategy` 类的非虚拟保护方法。

##### Send Interest action

```cpp
/** \brief send Interest to egress
*  \param pitEntry PIT entry
*  \param egress face through which to send out the Interest and destination endpoint
*  \param interest the Interest packet
*/
VIRTUAL_WITH_TESTS void
sendInterest(const shared_ptr<pit::Entry>& pitEntry,
           const FaceEndpoint& egress, const Interest& interest);
```

此操作以 `Strategy::sendInterest` 方法实现。参数包括一个PIT条目，一个出口 *Face* 和必须与PIT条目匹配的 `Interest` 。该操作将启动 *Outgoing Interest* 管道处理流程（第4.2.5节）。

调用本操作的策略负责检查的转发是否不违反基于命名空间的范围控制[10]。通常，该策略应使用在PIT *in-records* 中找到的传入 `Interest` 之一来调用此操作。 只要兴趣仍然与PIT条目相匹配，就可以制作一个兴趣的副本并修改其指导字段（通常指的是，修改 *LpHeaderField* ）。

```cpp
////////本段代码是NFD 0.7.0 新增的，展示了制作一个兴趣的副本，并移除一个 LpHeaderField////////////////
if (interest.getTag<lp::PitToken>() != nullptr) {		
    Interest interest2 = interest; // make a copy to preserve tag on original packet
    interest2.removeTag<lp::PitToken>();
    m_forwarder.onOutgoingInterest(pitEntry, egress, interest2);
    return;
}
//////////////////////////////////////////////////////////////////////////////////////////////
m_forwarder.onOutgoingInterest(pitEntry, egress, interest);
```

##### Send Data action

```cpp
/** \brief send \p data to \p egress
*  \param pitEntry PIT entry
*  \param data the Data packet
*  \param egress face through which to send out the Data and destination endpoint
*/
VIRTUAL_WITH_TESTS void
sendData(const shared_ptr<pit::Entry>& pitEntry, const Data& data, const FaceEndpoint& egress);
```

此操作以 `Strategy::sendData` 方法实现。参数包括PIT条目， `Data` 和下游 *Face* 。

此操作将删除PIT条目的记录中的内容，并进入 *Outgoing Data* 管道（第4.3.3节）。

在许多情况下，该策略可能希望将 `Data` 发送到每个下游。`Strategy::sendDataToAll` 方法是用于此目的的帮助程序，它接受PIT条目，`Data` 和 传入 `Data` 的 *Face* 。请注意，`Strategy::sendDataToAll` 会将数据发送到每个待处理的下游，除非待处理的下游 *Face* 与数据的传入 *Face* 相同，并且该 *Face* 不是临时的。

```cpp
BOOST_ASSERT(pitEntry->getInterest().matchesData(data));

shared_ptr<lp::PitToken> pitToken;
auto inRecord = pitEntry->getInRecord(egress.face);
if (inRecord != pitEntry->in_end()) {
    pitToken = inRecord->getInterest().getTag<lp::PitToken>();
}

// delete the PIT entry's in-record based on egress,
// since Data is sent to face and endpoint from which the Interest was received
pitEntry->deleteInRecord(egress.face);

if (pitToken != nullptr) {
    Data data2 = data; // make a copy so each downstream can get a different PIT token
    data2.setTag(pitToken);
    m_forwarder.onOutgoingData(data2, egress);
    return;
}
m_forwarder.onOutgoingData(data, egress);
```

##### Send Nack action

```cpp
/** \brief send Nack to egress
*  \param pitEntry PIT entry
*  \param egress face through which to send out the Nack and destination endpoint
*  \param header Nack header
*
*  The egress must have a PIT in-record, otherwise this method has no effect.
*/
VIRTUAL_WITH_TESTS void
sendNack(const shared_ptr<pit::Entry>& pitEntry,
       const FaceEndpoint& egress, const lp::NackHeader& header)
```

此操作以 `Strategy::sendNack` 方法实现。 参数包括PIT条目、下游 *Face* 和 *NackHeader* 。

此操将启动 *Outgoing Nack* 管道（第4.4.2节）处理流程。PIT条目中应存在一个下游 *Face* 的 *in-record* ，并且通过从该 *in-record* 中获取最后一个传入的 `Interest` 并添加指定的 *NackHeader* ，将构造一个 `Nack` 包。如果对应的PIT表项中缺少符合条件的 *in-record* ，则此操作无效。

```cpp
m_forwarder.onOutgoingNack(pitEntry, egress, header);
```

在许多情况下，该策略可能希望将 *Nacks* 发送到每个下游（同样的对每个要发送的下游 *Face* 都有匹配的 *in-record* ）。 `Strategy::sendNacks` 方法是用于此目的的辅助方法，它接受PIT条目和 *NackHeader* 。调用此帮助程序方法等效于为每个下游调用 *send Nack* 操作。

```cpp
/** \brief send Nack to every face-endpoint pair that has an in-record, except those in \p exceptFaceEndpoints
*  \param pitEntry PIT entry
*  \param header NACK header
*  \param exceptFaceEndpoints list of face-endpoint pairs that should be excluded from sending Nacks
*  \note This is not an action, but a helper that invokes the sendNack action.
*/
void
sendNacks(const shared_ptr<pit::Entry>& pitEntry, const lp::NackHeader& header,
        std::initializer_list<FaceEndpoint> exceptFaceEndpoints = {});
```

#### 5.1.3 Storage

策略被允许在PIT条目（ *PIT entries* ）、PIT入记录（ *PIT in-records* ）、PIT出记录（ *PIT out-records* ）和测量表条目（ *Measurements entries* ）中存储任意信息，所有这些可以存储信息的条目或者记录对象均继承自 `StrategyInfoHost` 类型 $^{10}$。在触发器内部，该策略已经可以访问PIT条目，并且可以查找所需的 *in-records* 和 *out-records* 。可以通过 `Strategy::getMeasurements` 方法访问测量条目（第3.7节）。策略的访问仅限于其控制下的命名空间下的 *Measurements* 条目（第3.7.2节）。

> $^{10}$ “Host” 是指持有策略信息，而不是端点/网络实体。

特定于策略的信息应包含在 `StrategyInfo` 的子类中。该策略可以随时在 `StrategyInfoHost` 上调用 `StrategyInfoHost::getStrategyInfo` ， `StrategyInfoHost::insertStrategyInfo` 和 `StrategyInfoHost:EraseStrategyInfo` 来存储和检索信息。请注意，该策略必须确保每个 `StrategyInfo` 具有不同的 *TypeId* 。 如果将相同的 *TypeId* 分配给多种类型，则NFD有可能崩溃。

由于可以在运行时更改名称空间的策略选择，因此NFD确保过渡名称空间下的所有策略存储项目（ *strategy-stored items* ）都将被销毁。因此，策略必须准备好应对某些实体可能没有 *strategy-stored items* 的情况；但是，如果存在某项，则可以保证其类型正确。存储项的析构函数还必须取消所有计时器，以使该策略不会在不再受其控制的实体上被激活。

策略只允许使用上述机制来存储信息。除此之外，策略对象（ `nfd::fw::Strategy` 的子类）本身应该是无状态的。