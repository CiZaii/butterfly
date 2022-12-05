---
title: MVCC底层原理
date: '2022/8/26 18:22'
swiper: true
swiperImg: >-
  https://zangzang.oss-cn-beijing.aliyuncs.com/img/eeb95678d9e0d7ad4c7bd6aa6e503216.jpg
cover: >-
  https://zangzang.oss-cn-beijing.aliyuncs.com/img/eeb95678d9e0d7ad4c7bd6aa6e503216.jpg
categories: MySQL
tags:
  - 锁
abbrlink: 11dc0777
---

## MVCC底层原理

假设现在有一个index表，只有一条数据

![dedf4345b7282d7a872b2bd1a11fa3c9](https://zangzang.oss-cn-beijing.aliyuncs.com/img/dedf4345b7282d7a872b2bd1a11fa3c9.png)

此时他是有两个隐藏列的，一个是trxid(事务id)，一个是roll pointer(回滚指针)

此时新建三个会话，每个会话创建一个事务，我这里创建了

![faca570a9ce125e0d6fd35096271340b](https://zangzang.oss-cn-beijing.aliyuncs.com/img/faca570a9ce125e0d6fd35096271340b.png)

这三个依次进行一次更新操作，因为只有更新操作的时候才会生成事务id

![9deada5bbd1abd1e869fcf02f352380a](https://zangzang.oss-cn-beijing.aliyuncs.com/img/9deada5bbd1abd1e869fcf02f352380a.png)

所以要先去操作别的表一下以便生成事务id

我们在进行第三个事务的时候更新了一条数据在数据库底层会帮我们做这样一件事情

创建一条新数据然后将我们的旧数据放到回滚日志里边，并且将回滚指针指向它

![a9b16bbf2819b3caa58f540ab6548d85](https://zangzang.oss-cn-beijing.aliyuncs.com/img/a9b16bbf2819b3caa58f540ab6548d85.png)

此时我们进行一个查询会生成一个快照，他由指向查询时所有未提交事务id数组，和已创建事务id组成，查询数据需要跟read-view作对比从而得到快照结果

![88dbc40a19db9f4ca587ed5c88333502](https://zangzang.oss-cn-beijing.aliyuncs.com/img/88dbc40a19db9f4ca587ed5c88333502.png)

很明显此时查询结果为臧臧，这里就不做讲解

此时事务id为100的一次进行了3条更新操作

还会生成版本链

![a7858897faa9d5b53cc2e07411e43749](https://zangzang.oss-cn-beijing.aliyuncs.com/img/a7858897faa9d5b53cc2e07411e43749.png)

此时橙色的为最新数据，而黄色的为在版本日志里的数据

下面进行一个新的查询

![3a352c2543f4b65816a9dc0aa58a223e](https://zangzang.oss-cn-beijing.aliyuncs.com/img/3a352c2543f4b65816a9dc0aa58a223e.png)

因为我们研究的是可重复读的情况所以会沿用上一次生成的快照

此时查询出来的数据还会是臧臧，那么这是为什么呢，分析一下

先说一些readview比对规则

>当执行查询sql时会生成一致性视图read-view，它由执行查询时所有未提交事务id数组(数组里最小的id为min_id)和已创建的最大事务id(max_id) 组成，查询的数据结构需要跟read-view做对比从而得到快照结果
>
>版本链对比规则：
>
>1. 如果在绿色部分(trx_id<min_id),表示这个版本是已经提交的事务生成的，这个数据是可见的；
>2. 如果落在红色部分(trx_id>max_id),表示这个版本是有将来启动的事务生成的，是肯定不可见的
>3. 如果落在黄色部分(min_id<trx_id<max_id)那就包括两个情况
> 4. 若row的trx_id在数组中，表示这个版本是由还没提交的事务生成的，不可见，当然了，自己肯定是可见的
> 5. 若row的trx_id不在数组中说明，这个版本是已经提交了的事务生成了的，可见
>
>对于删除的情况可以认为是update的特殊情况，会将版本链上最新的数据复制一份，然后将trx_id修改成删除操作的trx_id，同时在该条记录的头信息(record header)里的deleted_flag标记位写上true，来表示当前的记录已经被删除了，在查询时按照上边的规则查到对应记录如果delete_flag标记位true，意味当前记录已被删除，则不返回数据

![30c4cf79c88b157b2bd97608a34d9f1b](https://zangzang.oss-cn-beijing.aliyuncs.com/img/30c4cf79c88b157b2bd97608a34d9f1b.png)

1. 因为此时的readview是第一次生成的readview所以会进行比对事务id此时的事务id为100进行对比等于min_id(数组中最小事务id)，所以不可见，根据回滚指针去undolog里边找连续两个都是100，当到事务id为300的时候，符合第三个条件，然后进行判断不在未提交事务的数组中，所以可见，我们查到的数据为姓名为臧臧的数据，

2. 我们现在按顺序进行以下操作![9c98cba895b6eaf36cf50c31d7f35641](https://zangzang.oss-cn-beijing.aliyuncs.com/img/9c98cba895b6eaf36cf50c31d7f35641.png)

   此时再次进行查询，因为是可重复读状态依旧会使用上次生成的快照，进行对比此时版本链是这样的

![f1bb3f5f8618c8ee0078b0b8d5d136a9](https://zangzang.oss-cn-beijing.aliyuncs.com/img/f1bb3f5f8618c8ee0078b0b8d5d136a9.png)

3. 进行对比200在范围内，但是在未提交数组中所以不可见，以此类推，查出来的还是name=臧臧，

4. 此时我们进行这样的操作新建一个会话，然后进行同样的查询操作，这个时候生成的read-view是[200]，300.因为我们此时生成快照的时间在100和300都提交的时刻所以read-view是这样的继续对比200在范围内，并且在未提交数组中不可见，上边一样，一直到100的时候，小于min_id属于已提交的所有能读出name=朵橙