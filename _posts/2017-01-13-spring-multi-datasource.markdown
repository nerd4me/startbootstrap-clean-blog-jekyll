---
layout:     post
title:      "Spring实现数据源动态切换"
subtitle:   "Spring - multi datasource"
date:       2017-01-13 22:23:29 +0800
author:     "Nerd4me"
catalog:    true
tags:       [spring]
---

## 背景说明

在实际工作中，经常会遇到一个系统访问多个数据库的情况，那么问题来了，如何解决多数据源问题呢？不光是要配置多个数据源，还得在程序运行过程中动态的切换数据源。下面我们看一下如何利用Spring提供的工具类，优雅的实现项目多数据源的动态切换。

## 实现原理

扩展Spring提供的AbstractRoutingDataSource抽象类，该类实现了`java.sql.DataSource`接口，在程序运行过程中，充当数据源路由中介，具体看一下它的源码实现：

```java
@Override
public Connection getConnection() throws SQLException {
    return determineTargetDataSource().getConnection();
}

protected DataSource determineTargetDataSource() {
    Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
    Object lookupKey = determineCurrentLookupKey(); // 抽象方法
    DataSource dataSource = this.resolvedDataSources.get(lookupKey);
    if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
        dataSource = this.resolvedDefaultDataSource;
    }
    if (dataSource == null) {
        throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
    }
    return dataSource;
}
```

我们可以看到，该类内部维护了一个`java.util.Map`类型的成员变量`resolvedDataSources`，通过实现`determineCurrentLookupKey()`抽象方法，可以动态的获取`DataSource`对象。

## 举个栗子

动态数据源实现类：

```java
public class DynamicDataSource extends AbstractRoutingDataSource {
    private static final ThreadLocal<String> LOOKUP_KEY_HOLDER = new ThreadLocal<String>();

    @Override
    protected Object determineCurrentLookupKey() {
        return getCurrentLookupKey();
    }

    public static void setCurrentLookupKey(String lookupKey) {
        checkArgument(!isNullOrEmpty(lookupKey));
        LOOKUP_KEY_HOLDER.set(lookupKey);
    }

    public static String getCurrentLookupKey() {
        return LOOKUP_KEY_HOLDER.get();
    }
}
```

数据源配置：

```java
@Configuration
public class DBConfig {
    @Bean
    public DataSource dynamicDataSource() {
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        dynamicDataSource.setTargetDataSources(
                ImmutableMap.<Object, Object>of(
                        "h1", h2(), // h2
                        "hsql", hsql() // hsql
                )
        );
        dynamicDataSource.setDefaultTargetDataSource(h2());
        return dynamicDataSource;
    }

    @Bean(destroyMethod = "shutdown")
    public DataSource hsql() {
        return new EmbeddedDatabaseBuilder()
                .setType(HSQL)
                .addScripts("classpath:schema.sql", "classpath:test-data.sql")
                .continueOnError(true)
                .build();
    }

    @Bean(destroyMethod = "shutdown")
    public DataSource h2() {
        return new EmbeddedDatabaseBuilder()
                .setType(H2)
                .addScripts("classpath:schema.sql", "classpath:test-data.sql")
                .continueOnError(true)
                .build();
    }

    @Bean
    public JdbcTemplate jdbcTemplate() {
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        jdbcTemplate.setDataSource(dynamicDataSource());
        return jdbcTemplate;
    }
}
```

测试类：

```java
@ContextConfiguration(classes = { DBConfig.class })
public class DynamicDataSourceTest extends AbstractTransactionalJUnit4SpringContextTests {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    @Autowired
    @Qualifier("dynamicDataSource")
    public void setDataSource(DataSource dataSource) {
        super.setDataSource(dataSource);
    }

    @Test
    public void testGetUsers() {
        // 默认为‘h2’
        print(jdbcTemplate.queryForList("select * from sec_user"));
        jdbcTemplate.update("delete from sec_user");
        print(jdbcTemplate.queryForList("select * from sec_user"));

        // 切换数据源为‘hsql’
        DynamicDataSource.setCurrentLookupKey("hsql");
        print(jdbcTemplate.queryForList("select * from sec_user"));
    }

    private static void print(List<Map<String, Object>> result) {
        System.out.println(Joiner.on(", ").join(
                FluentIterable.from(result)
                        .transform(Functions.toStringFunction())
        ));
    }
}
```