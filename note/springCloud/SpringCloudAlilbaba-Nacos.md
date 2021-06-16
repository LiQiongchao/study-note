# 概要

Nacos 是 alibaba 出的一款配置中心 + 服务注册与发现。替换 Euraka 和 Config 配置中心。

[Nacos官网](https://nacos.io/zh-cn/index.html)

[Nacos Github demo](https://github.com/alibaba/spring-cloud-alibaba/tree/master/spring-cloud-alibaba-examples/nacos-example)

[相应练习代码](https://github.com/LiQiongchao/nacos-learn)

# 启动（单机模式）

## 下载Nacos

当前推荐的稳定版本为1.4.2或2.0.1。使用最新版的[2.0.1](https://github.com/alibaba/nacos/releases/tag/2.0.1)

```shell
# 使用的是window版本，下载后解压
wget https://github.com/alibaba/nacos/releases/download/2.0.1/nacos-server-2.0.1.zip
```

##  配置数据库

配置mysql数据（不配置也可以，默认使用的是内置数据库）详见[部署手册](https://nacos.io/zh-cn/docs/deployment.html)

>  要求mysql版本: 5.6.5+

1. 初始化 conf 文件夹下的 nacos-mysql.sql 文件。
2. 修改 `conf/application.properties` 文件，增加 mysql 的配置

```properties
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://11.162.196.16:3306/nacos_devtest?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=nacos_devtest
db.password=youdontknow
```

## 启动Nacos

```shell
# Linux
sh startup.sh -m standalone

# windows
startup.cmd -m standalone
```



# 配置中心

> [Nacos Spring Cloud 快速开始【官网】](https://nacos.io/zh-cn/docs/quick-start-spring-cloud.html)
>
> [Nacos config【Github】](https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-config)

## 配置数据

### Namespace

Nacos 可以使用不同的 Namespace 来区分不同的运行环境，默认使用的是 public 。当然也可以使用 profiles 来区别。

### 支持自定义的 Group 配置

在没有明确指定 `${spring.cloud.nacos.config.group}` 配置的情况下， 默认使用的是 DEFAULT_GROUP 。如果需要自定义自己的 Group，可以通过以下配置来实现：

```properties
spring.cloud.nacos.config.group=DEVELOP_GROUP
```

> 该配置必须放在 bootstrap.properties 文件中。并且在添加配置时 Group 的值一定要和 `spring.cloud.nacos.config.group` 的配置值一致。

### 配置Nacos数据

配置文件名格式: ${spring.application.name}-${profile}.${file-extension}

> ${spring.application.name}: 是在项目中 application.properties 中配置的
>
> ${profile}: 是在项目中 application.properties 中配置的 `spring.profiles.active`的值。
>
> ${file-extension}: 是在项目中 bootstrap.properties 中配置的 `spring.cloud.nacos.config.file-extension`。默认使用 properties 格式，一般修改为 yaml 格式。
>
> 配置格式类似于 SpringBoot 的 profile 配置，如 `nacos-config.yaml` 不管使用哪个 profile 都会加载。`nacos-config-dev.yaml` ，只能当`spring.profiles.active=dev`的时候才会被激活加载。

nacos-config.yaml

```yaml
Data ID:        nacos-config.yaml

Group  :        DEFAULT_GROUP

配置格式:        YAML

配置内容:        user.name: nacos-config-yaml
                user.age: 68
```

nacos-config-dev.yaml

```yaml
Data ID:        nacos-config-dev.yaml

Group  :        DEFAULT_GROUP

配置格式:        YAML

配置内容:        current.env: develop-env
```

nacos-config-prod.yaml

```yaml
Data ID:        nacos-config-prod.yaml

Group  :        DEFAULT_GROUP

配置格式:        YAML

配置内容:        current.env: product-env
```

### 配置程序

> 可以使用 alibaba 官方提供的[Java工种脚手架](https://start.aliyun.com/bootstrap.html)快速生成基础项目

bootstrap.properties

```yaml
# Nacos帮助文档: https://nacos.io/zh-cn/docs/concepts.html
# Nacos认证信息
#spring.cloud.nacos.config.username=nacos
#spring.cloud.nacos.config.password==nacos
#spring.cloud.nacos.config.contextPath=/nacos

# 配置支持yaml，默认支持properties。需要与配置中心的格式相对应
spring.cloud.nacos.config.file-extension=yaml

# 配置 Namespace, 默认是 public，NamespaceId 可以在命名空间列表查看
#spring.cloud.nacos.config.namespace=${NamespaceId}

# 动态刷新 nacos 配置（当配置中心修改后，服务可以自动加载配置），默认开
spring.cloud.nacos.config.refresh-enabled=true

# 设置配置中心服务端地址
spring.cloud.nacos.config.server-addr=127.0.0.1:8848
# Nacos 配置中心的namespace。需要注意，如果使用 public 的 namcespace ，请不要填写这个值，直接留空即可
# spring.cloud.nacos.config.namespace=
```

application.properties

```yaml
# 应用名称
spring.application.name=nacos-config

# 配置使用的profile
#spring.profiles.active=dev
spring.profiles.active=prod
```

NacosConfigApplication.java

```java
@SpringBootApplication
public class NacosConfigApplication {

    private static ConfigurableEnvironment environment;

    public static void main(String[] args) throws InterruptedException {
        ConfigurableApplicationContext applicationContext = SpringApplication.run(NacosConfigApplication.class, args);
        while(true) {
            //当动态配置刷新时，会更新到 Enviroment中，因此这里每隔一秒中从Enviroment中获取配置
            String userName = applicationContext.getEnvironment().getProperty("user.name");
            environment = applicationContext.getEnvironment();
            String userAge = environment.getProperty("user.age");
            String env = environment.getProperty("current.env");

            System.err.println("in "+env+" environEnv; user name :" + userName + "; age: " + userAge);
            TimeUnit.SECONDS.sleep(1);
        }
    }

}
```

启动 `NacosConfigApplication` 会配置不同的profile加载不同的数据。

```tex
# prod
in product-env environEnv; user name :nacos-config-yaml; age: 68

# dev
in dev-env environEnv; user name :nacos-config-yaml; age: 68

# 在配置中心修改完数据后，程序不需要重启，会动态加载新的配置数据
in develop-env environEnv; user name :nacos-config-yaml; age: 68
```

## 关闭配置中心功能

通过设置 spring.cloud.nacos.config.enabled = false 来完全关闭 Spring Cloud Nacos Config



# 服务发现











