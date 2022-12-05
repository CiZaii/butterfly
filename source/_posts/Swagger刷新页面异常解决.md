---
title: Swagger刷新页面异常解决
date: '2022/10/6 10:32'
swiper: true
swiperImg: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-yx76vg.jpg'
cover: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-yx76vg.jpg'
categories: 技巧
tags:
  - 异常记录
top: false
abbrlink: 10e6d135
---

直接引入maven依赖即可解决

```xml
            <dependency>
                <groupId>io.swagger</groupId>
                <artifactId>swagger-annotations</artifactId>
                <version>1.5.22</version>
            </dependency>
            <dependency>
                <groupId>io.swagger</groupId>
                <artifactId>swagger-models</artifactId>
                <version>1.5.22</version>
            </dependency>
```