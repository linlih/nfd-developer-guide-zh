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

> 下面插入的代码片段取自 => [`NFD/daemon/fw/forwarder.cpp` => `Forwarder::onIncomingInterest`](https://gitea.qjm253.cn/PKUSZ-future-network-lab/MIR/src/branch/master/daemon/fw/forwarder.cpp)

```cpp
void
Forwarder::onIncomingInterest(const FaceEndpoint& ingress, const Interest& interest)
```

**incoming Interest pipeline** 在`Forwarder::onIncomingInterest`方法中实现，并从`Forwarder::startProcessInterest`方法输入，该方法由`Face::afterReceiveInterest`信号触发。**incoming Interest pipeline** 的输入参数包括新接收到的`Interest`和对接收到该`Interest`的 *face* 的引用。

该管道包括以下步骤，总结在图8中：

![图8  incoming Interest pipeline](assets/1583392405437.png)

<center>图8  incoming Interest pipeline</center>

1. 第一步是检查收到的`Interest`是否违反`/localhost` *scope* [10] 限制。特别是，来自非本地 *face* 的`Interest`名称不允许以`/localhost`前缀开头，因为它是为`localhost`通信保留的。如果检测到违规，则立即删除`Interest`，并且不执行进一步的处理。此检查可防止恶意的发包者（ *malicious senders* ）。一个合规的转发器（ *forwarder* ）永远不会将以`/localhost`开头的`Interest`发送给非本地用户。请注意，此处不检查`/localhop` *scope* ，因为它的范围规则不限制传入的兴趣。

   ```cpp
   // /localhost scope control
   bool isViolatingLocalhost = ingress.face.getScope() == ndn::nfd::FACE_SCOPE_NON_LOCAL &&
       scope_prefix::LOCALHOST.isPrefixOf(interest.getName());
   if (isViolatingLocalhost) {
       NFD_LOG_DEBUG("onIncomingInterest in=" << ingress
                     << " interest=" << interest.getName() << " violates /localhost");
       // (drop)
       return;
   }
   ```

   > [10] J. Shi, “Namespace-based scope control,” https://redmine.named-data.net/projects/nfd/wiki/ScopeControl.

2. 对照 *Dead Nonce List* 表（第3.5节）检查传入 `Interest` 的 *Name* 和 *Nonce* 。如果找到匹配项，则将传入的 `Interest` 怀疑为一个循环的 `Interest`，并将其传递给给 **兴趣循环管道** （*Interest loop*）以进行进一步处理（第4.2.2节）。如果未找到匹配项，则处理继续进行到下一步。请注意，与通过PIT条目检测到的重复 *Nonce* 不同（如下所述），由 *Dead Nonce List* 检测到的重复不会导致创建PIT条目，因为为此传入 `Interest` 创建 *in-record* 会导致匹配的数据（如果有）被返回到下游，这是不正确的；另一方面，创建没有记录的PIT条目对将来重复Nonce检测没有帮助。

   ```cpp
   // detect duplicate Nonce with Dead Nonce List
   bool hasDuplicateNonceInDnl = m_deadNonceList.has(interest.getName(), interest.getNonce());
   if (hasDuplicateNonceInDnl) {
       // goto Interest loop pipeline
       this->onInterestLoop(ingress, interest);
       return;
   }
   ```

3. 如果 `Interest` 带有转发提示（ *forwarding hint* ），则该过程（ *reaching producer region ?* ）通过检查转发提示对象（ *forwarding hint object* ）中的任何委托名称（ *delegation name* ）是否是网络区域表（*network region table*， 第3.2节）中任何区域名称的前缀来确定 `Interest` 是否已到达生产者区域。 如果是这样，转发提示（ *forwarding hint* ）将被删除，因为它已经完成了将 `Interest` 引入生产者区域的任务，并且不再需要。

   ```cpp
   // strip forwarding hint if Interest has reached producer region
   if (!interest.getForwardingHint().empty() &&
       m_networkRegionTable.isInProducerRegion(interest.getForwardingHint())) {
       NFD_LOG_DEBUG("onIncomingInterest in=" << ingress
                     << " interest=" << interest.getName() << " reaching-producer-region");
       const_cast<Interest&>(interest).setForwardingHint({});
   }
   ```

4. 下一步是使用 `Interst` 包中指定的名称和选择器查找现有的或创建新的PIT条目。至此，PIT条目成为传入 `Interest` 和后续管道的处理对象。请注意，NFD在执行 `ContentStore` 查找之前创建了PIT条目。做出此决定的主要原因是，由于 `ContentStore` 可能明显大于PIT，所以查询 `ContentStore` 的代价是高于查询 PIT 的，在下面即将讨论的一些情况下，是可以跳过CS查找，所以在查询 `ContentStore` 之前查询PIT或创建相应的表项是有助于减小查询开销的。

   ```cpp
   // PIT insert
   shared_ptr<pit::Entry> pitEntry = m_pit.insert(interest).first;
   ```

5. 在进一步处理传入 `Interest` 之前，查询 PIT 中对应表项的 *in-records* （ *这边查询的是同一个 PIT entry 的 in-record 列表* ）。如果在不同 *Face* 的记录中找到匹配项（ *即找到一个 in-record 记录，它的 Nonce 和传入 `Interest` 的 Nonce 是一样的，但是是从不同的 Face传入的* ），则可能是由于 `Interest` 循环或者是同一个 `Interest` 沿多个不同的路径到达，传入 `Interest` 被视为重复（ *loop* ）。并传递给兴趣循环管道（ *Interest loop* ）进行进一步处理（第4.2.2节）。 否则，处理继续。

   > 请注意，如果在同一个 p2p *Face* 收到相同 *Nonce* 的同名 `Interest`，则该 `Interest` 被视为合法重发，因为在这种情况下不存在持续循环的风险。

   ```cpp
   // detect duplicate Nonce in PIT entry
   int dnw = fw::findDuplicateNonce(*pitEntry, interest.getNonce(), ingress.face);
   bool hasDuplicateNonceInPit = dnw != fw::DUPLICATE_NONCE_NONE;
   if (ingress.face.getLinkType() == ndn::nfd::LINK_TYPE_POINT_TO_POINT) {
       // for p2p face: duplicate Nonce from same incoming face is not loop
       hasDuplicateNonceInPit = hasDuplicateNonceInPit && !(dnw & fw::DUPLICATE_NONCE_IN_SAME);
   }
   if (hasDuplicateNonceInPit) {
       // goto Interest loop pipeline
       this->onInterestLoop(ingress, interest);
       return;
   }
   
   // 下面是 fw::findDuplicateNonce 的实现
   int
   findDuplicateNonce(const pit::Entry& pitEntry, uint32_t nonce, const Face& face)
   {
     int dnw = DUPLICATE_NONCE_NONE;
   
     for (const pit::InRecord& inRecord : pitEntry.getInRecords()) {
       if (inRecord.getLastNonce() == nonce) {
         if (&inRecord.getFace() == &face) {
           dnw |= DUPLICATE_NONCE_IN_SAME;
         }
         else {
           dnw |= DUPLICATE_NONCE_IN_OTHER;
         }
       }
     }
   
     for (const pit::OutRecord& outRecord : pitEntry.getOutRecords()) {
       if (outRecord.getLastNonce() == nonce) {
         if (&outRecord.getFace() == &face) {
           dnw |= DUPLICATE_NONCE_OUT_SAME;
         }
         else {
           dnw |= DUPLICATE_NONCE_OUT_OTHER;
         }
       }
     }
   
     return dnw;
   }
   ```

6. 接下来，由于新的有效兴趣已到达，因此取消了PIT条目上的到期计时器（ *expiry timer* ），延长PIT条目的寿命。稍后可以在兴趣处理路径中重置计时器，例如，如果在 `ContentStore` 中找到匹配的数据。

7. 然后，管道将测试 `Interest` 是否未决（ *is pending ?* ），即PIT条目是否已经具有相同或另一个传入Face的另一个记录。回想一下，NFD的PIT条目不仅可以代表未决 `Interest` ，而且还可以代表最近满足的 `Interest` （第3.4.1节）。此测试等效于CCN节点模型[9]中的“具有PIT条目”，其PIT仅包含未决兴趣。=> **可以简单的认为，检查传入的 `Interest` 的PIT表项中是否包含其它记录，如果包含，这个兴趣包就是未决的**

8.  如果 `Interest` 是非未决的（ *not pending* ），则去 `ContentStore` 查询是否有对应的匹配项（`Cs::find`，第3.3.1节）。否则，不需要进行CS查找，直接传递给 *ContentStore miss* 管道处理，因为未决的 `Interest` 意味着先前查询过 `ContentStore` ，且在 `ContentStore` 中未能命中 。如果 `Interest` 是未决的（ *pending* ），根据CS是否匹配，选择将 `Interest` 传递给 *ContentStore miss* 管道（第4.2.4节）还是 *ContentStore hit* 管道（第4.2.3节）处理。

   ```cpp
   // is pending?
   if (!pitEntry->hasInRecords()) { // 非未决
       m_cs.find(interest,
                 bind(&Forwarder::onContentStoreHit, this, ingress, pitEntry, _1, _2),
                 bind(&Forwarder::onContentStoreMiss, this, ingress, pitEntry, _1));
   }
   else { // 未决
       this->onContentStoreMiss(ingress, pitEntry, interest);
   }
   ```

#### 4.2.2 Interest Loop Pipeline

> 下面插入的代码片段取自 => [`NFD/daemon/fw/forwarder.cpp` => `Forwarder::onInterestLoop`](https://gitea.qjm253.cn/PKUSZ-future-network-lab/MIR/src/branch/master/daemon/fw/forwarder.cpp)

```cpp
void
Forwarder::onInterestLoop(const FaceEndpoint& ingress, const Interest& interest)
```

*Interest Loop* 管道在 `Forwarder::onInterestLoop` 方法中实现，当在 *incoming Interest* 管道（第4.2.1节）检测到兴趣循环时会触发 *Interest Loop* 管道的处理逻辑。该管道的输入参数包括 *Interest* 及其传入 *Face*。

如果传入 `Interest` 的 *Face* 是点对点的（ *point-to-point* ），则会向传入 *Face* 发送一个原因代码为“重复”（ *Duplicate* ）的 `Nack` 。 由于在多路访问链路（ *multi-access link* ）上未定义 `Nack` 的语义，因此，如果传入 *Face* 是多路访问的（ *multi-access* ），则会简单地丢弃循环的兴趣。

```cpp
// if multi-access or ad hoc face, drop
if (ingress.face.getLinkType() != ndn::nfd::LINK_TYPE_POINT_TO_POINT) {
    NFD_LOG_DEBUG("onInterestLoop in=" << ingress
                  << " interest=" << interest.getName() << " drop");
    return;
}

NFD_LOG_DEBUG("onInterestLoop in=" << ingress << " interest=" << interest.getName()
              << " send-Nack-duplicate");

// send Nack with reason=DUPLICATE
// note: Don't enter outgoing Nack pipeline because it needs an in-record.
lp::Nack nack(interest);
nack.setReason(lp::NackReason::DUPLICATE);
ingress.face.sendNack(nack, ingress.endpoint);
```

#### 4.2.3 ContentStore Hit Pipeline

> 下面插入的代码片段取自 => [`NFD/daemon/fw/forwarder.cpp` => `Forwarder::onContentStoreHit`](https://gitea.qjm253.cn/PKUSZ-future-network-lab/MIR/src/branch/master/daemon/fw/forwarder.cpp)

```cpp
void
Forwarder::onContentStoreHit(const FaceEndpoint& ingress, const shared_ptr<pit::Entry>& pitEntry,
                             const Interest& interest, const Data& data)
```

*ContentStore Hit* 管道在 `Forwarder::onContentStoreHit` 方法中实现，当在 *incoming Interest* 管道（第4.2.1节）中执行 `ContentStore` 查找（第3.3.1节）并找到匹配项之后触发 *ContentStore Hit* 管道处理逻辑。该管道的输入参数包括 `Interest`，其传入 *Face* ，PIT条目和匹配的数据包。/ localhost

如图9所示，此管道首先将 `Interest` 的到期计时器设置为当前时间，然后调用 `Interest` 所选策略的 `Strategy::afterContentStoreHit` 回调。

![image-20200715144230603](chapter4.assets/image-20200715144230603.png)

```cpp
NFD_LOG_DEBUG("onContentStoreHit interest=" << interest.getName());
++m_counters.nCsHits;

data.setTag(make_shared<lp::IncomingFaceIdTag>(face::FACEID_CONTENT_STORE));
// FIXME Should we lookup PIT for other Interests that also match the data?

pitEntry->isSatisfied = true;
pitEntry->dataFreshnessPeriod = data.getFreshnessPeriod();

// set PIT expiry timer to now
this->setExpiryTimer(pitEntry, 0_ms);

// dispatch to strategy: after Content Store hit
this->dispatchToStrategy(*pitEntry,
                         [&] (fw::Strategy& strategy) { strategy.afterContentStoreHit(pitEntry, ingress, data); });
```

#### 4.2.4 ContentStore Miss Pipeline

> 下面插入的代码片段取自 => [`NFD/daemon/fw/forwarder.cpp` => `Forwarder::onContentStoreMiss`](https://gitea.qjm253.cn/PKUSZ-future-network-lab/MIR/src/branch/master/daemon/fw/forwarder.cpp)

```cpp
void
Forwarder::onContentStoreMiss(const FaceEndpoint& ingress,
                              const shared_ptr<pit::Entry>& pitEntry, const Interest& interest)
```

*ContentStore Miss* 管道在 `Forwarder::onContentStoreMiss` 方法中实现，当在 *incoming Interest* 管道（第4.2.1节）中执行 `ContentStore` 查找（第3.3.1节）并没有找到匹配项之后触发 *ContentStore Hit* 管道处理逻辑。 该管道的输入参数包括 `Interest` ，其传入 *Face* 和PIT条目。

![image-20200715145909460](chapter4.assets/image-20200715145909460.png)

如图10所示，该管道执行以下步骤：

1. 根据传入的 `Interest` 以及对应的传入 *Face* 决定在相应 PIT 条目中插入或者更新 *in-record* 。如果相应 PIT 条目中已经存在相同传入 *Face* 的 *in-record* 记录，（例如，兴趣正在由同一下游重新传输），只需使用新观察到的 `Interest` 的 *Nonce* 和到期时间（*expiration time*）更新原来的 *in-record* 记录即可。记录中的到期时间由兴趣数据包中的 *InterestLifetime* 字段控制； 如果省略 *InterestLifetime* ，则使用默认的4秒。

   ```cpp
   NFD_LOG_DEBUG("onContentStoreMiss interest=" << interest.getName());
   ++m_counters.nCsMisses;
   
   // insert in-record
   pitEntry->insertOrUpdateInRecord(ingress.face, interest);
   ```

2. PIT条目上的到期计时器设置为最后一个PIT *in-record* 到期的时间。当到期计时器到期时，将执行 *Interest Finalize* 管道（第4.2.6节）。

   ```cpp
   // set PIT expiry timer to the time that the last PIT in-record expires
   auto lastExpiring = std::max_element(pitEntry->in_begin(), pitEntry->in_end(),
                                        [] (const auto& a, const auto& b) {
                                            return a.getExpiry() < b.getExpiry();
                                        });
   auto lastExpiryFromNow = lastExpiring->getExpiry() - time::steady_clock::now();
   this->setExpiryTimer(pitEntry, time::duration_cast<time::milliseconds>(lastExpiryFromNow));
   ```

3. 如果 `Interest` 在其NDNLPv2 header 中携带 *NextHopFaceId* 字段，则管道将遵循此字段。在FaceTable中查找所选的下一跳  *Face*。如果找到 *Face*，则执行 *outgoing Interest* 管道（第4.2.5节）；如果该 *Face* 不存在，则删除 `Interest`。

   ```cpp
   // has NextHopFaceId?
   auto nextHopTag = interest.getTag<lp::NextHopFaceIdTag>();
   if (nextHopTag != nullptr) {
       // chosen NextHop face exists?
       Face* nextHopFace = m_faceTable.get(*nextHopTag);
       if (nextHopFace != nullptr) {
           NFD_LOG_DEBUG("onContentStoreMiss interest=" << interest.getName()
                         << " nexthop-faceid=" << nextHopFace->getId());
           // go to outgoing Interest pipeline
           // scope control is unnecessary, because privileged app explicitly wants to forward
           this->onOutgoingInterest(pitEntry, FaceEndpoint(*nextHopFace, 0), interest);
       }
       return;
   }
   ```

4. 如果没有 *NextHopFaceId* 字段，则转发策略负责对 `Interest` 做出转发决策。因此，管道将调用“查找有效策略”算法（ *Find Effective Strategy algorithm* ，第3.6.1节）来确定要使用的策略，并调用所选策略对象的 `Strategy::afterReceiveInterest` 回调。传入 *Interest* ，其传入 *Face* 和PIT条目（第5.1.1节）。

   ```cpp
   // dispatch to strategy: after incoming Interest
   this->dispatchToStrategy(*pitEntry,
                            [&] (fw::Strategy& strategy) {
                                strategy.afterReceiveInterest(FaceEndpoint(ingress.face, 0), interest, pitEntry);
                            });
   ```

#### 4.2.5 Outgoing Interest Pipeline

> 下面插入的代码片段取自 => [`NFD/daemon/fw/forwarder.cpp` => `Forwarder::onOutgoingInterest`](https://gitea.qjm253.cn/PKUSZ-future-network-lab/MIR/src/branch/master/daemon/fw/forwarder.cpp)

```cpp
void
Forwarder::onOutgoingInterest(const shared_ptr<pit::Entry>& pitEntry,
                              const FaceEndpoint& egress, const Interest& interest)
```

*Outgoing Interest* 管道在 `Forwarder::onOutgoingInterest` 方法中实现，并一般在 `Strategy::sendInterest` 方法内调用（ *也可能如上述 4.2.4 节所述，兴趣包携带了 NextHopFaceId 字段，此时也可能不经过策略决策，直接触发 Outgoing Interest 管道* ），该方法处理策略的发送兴趣动作（第5.1.2节）。该管道的输入参数包括PIT条目，传出 *Face* 和 `Interest` 。请注意，`Interest` 不是进入管道时的参数。管道步骤要么直接使用PIT条目执行检查，要么获取对存储在PIT条目内的 `Interest` 的引用。

该管道首先在PIT条目中为指定的传出 *Face* 插入一个 *out-record*，或者为同一 *Face* 更新一个现有的 *out-record*。 在这两种情况下，PIT记录都将记住最后一个传出兴趣数据包的 *Nonce* ，这对于匹配传入的Nacks很有用，还有到期时间戳，它是当前时间加上 *InterestLifetime* 。最后， `Interest` 被发送到传出的 `Face` 。

> PIT 表的结构参见 => [3.4 PIT](https://sunnyqjm.github.io/nfd-developer-guide-zh/#/chapter3?id=_34-兴趣表（pit）)

```cpp
NFD_LOG_DEBUG("onOutgoingInterest out=" << egress << " interest=" << pitEntry->getName());

// insert out-record
pitEntry->insertOrUpdateOutRecord(egress.face, interest);

// send Interest
egress.face.sendInterest(interest, egress.endpoint);
++m_counters.nOutInterests;
```

#### 4.2.6 Interest Finalize Pipeline

> 下面插入的代码片段取自 => [`NFD/daemon/fw/forwarder.cpp` => `Forwarder::onInterestFinalize`](https://gitea.qjm253.cn/PKUSZ-future-network-lab/MIR/src/branch/master/daemon/fw/forwarder.cpp)

```cpp
void
Forwarder::onInterestFinalize(const shared_ptr<pit::Entry>& pitEntry)
```

*Interest Finalize* 管道在 `Forwarder::onInterestFinalize` 方法中实现，一般是由到期计时器（ *expiry timer* ）到期时触发。

管道首先确定是否需要将记录在PIT条目中的任何 *Nonces* 插入到 *Dead Nonce List* 中（第3.5节）。*Dead Nonce List* 是一个旨在检测循环兴趣的全局数据结构，我们希望插入尽可能少的 *Nonces* 以减小其大小。只需要插入传出的 *Nonces* （在外记录中），因为未发送出去的 *incoming Nonce* 是不可能环回的。

```cpp
NFD_LOG_DEBUG("onInterestFinalize interest=" << pitEntry->getName()
              << (pitEntry->isSatisfied ? " satisfied" : " unsatisfied"));

// Dead Nonce List insert if necessary
this->insertDeadNonceList(*pitEntry, nullptr);
```

我们可以利用 `ContentStore` 获得更多不插入 *Nonce* 到 *Dead Nonce List* 的机会：如果PIT条目被满足，并且 `ContentStore` 有能力在 *Dead Nonce List* 条目的生存期内满足后续的 *looping Interest* （ *可以认定循环兴趣会因为命中缓存而停止循环* ），且在此期间 *Data* 没有被从 `ContentStore` 中逐出，则此PIT条目中的随机数不需要插入 *Dead Nonce List* 。如果 `Interest` 没有设置 *MustBeFresh* 选择器，或者缓存的数据的 *FreshnessPeriod* 不小于 *Dead Nonce List* 条目的生存期，则认为 `ContentStore` 可以满足对应的循环 `Interest`。

如果确定应将一个或多个随机数插入到 *Dead Nonce List* 中，则将 `<Name, Nonce>` 元组添加到 *Dead Nonce List* 中（第3.5.1节）。

```cpp
void
Forwarder::insertDeadNonceList(pit::Entry& pitEntry, Face* upstream)
{
  // need Dead Nonce List insert?
  bool needDnl = true;
  if (pitEntry.isSatisfied) {
    BOOST_ASSERT(pitEntry.dataFreshnessPeriod >= 0_ms);
    needDnl = static_cast<bool>(pitEntry.getInterest().getMustBeFresh()) &&
              pitEntry.dataFreshnessPeriod < m_deadNonceList.getLifetime();
  }

  if (!needDnl) {
    return;
  }

  // Dead Nonce List insert
  if (upstream == nullptr) {
    // insert all outgoing Nonces
    const auto& outRecords = pitEntry.getOutRecords();
    std::for_each(outRecords.begin(), outRecords.end(), [&] (const auto& outRecord) {
      m_deadNonceList.add(pitEntry.getName(), outRecord.getLastNonce());
    });
  }
  else {
    // insert outgoing Nonce of a specific face
    auto outRecord = pitEntry.getOutRecord(*upstream);
    if (outRecord != pitEntry.getOutRecords().end()) {
      m_deadNonceList.add(pitEntry.getName(), outRecord->getLastNonce());
    }
  }
}
```

最后，将PIT条目从PIT中删除。

```cpp
// PIT delete
pitEntry->expiryTimer.cancel();
m_pit.erase(pitEntry.get());
```

### 4.3 数据包处理路径（Data Processing Path）

NFD中的 `Data` 处理分为以下管道：

- **Incoming Data**：处理传入 `Data`
- **Data unsolicited**：处理传入的未经请求的 `Data`
- **Outgoing Data**：准备并发送出 `Data`

#### 4.3.1 Incoming Data Pipeline

> 下面插入的代码片段取自 => [`NFD/daemon/fw/forwarder.cpp` => `Forwarder::onIncomingData`](https://gitea.qjm253.cn/PKUSZ-future-network-lab/MIR/src/branch/master/daemon/fw/forwarder.cpp)

```cpp
void
Forwarder::onIncomingData(const FaceEndpoint& ingress, const Data& data)
```

*Incoming Data* 管道在 `Forwarder::onIncomingData` 方法中实现，并由 `Forward::startProcessData` 方法内部调用，该方法由 `Face::afterReceiveData` 信号触发。该管道的输入参数包括 `Data` 及其传入的 *Face*。

![image-20200715161727678](chapter4.assets/image-20200715161727678.png)

如图11所示，该管道包括以下步骤：

1. 第一步是检查 `Data` 是否违反了 `/localhost` *scope* [10]。 如果 `Data` 来自非本地 *Face*，但其名称以 `/localhost` 前缀开头，则该 `Data` 将违反 *scope* 并被删除。此检查可防止恶意发送者。兼容的 *forwarder* 永远不会将 `/localhost` 数据发送到非本地 *Face* 。请注意，此处未选中 `/localhop` 范围，因为其范围规则并不限制传入的 `Data`。

   > [10] J. Shi, “Namespace-based scope control,” https://redmine.named-data.net/projects/nfd/wiki/ScopeControl.

   ```cpp
   // receive Data
   NFD_LOG_DEBUG("onIncomingData in=" << ingress << " data=" << data.getName());
   data.setTag(make_shared<lp::IncomingFaceIdTag>(ingress.face.getId()));
   ++m_counters.nInData;
   
   // /localhost scope control
   bool isViolatingLocalhost = ingress.face.getScope() == ndn::nfd::FACE_SCOPE_NON_LOCAL &&
       scope_prefix::LOCALHOST.isPrefixOf(data.getName());
   if (isViolatingLocalhost) {
       NFD_LOG_DEBUG("onIncomingData in=" << ingress << " data=" << data.getName() << " violates /localhost");
       // (drop)
       return;
   }
   ```

2. 
