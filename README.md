
EOSIO体系公链间跨链交互原理及实现
-------
(BOSCORE TEAM)

本文将介绍IBC技术原理，并对BOSCORE团队开发的IBC合约及插件进行深入介绍。

为了实现两条EOSIO体系区块链间token的跨链转移，首先需要解决两个难题，1.轻客户端如何实现，2.如何确保跨链交易的完整性和可靠性，
如何防止双花和重放攻击。下文均以EOS主网和BOS主网为例进行说明，但本文档适用于任何两条EOSIO体系的区块链。


### 一、术语
- BOSIBC  
  是指BOSCORE技术团队开发的IBC合约及插件。

### 二、关键概念及数据结构
- Simple Payment Verification (SPV)  
  简单支付验证技术最早在中本聪的比特币白皮书中提出[Bitcoin](https://bitcoin.org/bitcoin.pdf)，用于验证一笔交易存在于区块链中。
  `SPV client`存储着连续的区块头，但没有区块体，因此只需占用很小的存储空间。当获得一笔交易和这笔交易的`Merkle path`后，可以验证这笔交易是否
  存在于区块链上。
- Lightwight client (lwc)  
  轻客户端，即`SPV client`，即由区块头组成的一条轻量的链。
- Merkle path
  默克尔路径。为验证一笔交易是否存在于某个区块中，只需要提供交易原始数据和交易在所在区块的`merkle path`，而无需提供整个区块体，通过计算`merkle path`并和区块头中
  记录的`merkle root`对比，若相等，则说明此交易存在于此区块中。`merkle path`也被称为`merkle branch`  
- Block Producer Schedule  
  `BP Schedule`是基于DPOS机制的EOSIO体系公链用于决定生产区块权利的技术，新的`BP Schedule`是由上一批`BP Schedule`包含的`Block Producers`认证通过后生效
  以此确保严格的BP权利交接，在轻客户端中跟随对应主网的`BP Schedule`是IBC系统逻辑的一项核心技术。
- forkdb  
  EOSIO节点在运行时，有两个底层的db用于存储区块信息，一个是`blog`即`block log`，用于存储不可逆区块，一个是`forkdb`，用于存储可逆区块。
  forkdb存储的是当前区块链最顶端的一部分区块信息，一个区块首先要被forkdb接受才能最终进入不可逆区块，IBC系统在合约实现的轻客户端主要参考了forkdb的逻辑。

### 三、轻客户端
为了解决跨链问题，首先要解决的是轻客户端如何实现的问题。
1. 轻客户端运行在哪里合理，合约中还是合约外，例如插件中；
2. 若运行在合约中，是实时同步对方链的全部区块头数据，还是根据需要同步一部分区块数据来验证交易，因为如果同步全部区块数据会消耗两条链大量cpu资源。
3. 若运行在合约中，如何确保轻客户端的可信性，如何防止恶意攻击，如何做到完全去中心化，不依赖对任何中继节点的信任；

#### 3.1 轻客户端是否运行在合约中  
比特币的轻客户端最早是运行在单个节点上（如个人电脑或去中心化比特币手机钱包），用于验证交易是否存，并查看交易所在区块深度。具体实现可以参考
[breadwallet](https://github.com/breadwallet/breadwallet-core)。

IBC和去中心化钱包对轻客户端的需求是不同的，去中心化钱包一般运行在个人的手机App上，为用户个人提供交易验证服务，而IBC系统需要的轻客户端
需要对所有人公开可查可信，从这个角度看，一个能够获得大众信任的轻客户端只能运行在合约中，因为只有合约的数据是全局一致不可篡改的，
运行在合约外则无法实现一个可信的轻客户端，因此BOSIBC将轻客户端运行在合约中，
源码请查看[ibc.chain](https://github.com/boscore/ibc_contracts/tree/master/ibc.chain)。

#### 3.2 是否同步全量区块头信息
在比特币和以太坊的轻客户端中，会寻找一个起点和后续的一些验证点，轻客户端会同步起点后的全量的区块头信息。比特币每年产生的所有区块的区块头体积仅有4Mb，
按现在移动设备的存储能力，是完全可以容纳的，并且同步这些区块头也不会消耗移动设备大量计算资源。然而EOSIO的情况却很不同，
EOSIO每0.5秒一个区块，实际测试可知，每添加一个区块头到合约中需要消耗0.5毫秒cpu，每删除一个区块头需要0.2毫秒，因此每处理一个区块头需要0.7ms的cpu。
假设要同步对方EOSIO公链的全量区块头信息，按现在每个区块总的cpu时间200ms计算，也就是需要一条链全部计算的`0.7ms / 200ms = 0.35%`才能实时全量同步
另一条链的所有区块头， 按实际全网抵押总量为4亿token计算，如果再cpu繁忙时保证IBC系统正常工作，需要为push区块信息的账户抵押
`4亿 * 0.35% = 140万`token，这是个很大的数目。又因为EOSIO倡导多侧链的生态，假设未来有多条侧链和EOSIO主网实现跨链，并且侧链与侧链间
也实现了一对多的跨链，按1对10计算，每条链需要维护10个轻客户端，则只为了维护这些轻客户端就需要消耗3.5%的单条链全网cpu，这个比例实在是太高了，
因此需要寻找更合理的方案。

设计跨链通信的过程是一种寻找可信证据的过程，有没有一种方案即不需要同步全量区块信息，又可以保证轻客户端的可信性，EOSIO底层已经为实现这一目的有所准备。
我们先假设，如果BP schedule自始至终不会变化，那么任何时候，当ibc.chain合约中获得一连串的签名验证通过的区块头，比如第`n ~ n+336`个，
并且有2/3以上的活跃bp在出块，就可以确信第n个区块已经是不可逆的，可以用于验证跨链交易。
后面就是需要考虑有BP schedule更换的情况了，当出现BP schedule更换时，不在接受交易验证，直到更换完成，处理BP更换时相对复杂的过程，后续会更详细介绍，
因此使用这个方案就可以大大降低需要同步的区块头数量，只有在BP列表更换或有跨链交易时才需要同步区块。

为了实现这一目的，在ibc.chain中引入了概念`section`，一个section记录的是一段连续的区块头信息，section结构不存储具体的区块头信息，而是记录这一段
区块头的第一个区块编号(first)和最后一个区块编号(last)，具体区块头信息在`chaindb`中存储，每个section都有一个valid值，在没有bp schedule更替的时候，
只要有2/3的活跃BP在出块，并且`last - first > lib_depth`则认为`first ~ last - lib_depth`的区块是不可逆的，可以用于验证跨链交易，
当遇到BP schedule 更替，section的valid变为false，不再接受交易验证，直到schedule更替完成，valid重新变为true之后，继续验证跨链交易。


#### 3.3 如何确保轻客户端的可信性

**3.3.1 forkdb** 

*1.一个新的区块是如何追加到forkdb的*  

一个运行的nodeos节点维护着两个底层数据结构[blog](https://github.com/EOSIO/eos/blob/master/libraries/chain/include/eosio/chain/block_log.hpp)
和[forkdb](https://github.com/EOSIO/eos/blob/master/libraries/chain/include/eosio/chain/fork_database.hpp)，
blog用于存储不可逆的区块信息，其存储的数据是序列化的`signed_block`，forkdb用于存储可逆区块信息，其存储的数据是`block_state`，
`block_state`比`signed_block`包含更多区块相关信息。一个区块首先要被追加到forkdb，才可能最终变为不可逆区块而移除forkdb进入blog，
一个区块的`block_state`信息是如何获得的呢，并非生产区块的BP将所生产区块的`block_state`通过p2p网络传递给其他bp和全节点，p2p网络
只传递[signed_block](https://github.com/EOSIO/eos/blob/master/plugins/net_plugin/include/eosio/net_plugin/protocol.hpp#L142),
当一个节点通过p2p网络接收到一个`signed_block`后，它会使用此`signed_block`构建`block_state`并验证签名
[相关函数](https://github.com/EOSIO/eos/blob/master/libraries/chain/block_header_state.cpp#L144)，
其中需要说明的几个关键点是，1.`blockroot_merkle`，2.`get_scheduled_producer()`,3.`verify_signee()`。

blockroot_merkle  
EOSIO在`block_state::block_header_state`结构中维护了一个`blockroot_merkle`的`incremental_merkle`数据，`incremental_merkle`实际是
一个完整的`merkle`树的活跃节点，使用`incremental_merkle`只需维护极少的活跃节点信息即可不断累加并获得`merkle_root`，是`block_state`
中使用的一个关键技术。BOSIBC的ibc.chain合约同样使用了此数据结构。

`blockroot_merkle`从创世区块id不断累加，但是`signed_block`和`blog`中并没有这个数据，只有forkdb的每个`block_state`中记录着当前block的
`blockroot_merkle`，并且此值被用于计算区块签名。

get_scheduled_producer()  
此函数根据一个区块的`header.timestamp`计算出应该生成此区块的`producer_key`（见`block_header_state::next()`），为后续验证签名做准备。

验证签名相关函数如下
```
  digest_type   block_header_state::sig_digest()const {
     auto header_bmroot = digest_type::hash( std::make_pair( header.digest(), blockroot_merkle.get_root() ) );
     return digest_type::hash( std::make_pair(header_bmroot, pending_schedule_hash) );
  }

  public_key_type block_header_state::signee()const {
    return fc::crypto::public_key( header.producer_signature, sig_digest(), true );
  }

  void block_header_state::verify_signee( const public_key_type& signee )const {
     EOS_ASSERT( block_signing_key == signee, wrong_signing_key, "block not signed by expected key",
                 ("block_signing_key", block_signing_key)( "signee", signee ) );
  }
```
验证签名的第一步是获得区块摘要，即`sig_digest()`，此函数中用到了`header.digest()`,`blockroot_merkle.get_root()`和`pending_schedule_hash`；
第二步是获得签名公钥，即`signee()`，通过区块的`producer_signature`和`sig_digest()`计算BP公钥；
第三步是验证公钥是否正确，即`verify_signee()`，此函数在`block_header_state::next()`被调用；验证通过后一个区块被追加的forkdb中的分支中。

所以在forkdb中每添加一个区块都经过了非常严格全面的效验，核心是包括`blockroot_merkle`，`get_scheduled_producer()`和`verify_signee()`,
在ibc.chain合约完全继承了forkdb严格的效验。


*2.forkdb如何处理分叉*  
当添加一个新的区块导致fordb的head.id和controller_impl的head.id不同时，则重新选择分支。
源码参考`eosio::chain::controller_impl`的`push_block()`和`maybe_switch_forks()`;


*3.LIB如何确定*  
EOSIO目前使用的共识方式是dpos，当构造一个区块的`block_header_state`时会设定`required_confs`，此值为当前活跃BP数量的2/3+1，
在21个BP的情况下，`required_confs`为15。每个区块头中都有`header.confirmed`，用于对前面的区块进行确认，每个区块得到一个确认，
其`required_confs`会减1，当某个区块的`required_confs`减少到零时，此区块会被最新区块（即forkdb的head）提名为`dpos_proposed_irreversible_blocknum`，
当某个区块获得了2/3的BP提名后，其变为不可逆区块，即进入LIB。由于确认的信息是在header中传递的，因此一个区块从产生到进入LIB总共需要
两个2/3论，也就是 `12 *（ 14 * 2 ） = 336`才会进入LIB，考录到BP每次都是连续出12个区块，只有第一个区块的`header.confirmed`为非零，
因此当一个BP开始出块时，只有第一个区块会提升LIB，因此实际head和LIB的差距在325~336之间，但是在有BP丢块的情况下，head和LIB的差距可能出现小于
325~336。

*4.BP列表是如何更换的*  
在pow的区块链中，比如比特币和以太坊，是选择算力累加最大的分叉作为主链，一个区块只有包含一定的算力才有可能被认可，最终变为不可逆。
而在以dpos为共识算法的EOS中，区块被认可的标记是BP签名，因此BP列表在EOSIO中具有至关重要的地位。IBC的轻客户端同样需要维护BP列表和BP列表的更换，
因此需要透彻分析BP列表的更换逻辑。

第一步，在系统合约`eosio.system`的`onblock()`函数中，系统会每分钟一次尝试更新bp列表`update_elected_producers( timestamp )`，此函数最终调用
wasm接口`set_proposed_producers()`，通过一系列检查后，会将新的schedule和当前区块编号存到`global_property_object`对象中。

第二部，当此区块变为不可逆之后，会在当前的pending区块中设置新的名单`header.new_producers`，并删除`global_property_object`对象，当前
pending区块编号会被记录到`pending_schedule_lib_num`，此时在nodeos日志中可以看到新的名单；具体逻辑参考`controller::start_block() // Promote proposed schedule to pending schedule.`。
也就是说新的名单从`proposed schedule`变为`pending schedule`大约需要经历325~336个区块。从这里开始，
后面区块`block_header_state`的`pending_schedule.version`会比`active_schedule.version`大1.

第三步，当`pending_schedule_lib_num`变为不可逆后，`active_schedule`会被`pending schedule`替换，整个的BP更换过程完成。
从`pending schedule`出现到其变为`active_schedule`同样需要经历约325~336区块。


IBC系统的轻客户端同样需要继承forkdb的这些逻辑，才能实现可信的轻客户端。然而，轻客户端是在合约中实现，需要充分考虑合约的特性和限制，
因此在实现细节上，需要做诸多调整。

**3.3.2 eosio::table("chaindb"), ibc.chain合约中的forkdb**

*1.轻客户端(lwc)的LIB如何确定*  
有两种方案，一种是完全按forkdb的逻辑，维护一整套`confirm_count`和`confirmations`等`block_header_state`相关信息，每添加一个区块
计算一次LIB，这样做的优点是可以准确获得实时LIB值，然而对于轻客户端来说，其关心的是区块已经不可逆，而并非实时的精确LIB值。有没有更简单的方案呢，
根据上述的逻辑，如果有活跃的2/3以上的BP在出块，并且某个区块的深度超过336，则此区块一定是不可逆的，可以用于验证跨链交易；使用这种方案，
可以简化合约中forkdb的复杂度。

lic.chain合约中表`global`的`lib_depth`是一个深度值，当在一段连续的区块头中，某个区块头的深度超过此值时，则认为不可逆，可以用于验证交易了。
此值应该设置多少合适呢，可以设置成336，当轻客户端检查到有2/3以上的BP在出块，则认为深度超过了336的区块是不可逆的。然而在合约中添加和删除区块头
是非常耗cpu的，实际测试可知，每添加一个区块头需要消耗0.5毫秒cpu，每删除一个区块头需要0.2毫秒，因此每个区块头需要0.7ms的cpu。
是否会出现主网巨大波动，导致没有进lib的区块全部回滚呢，这是有可能的，实际也发生过，因此要必须确保一笔跨链交易所在区块进入lib，再开始处理。
插件在此时可以起到一定的作用，在BOSCORE技术团队研发的ibc_plugin中，设置了参数，只处理一定深度内的跨链交易，这样，插件中的深度和合约中的深度相加
超过336即可。这只是BOSIBC初期的做法，目的是避免消耗大量cpu，后续可能会考虑将合约中的深度设置为336，从而完全部依赖中继的深度，然而无论是插件
还是合约中增加深度值，都会直接延长跨链交易到账时间，从而影响用户体验，因此需要根据实际情况确定一个合理的值，从而即保证足够安全又不失良好的用户体验。

*2.表chaindb*  
表chaindb是ibc.chain合约中的forkdb，ibc.chain合约中没有blog结构，和eosio中forkdb使用的`block_header_state`不同，chaindb进行了大量精简，只保留了
`block_num、block_id、header、active_schedule、pending_schedule、blockroot_merkle、block_signing_key`7个数据，又因为
bp schedule需要占用大量空间，因此在另外的表`prodsches`中存储实际schedule信息，在chaindb中用id引用，以节省内存占用和wasm cpu消耗。

*3.轻客户端的创世区块*  
轻客户端需要一个可信的起点，此起点是轻客户端的创世区块头，后面所有区块头的验证都是基于创世区块头的信息。
创世区块头需要`block_header_state`中的`blockroot_merkle`，`active_schedule`和`header`信息，才能验证区块签名。
源码为`chain::chaininit()`，最重要一个限制是，创世区块头的`pending_schedule`必须和`active_schedule`相同，因为其不同意味着
此区块是bp列表更替过程中的一个区块，如果使用更替过程中的区块，需要同步后续的区块，知道`active_schedule`被`pending_schedule`替换，
增加了复杂度，因此这样的区块不适合作为创世区块。

*4.轻客户端是如何添加新区块的*  
轻客户端添加header的方式和和eosio的forkdb非常类似。源码见ibc.chain的`chain::pushheader()`。
第一步，通过区块编号验证是否能够连接到最新的section
第二部，是否需要处理分叉，删除旧数据。在ibc.chainz中不会同时保存多个分支，而是以后者替代前者的方式实现对分叉的处理。
第三步，通过区块id验证是否能够连接到最新的section
第四部，构造`block_header_state`，并验证BP签名

其中最核心的是构造`block_header_state`，在这个过程中处理的`BP schedule`的更换，确保有2/3以上活跃BP（见`section_type::add()`）。

*5. Section*
Section是chaindb的核心概念和创新，意思是一段连续的区块头，Section的管理也是最ibc.chain的核心的逻辑。
使用section的目的是降低cpu消耗，只有在BP schedule有变化或有跨链交易时才需要同步一部分区块头。
任何section的起始区块不能是 bp schedule 更换过程中的区块，也就是说，任何一个seciton的起始区块的`pending_schedule.version`必须等于
`active_schedule.version`，并且一个section的起始区块头的`active_schedule`必须和上一个section最后区块的`active_schedule`相同，
这样就保证了在任意两个section之间一定不存在BP schedule的更换，每一此BP schedule更换的完整过程必须在某个section中完成，从而确保
section数据的可信性。

### 四、跨链交易

*1. 跨链交易三部曲*  
一笔跨链交易分为三个过程，下面以将EOS从EOS主网跨链转账到BOS主网为例说明，首先用户在EOS主网发起一笔跨链建议`orig_trx`，在EOS侧`ibc.token`合约的
`origtrx`表中会记录此交易信息，当这笔交易所在区块进入lib后，EOS侧IBC中继(relay_eos)将此交易和交易相关信息(区块信息及Merkle路径)
传递到BOS侧中继(relay_bos)；relay_bos构造cash交易并调用BOS侧`ibc.token`合约的cash接口，如果调用成功，
cash函数中会给目标用户发行对应的token；等cash交易所在的区块进入lib后，relay_bos会将`cash_trx`和此交易相关信息（区块信息及Merkle路径）
传递到relay_eos，relay_eos构造cashconfirm交易并调用EOS侧`ibc.token`合约的cashconfirm接口，cashconfirm会删除EOS侧`ibc.token`合约
中对`orig_trx`的记录，至此一笔完整的跨链交易完成。

*2. 跨链失败的交易*  
跨链交易是可能失败的，比如指定的账户在对方链上不存在，或者由于网络环境恶劣，导致调用cash接口失败，未能成功跨链的交易会被回滚，即原路退还用户的资产，
然而现在的IBC系统是交易驱动的，失败的IBC交易需要等到有成功的IBC交易完成后才会被回滚。（注：后续版本升级会让失败的交易尽快回滚）

*3. 如何防止replay攻击，即双花攻击*  
防止双花攻击分为两个阶段：  
1，一笔成功的跨链交易只能执行一次cash，否则会造成重复cash。  
2，对于每一个cash交易，必须将其相关信息传回原链执行cashconfirm，以消除合约中记录的原始交易信息，否则会出现即在目的链上给用户发行了token，
又将原链的token退还给了用户。  

cash函数是ibc.token的核心逻辑，`ibc.token`合约中记录着最近执行cash的原始交易id，即`oirg_trx_id`，并且新的cash的`orig_trx`的区块编号必须
大于或等于所有`orig_trx`所在的区块编号，也就是说必须按原始交易在原链按区块的顺序进行cash，（执行cash时，原链某个区块内的跨链交易顺序是无关紧要的）
，再结合trx_id检查，可以确保一笔跨链交易只能执行一次cash。

同样，cashconfirm接口会检查cash交易的编号`seq_num`，此编号必须逐一递增，以确保所有在目的链上的cash交易都会删除在原链上的原始交易记录，
从而确保不会出现双花的情况。


### 五、插件

插件的作用分为两部分：1，轻客户端同步；2，跨链交易传递。  
核心逻辑请参考`ibc_plugin_impl::ibc_core_checker()`

ibc_plugin主要参考了net_plugin的框架。


### 六、问答

1. 问：IBC合约的多个action中用到了relay的权限，那么，本IBC系统是否依赖对中继的信任。  
答：验证relay权限处于两种考虑：1，ibc.chain合约使用了section的机制，现在的逻辑不允许为旧的section添加区块，
如果任何人都可以调用pushsection接口，假设应该push的区块范围是1000-1300，故意捣乱的人可能会抢险push 1100-1300，
从而导致1000-1100无法被push，进而导致一些跨链交易无法成功，（注，此问题会在后续版本中考虑优化）；2，考虑到IBC系统承载着
大量用户资产，并且本系统还未经过长期市场考验，因此增加了relay权限，以降低安全风险。

### 七、待解决的问题
 
*1. 多中继通道无法协调工作*
由于引入section机制，为整个IBC逻辑增加了负责度，当前多个中继通道并存会导致互相协调失败，从而导致IBC系统无法工作。
解决办法是在合约和插件中增强完善对sction的管理，从而做到多中继通道并存，提高整个IBC系统的可靠性。


### 八、升级计划

*1. 兼容pbft*

*2. 以更优雅的方式支持多条侧链&EOS主链互相跨链*

*3. 支持token以外其他类型数据的跨链*


### 九、参考
[Inter-blockchain Communication via Merkle Proofs with EOS.IO](https://steemit.com/eos/@dan/inter-blockchain-communication-via-merkle-proofs-with-eos-io)
[Bitcoin](https://bitcoin.org/bitcoin.pdf)
[Chain Interoperability](https://static1.squarespace.com/static/55f73743e4b051cfcc0b02cf/t/5886800ecd0f68de303349b1/1485209617040/Chain+Interoperability.pdf)