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



# 参考

- [https://www.jianshu.com/p/3e34e24e3144](https://www.jianshu.com/p/3e34e24e3144)

- [https://docs.sonarqube.org/latest/setup/install-server/](https://docs.sonarqube.org/latest/setup/install-server/)