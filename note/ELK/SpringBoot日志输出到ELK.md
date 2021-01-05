# SpringBoot日志输出到ELK

## 环境搭建

### 下载镜像

#### 手动下载ELK镜像

```shell
docker pull elasticsearch:6.4.0
docker pull logstash:6.4.0
docker pull kibana:6.4.0
```

### docker-compose

docker-compose理解成一个一次管理多个容器的工具。比如一个项目依赖 Redis, RabbitMQ, ES, 如果不借助 docker-compose 需要一个个地去配置，去启动，去关闭，非常麻烦，有了 docker-compose，可以统一管理起来。

#### 安装docker-compose

```shell
# 下载 docker-compose
curl -L https://get.daocloud.io/docker/compose/releases/download/1.24.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

# 修改权限
chmod +x /usr/local/bin/docker-compose

# 验证
docker-compose --version
```

### 配置ELK环境

#### docker-compose-env.yml 脚本

```yml
version: '3'

services:
  elasticsearch:
	image: elasticsearch:6.4.0
	# build: ./docker-images/es-ik-pinyin-6.4.0/
	container_name: elasticsearch
	environment:
	  - "cluster.name=elasticsearch" #设置集群名称为elasticsearch
	  - "discovery.type=single-node" #以单一节点模式启动
	  - "ES_JAVA_OPTS=-Xms512m -Xmx512m" #设置使用jvm内存大小
	volumes:
	  - /data/server-apps/elasticsearch/plugins:/usr/share/elasticsearch/plugins #插件文件挂载
	  - /data/server-apps/elasticsearch/data:/usr/share/elasticsearch/data #数据文件挂载
	ports:
	  - 9200:9200
	  
  kibana:
	image: kibana:6.4.0
	container_name: kibana
	links:
	  - elasticsearch:es #可以用es这个域名访问elasticsearch服务
	depends_on:
	  - elasticsearch #kibana在elasticsearch启动之后再启动
	environment:
	  - "elasticsearch.hosts=http://es:9200" #设置访问elasticsearch的地址
	ports:
	  - 5601:5601

  logstash:
    image: logstash:6.4.0
    container_name: logstash
    environment:
      - "xpack.monitoring.enabled=false" # 关闭 x-pack 的监控
    volumes:
      - /data/server-apps/logstash/logstash-springboot.conf:/usr/share/logstash/pipeline/logstash.conf #挂载logstash的配置文件
    depends_on:
      - elasticsearch #logstash在elasticsearch启动之后再启动
    links:
      - elasticsearch:es #可以用es这个域名访问elasticsearch服务
    ports:
      - 4560:4560
```

#### 配置ES

1. 设置系统内核参数，否则会因为内存不足无法启动

```shell
# 改变设置
sysctl -w vm.max_map_count=262144

# 使之立即生效
sysctl -p
```

2. 创建 /data/server-apps/elasticsearch/data 目录并设置权限，否则会因为无权限访问而启动失败。

```shell
# 创建目录
mkdir /data/server-apps/elasticsearch/data/

# 创建并改变该目录权限
chmod 777 /data/server-apps/elasticsearch/data
```

#### Logstash 配置文件

/data/server-apps/logstash/logstash-springboot.conf

```yaml
input {
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4560
    codec => json_lines
  }
}
output {
  elasticsearch {
    hosts => "es:9200"
    index => "springboot-logstash-%{+YYYY.MM.dd}"
  }
}
```

#### 启动ELK

```shell
docker-compose -f docker-compose-env.yml up -d
```

#### ES安装IK分词器

```shell
# 进入ES容器
docker exec -it elasticsearch /bin/bash

#此命令需要在容器中运行
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.4.0/elasticsearch-analysis-ik-6.4.0.zip

# 退出容器
Ctrl + p +q

# 重启ES
docker restart elasticsearch
```

#### 在 logstash 中安装 json_lines 插件

```shell
# 进入logstash容器
docker exec -it logstash /bin/bash
# 进入bin目录
cd bin/
# 安装插件
logstash-plugin install logstash-codec-json_lines
# 退出容器
Ctrl + p + q
# 重启logstash服务
docker restart logstash
```

#### 打开Kibana防火墙

```shell
# 查询是否开启
firewall-cmd --zone=public --query-port=5601/tcp
# 开启5601端口
firewall-cmd --zone=public --add-port=5601/tcp --permanent
firewall-cmd --zone=public --add-service=http --permanent
# 重新载入规则
firewall-cmd --reload
```

访问地址：http://127.0.0.1:5601/



## SpringBoot应用集成Logstash

### 加入logstash解析依赖

在pom.xml中加入logstash-logback-encoder的依赖

```xml
<!-- 集成logstash -->
<dependency>
	<groupId>net.logstash.logback</groupId>
	<artifactId>logstash-logback-encoder</artifactId>
	<version>${logstash.logback.encoder}</version>
</dependency>
```

### 配置logback

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!--
    see more:
        - https://juejin.im/post/5b51f85c5188251af91a7525#heading-8
        - https://juejin.im/post/5b128f326fb9a01e8b7814c4
    configuration四个属性
        scan:是否开启扫描检查配置文件，默认开启，当设置为TRUE时，每分钟扫描一次配置文件，文件加载新文件。
        scanPeroid: 设置scan的扫描间隔时间。如：scanPeroid="30 seconds"
        packagingData: 是否打印堆栈信息。
        debug: 是否开启实时查看运行状态。
-->
<configuration debug="true">
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    <include resource="org/springframework/boot/logging/logback/console-appender.xml"/>
    <!--<contextName>v2Log</contextName>-->
    <!--
        引用springBoot默认的配置文件base.xml， 里面包含defaults.xml，console-appender.xml（输出控制台），file-appender.xml日志文件存储
    -->
    <!--<include resource="org/springframework/boot/logging/logback/base.xml" />-->
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter"/>
    <conversionRule conversionWord="wex"
                    converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter"/>
    <conversionRule conversionWord="wEx"
                    converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter"/>

    <!-- log base path -->
    <springProperty scope="context" name="logPath" source="log.path" defaultValue="logs"/>

    <property name="APP_HOME" value="${logPath}"/>
    <property name="APP_NAME" value="customer"/>
    <property name="LOG_HOME_PATH" value="${APP_HOME}"/>
    <property name="DEBUG_LOG_FILE" value="${LOG_HOME_PATH}/${APP_NAME}"/>
    <property name="CONSOLE_LOG_PATTERN"
              value="${CONSOLE_LOG_PATTERN:-%clr(%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%t]){faint} %clr([%X{APP_KEY}]) %clr(%-10.50logger{39}-%line){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>
    <!--<property name="FILE_LOG_PATTERN" value="${FILE_LOG_PATTERN:-%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}} ${LOG_LEVEL_PATTERN:-%5p} ${PID:- } -&#45;&#45; [%t] %-40.40logger{39} : %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>-->
    <property name="FILE_LOG_PATTERN"
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} ${LOG_LEVEL_PATTERN:-%5p} ${PID:- } --- [%t] [%X{APP_KEY}] %-10.50logger{39}-%line : %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <appender name="DEBUG_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${DEBUG_LOG_FILE}.log</file>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <FileNamePattern>${DEBUG_LOG_FILE}.%d{yyyy-MM-dd}.log</FileNamePattern>
            <!-- 日志保留最大文件数 -->
            <MaxHistory>40</MaxHistory>
            <!-- 控制日志总大小 -->
            <totalSizeCap>40GB</totalSizeCap>
        </rollingPolicy>
        <!--
        filter其实是appender里面的子元素。它作为过滤器存在，执行一个过滤器会有返回DENY，NEUTRAL，ACCEPT三个枚举值中的一个。
            DENY：日志将立即被抛弃不再经过其他过滤器
            NEUTRAL：有序列表里的下个过滤器过接着处理日志
            ACCEPT：日志会被立即处理，不再经过剩余过滤器

        ThresholdFilter
        临界值过滤器，过滤掉低于指定临界值的日志。当日志级别等于或高于临界值时，过滤器返回NEUTRAL；当日志级别低于临界值时，日志会被拒绝。
        -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>
        <!--
        LevelFilter
        级别过滤器，根据日志级别进行过滤。如果日志级别等于配置级别，过滤器会根据onMath(用于配置符合过滤条件的操作)
        和 onMismatch(用于配置不符合过滤条件的操作)接收或拒绝日志
        下面配置是只打印info，其它都会deny拒绝
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        -->
    </appender>

    <!--输出到logstash的appender-->
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <!--可以访问的logstash日志收集端口-->
        <destination>127.0.0.1:4560</destination>
        <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder">
            <!--定义MDC中的字段名，会自动填充MDC中的值
				see more: https://github.com/logstash/logstash-logback-encoder#mdc-fields
			-->
            <includeMdcKeyName>appkey</includeMdcKeyName>
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <!-- 自定义字段 -->
            <customFields>{"project": "hwh-bus", "service_name": "${APP_NAME}"}</customFields>
        </encoder>
    </appender>

    <!--
        若设置 <logger > 后并且 additivity 设置值为 true 时就会进行传递导致 logger 打印日志后 root 又打印一次日志！
    -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="DEBUG_FILE"/>
        <appender-ref ref="LOGSTASH"/>
    </root>

</configuration>
```

主要是配置 `<appender name="LOGSTASH"></appender>` 及在 root 中配置输出 `<appender-ref ref="LOGSTASH"/>` 

启动项目日志即会输出到ES。

## Kibana查看日志

### 创建 index pattern

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-09-28_17-26-43.png)

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-09-28_17-28-00.png)

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-09-28_17-28-29.png)

### 查看日志

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-09-28_17-35-44.png)

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-09-28_17-36-30.png)



## Kibana安全

参考: [使用Nginx配置认证访问Kibana](https://github.com/LiQiongchao/tech-note/blob/master/note/ELK/%E4%BD%BF%E7%94%A8Nginx%E9%85%8D%E7%BD%AE%E8%AE%A4%E8%AF%81%E8%AE%BF%E9%97%AEKibana.md)



## 参考

> [使用Docker Compose部署SpringBoot应用](https://mp.weixin.qq.com/s?__biz=MzU1Nzg4NjgyMw==&mid=2247483800&idx=1&sn=b9e0b6c006bad05e4055a3c0bb61c815&scene=21#wechat_redirect)
>
> [mall在Linux环境下的部署（基于Docker Compose）](https://mp.weixin.qq.com/s/JYkvdub9DP5P9ULX4mehUw)
>
> [springboot应用整合elk实现日志收集](http://www.macrozheng.com/#/technology/mall_tiny_elk?id=springboot%e5%ba%94%e7%94%a8%e6%95%b4%e5%90%88elk%e5%ae%9e%e7%8e%b0%e6%97%a5%e5%bf%97%e6%94%b6%e9%9b%86)





