---
layout:     post
title:      Spring AOP中declare-parents为特定的类增加新的功能
subtitle:   Spring AOP中declare-parents为特定的类增加新的功能
date:       2017-09-08
author:     Jason
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - SPring
---


如果有这样一个需求，为一个已知的API添加一个新的功能。

由于是已知的API，我们不能修改其类，只能通过外部包装。但是如果通过之前的AOP前置或后置通知，又不太合理，最简单的办法就是实现某个我们自定义的接口，这个接口包含了想要添加的方法。

但是JAVA不是一门动态的语言，无法再编译后动态添加新的功能，这个时候就可以使用 aop:declare-parents 来做了.

如果是可以改写的类，直接实现自定义的接口就行了，下面看看AOP是如何做的！

最开始使用的类和接口：

package com.spring.test.declareparents;

public interface Chinese {
    public void Say();
}
public class LiLei implements Chinese{
    public void Say() {
        System.out.println("我是中国人！");
    }
}
　　想要添加的新功能和接口

package com.spring.test.declareparents;

public interface Add {
    public void Todo();
}
public class DoSomething implements Add{
    public void Todo() {
        System.out.println("我爱中国！");
    }
}
　　通过配置AOP，实现两种功能的耦合

复制代码
    <bean id="lilei" class="com.spring.test.declareparents.LiLei"/>
    <bean id="doSomething" class="com.spring.test.declareparents.DoSomething"/>
    
    <aop:config proxy-target-class="true">
        <aop:aspect>
            <aop:declare-parents 
            types-matching="com.spring.test.declareparents.LiLei"
            implement-interface="com.spring.test.declareparents.Add" 
            default-impl="com.spring.test.declareparents.DoSomething"/>
        </aop:aspect>
    </aop:config>
</beans>
复制代码
　　其中types-mathcing是之前原始的类，implement-interface是想要添加的功能的接口，default-impl是新功能的默认的实现。

　　在使用时，直接通过getBean获得bean转换成相应的接口就可以使用了。

        Chinese lilei = (Chinese)ctx.getBean("lilei");
        lilei.Say();
        
        Add lilei2 = (Add)ctx.getBean("lilei");
        lilei2.Todo();
 


