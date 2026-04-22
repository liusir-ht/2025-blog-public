## 问题背景
  最近想要研究一下AIOPS相关的东西，这篇文章记录一下从零开始搭建环境的记录，为后面回头做留存。

## 环境描述
|组件名称|组件版本| 组件作用|
|---|---|---|
|Tencent OS |3.3|腾讯云提供的Linux操作系统|
|Docker |Docker version 26.1.3, build b72abbb| 提供资源隔离、环境隔离的核心组件|
|Docker-compose| v2.23.0|提高docker部署容器的可维护性|

## 核心步骤

### Docker
1. 添加Docker Repo仓库
```
# 安装依赖
yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加 Docker 官方仓库
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

2. 安装Docker
```
yum install -y docker-ce docker-ce-cli containerd.io
```

3. 设置开机自启
```
# 启动 Docker 并设置开机自启
systemctl start docker
systemctl enable docker
```
4. 验证
```
docker --version
```

### Docker-compose

1. 下载
```
curl -L "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose
```
2. 验证
```
 docker-compose --version
```

## 结果验证
![](/blogs/TencentOS-install-docker/2b8afc05947f014f.png)


