## 问题背景
  最近在研究AIOPS，准备使用lightRAG作为LLM的外部知识库、包含常用告警处理、故障案例、服务关系等。

## 环境介绍
|组件名称|组件版本| 组件作用|
|---|---|---|
|Tencent OS |3.3|腾讯云提供的Linux操作系统|
|lightRAG| v1.4.16/0288| 类图RAG向量库，维护知识图谱的对应关系|
|qwen| 2.5(3b)|阿里开源的大模型|
|beg-m3| lastest|对中文友好的embedding模型|


## 核心步骤
### 部署lightRAG
```
git clone https://github.com/HKUDS/LightRAG.git
cd LightRAG

# 一键初始化开发环境（推荐）
# make dev 会安装测试工具链以及完整的离线依赖栈
# （API、存储后端与各类 Provider 集成），并构建前端；不会生成 .env。
# 启动服务前请先运行 make env-base，或手动从 env.example 复制并配置 .env。
make dev
#安装uv
curl -LsSf https://astral.sh/uv/install.sh | sh 
#安装bun
curl -fsSL https://bun.sh/install | bash 
#激活最新环境变量
source /root/.bash_profile 
# 激活虚拟环境 (Linux/macOS)
source .venv/bin/activate  

# 配置 env 文件
make env-base  # 或: cp env.example .env 后手动修改
# 启动API-WebUI服务
lightrag-server
```
### 目录结构
```
[root@xxxxx LightRAG]# tree  -a -L 1 .
.
├── AGENTS.md
├── assets
├── CLAUDE.md
├── .clinerules
├── config.ini
├── config.ini.example
├── data  #lightRAG的核心存储目录，分为（input、ragStore）
├── docker-build-push.sh
├── docker-compose-full.yml
├── docker-compose.yml
├── Dockerfile
├── Dockerfile.lite
├── .dockerignore
├── docs
├── .env  #核心配置，lightRAG读取的核心文件
├── env.docker-compose-full
├── env.example
├── examples
├── .git
├── .gitattributes
├── .github
├── .gitignore
├── inputs
├── k8s-deploy
├── LICENSE
├── lightrag
├── lightrag_hku.egg-info
├── lightrag.log
├── lightrag.service.example
├── lightrag_webui
├── Makefile
├── MANIFEST.in
├── .pre-commit-config.yaml
├── pyproject.toml
├── rag_storage
├── README.assets
├── README.md
├── README-zh.md
├── reproduce
├── requirements-offline-llm.txt
├── requirements-offline-storage.txt
├── requirements-offline.txt
├── scripts
├── SECURITY.md
├── setup.py
├── tests
├── uv.lock
└── .venv
```
### 准备测试文本
```
爱丽丝是一名软件工程师，她在OpenAI公司工作。她工作地点在旧金山。

鲍勃是一名数据科学家，他在Google公司任职。他居住在纽约。

查理是斯坦福大学的教授，他教授机器学习课程。

OpenAI是一家位于美国的组织，主要从事人工智能研究。

Google是一家科技公司，开发搜索引擎和云服务。

斯坦福大学位于加利福尼亚州，是一所著名大学。

爱丽丝和鲍勃合作进行一个自然语言处理项目。

鲍勃在加入Google之前曾就读于斯坦福大学。

查理在2020年发表了一篇关于深度学习的论文。

爱丽丝和鲍勃的项目开始于2023年。
```
## 结果验证
- 后台日志
![](/blogs/TencentOS-install-lightRAG/fb82a2a49e45d02f.png)
- Web UI
![](/blogs/TencentOS-install-lightRAG/013ab5d1d3588172.png)
- 查看知识图谱
![](/blogs/TencentOS-install-lightRAG/292b853ff7f05a88.png)

## 遇到的问题
- lightRAG上传测试文件时，出现维度不一致问题
```
ERROR: Failed to initialize LightRAG: Embedding dim mismatch, expected: 1024, but loaded: 3072
Traceback (most recent call last):
  File "/data/LightRAG/.venv/bin/lightrag-server", line 10, in <module>
    sys.exit(main())
             ^^^^^^
  File "/data/LightRAG/lightrag/api/lightrag_server.py", line 1523, in main
    app = create_app(global_args)
          ^^^^^^^^^^^^^^^^^^^^^^^
  File "/data/LightRAG/lightrag/api/lightrag_server.py", line 1060, in create_app
    rag = LightRAG(
          ^^^^^^^^^
  File "<string>", line 57, in __init__
  File "/data/LightRAG/lightrag/lightrag.py", line 712, in __post_init__
    self.entities_vdb: BaseVectorStorage = self.vector_db_storage_cls(  # type: ignore
                                           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<string>", line 9, in __init__
  File "/data/LightRAG/lightrag/kg/nano_vector_db_impl.py", line 61, in __post_init__
    self._client = NanoVectorDB(
                   ^^^^^^^^^^^^^
  File "<string>", line 6, in __init__
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/nano_vectordb/dbs.py", line 73, in __post_init__
    storage["embedding_dim"] == self.embedding_dim
AssertionError: Embedding dim mismatch, expected: 1024, but loaded: 3072
```
解决方案：
1. 查看embedding模型支持的维度
```
[root@xxxx LightRAG]# curl http://localhost:11434/api/embeddings   -s   -d '{
    "model": "bge-m3:latest",
    "prompt": "hello"
  }' | jq '.embedding | length'
1024
```
2. 修改lightRAG的`.env`里的`EMBEDDING_DIM=1024`,要跟embedding模型保持一致
3. 删除`rag_storage`目录，并重新创建
4. 重启服务即可
- lightRAG处理文档超时
```
ERROR: Failed to extract entities and relationships: C[1/6]: chunk-be79a2c6a466bfaa6a9d620769252fb7:
ERROR: Traceback (most recent call last):
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/httpx/_transports/default.py", line 101, in map_httpcore_exceptions
    yield
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/httpx/_transports/default.py", line 394, in handle_async_request
    resp = await self._pool.handle_async_request(req)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/httpcore/_async/connection_pool.py", line 256, in handle_async_request
    raise exc from None
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/httpcore/_async/connection_pool.py", line 236, in handle_async_request
    response = await connection.handle_async_request(
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/httpcore/_async/connection.py", line 103, in handle_async_request
    return await self._connection.handle_async_request(request)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/httpcore/_async/http11.py", line 136, in handle_async_request
    raise exc
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/httpcore/_async/http11.py", line 106, in handle_async_request
    ) = await self._receive_response_headers(**kwargs)
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/httpcore/_async/http11.py", line 177, in _receive_response_headers
    event = await self._receive_event(timeout=timeout)
            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/httpcore/_async/http11.py", line 217, in _receive_event
    data = await self._network_stream.read(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/httpcore/_backends/anyio.py", line 32, in read
    with map_exceptions(exc_map):
         ^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib64/python3.12/contextlib.py", line 158, in __exit__
    self.gen.throw(value)
  File "/data/LightRAG/.venv/lib64/python3.12/site-packages/httpcore/_exceptions.py", line 14, in map_exceptions
    raise to_exc(exc) from exc
httpcore.ReadTimeout
```
解决方案
1. 修改`.env`文件，设置MAX_ASYNC=1