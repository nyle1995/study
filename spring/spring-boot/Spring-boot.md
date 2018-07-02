# Spring Boot

- @SpringBootApplication
    - @Configuration
    - @EnableAutoConfiguration
        - import 了EnableAutoConfigurationImportSelector， 自动加载符合条件的Configuraion
            - 使用SpringFactoriesLoader来load
            - 搜索所有META-INF/spring.factories
    - @ComponentScan


### SpringApplication 执行流程

1. 判断是否是web工程，决定是AnnotationConfigEmbeddedWebApplicationContext 或者 ConfigurableWebApplicationContext
2. 创建配置Enviroment
3. 调用SpringApplictionRunListener的envriomentPrePared()
4. 展示Banner
5. 创建，初始化ApplicationContext
6. 调用ApplicationContextInitalizer 初始化
7. 调用SpringApplictionRunListener的contextPrepared
8. 将@enableautoConfigureation获取的配置 的ioc容器加载到ApplicationContext中国年
9. 调用SpringApplictionRunListener的contextLoaded
10. 调用ApplicationContext的refresh
11. 调用SpringApplictionRunListener的 finished

![](Springboot启动.png)