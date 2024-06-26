---
tags:
- Springboot
- 数据库

---

## 前言

Springboot具有自动推断数据库Driver Class的能力，当我们在配置数据源信息时，如果是一些常见的数据库如Mysql、Oracle，即使没有配置driver-class-name，程序也会自动推断出来。

## 验证Springboot自动推断Driver Class

### 正常推断的情况

创建一个没有配置driver-class-name的配置文件application-mysql-without-driver-class-name.yml。

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: root
```

编写单元测试类加载该配置文件，并验证实际的datasource中driverClassName信息。

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
@ActiveProfiles("mysql-without-driver-class-name")
class MysqlWithoutDriverClassNameTests {

    @Autowired
    HikariDataSource dataSource;

    /**
     * @param
     * @return void
     **/
    @Test
    public void testDriver() {
        assertEquals("com.mysql.cj.jdbc.Driver", dataSource.getDriverClassName());
    }

}
```

单元测试运行通过，说明Springboot确实有这样的能力。

### 无法推断的情况

然而Springboot并非对所有的数据库都能够自动推断，下面使用国产数据库OceanBase试一下。

同样新建一个没有配置driver-class-name的配置文件application-oceanbase-without-driver-class-name.yml。

```yaml
spring:
  datasource:
    url: jdbc:oceanbase://localhost:2883/test
    username: root
    password: root
```

编写单元测试类。

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
@ActiveProfiles("oceanbase-without-driver-class-name")
class OceanBaseWithoutDriverClassNameErrorTests {

    @Autowired
    HikariDataSource dataSource;

    @Test
    public void testDriver() {
        assertEquals("com.oceanbase.jdbc.Driver", dataSource.getDriverClassName());
    }

}
```

但是这次Springboot在启动过程中就直接报错了，并提示无法推断出一个合适驱动类。

![image-20240626114629252](http://cdn.road4code.com/image-bed/20240626114629.png)

### Springboot推断Driver Class源码分析

从数据源的配置类DataSourceConfiguration开始分析源码。

```java
	/**
	 * Hikari DataSource configuration.
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(HikariDataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource",
			matchIfMissing = true)
	static class Hikari {

		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.hikari")
		HikariDataSource dataSource(DataSourceProperties properties) {
			HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
			if (StringUtils.hasText(properties.getName())) {
				dataSource.setPoolName(properties.getName());
			}
			return dataSource;
		}

	}
```

找到Hikari数据源的配置，这里因为没有配置数据库连接池默认的Hikari，如果用的是其他连接池也可以参考，最终推断的方法是一样的。

查看下createDataSource方法的实现。

```java
	protected static <T> T createDataSource(DataSourceProperties properties, Class<? extends DataSource> type) {
		return (T) properties.initializeDataSourceBuilder().type(type).build();
	}
```

继续来到initializeDataSourceBuilder方法。

```java
	/**
	 * Initialize a {@link DataSourceBuilder} with the state of this instance.
	 * @return a {@link DataSourceBuilder} initialized with the customizations defined on
	 * this instance
	 */
	public DataSourceBuilder<?> initializeDataSourceBuilder() {
		return DataSourceBuilder.create(getClassLoader()).type(getType()).driverClassName(determineDriverClassName())
				.url(determineUrl()).username(determineUsername()).password(determinePassword());
	}
```

我们需要关注的便是determineDriverClassName方法了。

```java
	/**
	 * Determine the driver to use based on this configuration and the environment.
	 * @return the driver to use
	 * @since 1.4.0
	 */
	public String determineDriverClassName() {
		if (StringUtils.hasText(this.driverClassName)) {
			Assert.state(driverClassIsLoadable(), () -> "Cannot load driver class: " + this.driverClassName);
			return this.driverClassName;
		}
		String driverClassName = null;
		if (StringUtils.hasText(this.url)) {
			driverClassName = DatabaseDriver.fromJdbcUrl(this.url).getDriverClassName();
		}
		if (!StringUtils.hasText(driverClassName)) {
			driverClassName = this.embeddedDatabaseConnection.getDriverClassName();
		}
		if (!StringUtils.hasText(driverClassName)) {
			throw new DataSourceBeanCreationException("Failed to determine a suitable driver class", this,
					this.embeddedDatabaseConnection);
		}
		return driverClassName;
	}
```

从这段代码可以看到如果driverClassName为空，Springboot会通过DatabaseDriver.fromJdbcUrl(this.url)方法推断，该方法的实现主要是根据配置的url去枚举中通过前缀进行匹配的，很显然DatabaseDriver这个枚举类中并没有OceanBase的驱动配置。

```java
 public static DatabaseDriver fromJdbcUrl(String url) {
        if (StringUtils.hasLength(url)) {
            Assert.isTrue(url.startsWith("jdbc"), "URL must start with 'jdbc'");
            String urlWithoutPrefix = url.substring("jdbc".length()).toLowerCase(Locale.ENGLISH);
            DatabaseDriver[] var2 = values();
            int var3 = var2.length;

            for(int var4 = 0; var4 < var3; ++var4) {
                DatabaseDriver driver = var2[var4];
                Iterator var6 = driver.getUrlPrefixes().iterator();

                while(var6.hasNext()) {
                    String urlPrefix = (String)var6.next();
                    String prefix = ":" + urlPrefix + ":";
                    if (driver != UNKNOWN && urlWithoutPrefix.startsWith(prefix)) {
                        return driver;
                    }
                }
            }
        }

        return UNKNOWN;
    }
```

后面Springboot还会尝试通过this.embeddedDatabaseConnection.getDriverClassName()方法来推断嵌入式数据库驱动，同样也是匹配不上。

## 扩展Springboot自动推断Driver Class

根据源码分析可以知道如果数据库驱动没有在DatabaseDriver枚举中配置，那Springboot就无法推断Driver Class。这种写死在代码中的方式显然不太靠谱，需要更换一种更加合理的方式。

DriverManager.getDriver(String url)这个就是java提供的方法通过url来确定Driver Class的。从源码中可以看到会遍历所有加载的数据库驱动的acceptsURL，只要返回true则就是对应的Driver Class。

```java
/**
 * Attempts to locate a driver that understands the given URL.
 * The {@code DriverManager} attempts to select an appropriate driver from
 * the set of registered JDBC drivers.
 *
 * @param url a database URL of the form
 *     <code>jdbc:<em>subprotocol</em>:<em>subname</em></code>
 * @return a {@code Driver} object representing a driver
 * that can connect to the given URL
 * @throws SQLException if a database access error occurs
 */
@CallerSensitive
public static Driver getDriver(String url)
    throws SQLException {

    println("DriverManager.getDriver(\"" + url + "\")");

    ensureDriversInitialized();

    Class<?> callerClass = Reflection.getCallerClass();

    // Walk through the loaded registeredDrivers attempting to locate someone
    // who understands the given URL.
    for (DriverInfo aDriver : registeredDrivers) {
        // If the caller does not have permission to load the driver then
        // skip it.
        if (isDriverAllowed(aDriver.driver, callerClass)) {
            try {
                if (aDriver.driver.acceptsURL(url)) {
                    // Success!
                    println("getDriver returning " + aDriver.driver.getClass().getName());
                return (aDriver.driver);
                }

            } catch(SQLException sqe) {
                // Drop through and try the next driver.
            }
        } else {
            println("    skipping: " + aDriver.driver.getClass().getName());
        }

    }

    println("getDriver: no suitable driver");
    throw new SQLException("No suitable driver", "08001");
}
```

Java 中的 JDBC（Java Database Connectivity）是一种用于执行数据库操作的标准 API，所有数据库厂商需要对这些API进行具体实现。可以简单看下OceanBase对于acceptsURL方法的实现。

```java
public static boolean acceptsUrl(String url) {
    return url != null && url.startsWith("jdbc:oceanbase:") && !url.contains("disableMariaDbDriver");
}
```

这样以来我们就可以在DataSourceProperties这个Bean初始化完成后判断下driverClassName是否为空，如果为空，我们通过DriverManager.getDriver(String url)方法推断出driverClassName并set到DataSourceProperties中。

```java
@TestConfiguration
public class DatasourcePropertiesCustomizer implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof DataSourceProperties dataSourceProperties) {
            if (StringUtils.isBlank(dataSourceProperties.getDriverClassName())
                    && StringUtils.isNotBlank(dataSourceProperties.getUrl())) {
                String driverClassName = determineDriverClassName(dataSourceProperties.getUrl());
                if (StringUtils.isNotBlank(driverClassName)) {
                    log.info("success determine driver class name :{}", dataSourceProperties);
                    dataSourceProperties.setDriverClassName(driverClassName);
                }
            }
        }
        return bean;
    }

    /**
     * jdbc提供的推断driver class的方法
     * @param url
     * @return String
     **/
    private String determineDriverClassName(String url) {
        Driver driver = null;
        try {
            driver = DriverManager.getDriver(url);
        } catch (SQLException e) {
            log.warn("determine driver class name fail", e);
        }
        if (driver != null) {
            return driver.getClass().getName();
        }
        return null;
    }
}
```

再次通过单元测试验证下。

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
@ActiveProfiles("oceanbase-without-driver-class-name")
// 加载刚才写的完善Driver Class推断的
@Import(DatasourcePropertiesCustomizer.class)
class OceanBaseWithoutDriverClassNameSuccessTests {

    @Autowired
    HikariDataSource dataSource;

    @Test
    public void testDriver() {
        assertEquals("com.oceanbase.jdbc.Driver", dataSource.getDriverClassName());
    }

}
```

## 结语

相关源码请查考https://github.com/GuDaoCoder/code-demo/tree/main/determine-database-driver。