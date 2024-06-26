---
tags:
- Springboot
- Druid
---

## 前言

Druid是Java语言中最好的数据库连接池。Druid能够提供强大的监控和扩展功能。<!-- more -->
## 添加依赖
```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.8</version>
</dependency>
```
网上有不少教程建议的依赖是
```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.2.8</version>
</dependency>
```
其实Druid和Springboot官方已经提供了很好的集成，虽然二者都可以使用，但是druid-spring-boot-starter不需要编写配置类，简化了配置。
## 添加配置
```yaml
spring:
  datasource:
    username: root
      password: root
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://127.0.0.1:3306/test?useUnicode=true&characterEncoding=utf8&allowPublicKeyRetrieval=true&useSSL=false&serverTimezone=Asia/Shanghai
    type: com.alibaba.druid.pool.DruidDataSource
    # 数据源连接池配置
    druid:
      #   数据源其他配置
      initialSize: 5
      minIdle: 5
      maxActive: 20
      maxWait: 60000
      timeBetweenEvictionRunsMillis: 60000
      minEvictableIdleTimeMillis: 300000
      validationQuery: SELECT 1 FROM DUAL
      testWhileIdle: true
      testOnBorrow: false
      testOnReturn: false
      poolPreparedStatements: true
      maxPoolPreparedStatementPerConnectionSize: 20
      useGlobalDataSourceStat: true
      connectionProperties: druid.stat.mergeSql=true;druid.stat.logSlowSql=true;druid.stat.slowSqlMillis=1000;
      # 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
      filters: stat,wall,slf4j

      #配置监控属性： 在druid-starter的： com.alibaba.druid.spring.boot.autoconfigure.stat包下进行的逻辑配置
      # WebStatFilter配置，
      web-stat-filter:
        #默认为false，表示不使用WebStatFilter配置，就是属性名去短线
        enabled: true
        #拦截该项目下的一切请求
        url-pattern: /*
        #对这些请求放行
        exclusions: /druid/*,*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico
        session-stat-enable: true
        principal-session-name: session_name
        principal-cookie-name: cookie_name
      # StatViewServlet配置
      stat-view-servlet:
        #默认为false，表示不使用StatViewServlet配置，就是属性名去短线
        enabled: true
        #配置DruidStatViewServlet的访问地址。后台监控页面的访问地址
        url-pattern: /druid/*
        #禁用HTML页面上的“重置”功能，会把所有监控的数据全部清空，一般不使用
        reset-enable: false
        #监控页面登录的用户名
        login-username: admin
        #监控页面登录的密码
        login-password: 123456
        #白名单
        allow:
        #黑名单
        deny:
      #Spring监控配置，说明请参考Druid Github Wiki，配置_Druid和Spring关联监控配置
      aop-patterns: com.zzp.*
```
有几个地方需要注意下：

- 想要在日志中输出慢日志除了配置connectionProperties，还需要在filters属性中加入项目使用的日志框架如log4j或者slf4j，并**引入对应依赖，**否则日志中不会输出慢sql日志;
- 如果需要Spring监控，除了配置aop-patterns，还需要加入**spring-boot-starter-aop依赖，**否则Spring监控页面空白**。**
## 查看监控页面
访问http://ip:port/druid/login.html，输入账号密码，如果数据源页面各项参数和配置的参数一样，则说明集成成功。
![druid](http://cdn.road4code.com/image-bed/20240329175433.png)
调用接口，查看SQL监控和Spring监控页面是否正常显示。
## 去除广告
监控页面下方有阿里云的广告，可以通过代码去除。
```java
import com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceAutoConfigure;
import com.alibaba.druid.spring.boot.autoconfigure.properties.DruidStatProperties;
import com.alibaba.druid.util.Utils;
import org.springframework.boot.autoconfigure.AutoConfigureAfter;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.servlet.*;
import java.io.IOException;

/**
 * Druid广告配置
 *
 * @author zzp
 */
@Configuration
@ConditionalOnWebApplication
@AutoConfigureAfter(DruidDataSourceAutoConfigure.class)
@ConditionalOnProperty(
        name = "spring.datasource.druid.stat-view-servlet.enabled",
        havingValue = "true",
        matchIfMissing = true)
public class DruidAdConfig {

    /**
     * 去除监控页面底部广告
     *
     * @param properties
     * @return org.springframework.boot.web.servlet.FilterRegistrationBean
     */
    @Bean
    public FilterRegistrationBean removeDruidAdFilterRegistrationBean(
            DruidStatProperties properties) {
        // 获取web监控页面的参数
        DruidStatProperties.StatViewServlet config = properties.getStatViewServlet();
        // 提取common.js的配置路径
        String pattern = config.getUrlPattern() != null ? config.getUrlPattern() : "/druid/*";
        String commonJsPattern = pattern.replaceAll("\\*", "js/common.js");

        final String filePath = "support/http/resources/js/common.js";

        // 创建filter进行过滤
        Filter filter =
                new Filter() {
                    @Override
                    public void init(FilterConfig filterConfig) throws ServletException {}

                    @Override
                    public void doFilter(
                            ServletRequest request, ServletResponse response, FilterChain chain)
                            throws IOException, ServletException {
                        chain.doFilter(request, response);
                        // 重置缓冲区，响应头不会被重置
                        response.resetBuffer();
                        // 获取common.js
                        String text = Utils.readFromResource(filePath);
                        // 正则替换banner, 除去底部的广告信息
                        text = text.replaceAll("<a.*?banner\"></a><br/>", "");
                        text = text.replaceAll("powered.*?shrek.wang</a>", "");
                        response.getWriter().write(text);
                    }

                    @Override
                    public void destroy() {}
                };
        FilterRegistrationBean registrationBean = new FilterRegistrationBean();
        registrationBean.setFilter(filter);
        registrationBean.addUrlPatterns(commonJsPattern);
        return registrationBean;
    }
}

```
