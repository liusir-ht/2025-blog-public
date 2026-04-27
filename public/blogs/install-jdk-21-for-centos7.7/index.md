## 背景信息
  随着业务的发展，JAVA服务的站点需要使用新版本的JDK功能特性，针对讨论使用JDK21版本进行安装部署。

## 环境信息



| 组件名称 | 组件版本 | 组建功能 |
| --- | --- | --- |
| Centos  | 7.7 | Linux操作系统 |
| JDK | jdk-21.0.11| JAVA服务的依赖环境和编译环境 |

## 安装步骤

1. 下载JDK21的压缩包
```
wget https://download.oracle.com/java/21/latest/jdk-21_linux-x64_bin.tar.gz
```

2. 解压到java的常用目录
```
mkdir -p /usr/lib/jvm
tar -zxvf jdk-21_linux-x64_bin.tar.gz -C /usr/lib/jvm/
cd /usr/lib/jvm/
mv jdk-21.0.11/ java-21
```

3. 业务启动脚本添加以下内容（项目隔离）
```
export JAVA_HOME=/usr/lib/jvm/java-21
export PATH=$JAVA_HOME/bin:$PATH
```

## 结果验证

![](/blogs/install-jdk-21-for-centos7.7/314b758cbda97ed7.png)


## 总结
  在 CentOS 7 7.7 上安装 JDK 21 需要注意系统仓库限制，官方并未提供对应软件包，因此需要手动安装。
  
## 扩展知识
安装JDK21（系统级生效）
```
vim /etc/profile
在文件末尾添加：
export JAVA_HOME=/usr/lib/jvm/java-21
# 默认使用 JDK 21
export PATH=$JAVA_HOME/bin:$PATH
source /etc/profile #配置生效
java -version #验证
javac -version #验证
```


