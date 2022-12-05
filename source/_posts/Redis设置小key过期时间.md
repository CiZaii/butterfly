---
title: Redis设置小key过期时间
date: '2022/8/28 22:04'
swiper: true
swiperImg: >-
  https://zangzang.oss-cn-beijing.aliyuncs.com/img/b081857be95d8ae4c3cf8b1258109f1a.jpg
cover: >-
  https://zangzang.oss-cn-beijing.aliyuncs.com/img/b081857be95d8ae4c3cf8b1258109f1a.jpg
categories: 技巧
tags:
  - Redis
abbrlink: 8486078e
---

## 场景

首先是一个这样的业务场景，我们要做一个注册的功能，我们会通过用户输入的邮箱进行发送一个验证码，并且验证码有效期是3分钟，但是我们要去使用redis保存验证码，但是又不想用string去做。用hash去怎么实现呢

## 做法

### 保存

在我们redis中可以通过hash做，但是呢redis只提供了hash类型的大key的过期时间，这个时候问题就来了，我就想使用一个大key，然后每个邮箱的地址小key，验证码为value，这个时候我们只需要在验证码之后拼接一个时间

![image-20220706161815789](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220706161815789.png)

此时就是获取我们的当前时间然后偏移三分钟转换为字符串之后拼接到验证码之后

### 验证

![image-20220706163004834](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220706163004834.png)

我们这样的话取出来的时候就可以先把我们保存的过期时间取出来，然后获得当前时间进行比较如果当前时间在过期时间之后就代表我们的验证码已经过期了，如果没有的话就说明还没有过期，进行下边的思路

