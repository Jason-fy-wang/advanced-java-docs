---
tags:
  - mybatis
  - interview
---
### 1. Mybatis和JPA(Java persistence API)区别

1) 设计理念
	* Mybatis
		基于SQL 映射器（SQL Mapper) 模式,  开发者自己编写SQL, 并在XML 或注解中 把SQL 结果映射到 Java 对象
	* JPA (Java persistence API)
		基于ORM(对象关系映射) 思想, 强调POJO 即数据模型,  由框架负责生成SQL
		使用注解或XML定义实体类与数据库的映射关系, 屏蔽底层SQL 细节.

2) SQL 编写与自动化

| 特性     | mybatis                  | JPA(hibernate etc)                 |
| ------ | ------------------------ | ---------------------------------- |
| SQL 控制 | 开发者手写SQL(多表关联, 分页,动态SQL) | 自动生成SQL, 也可自定义SQL                  |
| 动态SQL  | 提供 <if> <foreach> 等标签支持  | 需借助 Criteria API 或 spring Data JPA |
| 复杂查询   | 完全由开发者控制,  性能可调优精细       | 对复杂关联查询支持一般, 需调优或手写原生SQL           |
|        |                          |                                    |


3) 映射与对象管理
	1. MyBatis
```shell
映射通过 <resultMap>  @Result 等手动配置, 科灵活映射一对一,  一对多
	不管理对象状态(即没有一级/二级缓存, 脏检查, 自动通过实体状态到数据库), 所有CRUD操作需要显式调用对应方法
```

	2. JPA
	
```
实体类由 EntityManager 统一管理, 支持一级/二级缓存, 懒加载/级联操作, 脏检查自动同步等特性
```



4) 性能与灵活度
     1. Myabtis 
		性能取决于开发者编写的SQL， 对SQL优化掌控力强
		对热点场景可精细化调优， 避免不必要的夺标Join 或深度前台查询
		
	2. JPA
		简化开发, 自动化程度高, 但在复杂查询, 批量操作时可能产生N+1 问题或不必要的SQL
		虽然可通过@BatchSize, Join FETCH, EntityGraph 等手段优化, 仍需要对ORM 行为较为了解才能避免性能陷阱


5) 开发成本与学习曲线
	Myabtis
	学习门槛地, 熟悉JDBC , SQL 即可上手
	大型项目中, 大量XML映射文件可能维护繁琐

	JPA
	学习曲线高, 理解实体状态转换, 级联, 事务, 延迟加载等
	一旦掌握, 使用注解/接口式 Repository 可以大幅减少模板化CRUD代码

6) 场景
	mybatis 
    1) 对SQL 由严格性能要求的场景
    2) 业务逻辑需要大量复杂或者动态SQL
    3) 团队对SQL优化经验丰富

	JPA
	1) CRUD 层较为简单, 业务以对象为中心
	2) 快速搭建, 迭代频繁, 对开发侠侣要求高
	3) 需要事务自动管理, 实体状体自动同步等ORM 特性



### 2.  统计mybatis 中 SQL 执行时间

 1) 使用mybatis interceptor
```java
@Intercepts({
  @Signature(type = Executor.class, method = "query", 
             args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
  @Signature(type = Executor.class, method = "update", 
             args = {MappedStatement.class, Object.class})
})
public class SqlTimeInterceptor implements Interceptor {
  @Override
  public Object intercept(Invocation invocation) throws Throwable {
    MappedStatement ms = (MappedStatement) invocation.getArgs()[0];
    String id = ms.getId();
    long start = System.currentTimeMillis();
    try {
      return invocation.proceed();
    } finally {
      long end = System.currentTimeMillis();
      long time = end - start;
      System.out.println("[MyBatis-SQL] " + id + " 执行耗时：" + time + " ms");
      // 或者用日志框架、上报到监控系统
    }
  }

  @Override
  public Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }

  @Override
  public void setProperties(Properties properties) { }
}
```

2) 使用Mybatis-plus performanceInterceptor
```java
@Bean
public PerformanceInterceptor performanceInterceptor() {
    PerformanceInterceptor p = new PerformanceInterceptor();
    // SQL 执行超过 maxTime(单位 ms) 会停止运行，有助于发现死循环
    p.setMaxTime(500);
    // SQL 美化，便于阅读
    p.setFormat(true);
    return p;
}

###output
==>  Preparing: SELECT * FROM user WHERE id = ?
==> Parameters: 1(Integer)
<==    Total: 5 ms

```

3) 在数据源层面使用代理(P6Spy / datasource-proxy)
	* P6Spy
		在classpath中加入 P6Spy, 并在 spy.properties中开始日志与耗时打印
	* datasource-proxy (net.ttddyy:datasource-proxy)
	```shell
	ProxyDataSource pds = ProxyDataSourceBuilder
    .create(originalDataSource)
    .logQueryBySlf4j(SLF4JLogLevel.INFO)
    .countQuery()         // 统计执行次数
    .asJson()             // JSON 格式输出
    .build();

```
		

4) Spring-APO
```java
@Aspect
@Component
public class MyBatisMetricsAspect {
    private final Timer.Builder timer = Timer.builder("mybatis.sql")
        .description("MyBatis SQL 执行时间")
        .publishPercentiles(0.5, 0.95, 0.99);

    @Around("execution(* org.apache.ibatis.executor.Executor.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        Timer.Sample sample = Timer.start();
        try {
            return pjp.proceed();
        } finally {
            String method = pjp.getSignature().getName();
            sample.stop(timer.tag("method", method).register(meterRegistry));
        }
    }
}

```


5) 开启Mybatis 自带的日志实现

```shell
# mybatis-config.xml
<configuration>
  <settings>
    <setting name="logImpl" value="STDOUT_LOGGING"/> <!-- 控制台打印 -->
    <!-- 或者使用 SLF4J: value="SLF4J" -->
  </settings>
</configuration>

```

