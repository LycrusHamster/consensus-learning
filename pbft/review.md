bft-tocs还没有阅读,接下来补完

首先,Liskov和Castro这两位大牛,直到最后的Proactive Recovery in a Byzantine-Fault-Tolerant System才把pbft的坑填上...

在早起的论文里,Liskov只是解读了当Primary产生byz-fault的时候,为了保证liveness,
backups可以发起viewchange来最终获得一个non-faulty的Primary....
但是如何鉴别,如何传递Primary产生错误的方式,完全没有提.只是从Quorum,Certificate论证了Safty和Liveness.

摘要
如果一个节点收到了**preprepare**+2f个prepare,其中prepare包括自己.那么就认为自己进入了prepared.并发送commit给他人.

如果没有收到preprepare,但是有2f+1个其他节点的prepare,能否进入prepared?答案是不能.
因为可能与Primary有network partition,会可能使得f个byz节点加上你这一个,超过限度.
也可能发现2f+1个prepare里面有人作恶,但是我们不能冒然发起viewchange,因为Primary可能不是作恶的节点.
所有必须不能进入prepared,然后超时发起viewchange.


如果有2f+1个节点进入了prepared的状态,我们就认为committed. (原文是f+1和non-faulty,我们取最坏的情况,事实上你完全不需要关心byz节点的状态).
(f+1) + (2f + 1) - (3f + 1) = 1 > 0, 至少有1个non-fault节点可以在2次Quorum传递信息的时候,把正确的信息传递下去,使得可以识别有byz节点在捣鬼.

如果一个节点prepared,并且收到2f+1个commit,那么他就将信息commit到状态机,进入committed-local的状态.

在Viewchange过程中,和所有的共识算法一样,要把之前已经commit到状态机里的状态传递下去,否则rsm就会崩溃..
committed-local -> 有2f+1个commit -> 有2f+1个节点进入了prepared, 且每个进入了prepared的节点收到了preprepare和2f和prepare
现在viewchange,Quorum intersection是 2*(2f+1) - (3f+1) = f+1, 有一个non-faulty节点存活,只要有byz-faulty捣鬼,就可以立马判断出来,

如果新的Quorum里面,有f个byz节点,1个non-faulty节点,f个刚上线的空白节点,那么但是究竟是谁捣鬼,正确的prepare信息是什么,还无法知晓.需要空白节点
走state-transfer来获取正确的数据,从使得的f个byz节点<f+1个non-faulty节点.
如果新的Quorum里面,没有空白节点,则直接f+1>f,搞定.
如果没有statetransfer,那总共的faulty节点不是f个,而是f+f个,这不满足safety和liveness的要求了.

关于prepared和commmitted.committed(2f+1个prepared)但没有被committed-local的请求,依然会被下一个viewchange包含.
这是因为2f+1=quorum.2个quorum intersection肯定有1个non-faulty,同上,传递下去.

所以只要committed了,哪怕没有committed-local,这个request也会被viewchange传递下去.

如果一个request没有被committed(2f+1个prepared),但是有至少被1个节点prepared.那就说不好了.如果viewchange的时候,Prepared Certificate包含了这个prepared的request,
那就会接力下去.如果没有,那就不会.这个Paxos的safe的准则是一样的,没有被超过2/3的节点认可的request,尽最大努力传递.

关于viewchange. 不考虑checkpoint,所以Prepared Certificate会包括从最开始的request.......我知道这很蠢,但是解释起来会更简单

viewchange过程中需要超过2/3的节点列举出所有的prepared request.只要request超过了2/3(也就是committed)了,那么一定会在前面的列举里.
新的Primary就可以把他加入到新view的preprepare请求里.

同时,新的newview请求要附上新Primary收集到的超过2f+1的viewchange请求,让backups可以确信新Primary没有作恶,否则继续viewchange.

seq-no是全局递增的.

viewchange的时候,所有节点的判断是互相独立的,但是只要收到f+1个viewchange请求的时候,非viewchange请求也应该进入viewchange状态,这是因为

至少有1个non-faulty节点请求viewchange,说明他可能有网络不良,或者鉴证了Primary作恶,我们应该相信他.虽然它可能是网络单项不连通,或者可能没有把证据发出来.
但是考虑到最多只有f个faulty节点,我们不能再允许哪怕1个节点有network partition. 因为这样会有f+1个faulty节点,虽然那1个节点是non-faulty的节点,
可以通过state transfer来追上进度....


为什么要有commit阶段?

因为上文解释,request被2f+1个节点认可后,才能顺利经历viewchange.
每个节点在做是否要prepare决策的时候,不能依赖其他节点,否则会陷入依赖死循环.
一个节点可能认可了request并广播,但是没有收到别人对request认可的广播.(异步网络)
当节点收到2f+1个prepare+preprepare的时候,怎么才能告诉其他节点自己认可了么?
他就得再一次告诉其他节点,自己prepared了.这个传递自己prepared的过程,就叫做commit.
同理,commit过程也是异步网络,但只要收到了2f+1个commit请求,那肯定是有超过2f+1个节点进入了prepared的了.

所以committed-local一定是commit,但只要是commit了,就可以经历viewchange了.只是节点没有上帝视角,他只能通过committed-local来感知.

为什么pbf需要3f+1?而不是像拜占庭将军问题那样,3f就够了呢?

原因是,拜占庭将军问题里,一个节点选择是否才行另一个节点的信息,需要通过递归访问/广播其他节点的补充信息,才能做判断.
首先f大了,递归效能吃不消,不practical.其次,节点必须先做出选择,然后告诉别人,而不是先等别人告诉自己足够的信息,再做抉择.因为超过2次递归,不被允许,
那么无法获得足够信息,所以只能先相信Primary.

那么Primary的作恶,会导致在某个non-faulty节点视角里,Primary的preprepare请求,和自己prepare请求(照搬Primary的preprepare),都发生变化,导致
共识无法进行下去.比如,Primary发送个3个节点ABC分别是0,1,1.下面的顺序是PABC

在A的视角里0,0,1,? (因为liveness,收到2f+1张票就要做出决定,所有这里的C的票被忽视)
在B的视角里1,?,1,1
在C的视角里1,?,1,1

显然A无法进行共识.A发现Primary作弊,广播证据,进行viewchange.

这时候,如果P,B,C达成了共识,viewchange太慢了,失败了,A其实应该被认为是另一种faulty节点.那么整个共识就被卡住了.因为超过了f个faulty节点了.

当然,如果协议有sync,也可以走sync.这里f个byz节点都拒绝sync请求,而我们认为A其实代表的是f个离线节点,那么剩下的BC代表的f+1个节点,必须都给出响应.这么sync可以收到f+1个相同的结果
sync可以成功.共识又活了.

如果A发现作弊,并且发送viewchange成功,那么其他的non-faulty节点也会进入viewchange,Primary被替换,皆大欢喜.

总结

2次quorum交换信息的时候,都最多只能有f个faulty,但是可以分别是不同的faulty.不能有超过f个faulty的情形.
如果有超过f个faulty的情形,只能靠其他手段弥补,比如sync/statetransfer.

疑问

prepare和commit阶段,是不是收到f+2个一致的消息就可以了?
不行(f+2) + (f+2) - (3f+1), 在f=3的时候 交集为0,表现为

P,X,Y 为byz节点
A,B互相收到了PXY和A/B的0
C,D互相收到了PXY和C/D的1
E,F,G互相收到了PXY和E/F/G/其中一个的2

那么最后的记过是,AB=0,CD=1,EFG=2.
下一轮的quorum时,q=5,完全没有办法统一.

sync是弥补sync那个seqno下,faulty节点超过f的情况.
next seqno 和 viewchange都是2多个quorum的碰撞,每个quorum只能有包含f个faulty.
1round,AB和F达成共识1,C离线.然后F离线,ABC进入下一轮round,那么quorum碰撞将失败,因为可能C上线后空白,quorum碰撞只有一个节点有数据.
所以无法判断真假.那个节点可以地上p-cert的信息来证明,这就等同于将上一轮的faulty定义为C.当然C也可以先完成上一轮的同步,将上一轮的faulty定义为F.


sync的时候,因为有P-cert,所以可以还原当时的quorum.并且只问1个节点索取信息.p-cert是>2/3的prepared,也就是committed,
如果没有p-cert,那么就要问所有节点,获得至少f+1个相同的prepare信息,且non-faulty节点必须是在**committed-local**的情况下?
我感觉这是对的,就把这个节点当做client来看到.

???
但是必须小心的是,如果节点进入了prepared,把>2/3的prepare信息加入了Pset,但是最后没有进入committed-local(由于byz节点装死),那么
pset会传递下去,但是byz不响应,pset不够了,所以就会宕住.但是non-faulty节点又没有办法知晓别的节点是否prepared了.
而空白节点,能获取f+1个节点进入了prepared,由于byz节点装死,所以仍然不该保证committed状态.所以提供pset(committed),
或者f+1节点+那f+1个诚实节点的必须committed-local(committed-local相当于承认了pset).
