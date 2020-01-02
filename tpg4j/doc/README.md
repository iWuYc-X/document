[TOC]

# tpg4j

线程池治理（Thread Pool Governance for Java）框架，用于统一管理项目中的线程池，避免到处创建线程池，通过配置可以快速改变线程池容量，动态调整/配置线程池。

在项目开发中，为了提高性能，我们通常会使用提高并发数的方式进行，这时通常是用线程池的方式进行线程管理与配置的。在项目中，为了方便集中管理线程池，我在这里使用了类似于log4j的方式进行配置。

> 配置文件中的名词解释：
>
> - cores:
>
> 表示的是宿主机的可用处理器数量。可用于做占位符，做乘法计算，目前只支持乘法计算。如：cores\*4表示宿主机可用处理器数量的四倍，若CPU是4核的宿主机，则cores\*4=4\*4=16，最终值为16。
>
> - 时间单位：
>
> 涉及到时间的配置，可使用[d,h,m,s,ms,ns]分别表示[天,时,分,秒,毫秒,纳秒]等单位。如：keepAliveTime:1h;表示keepAliveTime的值为1个小时。如果不填单位，则默认为s。keepAliveTime:60，表示keepAliveTime的值60秒。

## 配置示例
```yaml
tpg4j:
  commons:
    auto.scan: 10s
  conf:
    default:
      corePoolSize: 8
      factory: com.iwuyc.tools.tpg4j.thread.impl.DefaultExecutorServiceFactory
      # [h - hour;m - minute;s - second;ms - millisecond;ns - nanosecond;]default:s
      keepAliveTime: 60m
      maximumPoolSize: cores*2
      maxQueueSize: 1800
      daemon: false
    service:
      corePoolSize: cores
      factory: com.iwuyc.tools.tpg4j.thread.impl.DefaultExecutorServiceFactory
      # [h - hour;m - minute;s - second;ms - millisecond;ns - nanosecond;]default:s
      keepAliveTime: 60m
      maximumPoolSize: cores*2
      maxQueueSize: 1800
      daemon: true
  using:
    root: default
    com:
      - default
      - iwuyc:
          - service
          - tools:
              tpg4j: service1
              commons: service2

```
### 配置示例说明

> - tgp4j
>
> 配置文件的根节点，命名空间。
>
> - 模块说明(commons、conf、using)
>
>   - commons
>
>     通用配置模块。
>
>     - auto.scan：自动扫描间隔，默认为1m，可使用时间单位。整型，大于零，小于等于0的时候，默认间隔为60秒。
>
>   - conf
>
>     conf模块下的子节点名为线程池的名字，在接下来的using模块中，可作为引用的标记。
>
>     - 线程池名下，默认有[corePoolSize、factory、keepAliveTime、maximumPoolSize、maxQueueSize、daemon]配置项。如果是超出这些配置的，则可以在ThreadPoolConfig中的otherSetting中获取。
>       - corePoolSize：核心线程数，必须大于等于0。可使用cores的表达式，只支持乘法。2*cores表示宿主机cpu核心数的两倍。
>       - factory：创建线程池用的工厂类的全限定名，必须实现`com.iwuyc.tools.tpg4j.thread.ExecutorServiceFactory`接口，并且有无参构造函数。tpg4j默认实现类`com.iwuyc.tools.tpg4j.thread.impl.DefaultExecutorServiceFactory`支持创建jdk中的`java.util.concurrent.ThreadPoolExecutor`和`java.util.concurrent.ScheduledThreadPoolExecutor`两种线程池。
>       - keepAliveTime：线程存活时间，可使用时间单位，必须大于等于0。
>       - maximumPoolSize：最大线程数，必须大于0，并且大于corePoolSize的值。可使用cores的表达式，只支持乘法。4*cores表示宿主机cpu核心数的四倍。
>       - maxQueueSize：最大的队列大小。
>       - daemon：是否是守护线程。默认值为false。
>
>   - using
>
>     using表示配置各命名空间下所引用的线程池配置。
>
>     - root：为固定的名字，是命名空间的根节点，所有未直接指定并且父命名空间无指定引用的线程池名字的，都将使用root所引用的线程池配置。	
>
>     - 示例：
>
>       继承关系使用半角"."符号做分隔。
>
>       - 调用ThreadPoolsServiceHolder.getScheduleService(domain)获得线程池。其中domain="com.iwuyc.tools.tpg4j.thread.ConfigTest"，在配置示例中，最近的父命名空间为：com.iwuyc.tools，因此，会获取到名为"service1"配置对应的线程池，但因为conf模块中无"service1"的配置，因此，会回溯到更高一级的命名空间"com.iwuyc"中，最终获取到"service"的线程池。
>
>       - 若str="org.iwuyc.tools"，在using中未配置该命名空间，切不存在父命名空间，所以，最终获取到的将是"root"命名空间所对应线程池配置。
>
>       - 若使用类进行获取，最终获取到的将是类名对应的命名空间。如果是内部类，则，最终的子叶将是内部类的类名。例如：B为A的内部类。第一种情况：B.class.getName()为"A$B"，使用类的方式获取线程池ThreadPoolsServiceHolder.getScheduleService(A.B.class），则命名空间为“A.B”。第二种情况，如果使用字符串str="A$B",则，最终的命名空间为"A$B"。
>
>         using:
>
>         ​	root: default
>
>         ​	A$B: service1
>
>         ​	A: 
>
>         ​		B: service2
>
>         第一种情况将会获取到service2的线程池，而第二种情况将会获取到service1的线程池。
>
>     - 关于using配置的一些注意事项。
>
>       - using下的直属节点，必须是对象，不能是纯量或者数组。
>
>       - root下的直属节点只能是纯量，指向conf中的直属子节点，即线程池的名字。
>
>       - 命名空间下的直属节点，如果本空间下需要配置线程池的引用，则需要将本节点下的所有直属节点转换为数组，并且使用**最后一个**纯值作为当前节点的线程池引用名。如示例中的：
>
>         com:
>               \- default
>               \- iwuyc:
>                   \- service
>                   \- tools:
>                       tpg4j: service1
>                       commons: service11
>
>         其中default是纯值，表示com命名空间下引用的线程池名字，iwuyc为com的子命名空间。
>
>   - 

### 主要的接口有

```java
import com.iwuyc.tools.tpg4j.thread.ThreadPoolService;
import com.iwuyc.tools.tpg4j.thread.ExecutorServiceFactory;
import com.iwuyc.tools.tpg4j.thread.conf.Config;

```

### Quick Start

> 引入tpg4j的包

```xml
<dependencies>
    <dependency>
        <groupId>com.iwuyc.tools</groupId>
        <artifactId>tpg4j</artifactId>
        <version>0.1.1</version>
    </dependency>
</dependencies>
```

> 其中依赖了第三方的jar包，清单如下：

```xml
<dependencies>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.6.0</version>
    </dependency>
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>11.0</version>
    </dependency>
    <dependency>
        <groupId>org.yaml</groupId>
        <artifactId>snakeyaml</artifactId>
        <version>1.4</version>
    </dependency>
</dependencies>

```

​        以上的清单版本都是目前支持的最低版本（建议使用稳定的高版本工具包），并且需要主动声明引入。tpg4j不主动引入，主要考虑的是，这些工具包都是大家最常用的，为避免包冲突引起的问题，所以将这些包的范围都标记为"provided"，需用户自己主动引入。

> 下面的示例为当前最简单的使用示例，源码在 [tpg4j-demo](https://github.com/iWuYc-X/tpg4j-demo) 中可以查看并下载。

```java
package com.iwuyc.tools.demo.tpg4j;


import com.google.common.util.concurrent.SettableFuture;
import com.iwuyc.tools.tpg4j.thread.ThreadPoolService;
import com.iwuyc.tools.tpg4j.thread.ThreadPoolServiceHolder;
import com.iwuyc.tools.tpg4j.thread.conf.Config;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;

public class Bootstrap {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        // 配置文件。可以为空，如果为空，则使用默认的配置。
        String thframeYaml = null;
        ThreadPoolService service = Config.config(thframeYaml);

        // 方式1：由config返回的ThreadPoolsService实例获取线程池
        final SettableFuture<Void> end = SettableFuture.create();
        ExecutorService pools1 = service.getExecutorService(Bootstrap.class);
        pools1.submit(() -> {
            System.out.println("hello world!!");
            end.set(null);
        });
        System.out.println(pools1);

        // 方式2：由ThreadPoolsServiceHolder获取线程池
        ExecutorService pools2 = ThreadPoolServiceHolder.get(Bootstrap.class);
        // print: true
        assert  pools1 == pools2;
        end.get();

        // 最后在应用程序关闭的时候，应当把线程池服务关闭。
        ThreadPoolServiceHolder.getThreadPoolsService().shutdown();
    }
}

```
默认实现了jdk中的两种线程池的构造器：
```
com.iwuyc.tools.tpg4j.thread.impl.DefaultExecutorServiceFactory;
```
上述类实现了``` com.iwuyc.tools.tpg4j.thread.ThreadPoolService```接口，如果有需要支持拓展第三方的线程池（必须实现```java.util.concurrent.ExecutorService```接口）生成，则可以继承该接口，然后在```create(ThreadPoolConfig)```或```createSchedule(ThreadPoolConfig)```方法中实现相应的线程池构造。


## 默认配置
```yaml
tpg4j:
  commons:
    auto.scan: 1m
  conf:
    default:
      corePoolSize: cores
      factory: com.iwuyc.tools.tpg4j.thread.impl.DefaultExecutorServiceFactory
      keepAliveTime: 60m
      maximumPoolSize: cores*4
      maxQueueSize: 100000
      daemon: false
  using:
    root: default
```

