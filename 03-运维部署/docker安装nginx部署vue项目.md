## 前言
本文主要介绍如何通过docker安装nginx。

## docker安装nginx

### 安装临时nginx

先临时运行一个nginx容器，主要是为了获取里面的配置文件

```shell
docker run -d --name nginx -p 80:80 -p 443:443 nginx:latest
```

### 复制配置文件

在指定位置创建nginx目录

```shell
mkdir -p /home/docker/nginx
```

复制配置文件到宿主机目录，这里的9b7f001f4ea4是容器Id

```shell
docker cp 9b7f001f4ea4:/etc/nginx/nginx.conf /home/docker/nginx/
docker cp 9b7f001f4ea4:/etc/nginx/conf.d /home/docker/nginx/conf/
docker cp 9b7f001f4ea4:/usr/share/nginx/html /home/docker/nginx/html
docker cp 9b7f001f4ea4:/var/log/nginx/ /home/docker/nginx/logs/
```

### 正式安装nginx

删掉之前的临时nginx

```shell
docker stop 9b7f001f4ea4
docker rm 9b7f001f4ea4
```

重新启动nginx容器，并映射之前的配置文件

```shell
docker run -d  --name nginx -p 80:80 -p 443:443 -v /home/docker/nginx/nginx.conf:/etc/nginx/nginx.conf -v /home/docker/nginx/logs:/var/log/nginx -v /home/docker/nginx/html:/usr/share/nginx/html -v /home/docker/nginx/conf:/etc/nginx/conf.d --privileged=true     -e TZ=Asia/Shanghai nginx:latest
```

访问网页确认nginx正常启动

![image-20240409225821537](http://cdn.road4code.com/image-bed/20240409225821.png)

### 部署vue页面

将vue项目打包后的静态文件上传到之前的/home/docker/nginx/html目录下



![image-20240409230231190](http://cdn.road4code.com/image-bed/20240409230231.png)

