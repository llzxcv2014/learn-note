# Spring Security

Spring security的两个主要的领域：

1. **`authentication` 验证**
2. **`authorization` 授权/访问控制**

**`principle`**： 当事人（用户、设备或者其他应用）

Spring Secirity提供的验证方式：

* `HTTP BASIC`
* `HTTP DIGEST`
* `HTTP X.509证书`
* `LDAP`
* `Form-based（表单验证）`
* `OpenId`
* `Authentication based on pre-established request headers (such as Computer Associates Siteminder)`
* `Jasig Central Authentication Service (otherwise known as CAS, which is a popular open source single sign-on system)（单点登录）`
* `Transparent authentication context propagation for Remote Method Invocation (RMI) and HttpInvoker (a Spring remoting protocol)`
* `Automatic "remember-me" authentication (so you can tick a box to avoid re-authentication for a predetermined period of time)`
* `Anonymous authentication (allowing every unauthenticated call to automatically assume a particular security identity)`
* `Run-as authentication (which is useful if one call should proceed with a different security identity)`
* `Java Authentication and Authorization Service (JAAS)`
* `Java EE container authentication (so you can still use Container Managed Authentication if desired)`
* `Kerberos`
* `Java Open Source Single Sign-On (JOSSO)`
* `OpenNMS Network Management Platform`
* `AppFuse`
* `AndroMDA`
* `Mule ESB`
* `Direct Web Request (DWR)`
* `Grails`
* `Tapestry`
* `JTrac`
* `Jasypt`
* `Roller`
* `Elastic Path`
* `Atlassian Crowd`
* `own implemention`

## 模块介绍

### `Core`包

包含核心的 **验证** 和 **权限控制** 的 **类** 和 **接口**，远程支持和基本的供应接口

### `Remoting`包

和`Spring Remoting`整合

### `Web`包

包含过滤器和相关的网络安全的代码

### `Config`包

包含Spring security命名空间的解析代码和Java代码配置的相关代码

### `LDAP`包

LDAP相关的验证和整合

### `ACL`包

专门的域名访问控制列表的实现

### `CAS`包

单点登录的整合。

### `OpenId`包

`OpenId`的验证支持。

### `Test`包

测试的支持

## Spring Security配置

### Java配置

```java
import org.springframework.beans.factory.annotation.Autowired;

import org.springframework.context.annotation.*;
import org.springframework.security.config.annotation.authentication.builders.*;
import org.springframework.security.config.annotation.web.configuration.*;

@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

	@Bean
	public UserDetailsService userDetailsService() throws Exception {
		InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
		manager.createUser(User.withUsername("user").password("password").roles("USER").build());
		return manager;
	}
}
```

虽然没有很多配置，但是提供很多特性：

* 每条 `URL` 都需要验证
* 生成一个登录表单
* 允许用户名为user密码为password的用户通过验证。
* 允许用户登出
* `CSRF（跨站请求伪造）`的预防
* `Session fixation（固定SessionID漏洞攻击）`的预防
* Security Head整合
    
    1. `HTTP Strict Transport Security（HTTP严格安全传输）`
    2. `X-Content-Type-Options`的整合
    3. `Cache Control` (can be overridden later by your application to allow caching of your static resources)
    4. `X-XSS-Protection`整合
    5. `X-Frame-Options`整合防止`Clickjacking（点击劫持）`

* 和 `Servlet` API整合

    1. HttpServletRequest#getRemoteUser()
    2. HttpServletRequest.html#getUserPrincipal()
    3. HttpServletRequest.html#isUserInRole(java.lang.String)
    4. HttpServletRequest.html#login(java.lang.String, java.lang.String)
    5. HttpServletRequest.html#logout()

### AbstractSecurityWebApplicationInitializer

在`war`中注册`springSecurityFilterChain`

#### 不在spring容器中

```java
import org.springframework.security.web.context.*;

public class SecurityWebApplicationInitializer
	extends AbstractSecurityWebApplicationInitializer {

	public SecurityWebApplicationInitializer() {
		super(WebSecurityConfig.class);
	}
}
```

上面代码会做如下的事情

* Automatically register the springSecurityFilterChain Filter for every URL in your application
* Add a ContextLoaderListener that loads the WebSecurityConfig.

#### 和Spring MVC

```java
public class MvcWebApplicationInitializer extends
		AbstractAnnotationConfigDispatcherServletInitializer {

	@Override
	protected Class<?>[] getRootConfigClasses() {
		return new Class[] { WebSecurityConfig.class };
	}

	// ... other overrides ...
}
```

### HttpSecurity

`WebSecurityConfigurerAdapter`会提供一个默认的配置，通过`configure(HttpSecurity http)`方法

```java
protected void configure(HttpSecurity http) throws Exception {
	http
		.authorizeRequests()
			.anyRequest().authenticated()
			.and()
		.formLogin()
			.and()
		.httpBasic();
}
```

上面代码的作用：

* 确保任何请求都需要验证用户
* 允许用户通过表单验证
* 允许用户通过HTTP BASIC验证

`XML`的形式：

```xml
<http>
	<intercept-url pattern="/**" access="authenticated"/>
	<form-login />
	<http-basic />
</http>
```

### 配置登录页面

Spring Security会自动生成一个登录页面。配置自己的登录页面。

```java
protected void configure(HttpSecurity http) throws Exception {
	http
		.authorizeRequests()
			.anyRequest().authenticated()
			.and()
		.formLogin()
			.loginPage("/login")
			.permitAll();
}
```

### 请求权限验证

```java
protected void configure(HttpSecurity http) throws Exception {
    http
        .authorizeRequests()
            .antMatchers("/resources/**", "/signup", "/about").permitAll()
            .antMatchers("/admin/**").hasRole("ADMIN")
            .antMatchers("/db/**").access("hasRole('ADMIN') and hasRole('DBA')")
            .anyRequest().authenticated()
            .and()
        // ...
        .formLogin();
}
```

### 处理登出

当时使用`WebSecurityConfigurerAdapter`，登出会自动配置，默认访问`/logout`，进行以下操作：

* 使 `Http Session`失效
* 清理`Remember me`
* 清空`SecurityContextHolder`
* 重定向到`/login?logout`

```java
protected void configure(HttpSecurity http) throws Exception {
    http
        .logout()
            .logoutUrl("/my/logout")
            .logoutSuccessUrl("/my/index")
            .logoutSuccessHandler(logoutSuccessHandler)
            .invalidateHttpSession(true)
            .addLogoutHandler(logoutHandler)
            .deleteCookies(cookieNamesToClear)
            .and()
        ...
}
```

#### `LogoutHandler`

`LogoutHandler`在登出的时候会执行。他们应当被用来执行必要的清理，所以不能够抛出异常，几种提供的实现类

* `PersistentTokenBasedRememberMeServices`
* `TokenBasedRememberMeServices`
* `CookieClearingLogoutHandler`
* `CsrfLogoutHandler`
* `SecurityContextLogoutHandler`

#### `LogoutSuccessHandler`

`LogoutSuccessHandler`通过`LogoutFilter`在成功登出后调用，比如跳转到正确的地址。与`LogoutHandler`不同的是它可能抛出异常。实现类：

* `SimpleUrlLogoutSuccessHandler`
* `HttpStatusReturningLogoutSuccessHandler`

### 验证

#### In-Memory Authentication内存验证

```java
@Bean
public UserDetailsService userDetailsService() throws Exception {
    // ensure the passwords are encoded properly
    UserBuilder users = User.withDefaultPasswordEncoder();
    InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
    manager.createUser(users.username("user").password("password").roles("USER").build());
    manager.createUser(users.username("admin").password("password").roles("USER","ADMIN").build());
    return manager;
}
```

#### JDBC验证

```java
@Autowired
private DataSource dataSource;

@Autowired
public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
    // ensure the passwords are encoded properly
    UserBuilder users = User.withDefaultPasswordEncoder();
    auth
        .jdbcAuthentication()
            .dataSource(dataSource)
            .withDefaultSchema()
            .withUser(users.username("user").password("password").roles("USER"))
            .withUser(users.username("admin").password("password").roles("USER","ADMIN"));
}
```

#### LDAP验证

```java
@Autowired
private DataSource dataSource;

@Autowired
public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
    auth
        .ldapAuthentication()
            .userDnPatterns("uid={0},ou=people")
            .groupSearchBase("ou=groups");
}
```

#### AuthenticationProvider

实现`AuthenticationProvider`进行验证。

#### `UserDetailService`

实现`UserDetailService`接口

使用特定的密码加密方式

```java
@Bean
public BCryptPasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

### 多个`Multiple HttpSecurity`

```java
@EnableWebSecurity
public class MultiHttpSecurityConfig {
    @Bean
    public UserDetailsService userDetailsService() throws Exception {
        // ensure the passwords are encoded properly
        UserBuilder users = User.withDefaultPasswordEncoder();
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
        manager.createUser(users.username("user").password("password").roles("USER").build());
        manager.createUser(users.username("admin").password("password").roles("USER","ADMIN").build());
        return manager;
    }

    @Configuration
    @Order(1)
    public static class ApiWebSecurityConfigurationAdapter extends WebSecurityConfigurerAdapter {
        protected void configure(HttpSecurity http) throws Exception {
            http
                .antMatcher("/api/**")
                .authorizeRequests()
                    .anyRequest().hasRole("ADMIN")
                    .and()
                .httpBasic();
        }
    }

    @Configuration
    public static class FormLoginWebSecurityConfigurerAdapter extends WebSecurityConfigurerAdapter {

        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http
                .authorizeRequests()
                    .anyRequest().authenticated()
                    .and()
                .formLogin();
        }
    }
}
```

### 方法安全

#### `EnableGlobalMethodSecurity`

```java
@EnableGlobalMethodSecurity(securedEnabled = true)
public class MethodSecurityConfig {
// ...
}
```

```java
public interface BankService {

@Secured("IS_AUTHENTICATED_ANONYMOUSLY")
public Account readAccount(Long id);

@Secured("IS_AUTHENTICATED_ANONYMOUSLY")
public Account[] findAccounts();

@Secured("ROLE_TELLER")
public Account post(Account account, double amount);
}
```

使用`JSR-250`注解

```java
@EnableGlobalMethodSecurity(jsr250Enabled = true)
public class MethodSecurityConfig {
// ...
}
```

#### `GlobalMethodSecurityConfiguration`

当业务非常复杂，注解满足不了需求时，可以继承`GlobalMethodSecurityConfiguration`

```java
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class MethodSecurityConfig extends GlobalMethodSecurityConfiguration {
    @Override
    protected MethodSecurityExpressionHandler createExpressionHandler() {
        // ... create and return custom MethodSecurityExpressionHandler ...
        return expressionHandler;
    }
}
```

## Spring Security命名空间

命名空间是 Spring2.0 添加的功能。