sonarqube是一个代码质量检测的工具。从7.9版本以后sonar不再支持MySQL的数据库，所以关系型数据库使用postgresql。（如果不配置关系型数据库会使用内置的DB2数据库，内置数据库无法迁移和升级。）



# 安装postgresql

所有的安装都是使用Docker进行安装，方便快捷。

```shell
# 拉取镜像
docker pull postgres

# 创建同步数据的目录
mkdir -p /apps/server_apps/postgressql/data

# 启动postgresql
docker run --restart=always --name postgresql -p 5432:5432 \
-e POSTGRES_USER=sonar -e POSTGRES_PASSWORD=yBUOpHoUMEKy2TJH -e POSTGRE_DB=sonar \
-v /apps/server_apps/postgressql/data:/var/lib/postgresql/data -d postgres
```



# 安装sonar
```shell
# 拉取镜像
docker pull sonarqube:lts

# 创建需要同步的目录
mkdir -p /apps/server_apps/sonarqube/data
mkdir -p /apps/server_apps/sonarqube/extensions
mkdir -p /apps/server_apps/sonarqube/logs
chmod -R 777 /apps/server_apps/sonarqube

# 启动窗口
docker run --restart=always --name sonarqube --link postgresql -p 9000:9000 \
-e SONAR_JDBC_URL=jdbc:postgresql://postgresql:5432/sonar \
-e SONAR_JDBC_USERNAME=sonar \
-e SONAR_JDBC_PASSWORD=yBUOpHoUMEKy2TJH \
-v /apps/server_apps/sonarqube/data:/opt/sonarqube/data \
-v /apps/server_apps/sonarqube/extensions:/opt/sonarqube/extensions \
-v /apps/server_apps/sonarqube/logs:/opt/sonarqube/logs \
-d sonarqube
```

> 使用 sudo docker logs -f sonarqube 查看启动日志。



在浏览器中输入: http://HOST:9000 进行访问。

使用admin: admin登录。



# 安装插件

在【administrator】-【marketPlace】

安装插件名: PMD, Checkstyle, Chinese Pack, Findbugs



# 扫描代码

扫描代码主要是扫描后端Java代码和前端的vue代码。

扫描之前需要在 Sonar 服务上配置好项目名和 [AuthenticationToken](https://docs.sonarqube.org/8.9/user-guide/user-token/)（用于连接服务鉴权）

## Maven 扫描 Java 项目

> [官网说明](https://docs.sonarqube.org/8.9/analysis/scan/sonarscanner-for-maven/)

### 配置 Maven

配置 Maven 的 Settings.xml 文件

```xml
<settings>
    <pluginGroups>
        <pluginGroup>org.sonarsource.scanner.maven</pluginGroup>
    </pluginGroups>
    <profiles>
        <profile>
            <id>sonar</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <!-- Optional URL to server. Default value is http://localhost:9000 -->
                <sonar.host.url>
                  http://myserver:9000
                </sonar.host.url>
            </properties>
        </profile>
     </profiles>
</settings>
```

### 扫描项目

```shell
mvn clean verify sonar:sonar -Dsonar.projectKey=XX-web -Dsonar.login=YourAuthenticationToken
```

> 也可以使用 ` -Dsonar.host.url=http://172.21.20.88:9000` 来指定服务器地址。更多参数: [analysis-parameters](https://docs.sonarqube.org/8.9/analysis/analysis-parameters/)

## Jenkins 扫描 Vue 项目

配置实现在部署前端 Vue 项目时，对项目进行扫描检测。

### 配置 Jenkins

1. 安装插件

    【系统管理】-【插件管理】

    安装 [SonarQube Scanner for Jenkins](https://plugins.jenkins.io/sonar) 插件。

2. 配置凭证

    【系统管理】-【系统配置】-【Manage Credentials】-【Jenkins】-【全局凭证(unrestricted)】-【添加凭证】

   类型选择“Secret text”；Secret 写在 Sonar 服务上配置的 Authentication token；ID为凭证ID，可以自动生成。

3. 配置 SonarQube Server

    【系统管理】-【系统配置】

    配置 SonarQube server 。填写服务地址，选择 Server authentication token（第2步添加的）。

### 配置项目任务

进入项目任务，选择配置

在【构建环境】中选择 “Prepare SonarQube Scanner environment” ，选择第2步添加的凭证。

![image-20210617230304028](https://gitee.com/clancy/images/raw/master/img/20210617230312.png)

【构建】-【增加构建步骤】。然后拉到最上面，最先执行。

【Excute SonarQube Scanner】

![image-20210617230440213](https://gitee.com/clancy/images/raw/master/img/20210617230442.png)

配置 Ayalysis properties（其它项可以为空）

```properties
#projectKey项目的唯一标识，不能重复
sonar.projectKey=sanbao-web
sonar.projectName=sanbao-web
sonar.sourceEncoding=UTF-8
sonar.sources=src/
# 排除不扫描的目录
sonar.exclusions=node_modules/**, dist/**
```

> 为了避免扫描`node_modules` 和 `dist` 不需要分析的文件夹，所以在snoar扫描之前要把这两个文件删除了，或者单独建立一个扫描的工程。
>
> 上面是单模块的配置方式，多模块的参考 [使用Jenkins持续集成Vue项目配置Sonar任务](https://www.chinacion.cn/article/1320.html) 和 [Analysis Parameters-sonar.projectBaseDir](https://docs.sonarqube.org/latest/analysis/analysis-parameters/)



【执行shell】编译项目

![image-20210617230529096](https://gitee.com/clancy/images/raw/master/img/20210617230531.png)

【send files or execute commands over SSH】把编译后的dist上传到服务器

![image-20210617230841765](https://gitee.com/clancy/images/raw/master/img/20210617230843.png)

【执行shell】删除 `node_modules` 目录，不然 sonar 会扫描该目录，会导致 jenkins OOM

![image-20210617230955881](https://gitee.com/clancy/images/raw/master/img/20210617230957.png)

## Jenkins 扫描 Java 项目

 	配置 Java 项目扫描与 Vue 项目就最后一步不一样。
 	
 	进入 Java 项目任务，选择配置

- 在【构建环境】中选择 “Prepare SonarQube Scanner environment” ，选择第2步添加的凭证。

- 【构建】-【增加构建步骤】-【调用顶层 Maven 目标】。然后把模块拉到最上面，最先执行。

  Maven 指令填写 `$SONAR_MAVEN_GOAL`

要先在【全局工具配置】中配置 maven 在服务器中 maven 安装目录，然后在【调用顶层 Maven 目标】时选择上一步添加的 maven ，不然会报找不到 `mvn` 指令。后面打包，上传jar文件，可以继续添加其它的构建工具。

![image-20210617231233689](https://gitee.com/clancy/images/raw/master/img/20210617231235.png)



# 参考

- [https://www.jianshu.com/p/3e34e24e3144](https://www.jianshu.com/p/3e34e24e3144)
- [https://docs.sonarqube.org/latest/setup/install-server/](https://docs.sonarqube.org/latest/setup/install-server/)
- [sonar扫描代码配置](https://docs.sonarqube.org/8.9/analysis/overview/)
- [Jenkins 配置使用 Sonar 扫描 Vue项目](https://www.chinacion.cn/article/1320.html)