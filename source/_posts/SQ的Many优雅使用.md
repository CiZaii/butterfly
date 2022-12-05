---
title: SQ的Many优雅使用
date: '2022/10/28 21:20'
swiper: true
swiperImg: 'https://zangzang.oss-cn-beijing.aliyuncs.com/picGo/wallhaven-we8dv7.jpg'
cover: 'https://zangzang.oss-cn-beijing.aliyuncs.com/picGo/wallhaven-we8dv7.jpg'
categories: 技巧
tags:
  - stream-query
top: false
abbrlink: c0cbdbeb
---

## 使用实体类
```java
/**
 * <p>
 * 用户角色表
 * </p>
 *
 * @author 朵橙i
 * @since 2022-10-24
 */
@Getter
@Setter
@TableName("tb_user_role")
@ApiModel(value = "UserRole对象", description = "用户角色表")
public class UserRole implements Serializable {

    private static final long serialVersionUID = 1L;

    @ApiModelProperty("主键id")
    @TableId(value = "ID",type = IdType.ASSIGN_ID)
    private String id;

    @ApiModelProperty("用户id")
    @TableField("USER_ID")
    private String userId;

    @ApiModelProperty("角色id")
    @TableField("ROLE_ID")
    private String roleId;

    @ApiModelProperty("是否删除  0否 1是")
    @TableField(value = "IS_DELETE", fill = FieldFill.INSERT)
    @TableLogic(value = "0", delval = "1")
    private Integer isDelete;
}
```
## 进行操作
很明显这是一个中间表，我们现在有一个需求---> 我们要通过用户id查询出其所对应的所有的角色id，先看一下如果使用正常的情况下的逻辑处理
```java
LambdaQueryWrapper<UserRole> queryWrapper = new LambdaQueryWrapper<>();
queryWrapper.in(UserRole::getUserId, userIds);
List<UserRole> list = list(queryWrapper);
Map<String, List<UserRole>> collect = list.stream().collect(Collectors.groupingBy(UserRole::getUserId));
// 还要把map的value去转换成list<String>可太麻烦了

```
如果我们使用Collections.mapping方法给groupBy传入一个下游流进行操作的话会方便很多
下面示例是通过SQ的Many实现的代码编写更加优雅
```java
 // 根据userIds查询出所有的角色
        return Many.of(UserRole::getRoleId)
                .in(userIds) // 使用Steam对结果进行操作
                .query(steam // 根据userId分组然后list里边装有的是UserRole对象并不是我们想要的，我们只需要roleId
                ->
                steam.group(UserRole::getUserId,
                             // 传入一个下游收集器将每个所对应的多个userId封装到list里边
                Collective.mapping(UserRole::getUserId, Collectors.toList())));
```