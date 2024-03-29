---
layout: post
title: "微信红包原理"
date: 2015-05-04 22:32
comments: true
categories: 架构{arch}
---
#####注：本文是对微信群@QCon高可用架构群中的热火讨论，再加上微信同学的一些介绍所总结而出的一篇红包原理的文章

相信大家都玩过微信红包，今天就跟大家分享一下微信红包的原理，仅供参考。
###发红包过程

{% img  center /images/wei-xin-hong-bao-yuan-li/fa-hong-bao.png %}

微信发红包的过程如上图所示，详细过程如下：
####步骤1：
请求通过前端负载均衡到达业务层；
####步骤2：
业务层将红包信息以红包ID(sendId)为key写入到Cache层，红包记录包括红包个数Count、红包额度Money;
####步骤3：
业务层将红步骤2中的红包信息同时也写入到db;

#####注：步骤2和步骤3可以同时进行，执行完成之后业务层就可以将红包推送给群里用户了

###点击进入红包过程
{% img  center /images/wei-xin-hong-bao-yuan-li/dian-ji-jin-ru-hong-bao.png %}

用户点击进入红包的处理过程如上图所示，详细过程如下：
####步骤1：
请求通过前端负载均衡到达业务层；
####步骤2：
业务层根据红包id访问cache；
####步骤3：
cache层将红包信息返回给业务层；
####步骤4：
业务层根据红包的剩余个数(Count)决定客户端的最终结果，如果Count>0，表示红包可折，提示用户可以进行折红包，否则提示用户红包已被抢完；

###拆红包过程
{% img  center /images/wei-xin-hong-bao-yuan-li/cai-hong-bao.png %}

用户拆红包的过程图如上图所示，详细过程如下：
####步骤1：
请求通过前端负载均衡到达业务层；
####步骤2：
业务层根据红包id访问cache；
####步骤3：
cache层将红包信息返回给业务层，如果红包剩余个数（Count）等于0，说明红包已经被抢完，执行步骤7，直接返回提示用户红包已经被抢完，否则计算红包金额，计算的方式如下：生成从0.01元到剩余平均值2倍之间的一个随机数作为金额（当红包剩余个数为2时，有可能存在超过剩余金额的情况，但只要保证给最后一个红包剩余不少于0.01元的额度即可），最后一个红包直接取剩余额度即可。
步骤4：生成红包额度之后，通过Cache提供的 CAS操作更新红包信息(红包个数减1，剩余总额减去当成红包的额度)。
####步骤5：
更新Cache之后，更新DB，记录当前被抢了的红包数和金额；
####步骤6：
如果更新成功返回给业务层，更新失败，业务层可重试；
####步骤7：
红包剩余个数为0，则返回，提示用户红包已经被抢完，或者更新DB之后，返回用户抢到红包的额度；
#####注：有些同学讨论之后总结CAS操作在数据库层去执行，我觉得应该在Cache层更合适，因为Cache容易实现，而且Cache可以专门去做CAS操作，因此个人觉得Cache比较合适。

###查询红包过程
查询红包直接读Cache即可，如果红包未抢完，则返回已抢到红包的用户列表，否则返回用户列表的同时，还计算是最佳手气用户，最佳手气的影响因素有两个，一个是红包大小，一个是时间，红包最大者手气最佳，如果存在多个最大者，那就拼时间，最先抢到最大红包的用户手气最佳。

#####注：与红包相关的数据量不少，DB存储和Cache都做了Sharding，同时，红包是跟钱相关的，那么存储层的高可靠和高可用性是必须要考虑的一个问题，而且是一个比较大的话题，这里就不做介绍了，有兴趣的同学可以自己思考一下。