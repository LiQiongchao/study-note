## 配置Nginx代理

```properties
    server {
        listen       5605;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            #proxy_buffering off;
            proxy_pass http://localhost:5601;

            # 配置账号密码访问
            # 提示信息
            auth_basic 'login kibana!';
            # 配置账号密码文件，也可以使用绝对路径
            auth_basic_user_file password/httpPasswd;
            autoindex on;
        }

    }
```



## 生成用户名与密码

```shell
printf "YOUR_USERNAME:$(openssl passwd -crypt YOUR_PASSWD)\n" >> /usr/local/openresty/nginx/conf/password/httpPasswd
```



## 使用用户名与密码访问

![](https://gitee.com/clancy/images/raw/master/img/20201012161729.png)

在用户名与密码中输入上一步生成的用户名与密码即可登录功能！







