Gitlab是一个类似于Github的代码管理工具，为了安全，一般会自己搭建一个Gitlab服务器来管理公司代码，而不选择放在Github上，因为Github私有库是收费的。

下面就使用 Docker 快速搭建一个 Gitlab。

# 安装GitLab

## 下载Gitlab的Docker镜像

```shell
docker pull gitlab/gitlab-ce
```

## 启动Gitlab

Gitlab的http服务端口映射到宿主机的1080端口上，ssh端口映射到宿主机的1022端口上，这里我们将Gitlab的配置，日志以及数据目录映射到了宿主机的指定文件夹下，防止我们在重新创建容器后丢失数据。

```shell
docker run --detach \
  --publish 10443:443 --publish 4080:80 --publish 1022:22 \
  --name gitlab \
  --restart always \
  --volume /data/server-apps/gitlab/config:/etc/gitlab \
  --volume /data/server-apps/gitlab/logs:/var/log/gitlab \
  --volume /data/server-apps/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ce:latest
```

## 开启 Gitlab 的 http 及 ssh 的映射端口

```shell
# 开启4080端口
firewall-cmd --zone=public --add-port=4080/tcp --permanent 
# 开启1022端口
firewall-cmd --zone=public --add-port=1022/tcp --permanent 
# 重启防火墙才能生效
systemctl restart firewalld
# 查看已经开放的端口
firewall-cmd --list-ports
```



## 访问

在浏览器中输入 `http://your-host:port` 访问GitLab。Gitlab启动比较慢，未启动成功前访问会出现502的错误。

**第一次访问时，会让重置 root 用户的密码**，重置后，用 root 用户登录即可。

![](https://gitee.com/clancy/images/raw/master/img/20201014164144.png)

# 配置Gitlab

主要是基本的使用为主。如，创建组，项目，用户等。

## 配置用户

### 创建用户

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-10-14_15-32-39.png)

### 填写用户信息

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-10-14_15-34-53.png)

### 配置用户密码

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-10-14_15-37-48.png)

## 配置中文

可以根据自己的需求选择汉化，Gitlab 自带中文翻译功能。目前使用的13.12.0版本是支持的，不过汉化率只有72%，还为实验特性。

【个人中心】——【Preferences】——【Localization】

![image-20210527104233978](https://gitee.com/clancy/images/raw/master/img/image-20210527104233978.png)

【Save changes】后按 F5 刷新即可！

## 配置组与项目

### 配置组

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-10-14_15-28-23.png)

![](https://gitee.com/clancy/images/raw/master/img/20201014173133.png)



### 配置项目

![](https://gitee.com/clancy/images/raw/master/img/20201014164819.png)



![](https://gitee.com/clancy/images/raw/master/img/20201014173220.png)



# 使用 http 与 SSH 操作项目

Windows客户端GIT就是傻瓜式安装。

## 使用 http 操作

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-10-14_16-51-28.png)

复制地址之后把 HOST 部分修改成自己服务器的 IP 与端口。

```shell
# clone 项目到本地
http://192.168.31.120:4080/group-hello/hello-world.git

# 进入到项目目录
cd hello-world/

# 创建文件
echo hello Gitlab >> test.txt

# 提交到服务器
git add .
git commit -am 'test'
git push
```

刷新 Gitlab 的项目页，即可看到新提交的文件 text.txt

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-10-14_16-56-22.png)

## 使用 SSH 操作

使用 http 操作 Gitlab 相对简单，使用 SSH 方式相对麻烦些。

### 配置 SSH 公钥

把 `~/.ssh/id_rsa.pub` 内容复制。然后粘贴到 Gitlab 个人设置的 ssh-key中，写入 title 标题，然后添加即可。

![](https://gitee.com/clancy/images/raw/master/img/Snipaste_2020-10-14_17-16-49.png)



### ~~配置Clone的默认端口~~

正常情况下复制 ssh 的地址是不包含端口的，如：`ssh://git@c59c7990905c:group-hello/hello-world.git`

而配置后是 `ssh://git@c59c7990905c:1022/group-hello/hello-world.git`

这一步也可以不配置，因为后面在服务器配置完 ssh 端口后，无须再显示的指定端口。

修改 Gitlab 的配置文件

`vim /data/server-apps/gitlab/config/gitlab.rb `

```properties
### GitLab Shell settings for GitLab
# gitlab_rails['gitlab_shell_ssh_port'] = 22
gitlab_rails['gitlab_shell_ssh_port'] = 1022
```

重启 Gitlab 的容器

```shell
docker restart gitlab
```

启动时间较长，需要等一会再去访问，否则会报502的错误。

### 配置本地访问 SSH 的默认端口

这时候如果去 clone 项目会报22端口服务器拒绝。

这时候需要编辑 `~/.ssh/config ` 文件，config 文件不存在就创建一个。

```yaml
# host c59c7990905c
host 192.168.31.120
	user git
	hostname 192.168.31.120
	port 1022
```

也可以把 `host` 的值改的与 Gitlab 的 host 一致，那样就不用修改在 Gitlab 上复制的 ssh 的地址了，直接可以使用。

```shell
# 因为已经在 ~/.ssh/config 中配置端口，无须再指定端口了。
# git clone git@c59c7990905c:group-hello/hello-world.git
git clone git@192.168.31.120:group-hello/hello-world.git
```

这时候使用 ssh 的方式也可以像 http 方式一样操作 Gitlab 库了。

## 配置域名访问

- 配置配置文件

`vim /data/server-apps/gitlab/config/gitlab.rb`

```properties
## GitLab URL
##! URL on which GitLab will be reachable.
##! For more details on configuring external_url see:
##! https://docs.gitlab.com/omnibus/settings/configuration.html#configuring-the-external-url-for-gitlab
##!
##! Note: During installation/upgrades, the value of the environment variable
##! EXTERNAL_URL will be used to populate/replace this value.
##! On AWS EC2 instances, we also attempt to fetch the public hostname/IP
##! address from AWS. For more details, see:
##! https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html
# external_url 'GENERATED_EXTERNAL_URL'
external_url 'http://gitlab.haowaihao.com.cn/'
```

- 配置Nginx代理

```properties
    server {
        listen       80;
        server_name  gitlab.haowaihao.com.cn;

        #防止代码上传代码太大，无法上传
        client_max_body_size 128M;
        
        #charset koi8-r;
        #access_log  logs/host.access.log  main;

        location / {
            proxy_pass http://127.0.0.1:4080;
            # root   html;
            # index  index.html index.htm;
        }
    }
```

> 代码太大无法上传时会如下错误：
>
> error: RPC failed; HTTP 413 curl 22 The requested URL returned error: 413
> fatal: the remote end hung up unexpectedly







