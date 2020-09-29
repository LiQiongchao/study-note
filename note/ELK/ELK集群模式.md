# 自定义ES，Logstash镜像

下载方式有两种，一种是手动下载，启动后，进入到容器中ES下载分词器，Logstash下载Json解析工具。二就是使用Dockerfile在原有镜像的基础上打包成一个新镜像，直接启动容器即可，无需再进入容器下载分词器，Json解析等工具。

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

### docker-compose.yml

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







































# 参考

> [ElasticSearch优化系列一：集群节点规划](https://www.jianshu.com/p/4c57a246164c)