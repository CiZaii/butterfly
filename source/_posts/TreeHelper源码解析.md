---
title: TreeHelper源码解析
date: 2022/12/5 15:43
tags: stream-query
categories: 源码
sticky: 1
cover: https://zangzang.oss-cn-beijing.aliyuncs.com/img/20221205220741.png
---


## 关于TreeHelper的内部操作

### 我们先看一下源代码里边的属性

```java
// 获取当前节点id的操作    
private final SerFunc<T, R> idGetter;
// 获取当前节点的父节点
private final SerFunc<T, R> pidGetter;
// 父id的值
private final R pidValue;
// 判断是否是祖宗节点的
private final SerPred<T> parentPredicate;
// 获取子节点的操作
private final SerFunc<T, List<T>> childrenGetter;
// 保存子节点的操作
private final SerBiCons<T, List<T>> childrenSetter;
```

### 请求树先生的帮助

```
/**
 * <p>获取树先生的帮助</p>
 *
 * @param idGetter       获取节点id操作 {@link io.github.vampireachao.stream.core.lambda.function.SerFunc} object
 * @param pidGetter      获取父节点id操作 {@link io.github.vampireachao.stream.core.lambda.function.SerFunc} object
 * @param pidValue       父节点值
 * @param childrenGetter 获取子节点操作 {@link io.github.vampireachao.stream.core.lambda.function.SerFunc} object
 * @param childrenSetter 操作子节点 {@link io.github.vampireachao.stream.core.lambda.function.SerBiCons} object
 * @param <T>            树节点类型
 * @param <R>            父id类型
 * @return a {@link io.github.vampireachao.stream.core.business.tree.TreeHelper} object
 */
public static <T, R extends Comparable<R>> TreeHelper<T, R> of(SerFunc<T, R> idGetter,
                                                               SerFunc<T, R> pidGetter,
                                                               R pidValue,
                                                               SerFunc<T, List<T>> childrenGetter,
                                                               SerBiCons<T, List<T>> childrenSetter) {
    return new TreeHelper<>(idGetter, pidGetter, pidValue, null, childrenGetter, childrenSetter);
}
```

>这里其实就是先将我们的操作获取了给到树先生，然后树先生就可以帮我们干这些事了

### 客制化树先生

```java
/**
 * <p>ofMatch.</p>
 *
 * @param idGetter        获取节点id操作  {@link io.github.vampireachao.stream.core.lambda.function.SerFunc} object
 * @param pidGetter       获取父节点id操作 {@link io.github.vampireachao.stream.core.lambda.function.SerFunc} object
 * @param parentPredicate 是否是父节点断言操作 {@link io.github.vampireachao.stream.core.lambda.function.SerPred} object
 * @param childrenGetter  获取子节点操作 { {@link io.github.vampireachao.stream.core.lambda.function.SerFunc} object
 * @param childrenSetter  操作子节点  {@link io.github.vampireachao.stream.core.lambda.function.SerBiCons} object
 * @param <T>             树节点类型 T class
 * @param <R>             父id类型 R class
 * @return a {@link io.github.vampireachao.stream.core.business.tree.TreeHelper} object
 */
public static <T, R extends Comparable<R>> TreeHelper<T, R> ofMatch(SerFunc<T, R> idGetter,
                                                                    SerFunc<T, R> pidGetter,
                                                                    SerPred<T> parentPredicate,
                                                                    SerFunc<T, List<T>> childrenGetter,
                                                                    SerBiCons<T, List<T>> childrenSetter) {
    return new TreeHelper<>(idGetter, pidGetter, null, parentPredicate, childrenGetter, childrenSetter);
}
```

>这个方法是定制化我们的祖宗节点，我们有两种方法请求，第一种是上边那种传入一个值，这个值就是祖宗节点，另一种就是这个ofMatch使用断言去选择祖宗节点，如果符合断言就说明是一个祖宗节点

### 拾取果实构建树

```java
/**
 * <p>拾取果实构建树</p>
 *
 * @param list 果篮 {@link java.util.List} object
 * @return 树 {@link java.util.List} object
 */
public List<T> toTree(List<T> list) {
    // 这里判断断言是否存在，如果存在则使用断言的方法去构造祖宗节点，如果不存在使用pidVal去构造
    if (Objects.isNull(parentPredicate)) {
        // 过滤掉节点id为null的，只剩下不为null，然后根据节点的父id分组
        final Map<R, List<T>> pIdValuesMap = Steam.of(list).filter(e -> Objects.nonNull(idGetter.apply(e))).group(pidGetter);
        // 根据给的pid去拿到祖宗节点，构造出根节点来
        final List<T> parents = pIdValuesMap.getOrDefault(pidValue, new ArrayList<>());
        // 构造子节点
        getChildrenFromMapByPidAndSet(pIdValuesMap);
        return parents;
    }
    // 创建一个大小和节点相同的集合
    final List<T> parents = new ArrayList<>(list.size());
    // 在过滤断言的时候同时构造祖宗节点
    final Map<R, List<T>> pIdValuesMap = Steam.of(list).filter(e -> {
        if (parentPredicate.test(e)) {
            parents.add(e);
        }
        return Objects.nonNull(idGetter.apply(e));
    }).group(pidGetter);
    // 构造子节点
    getChildrenFromMapByPidAndSet(pIdValuesMap);
    return parents;
}
```

>注释已经写的很清楚了但是可能有人会疑问为什么你把pIdValuesMap给了getChildrenFromMapByPidAndSet最后返回的是parents但是里边的值已经改变了，那么此时说明还是不够掌握集合哦，因为我们parents里边的祖宗节点其实和pIdValuesMap那祖宗节点的部分地址是一样的可以看一下下边的图，这样就不难解释为什么我们可以返回parents了。

![image-20221205210754581](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221205210754581.png)

### getChildrenFromMapByPidAndSet

```java
private void getChildrenFromMapByPidAndSet(Map<R, List<T>> pIdValuesMap) {
    Steam.of(pIdValuesMap.values()).flat(Function.identity())
        // 遍历所有节点，如果节点的id不为null则进行保存操作
            .forEach(value -> {
                final List<T> children = pIdValuesMap.get(idGetter.apply(value));
                if (children != null) {
                    childrenSetter.accept(value, children);
                }
            });
}
```

### 将树上所有的果实摘取到果篮中

```java
/**
 * <p>将树上所有的果实摘取到果篮中</p>
 *
 * @param list 要摘取果实的树 {@link java.util.List} object
 * @return 果篮(包装果实的集合) {@link java.util.List} object
 */
public List<T> flat(List<T> list) {
    AtomicReference<Function<T, Steam<T>>> recursiveRef = new AtomicReference<>();
    // 对当前的数据进行操作拿到他的孩子做扁平化处理然后继续处理孩子的孩子最后与当前元素合并成一个新的流
    Function<T, Steam<T>> recursive = e -> Steam.of(childrenGetter.apply(e)).flat(recursiveRef.get()).unshift(e);
    recursiveRef.set(recursive);
    return Steam.of(list).flat(recursive).peek(e -> childrenSetter.accept(e, null)).toList();
}
```

>filter和forEach原理一样如下
>
>这个方法涉及到了一个递归的操作，原理和在1.1.17之前的Steam中的toTree一样可以去看我之前写过的文章，把链接放到这里了[Lambda也能写递归？](https://zverify.cn/cizai/43424dfc.html)

