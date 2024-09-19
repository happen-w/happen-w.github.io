---
title: "使用arthas分析代码消耗点"
categories: Java
---

### 1. 问题描述
产品反馈导出订单数据量一大，就会很慢

### 2 大致分析下问题
打开show sql配置,看到很多相同的sql.估计就是循环查询数据。mybais没有想hibernate一样，如果相同session如果通过id查询一样，就不发起sql查询

### 使用arthas分享耗时点

```shell
jps  # 查看java进程
./as.bat pid # 成功后会启动web页面
trace 类名 方法名  # 没什么不 类名#方法名, 从idea负责出来方法后还要再改下。。。
```
com.vehicle.platform.service.engine.OrderService#convertToMasterorderExports  方法耗时比较多，继续观察这个方法
![image](/assets/images/20240919/1.png)  

com.vehicle.platform.service.engine.OrderService#convertToMasterorderExports
![image](/assets/images/20240919/2.png)
直接报错了。这份方法太长了，70多个字段，然后我就refactor一下，抽成几个小的方法，然后用同意的的方法定位。


### 问题解决
 最后定位就是和开始猜想的一样，里面有循环查询，但每次查出来都一样，不止一处

- 在循环外面获取数据,不用每次去查询数据库
- 做线程本地缓存,将查询结果(空也要记录),避免每次查库(空间换时间)

修改前导出20条和导出100条
![image](/assets/images/20240919/3.png)
![image](/assets/images/20240919/4.png)
修改后4211条
![image](/assets/images/20240919/5.png)

### 总结
- 避免在循环中写sql,如果真的需要,最好测试性能,做缓存,避免每次查.或者使用sql链接
- arthas可以快速方便定位方法执行耗时,快速地位问题点,而不是一点点啃代码
- 避免方法写太长.arthas看了都摇头.不然定位耗时的时候,还要写重构下代码(尴尬)



