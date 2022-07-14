# PBFT
Knowledge about pbft.  
&emsp;&emsp;PBFT在保证可用性和安全性（liveness & safety）的前提下，提供了(n-1)/3的容错性，意思就是如果系统内有n台机  子，那么系统最多能容忍的作恶/故障节点为(n-1)/3个。（作恶节点可以不响应或者回应错误的信息）
---
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
&emsp;&emsp;(5)如果收到正确的 PREPARE 消息大于等于2f，就进入 COMMIT 阶段，广播 **<COMMIT,v,n,d,i>**。  
### REPLY
&emsp;&emsp;主节点(primary)、副节点(backups)收到 commit 消息，进行验证：  
&emsp;&emsp;(1)检查 COMMIT 消息签名是否正确、消息摘要d和消息m相匹配：d=hash(m);  
&emsp;&emsp;(2)节点当前处于视图v中;  
&emsp;&emsp;(3)h<=n<=H，H和h代表序号n的高低水位;  
&emsp;&emsp;(4)前3条验证通过，将收到的消息写入log;  
&emsp;&emsp;(5)如果收到正确的 COMMIT 消息大于等于2f+1个(包括自己)，就回复 **<<REPLY,v,t,c,i,r>>** 消息给 client， *t*是时间戳timestamp，*c*是client， *i*是节点i的编号，*r*是节点i的回复。  
## 三、垃圾回收
## 四、视图更换
![image](https://user-images.githubusercontent.com/82591506/178883318-4a5f15e6-6dba-4a09-987e-79d0dc07d6fa.png)
