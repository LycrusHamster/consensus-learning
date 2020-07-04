# tendermint

## round leader

tendermint是pbft的round leader版本,更加适合POS,可以让出块节点的工作量更加"公平",出块概率也更加"公平",虽然这并没有改善最终的结果.

pbft有着复杂的view change, tendermint的leader是prophetic的,也就是说,leader的轮流是事先决定好的.
可以是round robin,也可是是一个确定性的"伪随机"算法,也可以是........反正是可以预见的就可以了..

不像view change,新的Primary要收集2f+1个view change的请求,从而来确保n和request的正确性.
tendermint,无时无刻不在view change.但是如果向view change一样,带上足够的信息,这样效率太低.
当view change的primary提出的new view change请求,backups不接受的话,就会触发超时,为了liveness,进而继续view change.
tendermint的想法也是一样, 在new round的时候,不带上足够有用的信息,但是如果新的round 的leader propose的request不符合我的预期,不能保证safety,
我就直接不vote,(当然,为了加快liveness,vote nil),使得继续换leader,从而达成liveness.

## 差别

在pbft中,一个节点收到了1*pre-prepare+2f*prepare,就认为自己进入了**prepared**状态,这些被各自backup signed
pre-prepare和prepare,就是该backup的prepared的prepare-certificate.然后commit.

同理,在tendermint中,一个节点收到了proposal(视为等同prevote)+prevote,达到了2f+1,就进入locked(其实就是prepared in pbft)状态.
这2f+1个proposal/prevote,就形成了polka(其实就是prepare-certificate in pbft),然后才会针对该request发出precommit.

## lock,unlock,polka

我们知道,polka其实就是prepare-certificate.但是如何在round之间保持不出错呢?pbft的view change需要primary不出错,然后backups认同.
在tendermint里,primary没办法接受类似与view change的信息,那就只能依靠backups单纯的拒绝一些新的请求,保证safety.

## lock

根据之前提及的,当一个节点收到了2f+1个prevote的时候,他就会lock在r上,lock的内容是prevote里面包含的request(当然也可以叫proposal)

### prevote-the-lock

如果一个节点被lock在r上,request是A,那么在之后的rounds,只要没有unlock,他都必须propose或者prevote他锁lock的request,也就是A.

这因为该节点获得了polka,进入了prepared状态,如果他在之后round又投了其他request,B,那么可能使得一部分的节点在r上达成了共识A,一部分节点,在r+1上达成了共识B.
深入原因,是该节点在不同的leader下,给出了不同的request.不同的leader代表了不同的quorum.在不同的quorum之间回报不同的信息,是faulty节点的行为.
会破坏quorum本身的作用(byz节点除外).

举例,A是f个non-faulty,B是f个faulty,C是f个non-faulty,D是一个non-faulty.

在r轮,A+B+D都收到了2f+1个prevote进入了polka,然后A收到了来自A+B+D的precommit,满足2f+1,commit.
而D没有收到A的precommit,只能继续lock.
C认为网络分区了..

在r+1轮,B提出新的request B,并且得到C的响应.这时如果D没有坚守request A,而是也投了request B,那么C就可以接受
来自B+C+D,总共2f+1个prevote,进入polka,随后又收到2f+1个precommit.commit了request B...safety破坏.

这里的D坚持lock,坚持prepose/prevote上一轮lock住的request,就是体现出了2个quorum交集内f+1个节点中的那一个诚实节点的作用.

从原理上解释: 一个节点polka了,就是认为其他节点可能也想prevote这个request.但是通信是异步的,其他节点可能没有polka(自己没收到precommit nil),也可能polka了(自己还是没收到precommit).
不管如何,其他的节点存在着最后commit这个request的可能(其他节点polka了,并且收到了自己的precommit).那么就必须坚持这个request.

### unlock-on-polka

当前在r+1轮,如果一个节点在precommit阶段,收到了来自其他precommit(还有附带的签名已证实precommit背后的polka的2f+1个prevote的合法性),那么就把可能在r轮上的lock给unlock.

这也很好理解,因为节点可能被lock在了r轮一个polka上,而r轮可能没有其他节点收到polka,没有lock,转而在r+1轮可能达成了新的共识,或者lock在了新的request上.
自己不能一直那个锁到死.那么在r+2轮,自己可以参与到新的request上,并且认为r轮的request肯定不会达成.

这是因为如果r轮有committed(in pbft)的情况下,也就是说有2f+1个节点都进入了polka,至少有1个non-faulty节点会继续lock,从而使得新的request无法凑够2f+1张票(最多2f张).

## safety

如果一个request被committed-local,那么他一定是进入了committed状态,也就是有2f+1个节点进入了polka.有2f+1个节点lock在了这个request上.

在新一轮r下,至少有1个non-faulty节点lock在该request下,所以新的request最多2f个,无法在non-fault节点中达成polka.

## liveness

2f+1个non-faulty节点不可能总是lock不同的request上,因为可以unlock.这样最后至少有2f+1个节点可以达成共识,中间可能需要跳过多个faulty leader.

## 举例

有1,2,3,4,5,6,7,总共7个节点,f=2 N=7, Q=5.其中6和7是non-faulty节点.

在第1轮,6是leader,1被2367lock在了A上,但是23并没有被lock(可能是没有收到1的请求),45也是没有收到足够的prevote.

在第2轮,7是leader,23分别被4567lock在lB上,但是45并没有被lock,原因同上.1没有收到23的precommit,没有unlock.

在第3轮,6可能又成为了leader(只要最终leader人人有份就可以),4567无法满足2f+1,而1,23分别只和4567发生了通讯,都没有unlock.

......

但是最终,会有non-faulty节点成为leader,网络会通畅.
然后只要有f+1个non-faulty节点成功收到了其他剩下f个non-faulty的precommit,进而unlock,使得unlock的节点有f+1个,
然后和f个non-faulty的节点,组成2f+1个达成共识的节点,liveness达成.....虽然这很艰辛..

## liveness的...推演?
如果有f+1个节点lock在request A上,那么他必将达成.因为剩下的2f不足以unlock.

同一个round不可能lock2个不同的request.quorum intersection里至少包含了一个non-faulty节点,所以不可能一个round lock不同的request.

如果有f+1个节点被lock在了不同的round的不同的request上,那么剩下的f个节点只可能被lock在这些被lock的request里,或者没有被lock.
这是因为f个faulty节点如何欺骗,再加上f个non-faulty且没有lock的节点的网络都是单向通信,也只能凑够2f个.
如果要lock,必须满足2f+1,必然有1个节点来自那f+1个已经被lock的及节点.

如果2f+1个non-faulty被分别lock在了不同的request上,且每个request都达不到f+1(达到f+1将最终被共识,原因见前),那么只要网络最终通畅,
round在后的lock会被unlock,然后被lock再了其他的request上.最终某一个request会被凑到f+1个lock.
