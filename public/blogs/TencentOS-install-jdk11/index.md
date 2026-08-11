## 背景

业务上有一个新服务，需要使用 JDK 11 运行。服务器默认安装的 Java 版本较低，不符合业务要求，因此需要在 TencentOS Server 3.3 上安装 JDK 11.0.18，并使用 `alternatives` 命令切换系统默认的 Java 版本。

## 环境介绍

| 组件名称 | 组件版本 | 组件介绍 |
| ------ | ------ | ------------ |
| TencentOS Server | 3.3 | 腾讯云开源的 Linux 服务器操作系统 |
| OpenJDK | 11.0.18 | 开源的 Java 开发工具包 |

## 核心步骤

1. 查看当前 Java 版本

安装前先检查服务器当前使用的 Java 版本：

~~~bash
java -version
javac -version
~~~

如果服务器没有安装 Java，命令会提示 `command not found`。

2. 查看 YUM 仓库中的 JDK 11 版本

~~~bash
yum --showduplicates list java-11-openjdk java-11-openjdk-devel
~~~

检查输出结果中是否包含 `11.0.18`。

YUM 只能安装当前仓库仍然提供的版本。如果列表中没有 `11.0.18`，则无法通过 YUM 精确安装该历史版本，需要改用二进制压缩包安装。

3. 使用 YUM 安装 JDK 11.0.18

如果仓库中存在 `11.0.18`，可以执行以下命令安装：

~~~bash
yum install -y 'java-11-openjdk-11.0.18*' 'java-11-openjdk-devel-11.0.18*'
~~~

其中：

`java-11-openjdk`：提供 Java 运行环境

`java-11-openjdk-devel`：提供 `javac`、`javadoc`、`jar` 等 Java 开发工具

如果业务只要求使用 JDK 11，不强制要求具体的小版本，也可以直接安装仓库中的最新 JDK 11：

~~~bash
yum install -y java-11-openjdk java-11-openjdk-devel
~~~

4. 使用 alternatives 切换 Java 版本

查看并切换系统默认的 `java`：

~~~bash
alternatives --config java
~~~

查看并切换系统默认的 `javac`：

~~~bash
alternatives --config javac
~~~

命令会列出系统中已经注册的 Java 版本，根据提示输入 JDK 11 对应的序号并按回车键确认。

查看 `java` 和 `javac` 实际指向的路径：

~~~bash
readlink -f "$(command -v java)"
readlink -f "$(command -v javac)"
~~~

5. 配置 JAVA_HOME 环境变量

根据当前 `javac` 的实际路径获取 `JAVA_HOME`：

~~~bash
JAVA_HOME_PATH=$(dirname "$(dirname "$(readlink -f "$(command -v javac)")")")
echo "$JAVA_HOME_PATH"
~~~

创建系统环境变量配置文件：

~~~bash
cat >/etc/profile.d/java.sh <<EOF
export JAVA_HOME=$JAVA_HOME_PATH
export PATH=\$JAVA_HOME/bin:\$PATH
EOF
~~~

加载环境变量：

~~~bash
source /etc/profile.d/java.sh
~~~

## 结果验证

~~~bash
[root@xxxx ~]# java -version
openjdk version "11.0.18" 2023-01-17 LTS
OpenJDK Runtime Environment (build 11.0.18+10-LTS)
OpenJDK 64-Bit Server VM (build 11.0.18+10-LTS, mixed mode, sharing)

[root@xxxx ~]# javac -version
javac 11.0.18

[root@xxxx ~]# echo $JAVA_HOME
/usr/lib/jvm/java-11-openjdk-11.0.18.x-x.x86_64
~~~

如果 `java` 和 `javac` 均显示 `11.0.18`，并且 `JAVA_HOME` 指向 JDK 11 的安装目录，则说明安装和版本切换成功。