---
title: SpringSecurity认证流程
date: '2022/9/14 21:42'
swiper: true
swiperImg: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-8o2v82.jpg'
cover: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-8o2v82.jpg'
categories: 源码
tags:
  - 安全框架
sticky: 1
abbrlink: 7dde2051
---
## SpringSecurity认证流程

最近几天比较闲把SpringSecurity的源码看了一下，这里先讲一下认证的流程，debug级别的讲解

>注意此篇文章没有角色和鉴权，后续看完源码会发，还有如果不知道AuthenticationManager和ProviderManager还有AuthenticationProvider，可能有些地方会看不懂，后续会单独出一篇文章进行讲解

首先准备，创建一个springboot项目，然后引入必要的一些依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
        <version>2.6.4</version>
    </dependency>
    <dependency>
        <groupId>io.github.vampireachao</groupId>
        <artifactId>stream-core</artifactId>
        <version>1.1.8</version>
    </dependency>
    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-swagger2</artifactId>
        <version>2.9.2</version>
    </dependency>
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-generator</artifactId>
        <version>3.5.2</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.velocity</groupId>
        <artifactId>velocity-engine-core</artifactId>
        <version>2.3</version>
    </dependency>
    <!--freemarker模板-->
    <dependency>
        <groupId>org.freemarker</groupId>
        <artifactId>freemarker</artifactId>
    </dependency>
    <dependency>
        <groupId>io.github.vampireachao</groupId>
        <artifactId>stream-plugin-mybatis-plus</artifactId>
        <version>1.1.8</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

数据库数据，因为我懒的创建一个新表了，所以这里就使用了之前的表结构

```sql
-- ----------------------------
-- Table structure for tb_user_role
-- ----------------------------
DROP TABLE IF EXISTS `tb_user_role`;
CREATE TABLE `tb_user_role`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `user_id` int NULL DEFAULT NULL COMMENT '用户id',
  `role_id` int NULL DEFAULT NULL COMMENT '角色id',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 612 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Records of tb_user_role
-- ----------------------------
INSERT INTO `tb_user_role` VALUES (578, 2, 2);
INSERT INTO `tb_user_role` VALUES (579, 3, 2);
INSERT INTO `tb_user_role` VALUES (580, 4, 2);
INSERT INTO `tb_user_role` VALUES (581, 5, 2);
INSERT INTO `tb_user_role` VALUES (582, 1, 1);
INSERT INTO `tb_user_role` VALUES (583, 1, 2);
INSERT INTO `tb_user_role` VALUES (584, 6, 2);
INSERT INTO `tb_user_role` VALUES (585, 7, 2);
INSERT INTO `tb_user_role` VALUES (586, 8, 2);
INSERT INTO `tb_user_role` VALUES (587, 9, 2);
INSERT INTO `tb_user_role` VALUES (589, 11, 2);
INSERT INTO `tb_user_role` VALUES (590, 12, 2);
INSERT INTO `tb_user_role` VALUES (591, 13, 2);
INSERT INTO `tb_user_role` VALUES (592, 14, 2);
INSERT INTO `tb_user_role` VALUES (593, 15, 2);
INSERT INTO `tb_user_role` VALUES (595, 16, 2);
INSERT INTO `tb_user_role` VALUES (597, 17, 2);
INSERT INTO `tb_user_role` VALUES (598, 18, 2);
INSERT INTO `tb_user_role` VALUES (599, 19, 2);
INSERT INTO `tb_user_role` VALUES (600, 10, 2);
INSERT INTO `tb_user_role` VALUES (601, 10, 3);
INSERT INTO `tb_user_role` VALUES (602, 20, 2);
INSERT INTO `tb_user_role` VALUES (603, 21, 2);
INSERT INTO `tb_user_role` VALUES (604, 22, 2);
INSERT INTO `tb_user_role` VALUES (605, 23, 2);
INSERT INTO `tb_user_role` VALUES (606, 24, 2);
INSERT INTO `tb_user_role` VALUES (611, 33, 1);

-- ----------------------------
-- Table structure for tb_user_auth
-- ----------------------------
DROP TABLE IF EXISTS `tb_user_auth`;
CREATE TABLE `tb_user_auth`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `user_info_id` int NULL DEFAULT NULL COMMENT '用户信息id',
  `username` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '用户名',
  `password` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '密码',
  `secret_key` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '秘钥',
  `login_type` tinyint(1) NOT NULL DEFAULT 1 COMMENT '登录类型',
  `ip_address` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '用户登录ip',
  `ip_source` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT 'ip来源',
  `create_time` datetime NOT NULL COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT NULL COMMENT '更新时间',
  `last_login_time` datetime NULL DEFAULT NULL COMMENT '上次登录时间',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `username`(`username` ASC) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 30 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Table structure for tb_role_resource
-- ----------------------------
DROP TABLE IF EXISTS `tb_role_resource`;
CREATE TABLE `tb_role_resource`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `role_id` int NULL DEFAULT NULL COMMENT '角色id',
  `resource_id` int NULL DEFAULT NULL COMMENT '权限id',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 4373 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Records of tb_role_resource
-- ----------------------------
INSERT INTO `tb_role_resource` VALUES (4011, 2, 254);
INSERT INTO `tb_role_resource` VALUES (4012, 2, 267);
INSERT INTO `tb_role_resource` VALUES (4013, 2, 269);
INSERT INTO `tb_role_resource` VALUES (4014, 2, 270);
INSERT INTO `tb_role_resource` VALUES (4015, 2, 257);
INSERT INTO `tb_role_resource` VALUES (4016, 2, 258);
INSERT INTO `tb_role_resource` VALUES (4164, 3, 192);
INSERT INTO `tb_role_resource` VALUES (4165, 3, 195);
INSERT INTO `tb_role_resource` VALUES (4166, 3, 183);
INSERT INTO `tb_role_resource` VALUES (4167, 3, 246);
INSERT INTO `tb_role_resource` VALUES (4168, 3, 199);
INSERT INTO `tb_role_resource` VALUES (4169, 3, 185);
INSERT INTO `tb_role_resource` VALUES (4170, 3, 191);
INSERT INTO `tb_role_resource` VALUES (4171, 3, 254);
INSERT INTO `tb_role_resource` VALUES (4172, 3, 208);
INSERT INTO `tb_role_resource` VALUES (4173, 3, 234);
INSERT INTO `tb_role_resource` VALUES (4174, 3, 237);
INSERT INTO `tb_role_resource` VALUES (4175, 3, 213);
INSERT INTO `tb_role_resource` VALUES (4176, 3, 241);
INSERT INTO `tb_role_resource` VALUES (4177, 3, 239);
INSERT INTO `tb_role_resource` VALUES (4178, 3, 276);
INSERT INTO `tb_role_resource` VALUES (4179, 3, 205);
INSERT INTO `tb_role_resource` VALUES (4180, 3, 218);
INSERT INTO `tb_role_resource` VALUES (4181, 3, 223);
INSERT INTO `tb_role_resource` VALUES (4182, 3, 202);
INSERT INTO `tb_role_resource` VALUES (4183, 3, 230);
INSERT INTO `tb_role_resource` VALUES (4184, 3, 238);
INSERT INTO `tb_role_resource` VALUES (4185, 3, 232);
INSERT INTO `tb_role_resource` VALUES (4186, 3, 243);
INSERT INTO `tb_role_resource` VALUES (4187, 3, 196);
INSERT INTO `tb_role_resource` VALUES (4188, 3, 257);
INSERT INTO `tb_role_resource` VALUES (4189, 3, 258);
INSERT INTO `tb_role_resource` VALUES (4190, 3, 225);
INSERT INTO `tb_role_resource` VALUES (4191, 3, 231);
INSERT INTO `tb_role_resource` VALUES (4192, 3, 210);
INSERT INTO `tb_role_resource` VALUES (4282, 1, 165);
INSERT INTO `tb_role_resource` VALUES (4283, 1, 192);
INSERT INTO `tb_role_resource` VALUES (4284, 1, 193);
INSERT INTO `tb_role_resource` VALUES (4285, 1, 194);
INSERT INTO `tb_role_resource` VALUES (4286, 1, 195);
INSERT INTO `tb_role_resource` VALUES (4287, 1, 166);
INSERT INTO `tb_role_resource` VALUES (4288, 1, 183);
INSERT INTO `tb_role_resource` VALUES (4289, 1, 184);
INSERT INTO `tb_role_resource` VALUES (4290, 1, 246);
INSERT INTO `tb_role_resource` VALUES (4291, 1, 247);
INSERT INTO `tb_role_resource` VALUES (4292, 1, 167);
INSERT INTO `tb_role_resource` VALUES (4293, 1, 199);
INSERT INTO `tb_role_resource` VALUES (4294, 1, 200);
INSERT INTO `tb_role_resource` VALUES (4295, 1, 201);
INSERT INTO `tb_role_resource` VALUES (4296, 1, 168);
INSERT INTO `tb_role_resource` VALUES (4297, 1, 185);
INSERT INTO `tb_role_resource` VALUES (4298, 1, 186);
INSERT INTO `tb_role_resource` VALUES (4299, 1, 187);
INSERT INTO `tb_role_resource` VALUES (4300, 1, 188);
INSERT INTO `tb_role_resource` VALUES (4301, 1, 189);
INSERT INTO `tb_role_resource` VALUES (4302, 1, 190);
INSERT INTO `tb_role_resource` VALUES (4303, 1, 191);
INSERT INTO `tb_role_resource` VALUES (4304, 1, 254);
INSERT INTO `tb_role_resource` VALUES (4305, 1, 169);
INSERT INTO `tb_role_resource` VALUES (4306, 1, 208);
INSERT INTO `tb_role_resource` VALUES (4307, 1, 209);
INSERT INTO `tb_role_resource` VALUES (4308, 1, 170);
INSERT INTO `tb_role_resource` VALUES (4309, 1, 234);
INSERT INTO `tb_role_resource` VALUES (4310, 1, 235);
INSERT INTO `tb_role_resource` VALUES (4311, 1, 236);
INSERT INTO `tb_role_resource` VALUES (4312, 1, 237);
INSERT INTO `tb_role_resource` VALUES (4313, 1, 171);
INSERT INTO `tb_role_resource` VALUES (4314, 1, 213);
INSERT INTO `tb_role_resource` VALUES (4315, 1, 214);
INSERT INTO `tb_role_resource` VALUES (4316, 1, 215);
INSERT INTO `tb_role_resource` VALUES (4317, 1, 216);
INSERT INTO `tb_role_resource` VALUES (4318, 1, 217);
INSERT INTO `tb_role_resource` VALUES (4319, 1, 224);
INSERT INTO `tb_role_resource` VALUES (4320, 1, 172);
INSERT INTO `tb_role_resource` VALUES (4321, 1, 240);
INSERT INTO `tb_role_resource` VALUES (4322, 1, 241);
INSERT INTO `tb_role_resource` VALUES (4323, 1, 244);
INSERT INTO `tb_role_resource` VALUES (4324, 1, 245);
INSERT INTO `tb_role_resource` VALUES (4325, 1, 267);
INSERT INTO `tb_role_resource` VALUES (4326, 1, 269);
INSERT INTO `tb_role_resource` VALUES (4327, 1, 270);
INSERT INTO `tb_role_resource` VALUES (4328, 1, 173);
INSERT INTO `tb_role_resource` VALUES (4329, 1, 239);
INSERT INTO `tb_role_resource` VALUES (4330, 1, 242);
INSERT INTO `tb_role_resource` VALUES (4331, 1, 276);
INSERT INTO `tb_role_resource` VALUES (4332, 1, 174);
INSERT INTO `tb_role_resource` VALUES (4333, 1, 205);
INSERT INTO `tb_role_resource` VALUES (4334, 1, 206);
INSERT INTO `tb_role_resource` VALUES (4335, 1, 207);
INSERT INTO `tb_role_resource` VALUES (4336, 1, 175);
INSERT INTO `tb_role_resource` VALUES (4337, 1, 218);
INSERT INTO `tb_role_resource` VALUES (4338, 1, 219);
INSERT INTO `tb_role_resource` VALUES (4339, 1, 220);
INSERT INTO `tb_role_resource` VALUES (4340, 1, 221);
INSERT INTO `tb_role_resource` VALUES (4341, 1, 222);
INSERT INTO `tb_role_resource` VALUES (4342, 1, 223);
INSERT INTO `tb_role_resource` VALUES (4343, 1, 176);
INSERT INTO `tb_role_resource` VALUES (4344, 1, 202);
INSERT INTO `tb_role_resource` VALUES (4345, 1, 203);
INSERT INTO `tb_role_resource` VALUES (4346, 1, 204);
INSERT INTO `tb_role_resource` VALUES (4347, 1, 230);
INSERT INTO `tb_role_resource` VALUES (4348, 1, 238);
INSERT INTO `tb_role_resource` VALUES (4349, 1, 177);
INSERT INTO `tb_role_resource` VALUES (4350, 1, 229);
INSERT INTO `tb_role_resource` VALUES (4351, 1, 232);
INSERT INTO `tb_role_resource` VALUES (4352, 1, 233);
INSERT INTO `tb_role_resource` VALUES (4353, 1, 243);
INSERT INTO `tb_role_resource` VALUES (4354, 1, 178);
INSERT INTO `tb_role_resource` VALUES (4355, 1, 196);
INSERT INTO `tb_role_resource` VALUES (4356, 1, 197);
INSERT INTO `tb_role_resource` VALUES (4357, 1, 198);
INSERT INTO `tb_role_resource` VALUES (4358, 1, 257);
INSERT INTO `tb_role_resource` VALUES (4359, 1, 258);
INSERT INTO `tb_role_resource` VALUES (4360, 1, 179);
INSERT INTO `tb_role_resource` VALUES (4361, 1, 225);
INSERT INTO `tb_role_resource` VALUES (4362, 1, 226);
INSERT INTO `tb_role_resource` VALUES (4363, 1, 227);
INSERT INTO `tb_role_resource` VALUES (4364, 1, 228);
INSERT INTO `tb_role_resource` VALUES (4365, 1, 231);
INSERT INTO `tb_role_resource` VALUES (4366, 1, 180);
INSERT INTO `tb_role_resource` VALUES (4367, 1, 210);
INSERT INTO `tb_role_resource` VALUES (4368, 1, 211);
INSERT INTO `tb_role_resource` VALUES (4369, 1, 212);
INSERT INTO `tb_role_resource` VALUES (4370, 1, 277);

-- ----------------------------
-- Table structure for tb_role
-- ----------------------------
DROP TABLE IF EXISTS `tb_role`;
CREATE TABLE `tb_role`  (
  `id` int NOT NULL AUTO_INCREMENT COMMENT '主键id',
  `role_name` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '角色名',
  `role_label` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '角色描述',
  `is_disable` tinyint(1) NOT NULL DEFAULT 0 COMMENT '是否禁用  0否 1是',
  `create_time` datetime NOT NULL COMMENT '创建时间',
  `update_time` datetime NULL DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 5 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Records of tb_role
-- ----------------------------
INSERT INTO `tb_role` VALUES (1, '管理员', 'admin', 0, '2021-03-22 14:10:21', '2022-01-09 08:10:20');
INSERT INTO `tb_role` VALUES (2, '用户', 'user', 0, '2021-03-22 14:25:25', '2021-08-11 21:12:21');
INSERT INTO `tb_role` VALUES (3, '测试', 'test', 0, '2021-03-22 14:42:23', '2021-08-24 11:25:39');
INSERT INTO `tb_role` VALUES (4, 'test', 'test', 0, '2022-07-22 15:41:14', '2022-07-22 15:41:17');

```

然后使用mp自动生成一下entity，controller，service，mapper层，下面我就从源码的角度一步一步的来讲解一下整个认证的流程，注意这里需要有一定的springsecurity基础。

## 配置一下基本环境

首先我写了一些基本的配置然后从头走一遍认证流程

```java
package com.zang.securitysourcecodelearning.config.security;

import com.zang.securitysourcecodelearning.entity.UserAuth;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.password.NoOpPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

/**
 * @Author: ZVerify
 * @Description: TODO SpringSecurity配置类
 * @DateTime: 2022/9/14 17:07
 **/
@Configuration
public class ZVerifySpringSecurityConfig extends WebSecurityConfigurerAdapter {



    @Autowired
    private ZVerifyUserDetailService zVerifyUserDetailService;

    @Bean
    ZVerifyFilter zVerifyFilter() throws Exception {
        ZVerifyFilter zVerifyFilter = new ZVerifyFilter();
        // 设置登录的路径
        zVerifyFilter.setFilterProcessesUrl("/doLogin");
        // 设置表单登录的key
        zVerifyFilter.setPasswordParameter("pwd");
        zVerifyFilter.setUsernameParameter("user");
        // 设置自定义的AuthenticationManager
        zVerifyFilter.setAuthenticationManager(authenticationManagerBean());
        // 登录成功之后的处理器
        zVerifyFilter.setAuthenticationSuccessHandler((re,resp,au) ->{
            UserAuth details = (UserAuth) au.getPrincipal();
            System.out.println(details.getUsername());
        });
        return zVerifyFilter;
    }

    @Override
    @Bean
    // 为了将我的AuthenticationManager放到容器中
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth){
        // 创建者创建一个AuthenticationManager，并且将我自定义的authenticationProvider放入其中，这就表明不适用默认的authenticationProvider
        auth.authenticationProvider(dao());
    }
    
    @Bean
    public AuthenticationProvider dao(){
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        // 自定义获取用户信息逻辑
        provider.setUserDetailsService(zVerifyUserDetailService);
        // 自定义密码加密
        provider.setPasswordEncoder(getNoOpPasswordEncoder());
        return provider;
    }
    
    
    @Bean
    public PasswordEncoder getNoOpPasswordEncoder(){
        return NoOpPasswordEncoder.getInstance();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().anyRequest().authenticated().and().csrf().disable().formLogin();
        // 这个东西就是使用我自己创建的filter去过滤器链中替换掉UsernamePasswordAuthenticationFilter这个过滤器
        http.addFilterAt(zVerifyFilter(), UsernamePasswordAuthenticationFilter.class);

    }
}

```

## 启动项目

在我们进行登录的时候因为我们已经对UsernamePasswordAuthenticationFilter这个过滤器进行了替代，所以会进入我自定义的ZVerifyFilter中

```java
public class ZVerifyFilter extends UsernamePasswordAuthenticationFilter {

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        // 判断请求方式是否是POST
        if (!request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
        }
        // 拿到表单信息
        String userName = request.getParameter(getUsernameParameter());
        String passWord = request.getParameter(getPasswordParameter()) ;

        // 封装成UsernamePasswordAuthenticationToken
        UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(userName,
                passWord);
        setDetails(request, authRequest);
        // 进行认证
        return this.getAuthenticationManager().authenticate(authRequest);
    }
}
```

最后一句话会拿到当前的AuthenticationManager接口然后调用其实现类的authenticate()方法，但是AuthenticationManager接口真正的实现类只有ProviderManager，所以此时也就是调用ProviderManager的authenticate()方法此时会进入for循环然后进行判断

![image-20220914200622864](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914200622864.png)

此时会退出当前的循环，因为是第一次进入所以size是1，退出本次循环之后就退出for循环了，进入下面的if判断

![image-20220914200818752](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914200818752.png)

此时

![image-20220914200854595](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914200854595.png)

这个parent其实还是一个AuthenticationManager，所以就又进入了这个方法，这次不同的是![image-20220914201051167](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914201051167.png)

### 获取完整用户信息

此时的AuthenticationProvider是我们自定义的DaoAuthenticationProvider

很明显这次if条件不成立了

![image-20220914201246815](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914201246815.png)

然后会进入DaoAuthenticationProvider的authenticate()，首先进入的是抽象类AbstractUserDetailsAuthenticationProvider的authenticate()方法![image-20220914201507925](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914201507925.png)

首先取出userName，然后判断缓存中存不存在，不存在的话进入下边的方法

![image-20220914202000598](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914202000598.png)

注意当前的类是DaoAuthenticationProvider，所以会进入DaoAuthenticationProvider的retrieveUser()方法，

![image-20220914202127093](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914202127093.png)

这里会看到获取用户信息的方法，所以我想要自定义的实现获取用户信息，只需要创建一个类然后实现UserDetailsService方法，并且设置到我的DaoAuthenticationProvider，这样就可以使用我们的自定义逻辑

![image-20220914204148284](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914204148284.png)

拿到用户信息此时UserDetails对象已经是我们从数据库中查出来的UserAuth对象了![image-20220914204316759](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914204316759.png)

### 用户信息校验

拿到user之后会进行一些验证

![image-20220914204414887](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914204414887.png)

现在看很明显

![image-20220914204445987](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914204445987.png)

两个进行用户信息校验的都是使用的默认的，此时有了上面的经验我们如果不想用他默认的我们就可以自己创建一个Checker实现类，然后实现UserDetailsChecker接口，并且设置到我们的DaoAuthenticationProvider中就可以了![image-20220914204753476](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914204753476.png)

### 密码加密(自定义)

所以这里校验信息就不去看了直接进入additionalAuthenticationChecks()方法，这个方法是用来进行密码判断的，同样道理，因为我在配置类中指定了密码加密的方式![image-20220914205343252](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914205343252.png)

所以这个方法调用的是我们名文密码中的matches()

![image-20220914205421915](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914205421915.png)

直接进行判断

结束之后验证通过，然后还是会进行一个校验，这两个校验一个是看是否被锁定，以及各种信息，后面这个是判断密码是否过期，因为我们可能会有指定场景，写一个定时任务，在一个月之后设置为密码过期啥的。

### 密码加密（默认）

因为我使用了自定义的密码加密，所以这里就带大家走一遍源码就不进行debug了

在DaoAuthenticationProvider中有这样一段话![image-20220914211905574](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914211905574.png)

这是一个工厂![image-20220914212000660](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914212000660.png)

会创建一个DelegatingPasswordEncoder将默认的bcrypt和map放入其中，很明显这是策略模式

![image-20220914212153792](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914212153792.png)

在这里会先去调用extractId，将我们数据库中的密码拿出来看看有没有{xxx}xxxx这种类型的然后解析，解析出来从map中拿，如果没有的话就抛出异常，有的话就使用其对应的加密进行密码验证

### 密码加密升级

最后这个![image-20220914205837267](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914205837267.png)

createSuccessAuthentication()方法也很重要![image-20220914210024831](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914210024831.png)

首先会判断我们有没有自己实现密码升级的服务，如果实现了，并且是默认的密码加密的话才会进行密码升级服务，这个也是我们自己可以去定义

![image-20220914210347579](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914210347579.png)

但是别忘记要同时满足加密方式也可以进行密码升级服务哦，缺一不可

### 认证成功发布事件

然后认证成功之后会发布一个事件也就是登录成功回调方法

![image-20220914210643807](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914210643807.png)

![image-20220914210656012](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220914210656012.png)

我们只需要在filter中设置一下就好，将其默认的覆盖掉就可以使用我们自己的了。

>此时整个用户认证就结束了，可以说只要把源码搞清楚定制性是非常高的了