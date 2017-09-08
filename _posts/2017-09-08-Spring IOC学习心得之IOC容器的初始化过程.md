---
layout:     post
title:     Spring IOC学习心得之IOC容器的初始化过程
subtitle:   
date:       2017-09-08
author:     Jason
header-img: 
catalog: true
tags:
    - Spring
    - Java
    - Linux
---

注：本文大多数内容都是摘自《Spring技术内幕》这本书
简单来说，Ioc容器的初始化过程是在refresh（）方法中启动的，包括BeanDefinition的Resource定位，载入和注册三个过程。
第一：Resource的定位
这个Resource指的是BeanDefinition的资源定位，由ResourceLoader通过统一的Resource接口来完成。文件系统中的Bean定义信息可以使用FileSystemResource来进行抽象，类路径中的Bean定义信息可以使用ClassPathResource来抽象。

第二： BeanDefinition的载入
上面已经定位了资源Resource的位置，接下来就是将Bean定义表示成Ioc容器内部的结果，即读取进来封装成BeanDefinition。

第三：向IOc容器注册这些BeanDefinition
实际上就是将BeanDefinition注册到Hashmap中去，IOC就是通过这个HashMap来持有这些BeanDefinition数据的

但是请注意，上面三个过程不包含Bean依赖注入的过程，依赖注入一般发生在应用第一次通过getBean向容器索取Bean的时候，但是有一个例外是，可以设置Bean的lazyinit属性，那么这个Bean的依赖注入就再IOC容器初始化时就预先完成了
