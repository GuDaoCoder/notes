---
tags:
- JDK17
---

## 前言
本文主要介绍通过命令的方式在linux下安装JDK17。

## 下载安装包

* 创建安装目录

```shell
mkdir -p /opt/java/jdk17
```

* 下载安装包

```shell
wget https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.tar.gz -P /opt/java/jdk17
```

* 解压文件

```shell
cd /opt/java/jdk17
tar zxvf jdk-17_linux-x64_bin.tar.gz
```

## 配置环境变量

* 添加环境变量

```shell
echo '
export JAVA_HOME=/opt/java/jdk17/jdk-17.0.10
export CLASSPATH=.:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar
export PATH=$JAVA_HOME/bin:$PATH' >> /etc/profile
```

* 使变量生效

```shell
source /etc/profile
```

* 验证是否安装成功

```shell
[root@iZuf6bsr3117yu2qoyo5nvZ jdk-17.0.10]# java -version
java version "17.0.10" 2024-01-16 LTS
Java(TM) SE Runtime Environment (build 17.0.10+11-LTS-240)
Java HotSpot(TM) 64-Bit Server VM (build 17.0.10+11-LTS-240, mixed mode, sharing)
```

