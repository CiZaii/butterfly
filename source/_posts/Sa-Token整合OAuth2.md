
---
title: Sa-Token整合OAuth2
date: 2022/9/19 19:28
swiper: false # 是否将改文章放入轮播图中
swiperImg: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-v97wl5.png' # 该文章在轮播图中的图片，可以是本地目录下图片也可以是http://xxx图片
cover: https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-v97wl5.png # 该文章图片，可以是本地目录下图片也可以是http://xxx图片
categories: 第三方登录 # 分类
tags: [OAuth2] #标签
top: false

---

# Sa-Token整合OAuth2

开源地址 https://gitee.com/ZVerify/zverify-blog

## 为什么要整合OAuth2

有些时候我们自己写的网站注册过于繁琐需要每个用户花费时间去注册，所以我们可以添加各种第三方登录，方便网站的用户去登录。

## 写之前思考一下

我们先想一下我们在进行第三方登录的时候是怎样的一个步骤，首先第三方登录都需要遵守OAuth2的流程，这里我使用了授权码模式，对于其他三种授权模式请参考网络文章，因为我使用了授权码模式所以他的整体流程都是一样的，这时候我们可以考虑使用模板模式，然后我们可能需要整合多个第三方登录，因为要考虑到用户的体验性，如果你只写一个的话，用户可能没有这个账号，所以可能会造成体验性差。这时候我们就需要横向扩展所以我们可以使用策略模式和模板模式。

这里我就用Gitee登录作为例子来说一下

## 整合Gitee第三方登录

https://gitee.com/api/v5/oauth_doc#/list-item-1，giteeOauth官方文档

这里的策略模式我就不讲解了，不懂的去看之前的文件上传，然后讲一下我所设计的模板，首先我们要要遵守Oauth2的授权码流程，首先前端通过访问网站拿到授权的code，然后回调我们后端的接口，此时只有code是变化的所以只需要接收到code，然后获取access_token ，拿到access_token之后我们可以去获取第三方用户信息，获取完用户信息要检查一下是否已经在我们数据库中存在了，如果存在的话就更新一下数据就放行就好了，如果没有当前用户的话就去保存一下再去更新登录信息就好啦。

## 代码解析

### 总登录方法

```java
@Override
@Transactional
public Result<SaTokenInfo> login(String data, LoginTypeEnum loginTypeEnum) throws IOException {

    AtomicReference<Result<SaTokenInfo>> result = new AtomicReference<>();
    // 创建登录信息
    SocialTokenDTO socialToken = getSocialToken(data);

    // 获取用户ip信息
    String ipAddress = IpUtil.getIpAddress(request);
    String ipSource = IpUtil.getIpSource(ipAddress);

    // 获取第三方用户信息
    SocialUserInfoDTO socialUserInfo = getSocialUserInfo(socialToken);
    // 判断是否已经注册
    Opp.of(userAuthService.getUserAuth(loginTypeEnum, socialUserInfo))

            .ifPresent((user) -> result.set(getUserDetail(user, ipAddress, ipSource, loginTypeEnum.getType())))
            .orElseRun(() -> {
                try {
                    result.set(saveUserDetail(socialToken, ipAddress, ipSource, loginTypeEnum.getType(), socialUserInfo));
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            });

    Object loginId = result.get().getResult().getLoginId();

    if (StpUtil.isDisable(loginId)) {

        throw new DisableLoginException(loginTypeEnum.getDesc(), loginId, StpUtil.getDisableTime(loginId));
    }

    return result.get();

}
```

### 获取第三方登录token信息

这里对接的是gitee的登录所以是获取code然后调用gitee获取access_token的接口，通过clientId，redirectUri，clientSecret，和code

```java
@Override
public SocialTokenDTO getSocialToken(String data) throws IOException {

    String url = String.format(SocialLoginConstant.OAUTH2_GITEE_CODE, giteeConfigProperties.getClientId(), giteeConfigProperties.getRedirectUri(), giteeConfigProperties.getClientSecret(), data);

    CloseableHttpClient build = HttpClientBuilder.create().build();
    HttpPost httpPost = new HttpPost(url);

    HttpResponse response = build.execute(httpPost);

    String json = EntityUtils.toString(response.getEntity());

    JSONObject jsonObject = JSON.parseObject(json);
    String access_token = jsonObject.getString(SocialLoginConstant.ACCESS_TOKEN);
    return SocialTokenDTO.builder().accessToken(access_token).openId(data).loginType(LoginTypeEnum.GITEE.getType()).build();
}
```

拿到access_token封装到对象中

### 通过access_token获得用户信息

然后通过access_token去访问gitee提供的通过access_token拿到用户信息的接口

```java
@Override
public SocialUserInfoDTO getSocialUserInfo(SocialTokenDTO socialTokenDTO) throws IOException {

    String urlUser = String.format(SocialLoginConstant.OAUTH2_GITEE_USERINFO, socialTokenDTO.getAccessToken());

    HttpClient httpClientUser = HttpClientBuilder.create().build();
    HttpGet httpPostUser = new HttpGet(urlUser);

    HttpResponse responseUser = httpClientUser.execute(httpPostUser);

    String user = EntityUtils.toString(responseUser.getEntity());

    GiteeUserInfoDTO giteeUserInfoDTO = JSONUtil.toBean(user, GiteeUserInfoDTO.class);

    return SocialUserInfoDTO.builder()
            .nickname(giteeUserInfoDTO.getName())
            .avatar(giteeUserInfoDTO.getAvatarUrl())
            .build();
}
```

拿到用户信息之后可以进行封装并返回。

判断是否我们当前用户数据库中是否存在要登录的用户，我这里使用用户名和登录类型做了一下简单的判断，可以根据自己的需求进行更改。

如果可以从数据库中查询数据出来，我们就更新一下登录，如果没有查询出来就进行用户信息初始化进行保存数据库然后更新登录就好啦，这两个可以根据自己的需求和业务去改，我这里用了sa-token，就简单把代码放这了

```java
/**
 * 获取用户信息
 *
 * @param user      用户账号
 * @param ipAddress ip地址
 * @param ipSource  ip源
 * @param loginType 登录类型
 * @return {@link SaTokenInfo} 用户信息
 */
private Result<SaTokenInfo> getUserDetail(UserAuth user, String ipAddress, String ipSource, Integer loginType) {
    // 更新登录信息
    StpUtil.login(user.getUserInfoId(), SaLoginConfig
            .setExtra("name", user.getUsername())
            .setExtra("id", user.getUserInfoId())
            .setExtra("ipAddress", ipAddress)
            .setExtra("ipSource", ipSource)
    );
    // 返回信息

    SaTokenInfo tokenInfo = StpUtil.getTokenInfo();

    return Result.ok(ResultConstant.LoginMessage.LOGIN_SUCCESS, tokenInfo);
}

/**
 * 新增用户信息
 *
 * @param socialToken token信息
 * @param ipAddress   ip地址
 * @param ipSource    ip源
 * @return {@link Result<SaTokenInfo>} 用户信息
 */
private Result<SaTokenInfo> saveUserDetail(SocialTokenDTO socialToken, String ipAddress, String ipSource, Integer loginType, SocialUserInfoDTO socialUserInfo) throws IOException {

    // 保存用户信息
    UserInfo userInfo = UserInfo.builder()
            //
            .nickname(socialUserInfo.getNickname())
            .avatar(socialUserInfo.getAvatar())
            .build();

    userInfoService.save(userInfo);

    // 保存账号信息
    UserAuth userAuth = UserAuth.builder()
            .userInfoId(userInfo.getId())
            .username(userInfo.getNickname())
            .password(socialToken.getAccessToken())
            .loginType(loginType)
            .lastLoginTime(LocalDateTime.now(ZoneId.of(ZoneEnum.SHANGHAI.getZone())))
            .ipAddress(ipAddress)
            .ipSource(ipSource)
            .build();

    userAuthService.save(userAuth);

    // 绑定角色
    UserRole userRole = UserRole.builder()
            .userId(userInfo.getId())
            .roleId(RoleEnum.USER.getRoleId())
            .build();

    userRoleService.save(userRole);

    // 更新登录信息
    StpUtil.login(userAuth.getUserInfoId(), SaLoginConfig
            .setExtra("name", userAuth.getUsername())
            .setExtra("id", userAuth.getUserInfoId())
            .setExtra("ipAddress", ipAddress)
            .setExtra("ipSource", ipSource)
    );

    SaTokenInfo tokenInfo = StpUtil.getTokenInfo();

    return Result.ok(ResultConstant.LoginMessage.LOGIN_SUCCESS, tokenInfo);
}
```