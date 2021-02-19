# 连接池

## TCP是面向连接的基于字节流的协议

说明：

1.面向连接，意味着连接要先创建再使用，创建连接的三次握手有一些网络开销

2.基于字节流，意味着字节是发送数据的最小单元，TCP协议本身无法区分哪几个字节是完整的消息体，也无法感知是否有多个客户端在使用同一个TCP连接，TCP只是一个读写数据的管道

结论：如果没有使用连接池，那么每次建立连接都有TCP握手的开销，且TCP基于连接字节流，在多线程的情况下对同一连接进行复用，会出现线程安全问题

## 基于TCP连接的客户端，对外提供API的三种模式

1.连接池和连接分离的 API：有一个 XXXPool 类负责连接池实现，先从其获得连接 XXXConnection，然后用获得的连接进行服务端请求，完成后使用者需要归还连接。通常，XXXPool 是线程安全的，可以并发获取和归还连接，而 XXXConnection 是非线程安全的。

2.内部带有连接池的 API：对外提供一个 XXXClient 类，通过这个类可以直接进行服务端请求；这个类内部维护了连接池，SDK 使用者无需考虑连接的获取和归还问题。一般而言，XXXClient 是线程安全的。

3.非连接池的 API：一般命名为 XXXConnection，以区分其是基于连接池还是单连接的，而不建议命名为 XXXClient 或直接是 XXX。直接连接方式的 API 基于单一连接，每次使用都需要创建和断开连接，性能一般，且通常不是线程安全的。

![1685d9db2602e1de8483de171af6fd7e](E:\存款PGM\传帮带\pool\1685d9db2602e1de8483de171af6fd7e.png)

## 禁止通过Jedis直连Redis，使用try-with-resources 模式从 JedisPool 获得和归还 Jedis 实例，并建议通过shutdownhook，优雅关闭JedisPool

Jedis连接池配置说明：https://zhuanlan.zhihu.com/p/84481313

```

private static JedisPool jedisPool = new JedisPool("127.0.0.1", 6379);

new Thread(() -> {
    try (Jedis jedis = jedisPool.getResource()) {
        for (int i = 0; i < 1000; i++) {
            String result = jedis.get("a");
            if (!result.equals("1")) {
                log.warn("Expect a to be 1 but found {}", result);
                return;
            }
        }
    }
}).start();


@PostConstruct
public void init() {
    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
        jedisPool.close();
    }));
}
```

推荐：通过Spring整合Redis，配置Jedis连接池，详见各项目spring-redis.xml配置

进阶：自从spring boot 2.x版本后，默认的redis的链接池从JedisPool变成了LettucePool，Lettuce主要利用netty实现与redis的同步和异步通信。所以更安全和性能更好。

## 使用连接池务必保证复用，Apache HttpClients属于内部带有连接池的API，最佳复用方式为CloseableHttpClient 声明为 static，只创建一次，并且在 JVM 关闭之前通过 addShutdownHook 钩子关闭连接池，在使用的时候直接使用 CloseableHttpClient 即可，无需每次都创建。

```

private static CloseableHttpClient httpClient = null;
static {
    //当然，也可以把CloseableHttpClient定义为Bean，然后在@PreDestroy标记的方法内close这个HttpClient
    httpClient = HttpClients.custom().setMaxConnPerRoute(1).setMaxConnTotal(1).evictIdleConnections(60, TimeUnit.SECONDS).build();
    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
        try {
            httpClient.close();
        } catch (IOException ignored) {
        }
    }));
}

@GetMapping("right")
public String right() {
    try (CloseableHttpResponse response = httpClient.execute(new HttpGet("http://127.0.0.1:45678/httpclientnotreuse/test"))) {
        return EntityUtils.toString(response.getEntity());
    } catch (Exception ex) {
        ex.printStackTrace();
    }
    return null;
}
```

拓展阅读：不复用会有什么问题？：https://www.jianshu.com/p/82f30208ae56

推荐：通过Spring集成httpClient，不必每次创建连接池对象：https://www.jianshu.com/p/363e3d7c235b

## 数据库连接池最佳实践，推荐使用druid

常用连接池对别：

![连接池](E:\存款PGM\传帮带\pool\连接池.png)

druid配置详解：

```
<bean id="cartDataSource" class="com.alibaba.druid.pool.DruidDataSource"

​     init-method="init" destroy-method="close">

​    <property name="url" value="${cluster.jdbc.url}"/>

​    <property name="username" value="${cluster.jdbc.username}"/>

​    <property name="password" value="${cluster.jdbc.password}"/>

​    <property name="connectionInitSqls" value="set names utf8mb4"/>

​    <!-- 连接池初始连接数 -->

​    **<property name="****initialSize****" value="5" />**

​    <!-- 允许的最大同时使用中(在被业务线程持有，还没有归还给druid) 的连接数 。**建议值区间****20-50** -->

​    **<property name="****maxActive****" value="20" />**

​    <!-- 允许的最小空闲连接数，空闲连接超时踢除过程会最少保留的连接数 -->

​    **<property name="****minIdle****" value="5" />**

<!-- 从连接池获取连接的最大等待时间 5 秒-->

​    **<property name="****maxWait****" value=“5000" />**

​    <!-- 一条物理连接的最大存活时间 120分钟-->

​    <property name="phyTimeoutMillis" value="7200000"/>

​    **<!--** **强行关闭从连接池获取而长时间未归还给****druid****的连接****(****认为异常连接）****-->**

​    **<property name="****removeAbandoned****" value="true"/>**

 **<!--** **异常连接判断条件，超过****180** **秒 则认为是异常的，需要强行关闭** **-->**

​    **<property name="****removeAbandonedTimeout****" value="180"/>**

​    <!-- 从连接池获取到连接后，如果超过被空闲剔除周期，是否做一次连接有效性检查 -->

​    **<property name="****testWhileIdle****" value="true"/>**

​    <!-- 从连接池获取连接后，是否马上执行一次检查 -->

​    **<property name="****testOnBorrow****" value=“true"/>**

​    <!-- 归还连接到连接池时是否马上做一次检查 -->

​    <property name="testOnReturn" value="false"/>

​    <!-- 连接有效性检查的SQL -->

​    **<property name="****validationQuery****" value="SELECT 1"/>**

​    <!-- 连接有效性检查的超时时间 60 秒 -->

​    **<property name="****validationQueryTimeout****" value=“60"/>**

<!-- 周期性剔除长时间呆在池子里未被使用的空闲连接, 300秒一次-->

​    <property name="timeBetweenEvictionRunsMillis" value=“300000"/>

​    <!-- 空闲多久可以认为是空闲太长而需要剔除 480 秒-->

​    <property name="minEvictableIdleTimeMillis" value=“480000"/>

​    <!-- 如果空闲时间太长即使连接池所剩连接 < minIdle 也要被剔除 480秒 -->

​    <property name="maxEvictableIdleTimeMillis" value=“480000"/>

​    <!-- 是否设置自动提交，相当于每个语句一个事务 -->

​    <property name="defaultAutoCommit" value="true"/>

​    <!-- 记录被判定为异常的连接 -->

​    <property name="logAbandoned" value="true"/>

​    <!-- 网络读取超时，网络连接超时

​       socketTimeout : 对于线上业务小于5s，对于BI等执行时间较长的业务的SQL，需要设置大一点

​    -->

​    <property name="connectionProperties" value="socketTimeout=3000;connectTimeout=1000"/>

</bean>
```



## 配置连接超时和读取超时参数，必须端到端对超时有一致性评估，协同配合方平衡吞吐量和错误率

说明：

- 连接超时参数ConnectTimeout，配置建立TCP连接的最长等待时间
- 读取超时参数ReadTimeout，用来控制从Socket上读取数据的最长等待时间

【推荐】：

连接超时参数

- 连接超时参数不建议设置过长，因为TCP三次握手连接所需时间非常短，如果很久无法建立连接，基本上是网络不通，通常配置1-5秒即可
- 连接超时要清楚排查位置，直连排查服务端，通过反响代理（如Nginx）排查反响代理

读取超时参数：

- 读取超时不建议设置过短，读取超时指的是向Socket写入数据后，等到Socket返回数据的超时时间，其中包含的时间或者说绝大部分时间，是服务端处理业务逻辑的时间。
- 读取超时不建议配置太长，超过时间太长，客户端线程等待时间过长，可能导致线程累计，程序崩溃，一般不超过30秒
- 读取超时，不代表服务端的执行会中断，需要结合业务实际考虑如何进行后续处理

## Druid源码中的设计模式之职责链模式

**Chain Of Responsibility Design Pattern**：Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it.

**职责链模式**：将请求的发送和接收解耦，让多个接收对象有机会处理这个请求。将这些接收对象串成一条链，并沿着这条链传递这个请求，直到链上某个接收对象能够处理它为止。

**举个栗子**：冯巩、牛群两位德高望重的艺术家，曾经表演过一段讽刺相声《小偷公司》，其中有这么几句台词：

牛：为此警察堵了我们好几回。我一看这架式，赶紧写一报告：鉴于风声太紧，建议公司全体人员立即转移!大狗，请批示!
冯：递上去了吗?
牛：交给副组长了。副组长拿过来一看，先画了个圆圈儿。
冯：画圈儿?
牛：这叫圈阅儿。意思是基本同意。请组长酌定。
冯：这还是组长管的事儿
牛：组长拿过来一看，又画了个圈儿，请副科长酌定!画了个圈儿请科长酌定。科长先画了个圈儿，请副经理酌定，又画了个圈儿。
冯：画了五个圈儿这个
牛：请总经理酌定。 要说办事效率说我们总经理。拿过来这么一看，五个圈儿，明白了。提起笔来，唰唰地批了几个字。
冯：怎么批的?
牛：同意!到奥运会去偷!

在这段台词中，建议公司立即转移的报告就是请求，副组长、组长、科长等就是处理器，每个处理器各自承担各自的处理请求职责，整个处理流程形成一个链条，故称为职责链模式。需要说明的是，职责链模式有多个变种，GoF给出的定义中，如果某个处理器能够处理这个请求，那就不会继续往下传递请求。此外，另一种比较常见的方式为，请求会被所有的处理器都处理一遍，不存在中途终止的情况，显然《小偷公司》中的情形属于此类。

**代码示例**：

```

public abstract class Handler {
  protected Handler successor = null;

  public void setSuccessor(Handler successor) {
    this.successor = successor;
  }

  public final void handle() {
    doHandle();
    if (successor != null) {
      successor.handle();
    }
  }

  protected abstract void doHandle();
}

public class HandlerA extends Handler {
  @Override
  protected void doHandle() {
    //...
  }
}

public class HandlerB extends Handler {
  @Override
  protected void doHandle() {
    //...
  }
}

public class HandlerChain {
  private Handler head = null;
  private Handler tail = null;

  public void addHandler(Handler handler) {
    handler.setSuccessor(null);

    if (head == null) {
      head = handler;
      tail = handler;
      return;
    }

    tail.setSuccessor(handler);
    tail = handler;
  }

  public void handle() {
    if (head != null) {
      head.handle();
    }
  }
}

// 使用举例
public class Application {
  public static void main(String[] args) {
    HandlerChain chain = new HandlerChain();
    chain.addHandler(new HandlerA());
    chain.addHandler(new HandlerB());
    chain.handle();
  }
}
```

**为啥要用这个模式**：

当有新的处理器需要加时，比如小偷公司的审批流程中需要增加股长审批，那么我们可以在不改变原有代码的情况下，新增一个HandlerC()来满足业务扩展，符合开闭原则

**durid源码分析之filter-chain设计模式**：https://blog.csdn.net/lqzkcx3/article/details/78397500

**进阶**：职责链模式常用在框架的开发中，为框架提供扩展点，让框架的使用者在不修改框架源码的情况下，基于扩展点添加新的功能。实际上，更具体点来说，职责链模式最常用来开发框架的过滤器和拦截器。Servlet Filter、Spring Interceptor 这两个 Java 开发中常用的组件都使用了职责链模式，感兴趣请自行研究。



## 对类似数据库连接池的重要资源进行持续检测，并设置合理的报警阈值

进阶：https://github.com/prometheus/jmx_exporter