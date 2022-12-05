---
title: SpringSecurity鉴权源码
date: '2022/9/16 19:04'
swiper: true
swiperImg: >-
  https://zangzang.oss-cn-beijing.aliyuncs.com/img/391c44cd9190562f87f20329d2eb371.jpg
cover: >-
  https://zangzang.oss-cn-beijing.aliyuncs.com/img/391c44cd9190562f87f20329d2eb371.jpg
categories: 源码
tags:
  - 安全框架
sticky: 1
abbrlink: b43e2d04
---

SpringSecurity鉴权源码

之前写了一篇SpringSecurity的认证，下面接着来说一下鉴权对源码，SpringSecurity有一个专门对过滤器来进行鉴权FilterSecurityInterceptor，他是专门来进行鉴权对，下面来根据源码一点点看一下。

这次由于测试我们先写一下基本对数据，基本跟认证时候一样，不过要改一些配置

![image-20220916151258174](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916151258174.png)

也就是我们对hello请求需要ADMIN这个角色才能通过访问，所以为了测试我把ADMIN角色都给每个用户

![image-20220916151547734](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916151547734.png)

下面我们登陆之后把断点打到FilterSecurityInterceptor中看一下整体对流程

登陆就略过了，然后我们请求/hello接口

![image-20220916152501997](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916152501997.png)

通过源码可以看到FilterSecurityInterceptor实现类Filter所以核心方法应该是doFilter()方法

![image-20220916152552710](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916152552710.png)

调用了invoke方法

![image-20220916152838721](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916152838721.png)

进来之后先判断一下isApplied()，这个方法其实就是看一下requert似不似等于null，并且request没有FILTER_APPLIED这个常量的Attribute，很明显返回false，所以就进入下一个if判断observeOncePerRequest这个属性默认为true所以我们会给上个if判断的那个FILTER_APPLIED设置一个true，接下来就是进入我们的beforeInvocation()方法，这个是super调用的，我们去父类AbstractSecurityInterceptor看一眼

![image-20220916153925832](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916153925832.png)

这个方法其实就很重要了

![image-20220916170128400](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916170128400.png)

调用本类的SecurityMetadataSource，也就是继承抽象类的FilterSecurityInterceptor的这个方法也就是

![image-20220916170948014](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916170948014.png)FilterInvocationSecurityMetadataSource类下的默认实现类DefaultFilterInvocationSecurityMetadataSource的getAttributes方法

![image-20220916171243212](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916171243212.png)

这个方法里边有个requestMap其实就是我们配置对什么接口需要什么权限，key是请求路径，value是所需权限然后挨个比对，找到我们当前对请求之后就返回他所需要对权限集合，

![image-20220916172041224](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916172041224.png)

很明显都不成立，进去下边的authenticateIfRequired();这个方法就是进入ProviderManager进行一下认证,然后出来之后就是进行鉴权attemptAuthorization，也就是AccessDecisionManager接口对默认实现AffirmativeBased类的decide方法，这个方法就是一个投票器，会轮训所有配置的AccessDecisionVoter并在任何AccessDecisionVoter投赞成票时授予访问权限。仅当有拒绝投票且没有赞成票时才拒绝访问。

![image-20220916173021612](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916173021612.png)

## 动态鉴权

这就是基本的鉴权了，现在问题来了，难到我们每次所有对接口都要去配置文件里边配置吗，很明显笨死了就，但是我们该如何去定制化对设置动态鉴权呢。前面的讲解应该很清楚了，关键是一个getAttributes()方法获得当前请求路径所需要对角色，一个是accessDecisionManager.decide()的投票，我们的数据一般都是存在数据库中进行动态的权限，这里我简单的说一下要怎么去配置动态权限

![image-20220916190131154](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916190131154.png)

我们去自定义两个类

![image-20220916190221563](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916190221563.png)

![image-20220916190252256](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220916190252256.png)

可以根据自己的逻辑自行修改
