## 问题背景
  最近在研究AIOPS，准备使用lightRAG作为LLM的外部知识库、包含常用告警处理、故障案例、服务关系等。

## 环境介绍
|组件名称|组件版本| 组件作用|
|---|---|---|
|Tencent OS |3.3|腾讯云提供的Linux操作系统|
|lightRAG| v1.4.16/0288| 类图RAG向量库，维护知识图谱的对应关系|
|qwen| 3.5(2b)|阿里开源的大模型|
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
七个小矮人的故事

在遥远的神秘森林深处，住着七个可爱的小矮人，他们每天一起工作、一起生活，过着快乐而充实的日子。

七个小矮人的名字和特点：

1. 开心果（Happy）- 总是面带微笑，性格乐观开朗，喜欢讲笑话逗大家开心。他是团队中的快乐源泉，总能用自己的幽默感化解大家的烦恼。

2. 瞌睡虫（Sleepy）- 永远都是一副睡眼惺忪的样子，随时随地都能睡着。虽然他看起来总是很困，但工作起来却从不含糊，尤其是在守护森林安全的时候格外警觉。

3. 喷嚏精（Sneezy）- 患有严重的花粉过敏症，动不动就打喷嚏，一个喷嚏能震得整个小屋都在摇晃。不过他的喷嚏有时也能派上用场，比如吹走捣乱的乌鸦。

4. 害羞鬼（Bashful）- 性格腼腆内向，一说话脸就红得像苹果。他最喜欢照顾森林里的小动物，虽然不善于表达，但心地最为善良。

5. 爱生气（Grumpy）- 总是板着一张脸，看起来凶巴巴的，经常对大家发牢骚。但其实他有一颗最柔软的心，每次有人遇到困难，他总是第一个伸出援手。

6. 万事通（Doc）- 是七个小矮人中的智慧担当，戴着厚厚的眼镜，喜欢读书和研究。他负责记录森林里的各种草药知识，大家生病时都找他开药方。

7. 糊涂蛋（Dopey）- 从不说话，总是用行动表达自己。他天真可爱，经常闹出一些让人哭笑不得的小笑话，是大家最喜欢逗弄的对象。

他们的日常生活：

每天清晨，当第一缕阳光照进森林，开心果就会吹响他的小号角，叫醒还在睡觉的其他小矮人。瞌睡虫总是最后一个起床，需要万事通用冷水才能把他弄醒。

早餐后，七个小矮人会分成两组：一组由万事通带领去森林采集草药和果实，另一组由爱生气带领去矿洞开采宝石。他们每天都会带回丰富的收获，然后一起分享。

中午时分，害羞鬼会准备丰盛的午餐。他虽然不爱说话，但厨艺却是最好的。爱生气虽然嘴上总是挑剔，但每次都会把饭菜吃得干干净净。

下午是自由活动时间。开心果会组织游戏，喷嚏精会帮忙打扫屋子，糊涂蛋则常常惹出一些小麻烦，比如把面粉撒得到处都是，或者不小心把自己锁在储藏室里。

傍晚，七个小矮人会围坐在壁炉旁，听万事通讲述森林里的传说故事。这时候连爱生气都会安静下来，认真地听着。直到瞌睡虫发出均匀的鼾声，大家才知道该去睡觉了。

他们与白雪公主的友谊：

有一天，七个小矮人在森林里遇到了迷路的白雪公主。开心果第一个欢迎她到家里做客，害羞鬼红着脸为她准备了晚餐，万事通帮她找到了回家的路。

从此，白雪公主经常来看望七个小矮人，教他们唱歌跳舞，帮他们缝补衣服。爱生气虽然嘴上说不喜欢热闹，但每次白雪公主来的时候，他都会偷偷把自己珍藏的宝石送给她。

森林的守护者：

七个小矮人不仅是快乐的小伙伴，更是这片神秘森林的守护者。他们知道每一棵树的年龄，了解每一条溪流的源头，保护着森林里所有的小动物。

当暴风雨来临时，他们会提前帮助小动物们转移到安全的地方。当有陌生人进入森林时，喷嚏精会用他的喷嚏制造声响吓退不速之客。

智慧的传承：

万事通把自己积累的知识都记录在一本厚厚的皮面笔记本里，包括各种草药的功效、宝石的种类、季节变化的规律。他打算把这本笔记本传给森林里的下一代，让他们继续守护这片美丽的家园。

七个小矮人的故事告诉我们，每个人都有自己的独特之处，正是因为彼此的不同，才能让生活变得如此丰富多彩。他们用团结、友爱和智慧，守护着属于自己的幸福生活。

这就是七个小矮人的故事，他们依然快乐地生活在神秘森林里，每天都有新的冒险，每天都有新的欢笑。
```
## 结果验证
- 后台日志
![](/blogs/TencentOS-install-lightRAG/fb82a2a49e45d02f.png)
- Web UI
![](/blogs/TencentOS-install-lightRAG/013ab5d1d3588172.png)

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