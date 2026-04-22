## 问题背景
  最近在研究AIOPS，准备使用PostgreSQL数据库进行存储告警信息、动作记录、行为操作等，尽量保持方便快捷的方式。

## 环境介绍
|组件名称|组件版本| 组件作用|
|---|---|---|
|Tencent OS |3.3|腾讯云提供的Linux操作系统|
|Docker |Docker version 26.1.3, build b72abbb| 提供资源隔离、环境隔离的核心组件|
|Docker-compose| v2.23.0|提高docker部署容器的可维护性|
|PostgreSQL| 16|新兴的关系数据库|


## 核心步骤

1. 创建数据目录
```
mkdir /data/postgres
```

2. 配置Docker-compose.yml文件
```
version: '3.8'

services:
  # PostgreSQL（独立使用）
  postgres:
    image: pgvector/pgvector:pg16
    container_name: my-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: mydb
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    ports:
      - "5432:5432"
```
3. 启动pg
```
# 拉取镜像并启动
docker-compose up -d

# 查看启动日志
docker-compose logs -f
```

## 结果验证
```
[root@xxxx ~]# docker exec my-postgres pg_isready -U admin
/var/run/postgresql:5432 - accepting connections
```