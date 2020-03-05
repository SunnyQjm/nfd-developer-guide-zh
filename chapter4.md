NFD具有智能转发平面（ *smart forwarding plane* ），该平面由 **转发管道** （ *forwarding pipelines* ，第4节）和 **转发策略** （ *forwarding strategies* ，第5节）组成。 **转发管道（或管道）由数据包上的一系列处理步骤组成**。 此外，当某个事件被触发（ *an event is triggered* ）并且匹配一定条件（ *a condition is matched* ）时将开启 *pipeline* 处理流程（ *a pipeline is entered* ）。例如，在接收到`Interest`时，在检测到接收到循环的`Interest`时，在准备将`Interest`从 *face* 转发出去时等。转发策略（或策略）在`packet`转发时进行决策，包括是否，何时以及在何处转发`packet`。NFD可以有多个服务于不同名称空间的策略，并且管道将相应地向策略提供`packet`。

图7显示了转发管道和策略的总体工作流程，其中蓝色框代表管道，白色框代表策略的决策点。

![图7  转发和策略的整体工作流程](assets/1583327861825.png)

<center>图7  转发和策略的整体工作流程</center>

### 4.1 转发管道（Forwarding Pipelines）

管道（ *pipelines* ）处理网络层数据包（`Interest`、`Data`或`Nack`），并且每个数据包都从一个管道传递到另一个管道（在某些情况下通过策略决策点），直到所有处理流程完成为止。管道内的处理使用CS、PIT、Dead Nonce List、FIB、网络区域表和策略选择表，但是管道对后三个表仅具有只读访问权限，因为这些表由相应的管理器管理，并且不直接受数据面流量的影响。

`FaceTable`跟踪NFD中所有活动的 *face* 。它是进入网络层数据包从中进入转发管道进行处理的入口点。管道还可以通过 *face* 发送数据包。

NDN中对`Interest`、`Data`和`Nack`数据包的处理是完全不同的。我们将转发管道分为 **兴趣处理路径** （ *Interest processing path* ）、 **数据处理路径** （ *Data processing path* ）和 **Nack处理路径** （ *Nack processing path* ），这将在以下各节中进行介绍。

### 4.2 兴趣处理路径（Interest Processing Path）

NFD将`Interest`处理分为以下管道：

- **Incoming Interest** ：处理收到的`Interest`
- **Interest loop** ：处理收到的循环`Interest`
- **ContentStore hit** ：处理可通过缓存的数据满足的传入`Interest`
- **ContentStore miss** ：处理缓存数据无法满足的传入`Interest`
- **Outgoing Interest** ：准备并发出`Interest`
- **Interest finalize** ：删除PIT条目

#### 4.2.1 Incoming Interest Pipeline

**incoming Interest pipeline** 在`Forwarder::onIncomingInterest`方法中实现，并从`Forwarder::startProcessInterest`方法输入，该方法由`Face::afterReceiveInterest`信号触发。**incoming Interest pipeline** 的输入参数包括新接收到的`Interest`和对接收到该`Interest`的 *face* 的引用。

该管道包括以下步骤，总结在图8中：

![图8  incoming Interest pipeline](assets/1583392405437.png)

<center>图8  incoming Interest pipeline</center>

1. 第一步是检查收到的`Interest`是否违反`/localhost` *scope* [10] 限制。特别是，来自非本地 *face* 的`Interest`名称不允许以`/localhost`前缀开头，因为它是为`localhost`通信保留的。如果检测到违规，则立即删除`Interest`，并且不执行进一步的处理。此检查可防止恶意的发包者（ *malicious senders* ）。一个合规的转发器（ *forwarder* ）永远不会将以`/localhost`开头的`Interest`发送给非本地用户。请注意，此处不检查`/localhop` *scope* ，因为它的范围规则不限制传入的兴趣。
2. 对照暂无现名清单（第3.5节）检查传入兴趣的名称和现名。 如果找到匹配项，则将传入的兴趣怀疑为循环，并将其提供给兴趣循环管道以进行进一步处理（第4.2.2节）。 如果未找到匹配项，则处理继续进行到下一步。 请注意，与通过PIT条目检测到的重复Nonce不同（如下所述），由Dead Nonce List检测到的重复不会导致创建PIT条目，因为为该传入兴趣创建记录中记录将导致匹配数据，如果 任何要返回到下游的错误信息； 另一方面，创建没有记录的PIT条目对将来重复Nonce检测没有帮助。