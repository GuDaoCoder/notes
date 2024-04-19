---
tags:
- Maven
---

## 前言
本文主要介绍通过命令的方式在linux下安装Maven。

## 下载安装包

- 创建仓库下载目录

```shell
mkdir -p /home/maven/repo

```

- 下载安装包

```shell
wget https://dlcdn.apache.org/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz -P /home/maven

```

- 解压文件

```shell
cd /home/maven
tar zxvf apache-maven-3.9.6-bin.tar.gz

```

## 修改配置文件
这里主要是把仓库下载目录修改成之前仓库的路径，并且添加阿里云的镜像源

```shell
vim /home/maven/apache-maven-3.9.6/conf/settings.xml
```

```xml
<localRepository>/home/maven/repo</localRepository>
```

```xml
<mirror>
    <id>alimaven</id>
    <name>aliyun maven</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    <mirrorOf>central</mirrorOf>
</mirror>
```



## 配置环境变量

- 添加环境变量

```shell
echo '
export M2_HOME=/home/maven/apache-maven-3.3.9
export CLASSPATH=$CLASSPATH:$M2_HOME/lib
export PATH=$PATH:$M2_HOME/bin' >> /etc/profile

```

- 使变量生效

```shell
source /etc/profile

```

- 验证是否安装成功

```shell
[root@iZuf6bsr3117yu2qoyo5nvZ conf]# mvn -v
Apache Maven 3.9.6 (bc0240f3c744dd6b6ec2920b3cd08dcc295161ae)
Maven home: /home/maven/apache-maven-3.9.6
Java version: 17.0.10, vendor: Oracle Corporation, runtime: /opt/java/jdk17/jdk-17.0.10
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-1160.108.1.el7.x86_64", arch: "amd64", family: "unix"

```

