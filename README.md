# PBFT
Knowledge about pbft.  
&emsp;&emsp;PBFT在保证可用性和安全性（liveness & safety）的前提下，提供了(n-1)/3的容错性，意思就是如果系统内有n台机  子，那么系统最多能容忍的作恶/故障节点为(n-1)/3个。（作恶节点可以不响应或者回应错误的信息）  

**为了保证pbft算法的正确性，节点总数量 n 和作恶节点数量 f 必须满足 n>3f**  
&emsp;&emsp;**reason**：假设有f个作恶节点，所以必须在*n-f*个状态复制机(SMR)的沟通下就要做出决定（因为在设计异步通信算法的时候，我们不知道那f个节点是恶意节点还是故障节点，这f个节点可以不发送消息，也可以发送错误的消息，所以在设计阈值的时候，我们要保证必须在n-f个状态复制机的沟通内，就要做出决定。因为如果阈值设置为需要n-f+1个消息，那么如果这f个作恶节点全部不回应，那这个系统根本无法运作下去）。
&emsp;&emsp;在n-f个状态复制机的沟通内，就要做出决定。我们无法预测这f个作恶节点做了什么（错误消息/不发送），所以并不知道这n-f个里面有几个是作恶节点。我们必须保证正常的节点大于作恶节点数，即有 n - f - f > f，从而得出了n > 3f。

![image](https://user-images.githubusercontent.com/82591506/178739325-81a80f71-d50a-4864-9014-4c98e4795043.png)
## 一、概念
### 1.视图(view)
  将时间段分为一段一段的view，在一个view内包括一个primary和其他backups。
### 2.客户端(client)
  客户端节点，负责发送交易请求。
### 3.主节点(primary)
  决定client请求操作的执行顺序(排序)，并打包发给backups。
### 4.副节点(backups)
  验证消息的合法性，对共识消息达成一致。
  
## 二、算法流程
### 三阶段 pre-prepare/prepare/commit
### REQUEST
&emsp;&emsp;首先，客户端(client)向主节点0发起请求 **<<REQUEST,o,t,c>>**，其中*t*是时间戳，*o*表示操作，c是client.
### 1.pre-prepare
&emsp;&emsp;主节点(primary)收到客户端(client)请求REQUEST。对REQUEST消息处理：  
&emsp;&emsp;(1)判断消息是否超出限制;  
&emsp;&emsp;(2)生成序列号n;  
&emsp;&emsp;(3)主节点(primary)向副节点(backups)发送 pre-prepare 消息 **<<PRE_PREPARE,v,n,d>,m>**，其中*v*代表视图编号、*n*代表序号（主节点收到客户端的每个请求都以一个编号来标记）、*d*代表m的消息摘要，*m*代表原始消息数据。  
### 2.prepare
&emsp;&emsp;副节点backups收到 pre-prepare 消息，进行以下消息验证：  
&emsp;&emsp;(1)消息m的签名合法性，并且消息摘要d和消息m相匹配：d=hash(m);  
&emsp;&emsp;(2)节点当前处于视图v中；  
&emsp;&emsp;(3)节点当前在同一个(view v ，sequence n)上没有其它pre-prepare消息，即不存在另外一个m'和对应的d' ，d'=hash(m')；  
&emsp;&emsp;(4)h<=n<=H，H和h代表序号n的高低水位。当前副节点(backups)同意请求后会向其它节点(backups、primary)发送 prepare 消息 **<PREPARE,v,n,d,i>**，*v*是view，*n*是序列号*d*是m的消息摘要，*i*是节点的编号。  
### 3.commit
&emsp;&emsp;主节点(primary)、副节点(backups)收到 prepare 消息，进行验证：  
&emsp;&emsp;(1)检查 PREPARE 消息签名是否正确、消息摘要d和消息m相匹配：d=hash(m);  
&emsp;&emsp;(2)节点当前处于视图v中;  
&emsp;&emsp;(3)h<=n<=H，H和h代表序号n的高低水位;  
&emsp;&emsp;(4)前3条验证通过，将收到的消息写入log;  
&emsp;&emsp;(5)如果收到正确的 PREPARE 消息大于等于 **2f+1**，就进入 COMMIT 阶段，广播 **<COMMIT,v,n,d,i>**。  
### REPLY
&emsp;&emsp;主节点(primary)、副节点(backups)收到 commit 消息，进行验证：  
&emsp;&emsp;(1)检查 COMMIT 消息签名是否正确、消息摘要d和消息m相匹配：d=hash(m);  
&emsp;&emsp;(2)节点当前处于视图v中;  
&emsp;&emsp;(3)h<=n<=H，H和h代表序号n的高低水位;  
&emsp;&emsp;(4)前3条验证通过，将收到的消息写入log;  
&emsp;&emsp;(5)如果收到正确的 COMMIT 消息大于等于 **2f+1**个(包括自己)，就回复 **<<REPLY,v,t,c,i,r>>** 消息给 client， *t*是时间戳timestamp，*c*是client， *i*是节点i的编号，*r*是节点i的回复。
*为什么prepare和commit阶段需要2f+1个节点的反馈？*
&emsp;&emsp;考虑最坏的情况：我们假设收到的有f个是正常节点发过来的，也有f个是恶意节点发过来的，那么，第2f+1个只可能是正常节点发过来的。（因为我们限制了最多只有f个恶意节点）由此可知，“大多数”正常的节点还是可以让系统工作下去的，所以2f+1这个参数和n>3f+1的要求是逻辑自洽的。
## 三、checkpoint 、stable checkpoint、高低水位
### checkpoint 
&emsp;&emsp;checkpoint 就是当前节点处理的最新请求序号。前文已经提到主节点收到请求是会给请求记录编号的。比如一个节点正在共识的一个请求编号是101，那么对于这个节点，它的 checkpoint 就是101。
### stable checkpoint
&emsp;&emsp;stable checkpoint 就是大部分节点 （2f+1） 已经共识完成的最大请求序号。比如系统有 4 个节点，三个节点都已经共识完了的请求编号是 213 ，那么这个 213 就是 stable checkpoint 了。 **作用**：最大的目的就是减少内存的占用。因为每个节点应该记录下之前曾经共识过什么请求，但如果一直记录下去，数据会越来越大，所以应该有一个机制来实现对数据的删除。那怎么删呢？很简单，比如现在的稳定检查点是 213 ，那么代表 213 号之前的记录已经共识过的了，所以之前的记录就可以删掉了。
### 高低水位
![image](https://user-images.githubusercontent.com/82591506/179181622-041a8e22-d15b-4c7d-b27e-4c99feba980a.png)  
&emsp;&emsp;图中A节点的当前请求编号是 1039，即checkpoint为1039，B节点的 checkpoint 为1133。当前系统 stable checkpoint 为 1034 。那么1034这个编号就是低水位，而高水位 H=低水位 h+L ，其中L是可以设定的数值。因此图中系统的高水位为 1034+100=1134。如果 B 当前的 checkpoint 已经为 1134，而A的 checkpoint 还是 1039 ，假如有新请求给 B 处理时，B会选择等待，等到 A 节点也处理到和 B 差不多的请求编号时，比如 A 处理到 1112 了，这时会有一个机制更新所有节点的 stabel checkpoint ，比如可以把 stabel checkpoint 设置成 1100 ，于是 B 又可以处理新的请求了，如果 L 保持100 不变，这时的高水位就会变成 1100+100=1200 了。  

## 四、垃圾回收

&emsp;&emsp;讨论从日志中清理消息的机制。为了保证Safty属性，必须将消息保存在replica的日志中，直到确信相关的请求已由至少f+1个非恶化replica执行，并且在视图切换时能够向别人证明这一点。此外，如果某些replica丢失了一些被所有的非恶化replica都已经清理掉的消息，则需要通过同步全部或部分服务状态使其得到更新。因此，replica也需要一些证明显示其状态是正确的。 

&emsp;&emsp;在每次执行操作后都去生成这些证明是代价高昂的。取而代之的是定期生成方式，比如当被执行的请求序号能够被某个常数（例如 100）整除时。我们将达到定期生成条件的请求被执行之后所产生的状态称为检查点，并规定带有证明的检查点就是稳定检查点。 

&emsp;&emsp;一个replica会维护服务状态的多个逻辑备份：最后一个稳定检查点，零个或多个不稳定检查点以及当前状态。正如第6.3节所述，写时复制技术（Copy-on-write）可用于减少存储状态的额外备份而带来的空间开销。 

&emsp;&emsp;检查点的正确性证明生成方式如下。当一个replica i 产生检查点时，它将广播一条检查点消息（以下引用原文称为checkpoint消息）$<{CHEKCPOINT,n,d,i}>_{\sigma_i}$到其他replica，其中 n 是最后一个请求序号，该请求的执行结果将反映在其状态中，而 d 是状态的摘要。每个replica都将一直接收并保存检查点消息在其日志中，直到其拥有 2f+1 个（可以包括自己在内）请求序号为 n、状态摘要为 d、且由不同replica签名的checkpoint消息。这 2f+1 条消息是检查点的正确性证明。 

&emsp;&emsp;具有证明的检查点将变得稳定，replica将丢弃其日志中请求序号小于或等于n的所有pre-prepare、prepare和commit消息；它还会丢弃所有较早的检查点和checkpoint消息。

&emsp;&emsp;检查点证明的计算是高效的，因为状态摘要可以使用第6.3节中讨论的增量加密[1]进行计算，并且很少生成证明。检查点协议用于推进低水位线标记和高水位线标记（这限制了哪些消息可以被接收）。低水位标记 h 等于最后一个稳定检查点的请求序号。高水位线标记H=h+k，其中 k 足够大，因此replica不会因等待检查点稳定而停止工作。例如，如果检查点是每100个请求生成，k 则可能是200。

## 五、视图更换 

&emsp;&emsp;当主节点挂了（超时无响应）或者从节点集体认为主节点是问题节点时，就会触发 ViewChange 事件， ViewChange 完成后，视图编号将会加 1 。 

![image](https://user-images.githubusercontent.com/82591506/178883318-4a5f15e6-6dba-4a09-987e-79d0dc07d6fa.png)  

如果backup i 的计时器在视图 v 中超时，则 i 将发起视图切换以将系统切换至视图 v+1 。它将停止接收消息（checkpoint，view-change和new-view消息除外），并广播一个视图切换消息（以下引用原文称为view-change消息）$<{VIEW-CHANGE,v+1,C,P,i}>_{\sigma_i}$给所有replica。 

&emsp;&emsp;n 是 i 已知的最近一个稳定检查点 S 的请求序号， *C* 是证明 S 正确性的 2f+1 个有效的checkpoint消息集合，*P* 是一个包含集合的集合，其中 m 表示 i 已经prepare过的每一个请求序号大于 n 的请求。每个集合$P_m$包含一个有效的pre-prepare消息（不包含对应的client请求）和 2f 个匹配项，即是视图编号、请求序号和请求摘要相同，由不同backup签名的有效的prepare消息。  

当视图 v+1 的primary p 从其他replica接收到 2f 个切换至视图 v+1 的有效view-change消息时，它将广播一个新视图消息（以下引用原文称为new-view消息）$<{NEW-VIEW,v+1,V,O}>_{\sigma_p}$给所有其他replica。

&emsp;&emsp;其中 *V* 是一个集合，包含primary接收的有效view-change消息以及primary已经发送（或还未来得及发送的）的切换至视图 v+1 的view-change消息，而 *O* 是一组pre-prepare消息（不附带请求内容）。O 的计算如下： 

1. primary确定 V 中最新的稳定检查点的请求序号min-s和 V 中所有prepare消息里请求序号的最大值max-s。 

2. primary为介于min-s和max-s之间的每个序号 n 以视图 v+1 分别创建新的pre-prepare消息。有两种情况：（1）在 V 中的某些view-change消息的 P 集合中，已经至少存在一个请求序号 n 了，或者（2）不存在这样的情况。在第一种情况下，primary创建一个新消息$<{PRE-PREPARE,v+1,n,d}>_ {\sigma_p}$，其中 d 是 V 中请求序号为 n ，且具有最高视图编号的pre-prepare消息的请求摘要。在第二种情况下，它将创建一个新的pre-prepare消息$<{PRE-PREPARE,v+1,n,d^{null}}>_{\sigma_p}$，其中$d^{null}$是一个特殊的空请求的请求摘要；空请求像其他请求一样遵从协议，但是它执行的是空操作。（Paxos [18]使用了类似的技术来补齐。） 

接下来，primary将 O 中的消息追加写入到其日志中。如果min-s大于其最新的稳定检查点的请求序号，则primary还将在其日志中插入请求序号为min-s的检查点的稳定性证明，并如第4.3节中所述清理日志信息。然后它进入视图 v+1 ：此时，它可以接受视图 v+1 的消息。

backup将接受切换至视图 v+1 的new-view消息，如果其签名正确，包含的view-change消息都是切换至视图 v+1 的有效消息，并且集合 O 是正确的；backup通过执行类似于primary创建 O 的计算过程来验证 O 的正确性。然后，跟primary一样将这些新信息追加写入到其日志中，为 O 中每个消息广播一个prepare消息到所有其他replica，把这些prepare消息追加写入到其日志中，并进入视图 v+1 。 

此后，按第4.2节中所述继续执行协议。replica其实是按照协议重做了min-s和max-s之间的所有消息，但是它们避免重新执行client的请求（通过使用它们存储并最后发送给每个client的响应信息）。 

一个replica可能缺少某些请求消息 m 或一个稳定检查点（因为他们不会在 new-view消息中发送）。它可以从另一个replica中获取丢失的信息。例如，replica i 可以从任意一个在 V 中已经证明了其检查点消息的正确性的那些replica中获得丢失的检查点状态 S 。由于这样的replica中有 f+1 个是正确的，因此replica i 将始终能够获得 S 或一个稍后验证过的稳定检查点。我们可以通过对状态进行分区并为每个分区标记上修改它的最后一个请求序号，来避免发送整个检查点。要将replica同步至最新，只需发送并更新它那些已经过期的分区即可，而不是整个检查点。

