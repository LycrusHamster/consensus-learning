# Failure model

我要说的不是什么stop-fail, recoverable, byzantine failure.

而是**node failure**和**network failure**.

基于异步网络,async network,当不能收到一个节点在一个确定期限内的反馈时(反馈超时),无法区分一下两种场景:

1. 无法识别是node跪了,或者是node故意不响应.前者可能是普通的fail-stop,后者可能是byztine.但至少都是node failure.
node failure的情况下,一个节点可能在不能quorum给出不一样的答复.也可能在同一个quorum内给不同的人不同的答复.

2. 如果node响应了,但是网络某些因素导致这次响应在这个期限内不可达.这是network failure.这次响应可能被延迟送达,可能丢失在网络中,甚至多次响应乱序到达.

针对与pbft算法里的N>3f+1,这里的f变得尤其.....头大.而且根据signature和mac的差别,更加更加头大.

# MAC 与 signature
signature有着不可抵赖性,irrefutable.

## signature
当prepared的时候,制作prepare certificate包含了signed pre-prepare + 2f * singed prepare.因为签名不可伪造.
那么在f+1个节点中,只要有一个non-faulty节点,PC就可以被传递.因为f个节点无法伪造那一个non-faulty节点的签名.如果f个节点强行改变f个节点的签名,保留那
1个non-faulty节点的签名,就立马会被发现.因为通过那一个non-faulty节点,可以发现f个节点签了不能的签名.

因此,只要每个节点保留了当时数据签名,(当时数据不论是什么,也不论当时需要多少签名),只要有1个non-faulty节点,就可以正确还原数据.

所以,**在确认时**(也就是读),只需要f+1个结果就可以了,但是在**形成决策时**(也就是写),仍然需要2f+1个相同结论, 这是因为quorum不重叠到f+1的时候,
无法保证有1个non-faulty节点,就无法保证不split-brain.


比如:
f=3 N=10

|全局|1|2|3|4|5|6|7|8*|9*|10*,primary|
|---|---|---|---|---|---|---|---|---|---|---|
|1| a|?|?|?|?|?|?|a|a|a|
|2|?|b|?|?|?|?|?|b|b|b|

这里由于backup无脑选择相信primary,我们把情况推广到f+2,f个faulty,1个自己排除,1个non-faulty

|全局|1|2|3|4|5|6|7|8*|9*|10*,primary|
|---|---|---|---|---|---|---|---|---|---|---|
|1&7| a|?|?|?|?|?|a|a|a|a|
|2&6|?|b|?|?|?|b|?|b|b|b|

典型的脑裂.所以在决策/写入时,仍然需要2f+1.

## Mac
Mac的数据只有双方在交互时作为验证,事后其他人完全没有办法正式.所以 prepare certificate只是包含了pre-prepare + 2f * prepare.
这样的pc完全没有证据作用. 所以当复现当时的2f+1的数据事,必须也要有2f+1个节点回应. 这样才能保证f+1个non-faulty占据多数票.

所以,**在确认时**(也就是读),只要有f+ 个**相同**的结果,就可以了.但是在**形成决策时**(也就是写),仍然需要2f+1个相同结论, 这是因为quorum不重叠到f+1的时候,
**在确认时**(也就是读),只要有f+1个**相同**的结果,是因为f+1对于2f+1已经形成了多数派.

这里要注意,是明确给出的结果,如果不能给出结果,则不能被算到f+1内.否则就和fast paxos一样,quorum就得扩大了.
所以这里的quorum*其实是不是真正的quorum.这里的quorum*是收到2f+1张有效的票,也就是PC.而不是2f+1个有效的回应.


## 针对Safety

在至多f个node failure的情况下,safety可以保证.
注意safety的定义,通俗的讲,坏的事情不能发生.当然这和好的事情能不能发生,算法能不能有效推进,是两码事.

## 针对Liveness

在至多f个**failure**的情况下,liveness才能保证.
即,node failure + network failure <= f
faulty = node failure + network failure

## 举例

例如是3f+1, f个normal节点A,f个node failure节点B,f个network failure的节点C,以及1个normal节点D. 注意这里的A,B,C是节点群.

A,B和D在不顾C的情况下,在seqNo n下达成了共识.之后B选择不回应请求,C无论是被B欺骗,还是network failure,都没有收集到足够的PC.
这种情况下,如果在n+1轮发生view change

### 情形:

1. 如果采用mac,需要2f+1个prepare "certificate"(1 PC = collection of {1 pre-prepare + 2f prepares}),**重新构建quorum**.
或者需要f+1个相同的prepare "certificate",**确认**当时的quorum.
因为C在n轮下由于network failure缺席,所以他们无法给出PC,不能算他们的结论.由此2f+1个PC只能由A+B来完成.

这时进行view change,quorum*抽中了B+C+D. C在n轮被视为network failure节点,只能给出null PC.B都给出了假的信息.
D给出了正确的信息.显然根据上述情形,没有办法达成safety,所以得继续向A获得请求,最后获得了A+D个f+1个PC,才能继续.
这里很好地体现了quorum*的现象.

显然根据liveness的要求,最多只有能n个failure.但是最坏的情况西,收集到了所有的A+B+C+D的请求...
这没有打破liveness的要求.因为之前的C节点得通过某种方式sync,重新获得数据.而他们的sync的方式是,向A+B+D获取有效数据,而不是向C内的其他成员获取无效数据.
为了获取有效数据,quorum*会偏移成包含A+D,C的票认为是废票.

为了突破这种情况

所以Mac的情况下,需要N=4f+1,Q=3f+1,InterSection = 2f+1.这样3f+1个选票内,至少有2f+1个有效票,至多f个废票.
2f+1个有效票内,f+1 outvote了 f,可以传递.

这种情况从本质上说,是Mac只能认票数,没有办法校验真伪,只能靠数量取胜.


2. 如果采用signature,需要f+1个prepare certificate(1 PC = collection o{1 signed pre-prepare + 2f signed prepares).
这种情况下可以继续共识(liveness下面再说).而且因为有至少1个non-faulty(这里的faulty包括上述2种),
所以原有的committed request(注意是committed,不是committed-local)可以被传递到下一个quorum(意味着保证了view change的safety)
safety得以保证.

A+B+D形成了committed,即A和D都收到了来自A+B+D的pre/prepare请求,进入prepared.虽然最后的commit阶段不一定成功,即committed-local不一定成功.
但是如果进入了committed,那么PC就会被保存下来,里面signed pre/prepare的签名都有保留.
现在发生view change,view change收到的是B+C+D的请求.B给出了假的PC,C显然没有PC,D给出了真是的PC.根据之前关于signature的描述,
在确认时,只需要f+1个结果就可以了.D成功地传达了信息,新的view change不会误伤到之前的n.
这里的B是node failure.D给出了null的PC,所以是正常的,不属于faulty.

# 总结
如果是Mac,那么null票将不能被认为是可靠的,quorum就必须扩大.这个想法和fast paxos很类似. N=4f+1,Q=3f+1
如果是Sig,non-faulty节点null票也是真实地反应了当时的情形.

不在深入Mac的情形....现在Signature是标配.