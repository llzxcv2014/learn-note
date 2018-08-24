# Spring Boot Note

## Spring boot初探

### 基本的springboot注解

1. `@SpringBootApplication`    
    package: org.springframework.boot.autoconfigure   
    作用：表明是一个SpringBoot应用。
2. `@SpringBootConfiguration`     
    package: org.springframework.boot    
    被Spring的`@Configuration`标记
3. `@EnableAutoConfiguration`    
    package:org.springframework.boot.autoconfigure    
    作用：开启在Spring应用中可以自动配置需要的类。
    原理：
    ```java
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @AutoConfigurationPackage
    @Import(AutoConfigurationImportSelector.class)
    public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";


	Class<?>[] exclude() default {};

	String[] excludeName() default {};

    }
    ```
    `@AutoConfigurationPackage`自动配置包
    ```java
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @Import(AutoConfigurationPackages.Registrar.class)
    public @interface AutoConfigurationPackage {

    }
    ```
    `@Import`注解是Spring的底层注解，给容器中导入一个组件
    @Import的Javadoc对这个注解的解释
    > Indicates one or more @Configuration classes to import.(与Spring XML中的`<import>`标签作用相同)