---
title: Swagger刷新页面异常解决
date: 2022/10/6 10:32
swiper: true # 是否将改文章放入轮播图中
swiperImg: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-yx76vg.jpg' # 该文章在轮播图中的图片，可以是本地目录下图片也可以是http://xxx图片
cover: https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-yx76vg.jpg # 该文章图片，可以是本地目录下图片也可以是http://xxx图片
categories: 技巧 # 分类
tags: [异常记录 ] #标签
top: false
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