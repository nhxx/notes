---
title: spring源码解析01：资源
date: 2018-01-03 21:19:58
tags:
---

## 01 Resource

spring core的resource类图
![resource.png](/images/Resource.png)
Resource接口抽象了实际具体资源形式；
常见的资源实现：

> * PathResource：依赖java.nio.file.Path处理，支持可写
> * FileSystemResource：依赖java.nio.File处理，支持可写
> * ClassPathResource：支持类路径上的资源加载，依赖ClassLoader或Class加载对应Path上的File（but not for resources in a JAR？？）
> * UrlResource：依赖java.net.URL定位对应的资源；

## 02 ResourceLoader

![ResourceLoader.png](/images/ResourceLoader.png)

### ResourceLoader

加载资源的策略接口，常见的加载策略包括类路径形式的资源（ClassRelativeResourceLoader） 或 文件资源（FileSystemResourceLoader）；

### ResourcePatternResolver

将pattern（比如基于ant形式的pattern）解析出多个resource对象；

### PathMatchingResourcePatternResolver

ResourcePatternResolver的唯一实现类；
核心函数是getResources

```
@Override
public Resource[] getResources(String locationPattern) throws IOException {
   if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
      if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
         return findPathMatchingResources(locationPattern);
      }
      else {
         return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
      }
   }
   else {
      int prefixEnd = (locationPattern.startsWith("war:") ? locationPattern.indexOf("*/") + 1 :
            locationPattern.indexOf(":") + 1);
      if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
         return findPathMatchingResources(locationPattern);
      }
      else {
         return new Resource[] {getResourceLoader().getResource(locationPattern)};
      }
   }
}
```

总结一下：如果符合Ant Pattern走findPathMatchingResources，如果开头是classpath*:，但不符合Ant Pattern，走findAllClassPathResources，都不符合走getResourceLoader().getResource(locationPattern)。
