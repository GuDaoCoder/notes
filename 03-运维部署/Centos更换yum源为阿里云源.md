---
tags:
- centos
- yum
- 运维
---
## 前言
CentOS，是基于Red Hat Linux提供的可自由使用源代码的企业级Linux发行版本；是一个稳定，可预测，可管理和可复制的免费企业级计算平台。<!-- more -->
## 参考文档
[https://developer.aliyun.com/mirror/centos?spm=a2c6h.13651102.0.0.3e221b111V0STq](https://developer.aliyun.com/mirror/centos?spm=a2c6h.13651102.0.0.3e221b111V0STq)
## 具体步骤
### 备份原来的源
```shell
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```
### 根据系统版本和工具下载对应的阿里云源
#### centos6
```shell
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-vault-6.10.repo
```
```shell
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-vault-6.10.repo
```
#### centos7
```shell
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
```
```shell
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
```
#### centos8
```shell
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-vault-8.5.2111.repo
```
```shell
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-vault-8.5.2111.repo
```
### 生成缓存
```shell
yum makecache
```
