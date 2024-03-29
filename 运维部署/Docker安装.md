## 前言
Docker是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的[镜像](https://baike.baidu.com/item/%E9%95%9C%E5%83%8F/1574)中，然后发布到任何流行的 Linux或Windows操作系统的机器上，也可以实现虚拟化。容器是完全使用[沙箱]机制，相互之间不会有任何接口。以下操作步骤基于centos系统。
## Docker安装步骤
### 卸载旧版本
```shell
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```
### 安装包
```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
```
### 设置阿里镜像仓库
yum-config-manager是一个命令。包含在yum-utils这个包内。即，安装yum-utils之后才能运行yum-config-manager命令。
```shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
### 更新yum软件包索引
```shell
yum makecache fast
```
### 通过yum安装docker
Docker有两个分支版本：Docker CE和Docker EE，即社区版和企业版，这里安装Docker CE。
```shell
yum -y install docker-ce
```
### 启动docker
```shell
systemctl start docker
```
### 验证
```shell
docker ps
```
输出以下内容则表示安装成功
![](dockerps.png)
### 设置docker开机自启
```shell
systemctl enable docker.service
```
## 设置阿里云镜像加速
登录[容器镜像服务控制台](https://cr.console.aliyun.com/)，按照提示执行。
![](aliyun.png)
```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://cxyncpk9.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
## 安装docker-compose
docker-compose是Docker提供的一个命令行工具，用来定义和运行由多个容器组成的应用。使用compose，我们可以通过YAML文件声明式的定义应用程序的各个服务，并由单个命令完成应用的创建和启动。

### 下载docker-compose
从github下载，可能比较慢
```shell
curl -L "https://github.com/docker/compose/releases/download/2.10.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
如果实在下载不下来，可以通过科学上网访问[docker-compose](https://github.com/docker/compose/releases/)。下载自己系统对应的文件改名为docker-compose并上传到/usr/local/bin/目录下。
### 赋予权限
```shell
chmod +x /usr/local/bin/docker-compose
```
### 创建软链
```shell
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
### 验证是否安装成功
```shell
docker-compose version
```
## 常见问题
### 修改容器时区
有时候我们会遇到容器时间跟宿主机时间相差8个小时的情况，这是因为我们在创建容器时没有指定时区。通过以下方式我们可以在容器创建完成后修改时区。
#### 将宿主机器时区文件拷贝至容器
```shell
docker cp /usr/share/zoneinfo/Asia/Shanghai [容器ID]:/usr/share/zoneinfo/Asia/Shanghai
```
#### 创建链接
```shell
docker exec -it [容器ID] sh -c  'ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime'
```
#### 查看时间验证是否修改成功
```shell
docker exec -it [容器ID] sh -c  'date'
```
