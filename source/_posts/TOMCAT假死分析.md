---
title: TOMCAT假死分析
date: 2017-01-05 20:29:14
tags:
- tomcat
- dbcp
category:
- CODE
---
#### 现象
+ tomcat假死，无法响应任何请求。
+ CPU、内存等均无告警，假死之后CPU占用率变得很低。
+ 无任何异常日志，CLOSE_WATI数正常。
+ 静态资源也无法访问
+ 通过命令查看线程数
```
ps -ef|grep tomcat --获取进程ID
ps -T -p <pid>|wc -l -- 获取tomcat下线程数
```

发现有近1500个线程，这已经到达tomcat线程上限。

#### 获取DUMP日志
因为生产环境没有装JDK，只有JRE环境，费了好一番功夫才发现有个神奇的命令.
```
kill -3 <pid>
```

这个命令并不会导致进程被杀，并且会将相应的线程堆栈信息和大致的内存占用情况输出到tomcat目录下的catalina.out文件中。
因为这个文件往往较大，所以DUMP前可以先清空这个日志文件。
```
echo "">catalina.out -- 这个命令也可以用于运行时释放日志
```

拿到DUMP后，问题开始明朗起来：
```
"http-bio-443-exec-1151" daemon prio=10 tid=0x00007fd1c96c9000 nid=0x26cb in Object.wait() [0x00007fd0f914e000]
   java.lang.Thread.State: WAITING (on object monitor)
    at java.lang.Object.wait(Native Method)
        - waiting on <0x00000007f5b040f8> (a org.apache.commons.pool.impl.GenericObjectPool$Latch)
            at java.lang.Object.wait(Object.java:503)
                at org.apache.commons.pool.impl.GenericObjectPool.borrowObject(GenericObjectPool.java:1118)
                    - locked <0x00000007f5b040f8> (a org.apache.commons.pool.impl.GenericObjectPool$Latch)
                        at org.apache.commons.dbcp.PoolingDataSource.getConnection(PoolingDataSource.java:106)
                            at org.apache.commons.dbcp.BasicDataSource.getConnection(BasicDataSource.java:1044)
                                at org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource.getConnection(AbstractRoutingDataSource.java:164)
                                    at org.springframework.jdbc.datasource.DataSourceTransactionManager.doBegin(DataSourceTransactionManager.java:205)
                                        at org.springframework.transaction.support.AbstractPlatformTransactionManager.getTransaction(AbstractPlatformTransactionManager.java:373)
                                            at org.springframework.transaction.interceptor.TransactionAspectSupport.createTransactionIfNecessary(TransactionAspectSupport.java:420)
                                                at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:257)
                                                    at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:95)
                                                        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
                                                            at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:646)


```

有近千个线程处于WAITING状态，都是卡在获取数据库连接这一步上。
反查数据库中的连接数:
```
SELECT COUNT(1) FROM GV$SESSION WHERE machine = '主机名'
```

结果为100，并且这些连接全部处于INACTIVE状态，而数据库连接池配置的maxActive数就是100个。
说明数据库连接池泄露了。

#### 分析
应用框架采用的是spring+mybatis+dbcp1.4。
由于并不需要手动关闭数据库连接，所以业务代码导致这个问题的可能性不大。
google之后发现dbcp官方JIRA上也report了这个问题，据说是一个BUG，升级到1.5.3版本能解决这个问题。
但是奇怪的是，应用已经正常运行两年多了，为什么最近才出现这个问题呢？
难道是因为割接的地市越来越多，导致服务器压力增大，进而导致这个问题的发生？
如果是DBCP的BUG，那么升级版本或者替换为C3P0应该能够解决这个问题。
但我不确定是否真的是这个原因，或许业务代码在某种极为巧合的情形下的确会导致连接无法正常关闭；那么鲁莽的行为只会掩盖这个问题，并且在日后造成更大的麻烦。
所以最好的解决方式是找到连接泄漏的位置。
通过采用DBCP配置：
```
maxWait=5000
removeAbandoned=true
removeAbandonedTimeout=60
logAbandoned=true
```

来定位问题代码的位置。
设置的具体含义在官方文档上有，简言之，这样设置之后，在一定条件下，会触发DBCP的回收机制。当一个连接超过一定时间没有被使用，那么就视为abandoned连接，删除之，并记录下该连接的上下文和调用栈。

#### 继续跟踪
目前连接数还没有到达指标处，继续跟踪，希望明天就能解决这个问题。




