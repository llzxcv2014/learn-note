# Spring Base Note

## Spring注解方式(`@Configuration`)

```java
@Configuration
public class AppConfig {
    
    @Bean
    public MyBean myBean() {
        // 实例化、配置、返回
    }
}
```
通过`AnnotationConfigApplicationContext`启动注解类的Spring容器，web环境下通过`org.springframework.web.context.support.AnnotationConfigWebApplicationContext`启动。

```java
 AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
 ctx.register(AppConfig.class);
 ctx.refresh();
 MyBean myBean = ctx.getBean(MyBean.class);
 // use
```

### 通过Spring XML方式 
```xml
<beans>
    <context:annotation-config/>
    <bean class="com.xxx.AppConfig"/>
 </beans>
```

### 为注册的Bean设置具体的值
通过`org.springframework.core.env.Environment`
```java
@Configuration
public class AppConfig {
    @AutoWired Environment env;

    @Bean
    public MyBean myBean() {
        MyBean myBean = new MyBean();
        myBean.setName(env.getProperty("bean.name"));
        return myBean;
    }
}
```

### 通过`@PropertySource`获取properties文件的属性值
```java
@Configuration
@PropertySource("classpath:/com/xxx/app.properties")
public class AppConfig {
   
    @Autowired Environment env;

    @Bean
    public MyBean myBean() {
        return new MyBean(env.getProperty("bean.name"));
    }
}
```

### 使用`@Value`注解
```java
@Configuration
@PropertySource("classpath:/com/acme/app.properties")
public class AppConfig {
    @Value("${bean.name}") String beanName;

    @Bean
    public MyBean myBean() {
        return new MyBean(beanName);
    }
}
```

### 使用`@Import`
可以引入其他的配置。
```java
@Configuration
public class DatabaseConfig {

    public DataSource dataSource() {
        //...
    }
}

@Configuration
@Import(DatabaseConfig.class)
public class AppConfig {

    private final DatabaseConfig dataConfig;

    public AppConfig(DatabaseConfig dataConfig) {
        this.dataConfig = dataConfig;
    }

    @Bean
    public MyBean myBean() {
        return new MyBean(dataConfig.dataSource());
    }
}
```

### 使用`@Profile`标签

`@Profile`标签可以区分生产环境和开发环境
```java
@Profile("development")
@Configuration
public class EmbeddedDatabaseConfig {
    @Bean
    public DataSource dataSource() {
        //...
    }
}

@Profile("production")
@Configuration
public class ProductionDatabaseConifg {

    @Bean
    public DataSource dataSource() {
        //...
    }
}
```
也可以在方法上使用该注解。

### 使用`@ImportSource`引入Spring XML的注解
在使用Java类作为Spring的配置时可以使用`@ImportSource`注解引入
```java
@Configuration
@ImportSource("classpath:/com/xxx/database-config.xml")
public class AppConfig {
    @Override DataSource dataSource;

    @Bean
    public MyBean myBean() {
        return new MyBean(this.dataSource);
    }
}
```

### 内部类
`@Configuration`可以标注内部类。
```java
@Configuration
public class AppConfig {
    @Override DataSource dataSource;

    public MyBean myBean() {
        return new MyBean(dataSource);
    }

    @Configurations
    static class DatabaseConfig {
        
        @Bean
        DataSource dataSource() {
            return new EmbeddedDatabaseBuilder().build();
        }
    }
}
```

### 配置延迟初始化
在默认情况下，被`@Bean`标签注解的方法会在容器启动的时候直接实例化。可以使用`@Lazy`注解配置延迟实例化，也可以标记与方法上。


### 测试环境
可以通过Spring-test模块使用Spring TextContext框架，用`@ContextConfiguration`注解便可使用。
```java
@RunWith(SpringJunit4ClassRunner.class)
@ContextConfiguration(classes={AppConfig.class, DatabaseConfig.class})
public class MyTests {

    @Autowired MyBean myBean;

    @AutoWired DataSource dataSource;

    @Test
    public void test() {
        //
    }
}
```

### 使用`@Configuration`的注意点

* __Configuration注解类必须通过类不能通过factory的方式获取，为了可以在运行时通过子类扩展。__
* __类不能是`final`__
* __类不能是本地的（如：不能在一个方法里声明）__
* __任何内部类的配置类必须用`static`声明__
* __被`@Bean`标注的方法再返回一个配置类（任何这种类会被视为一般的bean，它们的`@Configuraiton`不会被察觉）__ 
