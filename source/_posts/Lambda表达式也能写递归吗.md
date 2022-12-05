---
title: Lambda表达式也能写递归吗
date: '2022/8/26 23:12'
swiper: true
swiperImg: >-
  https://zangzang.oss-cn-beijing.aliyuncs.com/img/b326474acb2e57f6b5dedf7fddf403a0.jpg
cover: >-
  https://zangzang.oss-cn-beijing.aliyuncs.com/img/b326474acb2e57f6b5dedf7fddf403a0.jpg
categories: 开源
tags:
  - Steam
abbrlink: 43424dfc
---
##  🍑当你点进这篇文章的时候可能会有些疑问，什么Lambda表达式也能写递归？
没错是这样的，我们在很多时候会用到递归树但是如果在数据库去写递归的SQL对数据库的压力就太大了，通常我们会一次性的都查出来在Java去进行递归的操作，我们这个操作要写好多代码而且思想基本都差不多，所以我们的Steam提供了这样一个方法toTree()，他可以定制的去进行集合转换为树的操作。
这里我感觉lambda能写递归感到这个思想很好玩所以这里给大家讲一下我写的源码
```java
/**
 * 将集合转换为树，自定义树顶部的判断条件，内置一个小递归(没错，lambda可以写递归)
 * 因为需要在当前传入数据里查找，所以这是一个结束操作
 *
 * @param idGetter        id的getter对应的lambda，可以写作 {@code Student::getId}
 * @param pIdGetter       parentId的getter对应的lambda，可以写作 {@code Student::getParentId}
 * @param childrenSetter  children的setter对应的lambda，可以写作 {@code Student::setChildren}
 * @param parentPredicate 树顶部的判断条件，可以写作 {@code s -> Objects.equals(s.getParentId(),0L) }
 * @param <R>             此处是id、parentId的泛型限制
 * @return list 组装好的树
 * eg:
 * {@code List studentTree = Steam.of(students).toTree(Student::getId, Student::getParentId, Student::setChildren, Student::getMatchParent) }
 */
public <R extends Comparable<R>> List<T> toTree(Function<T, R> idGetter,
                                                Function<T, R> pIdGetter,
                                                BiConsumer<T, List<T>> childrenSetter,
                                                Predicate<T> parentPredicate) {
    List<T> list = toList();
    List<T> parents = Steam.of(list).filter(e -> Opp.of(e).is(parentPredicate)).toList();
    return getChildrenFromMapByPidAndSet(idGetter, childrenSetter, Steam.of(list).group(pIdGetter), parents);
}
/**
 * toTree的内联函数，内置一个小递归(没错，lambda可以写递归)
 * 因为需要在当前传入数据里查找，所以这是一个结束操作
 *
 * @param idGetter       id的getter对应的lambda，可以写作 {@code Student::getId}
 * @param childrenSetter children的setter对应的lambda，可以写作 {@code Student::setChildren}
 * @param pIdValuesMap   parentId和值组成的map，用来降低复杂度
 * @param parents        顶部数据
 * @param <R>            此处是id的泛型限制
 * @return list 组装好的树
 */
private <R extends Comparable<R>> List<T> getChildrenFromMapByPidAndSet(Function<T, R> idGetter,
        BiConsumer<T, List<T>> childrenSetter,
        Map<R, List<T>> pIdValuesMap,
        List<T> parents) {
        AtomicReference<Consumer<List<T>>> recursiveRef = new AtomicReference<>();
        Consumer<List<T>> recursive = values -> Steam.of(values).forEach(value -> {
        List<T> children = pIdValuesMap.get(idGetter.apply(value));
        childrenSetter.accept(value, children);
        recursiveRef.get().accept(children);
        });
        recursiveRef.set(recursive);
        recursive.accept(parents);
        return parents;
        }
```
这里主要讲解一下getChildrenFromMapByPidAndSet()这个私有方法，因为递归主要是在这里做的

造一些数据debug一下更好理解

我们测试用这些数据

```java
List<Student> studentTree = Steam
                    .of(
                            Student.builder().id(1L).name("dromara").build(),
                            Student.builder().id(2L).name("baomidou").build(),
                            Student.builder().id(3L).name("hutool").parentId(1L).build(),
                            Student.builder().id(4L).name("sa-token").parentId(1L).build(),
                            Student.builder().id(5L).name("mybatis-plus").parentId(2L).build(),
                            Student.builder().id(6L).name("looly").parentId(3L).build(),
                            Student.builder().id(7L).name("click33").parentId(4L).build(),
                            Student.builder().id(8L).name("jobob").parentId(5L).build()
                    )
                    // just 3 lambda,top parentId is null
                    .toTree(Student::getId, Student::getParentId, Student::setChildren);
```

我们测试使用的顶部数据是parentId为null的

当进去方法之后我买会根据parentId去进行分组存到map中方便后续使用

![image-20220826223948668](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220826223948668.png)

进入getChildrenFromMapByPidAndSet()方法中这里就不对参数做说明了上边源码中解释的很清楚

我们创建一个原子引用类存放一个Consumer是对list类型的操作

然后写一下这个consumer所进行的操作，具体操作后边说

![image-20220826224729454](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220826224729454.png)

到这里的时候会将这个consumer对象存放到recursiveRef中然后下面对其进行操作就进如这个consumer的操作了

![image-20220826225057371](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220826225057371.png)

此时我们要操作的是顶部数据

![image-20220826225412081](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220826225412081.png)

上边这步意思是通过我们现在的顶部id取出parentId为我们顶部数据id的对象然后对其进行我们传入的set操作将其放入顶部数据的子节点中然后最妙的地方出现了我们此时还没有到第二个根节点，此时从recursiveRef中取出来将刚刚保存的子节点取进行consumer操作也就是看看齐有没有子节点有的话继续进行操作，其实如果有一些基础的话我讲到这应该已经知道这个使用的思想是dfs(深度优先遍历)，此时看一下![image-20220826230254127](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220826230254127.png)

下一次进行操作的是name为hutool的这个对象，很明显证实了我的话

![image-20220826230406005](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220826230406005.png)

第三次就是name为looly这个对象。说到这应该已经很清楚了。

>这个递归的操作最妙的就是使用一个AtomicReference去存放我买的消费操作然后在每一次操作的时候从原子类中取出来再次进行消费。

