# 自定义ES，Logstash镜像

下载方式有两种，

一种是手动下载，启动后，再进入到容器中ES下载分词器，Logstash下载Json解析工具。

二就是使用Dockerfile在原有镜像的基础上打包成一个新镜像，直接启动容器即可，无需再进入容器下载分词器，Json解析等工具。

## 使用Dockfile下载打包镜像

### es 的 Dockfile

```shell
FROM elasticsearch:6.4.0

ENV VERSION=6.4.0

RUN sh -c '/bin/echo -e "y" | ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v${VERSION}/elasticsearch-analysis-ik-${VERSION}.zip'
RUN sh -c '/bin/echo -e "y" | ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v${VERSION}/elasticsearch-analysis-pinyin-${VERSION}.zip'
```

### logstash 的 Dockfile

```shell
FROM logstash:6.4.0

RUN sh -c '/bin/echo -e "y" | ./bin/logstash-plugin install logstash-codec-json_lines'
```

可以先使用 `docker build -t myName:1.0.0 .` 来打包镜像，也可以使用 docker-compose 直接打包启动。

### 使用 docker-compose 打包

关于使用 docker-compose 打包镜像可以参考下面配置。

> [SpringBoot日志输出到ELK]: SpringBoot日志输出到ELK.md

> build 配置查看[官网说明](https://docs.docker.com/compose/compose-file/#build)

```yaml
version: '3'

services:
  elasticsearch:
	# build 后面指定 Dockerfile 的路径
	build:
	  context: ./es-ik-pinyin-6.4.0/
	  dockerfile: Dockerfile-name # 如果名字是 Dockerfile 可以省略
	image: oio-es:1.0.0
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
```



# 配置文件

配置文件主要是 docker-compose, logstahsh, kibana的配置文件。es的配置文件是通过 compose 的 environment 配置给覆盖的。

## docker-compose.yml

```yaml
version: '3'

services:
  es01:
	image: oio-es:1.0.0
	container_name: es01
	logging:
	  options:
		max-size: "3g" # 设置日志文件大小最大为3G
		max-file: "5"  # 设置日志文件个数最大为5个
	environment:
	  - "cluster.name=oio-elasticsearch" #设置集群名称为elasticsearch
	  - "node.master=true"
	  - "node.data=false"
	  - "discovery.zen.ping.unicast.hosts=es02, es03"
	  - "ES_JAVA_OPTS=-Xms512m -Xmx512m" #设置使用jvm内存大小
	volumes:
	  - /data/server-apps/oio-elasticsearch/data01:/usr/share/elasticsearch/data #数据文件挂载
	ports:
	  - 9201:9200
	  - 9301:9300
  es02:
	image: oio-es:1.0.0
	container_name: es02
	logging:
	  options:
		max-size: "3g"
		max-file: "5"
	environment:
	  - "cluster.name=oio-elasticsearch" #设置集群名称为elasticsearch
	  - "discovery.zen.ping.unicast.hosts=es01, es03"
	  - "ES_JAVA_OPTS=-Xms512m -Xmx512m" #设置使用jvm内存大小
	volumes:
	  - /data/server-apps/oio-elasticsearch/data02:/usr/share/elasticsearch/data #数据文件挂载
	ports:
	  - 9202:9200
	  - 9302:9300
  es03:
	image: oio-es:1.0.0
	container_name: es03
	logging:
	  options:
		max-size: "3g"
		max-file: "5"
	environment:
	  - "cluster.name=oio-elasticsearch" #设置集群名称为elasticsearch
	  - "discovery.zen.ping.unicast.hosts=es01, es02"
	  - "ES_JAVA_OPTS=-Xms512m -Xmx512m" #设置使用jvm内存大小
	volumes:
	  - /data/server-apps/oio-elasticsearch/data03:/usr/share/elasticsearch/data #数据文件挂载
	ports:
	  - 9203:9200
	  - 9303:9300
          
  kibana:
	image: kibana:6.4.0
	container_name: kibana
	logging:
	  options:
		max-size: "3g"
		max-file: "5"
	# links:
	  # - es01:es #可以用es这个域名访问elasticsearch服务
	depends_on:
	  - es01 #kibana在elasticsearch启动之后再启动
	  - es02
	  - es03
	volumes:
	  - /data/server-apps/kibana/kibana.yml:/usr/share/kibana/config/kibana.yml:rw
	ports:
	  - 5601:5601

  logstash-bus:
    image: oio-logstash:1.0.0
    container_name: logstash-bus
    logging:
      options:
        max-size: "3g"
        max-file: "5"
    volumes:
      - /data/server-apps/logstash/logstash-bus.conf:/usr/share/logstash/pipeline/logstash.conf #挂载logstash的配置文件
    depends_on:
      - es01 #logstash在elasticsearch启动之后再启动
      - es02
      - es03
    environment:
      - "xpack.monitoring.enabled=false" # 关闭 x-pack 的监控
    ports:
      - 4560:4560
      - 4561:4561
      - 4562:4562
      - 4563:4563
      - 4565:4565
```



## logstash配置文件

logstash-bus.conf

```json
input {

  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4560
    codec => json_lines
    type => "customer-test"
  }
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4561
    codec => json_lines
    type => "receiver-test"
  }
  # 存储开发环境的所有的日志
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4565
    codec => json_lines
    type => "dev"
  }

}

filter{
  #if [type] == "customer-test" {
  #  mutate {
  #    remove_field => "port"
  #    remove_field => "host"
  #    remove_field => "@version"
  #  }
  #  json {
  #    source => "message"
  #    remove_field => ["message"]
  #  }
  #}
  # 把message保存至source方便搜索 
  
  json {
        source => "message"
        remove_field => ["message"]
  }
}

output {
  elasticsearch {
    hosts => ["es01:9200", "es02:9200", "es03:9200"]
    action => "index"
    codec => json
    index => "logstash-bus-%{type}-%{+YYYY.MM.dd}"
    template_name => "logstash-bus"
  }
}
```

> input 是配置对外信息
>
> output 的 hosts 配置的是各个es的服务器。

上面logstash是以模块来区分的，同样也可以以日志级别来区分。



## Logback配置

```xml
    <!--输出到logstash的appender-->
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <!--可以访问的logstash日志收集端口-->
        <destination>${logLogstash}</destination>
        <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- 自定义字段 -->
            <customFields>{"project": "hwh-bus", "service_name": "${APP_NAME}"}</customFields>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="LOGSTASH"/>
    </root>
```



## kibana配置文件

kibana.yml

```yaml
server.name: kibana
server.host: "0"
elasticsearch.url: http://es01:9200
xpack.monitoring.ui.container.elasticsearch.enabled: false
```

> 本来是不打算配置 kibana 的配置文件的，想着使用 compose 的 environment 把 `elasticsearch.url` 给覆盖掉，后来发现无法覆盖。所以又增加了个 kibana 的配置文件。



# 启动

使用 docker-compose 的命令启动

```shell
docker-compose -f docker-compose.yml up -d
```

> -f : 是指定 compose 的配置文件名，如果是 docker-compose.yml 和 docker-compose.yaml，可以不指定。
>
> up : 表示启动
>
> -d : 表示后台启动，否则就以当前窗口启动。



# kibana 配置

参考另一篇 [SpringBoot日志输出到ELK](SpringBoot日志输出到ELK.md)



# Kibana安全

参考: [使用Nginx配置认证访问Kibana](https://github.com/LiQiongchao/tech-note/blob/master/note/ELK/%E4%BD%BF%E7%94%A8Nginx%E9%85%8D%E7%BD%AE%E8%AE%A4%E8%AF%81%E8%AE%BF%E9%97%AEKibana.md)



# 参考

> [你居然还去服务器上捞日志，搭个日志收集系统难道不香么！](http://www.macrozheng.com/#/reference/mall_elk_advance)
>
> [ElasticSearch优化系列一：集群节点规划](https://www.jianshu.com/p/4c57a246164c)
>
> [elasticsearch-6.4.0 reference](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/discovery-settings.html)
>
> [kibana-6.4.0在docker中的配置【官网】](https://www.elastic.co/guide/en/kibana/6.4/docker.html)
>
> [docekr-compose官网environment的配置](https://docs.docker.com/compose/environment-variables/#set-environment-variables-in-containers)
>
> [logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder#context-fields)