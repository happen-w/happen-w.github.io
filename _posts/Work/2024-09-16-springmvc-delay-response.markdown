---
title: "Sleep使用错误问题"
categories: Java
---

入职后开始开代码，然后发现代码中有一些写法很多有问题。做了下记录和正确的解决方法
### 1 问题描述
一些接口在controller对方法进行try-catch后，发现异常后，sleep 1s后才返回。我猜大概是为了对于错误的请求，
不那么快返回，避免对面一直不断重试。可能这边的数据也需要去别的系统拉回来，sleep 1s后如果对面再来请求,就有结果了。
代码大概如下: 

```
@GetMapping("/***")
public String getSomeTime() {
    try{
        // do something    
    }catch(Exception e){
		Thread.sleep(1000);
		throw new RuntimeException("error");
    }
}
```

### 2 大致分析下问题
1. 跑这段代码是跑在tomcat的线程上，sleep后，回导致tomcat无法接受其他请求，线程的数量也是有限的，这样会大大的降低并发数。
当请求变多时，很多线程都在无意义的sleep,进而无法进行服务
2. 猜测可能的解决方案，这里应该时可以直接返回,然后线程可以继续服务，然后1s后，再将请求放回去
3. 查了下资料， servlet3.0 支持异步， springMVC也有对应的解决方案。


### 3 jemeter压测暴露问题
我们写一个最简单的接口，这个接口会处理完结果后，1s后返回。
然后测试当并发量大的时候，预期时会有大量的接口超时。

```shell
@GetMapping("/hello")
public String hello(@RequestParam(value = "name", defaultValue = "World") String name) throws Execption {
        Thread.sleep(1000);
		return String.format("Hello %s!", name);
}
```
1. 压测结果
 ![image](/assets/images/1sleep_请求情况.png)  

2. 可以看到1w个请求，平均需要4s后才能返回，吞吐量也只有195.5/sec (我猜tomcat最大的线程数时200个)


### 4 springmvc 异步返回
网上搜了下,springMVC还真有提供类似的解决方案。[mvc-ann-async][mvc-ann-async]
下面使用DeferredResult,达到我们想要的效果代码如下

```shell
private static long i = 0;
@GetMapping("/hello2")
public DeferredResult<String> hello2(@RequestParam(value = "name", defaultValue = "World") String name){
    long id;
    synchronized (this){
        i++;
        log.info("第" + i + "个请求");
        id = i;
    }

    DeferredResult<String> result = new DeferredResult<>();
    // 获取当前时间
    Instant now = Instant.now();
    // 将当前时间加1秒
    Instant later = now.plus(1, ChronoUnit.SECONDS);
    Date date = Date.from(later);
    caller.put(new Key(id, date), ()->{
        result.setResult(String.format("Hello %s!", name));
    });
    return result;
}

public static ConcurrentHashMap<Key, Runnable > caller = new ConcurrentHashMap<>();
public record Key(long id,Date date){};

// 使用一个公共的线程,每秒钟循环caller,拿到到期的请求返回结果
@PostConstruct
public void init() {
    new Thread(() -> {
        while (true) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }

            Iterator<Map.Entry<Key, Runnable>> iterator = caller.entrySet().iterator();
            while (iterator.hasNext()){
                Map.Entry<Key, Runnable> next = iterator.next();
                if(new Date().after(next.getKey().date)){
                    next.getValue().run();
                    iterator.remove();
                }
            }
        }
    }).start();
}
```

### 5 再次使用jemeter压测
1. 压测结果
![image](/assets/images/DeferredResult.png)
2. 可以看到，1w个请求，平均需要1.6s后才能返回，吞吐量有提升。
3. 去掉循环线程的sleep 1s还可以更好
![image](/assets/images/DeferredResult2.png)


### 6 总结
1. Java的业务代码里面，用到sleep要很小心，基本都有更好的方法替代。写sleep基本都是有问题的代码。
2. 这种思想，其实就和 IO多路复用时一样的。 

[mvc-ann-async]: https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html