---
title: java.lang.IllegalArgumentException:Can not set XXX to com.sun.proxy.$Proxy28
date: 2018-07-10 09:09:33
categories: 
- Spring
tags:
- Spring
- proxy
- IllegalArgumentException
---
最近Spring项目启动报错java.lang.IllegalArgumentException: Can not set XXX to com.sun.proxy.$Proxy28经过检查发现是使用 @Autowired 时，写在了接口的实现类上面，由于spring AOP动态代理是通过接口，如果不做配置的话一般情况使用的是Java原生的动态代理，所以注入地方都要通过接口进行注入，如果通过实现类进行注入就会报这个错
代码如下：
<!--more-->

```
public class MonitorService {

    @Autowired
    private JobFlowService jobFlowService;

}	
```
```
@Service("jobFlowService")
public class JobFlowService implements IScheduleExecutor {
	
}
```
有两种解决方案

- 1.在你的项目spring配置文件中添加一行配置:

```
<aop:aspectj-autoproxy proxy-target-class="true"/>
```
强制让Spring使用cglib生成代理类，就可以通过接口的实现类上@Service等类似注解进行注入
- 2.更改代码，写成用接口注入如下：

```
public class MonitorService {

    @Autowired
    private JobFlowServiceImpl jobFlowService;

}	
```