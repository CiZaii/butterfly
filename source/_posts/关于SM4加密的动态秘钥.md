
---
title: 关于SM4加密的动态秘钥
date: 2022/9/1 21:12
swiper: true # 是否将改文章放入轮播图中
swiperImg: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/logo.png' # 该文章在轮播图中的图片，可以是本地目录下图片也可以是http://xxx图片
cover: https://zangzang.oss-cn-beijing.aliyuncs.com/img/logo.png # 该文章图片，可以是本地目录下图片也可以是http://xxx图片
categories: 技巧 # 分类
tags: [ 安全框架] #标签
top: false

---
## 前提

前段时间我听说有一款国人开发的安全框架[Sa-Token](https://sa-token.dev33.cn/doc/index.html)，打算去支持一下于是看了一下官方文档就准备把自己之前的项目重构一下，我自己的项目中权限框架用的springsecurity做用户密码加密的时候直接用自带的就行，但是换成Sa-Token之后据我现在所知里边没有可用的对密码进行加密的工具类，所以我选择的国密SM4加密。

但是有一个问题因为我们的数据库可能随时挂掉，这就有一个问题了，在SM4加密的时候，我们的秘钥是动态的，每次产生的秘钥都是不一样的，他保存在类似于ThreadLocal这种上下文中，我们下次用的时候会从上下文中取出来，但是一旦我们的服务器挂掉之后，我们就会丢失秘钥，导致用户下次登录的时候没有办法解密从而无法登录。这个时候我是这样解决这个问题的，往下看

## 解决

![image-20220706165930788](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220706165930788.png)

我们先获得一个秘钥，通过这个秘钥对密码进行加密，然后把这个密码和对密码加密的秘钥同时存放在数据库中，这样我们就不会因为服务器挂掉而导致用户无法登录了，

![image-20220706172841345](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220706172841345.png)

> 这样在我们登录的时候只需要把当前的密码通过秘钥解密之后，与用户输入的密码进行比对就可以了
