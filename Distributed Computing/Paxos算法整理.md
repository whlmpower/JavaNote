# Paxos 算法整理
## 提案的选定    
足够多的Acceptor是整个Acceptor集合的一个子集，并且让这个大集合大的可以包含Acceptor集合中的大多数成员，因为任意两个包含大多数Acceptor的自己至少有一个公众成员。      

## 推导过程   
P1： 一个Acceptor必须批准它收到的第一个提案。     

P2: 如果编号为M0、Value 值为V0的提案（[M0, V0]）被选定了，那么所有比编号M0更高的，且被选定的提案，其值也是V0.     

条件P2就保证了只有一个value值被选定这个关键安全性属性。同时，一个提案被选定，其首先必须被至少一个Acceptor批准。      

P2a: 如果编号为M0,Value值为V0的提案被选定了，那么所有比编号M0更高的，且被Acceptor批准的提案，其值也必须是V0。     
结合P1 和 P2a，需要对P2a进行强化：     

P2b： 如果一个提案[Mo,V0]被选定了，那么之后任何Proposer产生的编号更高的提案，其值都是V0。     

重点论证P2b 成立即可。        

通过对Mn进行第二数学归纳法来进行证明，证明以下结论成立：     
假设编号在M0到Mn-1之间的提案，其value值都是V0，证明编号为Mn的天的Value值也是V0.      
因为编号为M0 的 提案已经被选定了，这就意味着肯定存在一个由半数以上的
Acceptor组成的集合C，C中的每一个Acceptor都批准了这个提案，再结合归纳假设，“编号为M0的提案被选定”意味着：       

**C中的每个Acceptor都批准了一个编号在M0到Mn-1范围内的提案，并且每个编号在M0到Mn-1范围内的被Acceptor批准的提案，其值都是V0**       

因为任何包含了半数以上Acceptor的集合S都至少包含C的一个成员，因此我们可以任务如果保持了下面P2c的不变性，那么编号为Mn的提案的value也为V0。   

P2c: 对于任意的Mn和Vn，如果提案[Mn,Vn]被提出，那么肯定存在一个由半数以上的Acceptor组成的集合S,满足以下两个条件中的任何一个：    
- S中不存在任何批准过编号小于Mn的提案Acceptor     
- 选取S中所有Acceptor批准的编号小于Mn的提案，其中编号最大的那个提案，其Value值也是Vn    

我们需要进行反向推导： P2c==>P2b==>P2, 然后通过P2 P1来保证一致性       

### 我们先来证明P2c:   

**首先假设提案[M0, V0]被选定了，设比该提案编号更大的提案[Mn, Vn]，我们需要证明就是在P2c的前提下，对于所有的[Mn, Vn]，存在Vn = V0；**     
1.当Mn = M0 + 1 时，如果有这样一个编号为Mn的提案，首先我们知道[M0, V0]已经被选定了，那么就一定存在一个Acceptor的子集S， 且S中的Acceptor已经批准了小于Mn 的提案，那么就一定存在一个Acceptor的子集S，且S中的Acceptor已经批准了小于Mn的提案，于是，Vn 只能是多数集中小于Mn 但为最大编号的那个提案的值。而此时因为Mn = M0 + 1,因此理论上编号小于Mn但为最大编号的那个提案肯定是[M0, V0]，同时由于S和通过[M0, V0]的Acceptor集合都是多数集，也就是二者之间肯定有交集————这样Proposer在确定Vn取值的时候，就一定会选择V0.    

2.根据假设，编号在M0+1 到 Mn-1区间内的所有提案的Value值都为V0，需要证明的是编号为Mn的提案的Value值也是V0.根据P2c，首先同样一定存在一个Acceptor的子集S，且S中的Acceptor已经批准了小于Mn 的提案，那么编号为Mn的提案的Value值也只能是这个多数集中编号小于Mn 但为最大编号的那个提案的值。如果这个编号落在M0+1 到 Mn-1区间内，那么Value值肯定为V0，如果不落在M0+1 到 Mn-1区间内，那么它的编号不可能比M0更小了，肯定就是M0，因为S 也肯定会批准[M0, V0]这个提案的Acceptor集合S有交集，而如果是M0,那么他的Value值也是V0，由此得证。     

## Proposer 提案的生成     
在P2c 的基础上，Proposer在产生一个编号为Mn的提案时，必须要知道当前某一个将要或者已经过半数以上Acceptor批准的编号小于Mn 但是为最大编号的Mn 的提案。   
1.Proposer选择一个新的提案编号Mn，然后向某个Acceptor集合的成员发起请求，要求该集合中的Acceptor做如下回应：    

- 向Proposer承诺，保证不会再批准任何编号小于Mn 的提案。    
- 如果Acceptor已经批准过任何提案，那么就向Proposer反馈当前该Acceptor已经批准的编号小于Mn但是为最大编号的那个提案的值。     
我们将该请求称为编号为Mn的Prepare请求     


2.如果Proposer收到了来自半数以上的Acceptor的响应结果，就可以产生编号为Mn、Value值为Vn的提案，这里的Vn是所有响应中编号最大的提案Value值，当然还存在另外一种情况，就是过半的Acceptor都没有批准过任何提案，即响应不包含任何的提案，那么此时Vn的值就可以由Proposer任意选择。      

在确定提案之后，Proposer就会将该提案再次发送给某个Acceptor集合，并期望获得他们的Accept请求。需要注意的是，此时接受Accept请求的Acceptor集合并不一定是之前响应Prepare请求的Acceptor集合。       

## Acceptor批准提案    
P1a: 一个Acceptor只要尚未响应过任何编号大于Mn的Prepare请求，你们它就可以接受这个编号为Mn 的提案      
P1a包含了P1，值得一提的是，Paxos 允许Acceptor忽略任何请求而不用担心破坏其算法安全性。    
## 算法优化：     
Acceptor可以忽略这样的Prepare请求（请求编号为Mn），同时也可以忽略掉它已经批准提案的Acceptor请求。       


## 算法陈述   
**阶段一：**   
1.Proposer选择一个提案编号为Mn，然后向Acceptor的某个超过半数的子集成员发送编号为Mn的Prepare请求。     
2.如果一个Acceptor收到了一个编号为Mn的Prepare请求，且编号Mn大于该Acceptor已经响应的所有Prepare请求的编号，那么它就会将它已批准的最大编号的提案作为响应反馈给Proposer，同时承诺不会批准任何编号小于Mn的提案。     

**阶段二：**     
1.如果Proposer收到来自半数以上的Acceptor对于其发出编号为Mn 的Prepare请求的响应，那么它就会发送一个针对[M0, Vn]提案的Acceptor请求给Acceptor。注意，如果Vn的值就是收到响应中编号最大的提案的值，如果响应中不包含任何值，那么他就是任意值。    
2.如果Acceptor收到这个针对[Mn,Vn]提案的Acceptor请求，只要该Acceptor尚未对编号大于Mn的Prepare请求做出响应，它就可以通过这个提案。    


## 提案的获取   

主要考虑方案三： （具体略，以后有时间再补充吧）    

## 通过选取主Proposer保证算法的活性    
避免产生死循环，必须选择一个主Proposer，并规定只有主Proposer才能提出议案。 只要主Proposer和过半的Acceptor能够正常进行网络通信，那么但凡主Proposer提出一个编号更高的提案，该提案最终就会被通过。    
    


     


