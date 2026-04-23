## 问题背景
  最近在研究AIOPS准备使用ollama+qwen+beg作为ligtRAG的模型支持，qwen在国内属于模型的第一梯队，beg模型在embedding模型中对中文的支持最强。

## 环境介绍
|组件名称|组件版本| 组件作用|
|---|---|---|
|Tencent OS |3.3|腾讯云提供的Linux操作系统（机型为CPU）|
|Docker |Docker version 26.1.3, build b72abbb| 提供资源隔离、环境隔离的核心组件|
|ollama| 0.21|快速部署各个厂商的大模型工具|
|qwen | 2.5（3b）| 阿里开源的大模型|
|beg | latest|针对RAG检索时需要的大模型|


## 核心步骤
### 部署ollama

```
docker run -itd -v /data/ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama:0.21.1
```

#### 参数解释
| 参数 | 说明 |
|------|------|
| `docker run` | 运行一个新的容器 |
| `-itd` | 组合参数：<br>- `-i`：保持标准输入打开<br>- `-t`：分配一个伪终端<br>- `-d`：后台运行容器（守护模式） |
| `-v /data/ollama:/root/.ollama` | 挂载卷（数据持久化）：<br>- 宿主机目录：`/data/ollama`<br>- 容器内目录：`/root/.ollama`<br>用于保存模型文件等数据 |
| `-p 11434:11434` | 端口映射：<br>- 宿主机端口：`11434`<br>- 容器端口：`11434`<br>可通过 `localhost:11434` 访问服务 |
| `--name ollama` | 指定容器名称为 `ollama` |
| `ollama/ollama:0.21.1` | 使用的镜像名称和版本标签 |


### 使用ollama部署qwen2.5
```
docker exec -it ollama bash -c "ollama run qwen2.5:3b"

```
#### 参数解释
| 参数 | 说明 |
|------|------|
| `docker exec` | 在已运行的容器中执行命令 |
| `-it` | 组合参数：<br>- `-i`：保持标准输入打开<br>- `-t`：分配一个伪终端<br>两者结合可实现交互式操作 |
| `ollama` | 目标容器名称（与之前创建的容器名称对应） |
| `bash` | 在容器内启动bash shell |
| `ollama run` | Ollama命令，用于运行指定的AI模型 |
| `qwen2.5:3b` | 模型名称和参数规模：<br>- `qwen2.5`：通义千问2.5模型系列<br>- `3b`：30亿参数版本（轻量级模型） |

### 使用ollama部署beg-m3
```
docker exec -it ollama bash -c "ollama run bge-m3"

```
#### 参数解释
| 参数 | 说明 |
|------|------|
| `docker exec` | 在已运行的容器中执行命令 |
| `-it` | 组合参数：<br>- `-i`：保持标准输入打开<br>- `-t`：分配一个伪终端<br>两者结合实现交互式操作 |
| `ollama` | 目标容器名称（与之前创建的容器名称对应） |
| `bash -c` | 通过bash执行字符串中的命令：<br>- `bash`：启动bash shell<br>- `-c`：执行后面引号中的命令字符串 |
| `"ollama run bge-m3"` | 要执行的命令：<br>- `ollama run`：Ollama运行模型命令<br>- `bge-m3`：BGE-M3模型名称（BAAI/BGE-M3，用于嵌入和重排序的多功能模型） |

## 结果验证
### ollama运行状况
![](/blogs/Docker-intall-ollama0.21/969060f2805bde7d.png)
### qwen2.5部署状况
![](/blogs/Docker-intall-ollama0.21/5ed415503373fbc7.png)
### beg-m3部署状况
![](/blogs/Docker-intall-ollama0.21/cf237a55795f4dd1.png)