---
title: MapStruct的高级用法
date: '2022/12/24 21:25'
updated: '2022/12/24 21:25'
tags: stream-query
categories: 技巧
sticky: 1
cover: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/20221224183454.png'
abbrlink: 4cab6938
---
## MapStruct的高级用法

最近两天学了一种MapStruct的高级用法，配合stream-query使用更佳哦

首先我们大多数项目应该都会用lombok，我就以整合lombok来讲

首先引入maven依赖

```xml
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <optional>true</optional>
</dependency>
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct</artifactId>
  <version>1.4.1.Final</version>
</dependency>
<dependency>
   <groupId>org.projectlombok</groupId>
   <artifactId>lombok-mapstruct-binding</artifactId>
   <version>0.2.0</version>
</dependency>
<dependency>
   <groupId>org.mapstruct.extensions.spring</groupId>
   <artifactId>mapstruct-spring-annotations</artifactId>
   <version>0.1.2</version>
</dependency>

<!--引入插件-->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>${maven-compiler-plugin.version}</version>
    <configuration>
        <source>${java.version}</source>
        <target>${java.version}</target>
        <annotationProcessorPaths>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>1.4.1.Final</version>
            </path>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
            </path>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok-mapstruct-binding</artifactId>
                <version>0.1.0</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

### 编写获取Bean工具类SpringContextHolder

```java
/**
 * @author Cizai
 * @since 2022/12/23 22:35
 **/
@Lazy(false)
@Component("SpringContextHolder")
public class SpringContextHolder implements ApplicationContextAware, DisposableBean {

    private static ApplicationContext applicationContext;
    private static final String ERROR_MESSAGE = "applicationContext尚未注入";

    @Override
    public void destroy() {
        applicationContext = null;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        SpringContextHolder.applicationContext = applicationContext;
    }

    public static <T> T getBean(Class<T> type) {
        return Optional.ofNullable(applicationContext).orElseThrow(() -> new IllegalStateException(ERROR_MESSAGE)).getBean(type);
    }

    @SuppressWarnings("unchecked")
    public static <T> T getBean(String name) {
        return (T) Optional.ofNullable(applicationContext).orElseThrow(() -> new IllegalStateException(ERROR_MESSAGE)).getBean(name);
    }

}
```

### 编写自定义转换接口

```java
/**
 * 自定义转换接口
 *
 * @author CiZai
 * @since 2022/12/23 22:38
 */
public interface Convertable<T extends Convertable<T>> {

    default T convert(Object o) {
        return (T) ConvertUtil.convert(o, this.getClass());
    }
}
```

### 自定义转换Mapper

```java
/**
 * MapUserMapper PO转VO
 *
 * @author CiZai
 * @since 2022/12/23 22:12
 */
@Mapper(componentModel = "spring")
public interface ArticlePoToVoMapper extends Converter<Article, ArticleVO> {

    /**
     * Convert the source object of type {@code S} to target type {@code T}.
     *
     * @param source the source object to convert, which must be an instance of {@code S} (never {@code null})
     * @return the converted object, which must be an instance of {@code T} (potentially {@code null})
     * @throws IllegalArgumentException if the source cannot be converted to the desired target type
     */
    @Override
    @Mapping(source = "id", target = "vId")
    ArticleVO convert(@NotNull Article source);
}

```

### 编写配置类

```java
/**
 * @author Cizai
 * @since 2022/12/23 19:07
 **/
@Configuration
public class ConvertConfig{
    /**
     * 注册我们自定义的转换器
     *
     * @param converters        转换器列表
     * @param conversionService 转换服务
     * @param <T>               转换源泛型
     * @param <R>               转换目标泛型
     */

    public <T, R> ConvertConfig(List<Converter<T, R>> converters, GenericConversionService conversionService) {
        converters.forEach(conversionService::addConverter);
    }
}
```

>在使用的过程中只需要将VO对象实现Convertable接口就可以了

### 测试

![image-20221224182710127](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221224182710127.png)

#### 配合stream-query操作

![image-20221224182958583](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221224182958583.png)