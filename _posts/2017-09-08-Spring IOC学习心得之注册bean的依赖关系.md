---
layout:     post
title:      Spring IOC学习心得之注册bean的依赖关系
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

registerDependentBean方法的解析（注册bean的依赖关系）
源码如下：
```
public void registerDependentBean(String beanName, String dependentBeanName) {  
        // A quick check for an existing entry upfront, avoiding synchronization...  
        String canonicalName = canonicalName(beanName);  
        /*  
        dependentBeanMap中存储的是目前已经注册的依赖这个bean的所有bean,这里从这个集合中获取目前所有已经注册的依赖beanName的bean集合,  
        然后看这个集合中是否包含dependentBeanName,即是否已经注册,如果包含则表示已经注册,则直接返回,否则,将bean依赖关系添加到两个map缓存即完成注册  
            */  
        Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);  
        if (dependentBeans != null && dependentBeans.contains(dependentBeanName)) {  
            return;  
        }  
  
        // No entry yet -> fully synchronized manipulation of the dependentBeans Set  
        synchronized (this.dependentBeanMap) {  
            dependentBeans = this.dependentBeanMap.get(canonicalName);  
            if (dependentBeans == null) {  
                dependentBeans = new LinkedHashSet<>(8);  
                this.dependentBeanMap.put(canonicalName, dependentBeans);  
            }  
            dependentBeans.add(dependentBeanName);  
        }  
        synchronized (this.dependenciesForBeanMap) {  
            Set<String> dependenciesForBean = this.dependenciesForBeanMap.get(dependentBeanName);  
            if (dependenciesForBean == null) {  
                dependenciesForBean = new LinkedHashSet<>(8);  
                this.dependenciesForBeanMap.put(dependentBeanName, dependenciesForBean);  
            }  
            dependenciesForBean.add(canonicalName);  
        }  
    }  
```
解释下上述代码中用到的两个map缓存集合，如下：
private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<String, Set<String>>(); //指定的bean与目前已经注册的依赖这个指定的bean的所有bean的依赖关系的缓存（我依赖的）
private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<String, Set<String>>(); //指定bean与目前已经注册的创建这个bean所需依赖的所有bean的依赖关系的缓存（依赖我的）
知道了这两个集合的意思，再参考上述源码中的注释，就不难理解这段代码的意思了
