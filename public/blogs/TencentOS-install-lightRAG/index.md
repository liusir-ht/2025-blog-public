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
### .env文件配置
```
### All configurable environment variable must show up in this sample file in active or comment out status
### Setup tool `make env-*` uses this file to generate final .env file
### These are placeholders and setup tool should not be substituted with actual values in this lines.

### Target environment of this env file: host/compose (compose is for Dokcer or Kubernetes)
LIGHTRAG_RUNTIME_TARGET=host

###########################
### Server Configuration
###########################
HOST=0.0.0.0
PORT=9621
WEBUI_TITLE='My Graph KB'
WEBUI_DESCRIPTION='Simple and Fast Graph Based RAG System'
# WORKERS=2
### gunicorn worker timeout(as default LLM request timeout if LLM_TIMEOUT is not set)
# TIMEOUT=150
# CORS_ORIGINS=http://localhost:3000,http://localhost:8080

### Optional SSL Configuration
### Docker note: generated compose files mount staged certs at /app/data/certs/ inside the container
# SSL=true
# SSL_CERTFILE=/path/to/cert.pem
# SSL_KEYFILE=/path/to/key.pem

### Directory Configuration (defaults to current working directory)
### Default value is ./inputs and ./rag_storage
# INPUT_DIR=<absolute_path_for_doc_input_dir>
# WORKING_DIR=<absolute_path_for_working_dir>

### Tiktoken cache directory (Store cached files in this folder for offline deployment)
# TIKTOKEN_CACHE_DIR=/app/data/tiktoken

### Ollama Emulating Model and Tag
# OLLAMA_EMULATING_MODEL_NAME=lightrag
OLLAMA_EMULATING_MODEL_TAG=latest

### Max nodes for graph retrieval (Ensure WebUI local settings are also updated, which is limited to this value)
# MAX_GRAPH_NODES=1000

### Logging level
# LOG_LEVEL=INFO
# VERBOSE=False
# LOG_MAX_BYTES=10485760
# LOG_BACKUP_COUNT=5
### Logfile location (defaults to current working directory)
# LOG_DIR=/path/to/log/directory
# LIGHTRAG_PERFORMANCE_TIMING_LOGS=false

#####################################
### Login and API-Key Configuration
#####################################
# AUTH_ACCOUNTS='admin:admin123,user1:{bcrypt}$2b$12$S8Yu.gCbuAbNTJFB.231gegTwr5pgrFxc8H9kXQ4/sduFBHkhM8Ka'
# TOKEN_SECRET=lightrag-jwt-default-secret-key!
# JWT_ALGORITHM=HS256
# TOKEN_EXPIRE_HOURS=48
# GUEST_TOKEN_EXPIRE_HOURS=24

### Token Auto-Renewal Configuration (Sliding Window Expiration)
### Enable automatic token renewal to prevent active users from being logged out
### When enabled, tokens will be automatically renewed when remaining time < threshold
# TOKEN_AUTO_RENEW=true
### Token renewal threshold (0.0 - 1.0)
### Renew token when remaining time < (total time * threshold)
### Default: 0.5 (renew when 50% time remaining)
### Examples:
###   0.5 = renew when 24h token has 12h left
###   0.25 = renew when 24h token has 6h left
# TOKEN_RENEW_THRESHOLD=0.5
### Note: Token renewal is automatically skipped for certain endpoints:
###   - /health: Health check endpoint (no authentication required)
###   - /documents/paginated: Frequently polled by client (5-30s interval)
###   - /documents/pipeline_status: Very frequently polled by client (2s interval)
###   - Rate limit: Minimum 60 seconds between renewals for same user

### API-Key to access LightRAG Server API
### Use this key in HTTP requests with the 'X-API-Key' header
### Example: curl -H "X-API-Key: your-secure-api-key-here" http://localhost:9621/query
# LIGHTRAG_API_KEY=your-secure-api-key-here
# WHITELIST_PATHS=/health,/api/*

######################################################################################
### Query Configuration
###
### How to control the context length sent to LLM:
###    MAX_ENTITY_TOKENS + MAX_RELATION_TOKENS < MAX_TOTAL_TOKENS
###    Chunk_Tokens = MAX_TOTAL_TOKENS - Actual_Entity_Tokens - Actual_Relation_Tokens
######################################################################################
# LLM response cache for query (Not valid for streaming response)
# ENABLE_LLM_CACHE=true
# COSINE_THRESHOLD=0.2
### Number of entities or relations retrieved from KG
# TOP_K=40
### Maximum number or chunks for naive vector search
# CHUNK_TOP_K=20
### control the actual entities send to LLM
# MAX_ENTITY_TOKENS=6000
### control the actual relations send to LLM
# MAX_RELATION_TOKENS=8000
### control the maximum tokens send to LLM (include entities, relations and chunks)
# MAX_TOTAL_TOKENS=30000

### chunk selection strategies
###     VECTOR: Pick KG chunks by vector similarity, delivered chunks to the LLM aligning more closely with naive retrieval
###     WEIGHT: Pick KG chunks by entity and chunk weight, delivered more solely KG related chunks to the LLM
###     If reranking is enabled, the impact of chunk selection strategies will be diminished.
# KG_CHUNK_PICK_METHOD=VECTOR

### maximum number of related chunks per source entity or relation
###     The chunk picker uses this value to determine the total number of chunks selected from KG(knowledge graph)
###     Higher values increase re-ranking time
# RELATED_CHUNK_NUMBER=5

#########################################################
### Reranking configuration
### RERANK_BINDING type: null, cohere, jina, aliyun
### For rerank model deployed by vLLM use cohere binding
### If LightRAG deployed in Docker:
###    uses host.docker.internal instead of localhost in RERANK_BINDING_HOST
#########################################################
RERANK_BINDING=null
# RERANK_MODEL=BAAI/bge-reranker-v2-m3
# RERANK_BINDING_HOST=http://localhost:8000/rerank
# RERANK_BINDING_API_KEY=your_rerank_api_key_here

### rerank score chunk filter(set to 0.0 to keep all chunks, 0.6 or above if LLM is not strong enough)
# MIN_RERANK_SCORE=0.0
### Enable rerank by default in query params when RERANK_BINDING is not null
# RERANK_BY_DEFAULT=True

### Cohere AI
# # RERANK_MODEL=rerank-v3.5
# # RERANK_BINDING_HOST=https://api.cohere.com/v2/rerank
# # RERANK_BINDING_API_KEY=your_rerank_api_key_here
### Cohere rerank chunking configuration (useful for models with token limits like ColBERT)
# RERANK_ENABLE_CHUNKING=true
# RERANK_MAX_TOKENS_PER_DOC=480

### Aliyun Dashscope
# # RERANK_MODEL=gte-rerank-v2
# # RERANK_BINDING_HOST=https://dashscope.aliyuncs.com/api/v1/services/rerank/text-rerank/text-rerank
# # RERANK_BINDING_API_KEY=your_rerank_api_key_here

### Jina AI
# # RERANK_MODEL=jina-reranker-v2-base-multilingual
# # RERANK_BINDING_HOST=https://api.jina.ai/v1/rerank
# # RERANK_BINDING_API_KEY=your_rerank_api_key_here

### For local deployment Embedding and Reranker with vLLM (OpenAI-compatible API)
### Wizard metadata used to preserve the chosen deployment provider across setup reruns
# LIGHTRAG_SETUP_EMBEDDING_PROVIDER=vllm
# LIGHTRAG_SETUP_RERANK_PROVIDER=vllm
# VLLM_EMBED_MODEL=BAAI/bge-m3
# VLLM_EMBED_PORT=8001
# VLLM_EMBED_DEVICE=cpu
### VLLM_EMBED_API_KEY is passed as --api-key to vLLM; synced to EMBEDDING_BINDING_API_KEY; auto-generated if blank
# VLLM_EMBED_API_KEY=
# VLLM_EMBED_EXTRA_ARGS=
# VLLM_RERANK_MODEL=BAAI/bge-reranker-v2-m3
# VLLM_RERANK_PORT=8000
# VLLM_RERANK_DEVICE=cuda
### VLLM_RERANK_API_KEY is passed as --api-key to vLLM; synced to RERANK_BINDING_API_KEY; auto-generated if blank
# VLLM_RERANK_API_KEY=
### Use float16 for GPU mode. CPU mode uses the official vLLM CPU image.
# VLLM_USE_CPU=1
### Set to 1 for CPU mode, unset for GPU mode
# CUDA_VISIBLE_DEVICES=-1
### Set to -1 to disable CUDA (CPU mode), or specific GPU IDs for GPU mode
# NVIDIA_VISIBLE_DEVICES=0
### Optional Docker runtime equivalent; generated GPU compose honors either variable.
# VLLM_RERANK_EXTRA_ARGS=

########################################
### Document processing configuration
########################################
ENABLE_LLM_CACHE_FOR_EXTRACT=true

### Document processing output language: English, Chinese, French, German ...
SUMMARY_LANGUAGE=English

### File upload size limit (in bytes)
### Default: 104857600 (100MB)
### Set to 0 or None for unlimited upload size
### Examples:
###   52428800  = 50MB
###   104857600 = 100MB (default)
###   209715200 = 200MB
### Note: If using Nginx as reverse proxy, also configure client_max_body_size
# MAX_UPLOAD_SIZE=104857600

### Entity types that the LLM will attempt to recognize
# ENTITY_TYPES='["Person", "Creature", "Organization", "Location", "Event", "Concept", "Method", "Content", "Data", "Artifact", "NaturalObject"]'

### Chunk size for document splitting, 500~1500 is recommended
CHUNK_SIZE=400
CHUNK_OVERLAP_SIZE=50
MIN_CHUNK_SIZE=100

### Number of summary segments or tokens to trigger LLM summary on entity/relation merge (at least 3 is recommended)
# FORCE_LLM_SUMMARY_ON_MERGE=8
### Max description token size to trigger LLM summary
# SUMMARY_MAX_TOKENS = 1200
### Recommended LLM summary output length in tokens
# SUMMARY_LENGTH_RECOMMENDED=600
### Maximum context size sent to LLM for description summary
# SUMMARY_CONTEXT_SIZE=12000
### Maximum token size allowed for entity extraction input context
# MAX_EXTRACT_INPUT_TOKENS=20480

### control the maximum chunk_ids stored in vector and graph db
# MAX_SOURCE_IDS_PER_ENTITY=300
# MAX_SOURCE_IDS_PER_RELATION=300
### control chunk_ids limitation method: FIFO, KEEP
###    FIFO: First in first out
###    KEEP: Keep oldest (less merge action and faster)
# SOURCE_IDS_LIMIT_METHOD=FIFO

# Maximum number of file paths stored in entity/relation file_path field (For displayed only, does not affect query performance)
# MAX_FILE_PATHS=100

### PDF decryption password for protected PDF files
# PDF_DECRYPT_PASSWORD=your_pdf_password_here

###############################
### Concurrency Configuration
###############################
### Max concurrency requests of LLM (for both query and document processing)
MAX_ASYNC=1
### Number of parallel processing documents(between 2~10, MAX_ASYNC/3 is recommended)
MAX_PARALLEL_INSERT=1
### Max concurrency requests for Embedding
# EMBEDDING_FUNC_MAX_ASYNC=8
### Num of chunks send to Embedding in single request
EMBEDDING_BATCH_NUM=1
EMBEDDING_BATCH_SIZE=1


###########################################################################
### LLM Configuration
### LLM_BINDING type: openai, ollama, lollms, azure_openai, aws_bedrock, gemini
### LLM_BINDING_HOST: Service endpoint (left empty if using default endpoint provided by openai or gemini SDK)
### LLM_BINDING_API_KEY: api key
### If LightRAG deployed in Docker:
###    uses host.docker.internal instead of localhost in LLM_BINDING_HOST
###########################################################################
### LLM request timeout setting for all llm (0 means no timeout for Ollma)
LLM_TIMEOUT=600

LLM_BINDING=ollama
# LLM_BINDING=openai
LLM_BINDING_HOST=http://localhost:11434
# LLM_BINDING_HOST=https://api.openai.com/v1
LLM_BINDING_API_KEY=your_api_key
LLM_MODEL=qwen2.5:3b
# LLM_MODEL=gpt-5-mini

### use the following command to see all support options for OpenAI, azure_openai or OpenRouter
### lightrag-server --llm-binding openai --help
### OpenAI Specific Parameters
# OPENAI_LLM_REASONING_EFFORT=minimal
### OpenRouter Specific Parameters
# OPENAI_LLM_EXTRA_BODY='{"reasoning": {"enabled": false}}'
### Qwen3 Specific Parameters deploy by vLLM
# OPENAI_LLM_EXTRA_BODY='{"chat_template_kwargs": {"enable_thinking": false}}'

### OpenAI Compatible API Specific Parameters
### Increased temperature values may mitigate infinite inference loops in certain LLM, such as Qwen3-30B.
# OPENAI_LLM_TEMPERATURE=0.9
### Set the max_tokens to mitigate endless output of some LLM (less than LLM_TIMEOUT * llm_output_tokens/second, i.e. 9000 = 180s * 50 tokens/s)
### Typically, max_tokens does not include prompt content
### For vLLM/SGLang deployed models, or most of OpenAI compatible API provider
# OPENAI_LLM_MAX_TOKENS=9000
### For OpenAI o1-mini or newer modles utilizes max_completion_tokens instead of max_tokens
# OPENAI_LLM_MAX_COMPLETION_TOKENS=9000

### Azure OpenAI example
### Use deployment name as model name or set AZURE_OPENAI_DEPLOYMENT instead
# AZURE_OPENAI_API_VERSION=2024-08-01-preview
# # LLM_BINDING=azure_openai
# # LLM_BINDING_HOST=https://xxxx.openai.azure.com/
# # LLM_BINDING_API_KEY=your_api_key
# # LLM_MODEL=my-gpt-mini-deployment

### Openrouter example
# # LLM_BINDING=openai
# # LLM_BINDING_HOST=https://openrouter.ai/api/v1
# # LLM_BINDING_API_KEY=your_api_key
# # LLM_MODEL=google/gemini-2.5-flash

### Google Gemini example (AI Studio)
# # LLM_BINDING=gemini
# # LLM_BINDING_API_KEY=your_gemini_api_key
# # LLM_BINDING_HOST=https://generativelanguage.googleapis.com
# # LLM_MODEL=gemini-flash-latest

### use the following command to see all support options for OpenAI, azure_openai or OpenRouter
### lightrag-server --llm-binding gemini --help
### Gemini Specific Parameters
# GEMINI_LLM_MAX_OUTPUT_TOKENS=9000
# GEMINI_LLM_TEMPERATURE=0.7
### Enable or disable thinking
# GEMINI_LLM_THINKING_CONFIG='{"thinking_budget": -1, "include_thoughts": true}'
# # GEMINI_LLM_THINKING_CONFIG='{"thinking_budget": 0, "include_thoughts": false}'

### Google Vertex AI example
### Vertex AI use GOOGLE_APPLICATION_CREDENTIALS instead of API-KEY for authentication
### LLM_BINDING_HOST=DEFAULT_GEMINI_ENDPOINT means select endpoit based on project and location automatically
# # LLM_BINDING=gemini
# # LM_BINDING_HOST=https://aiplatform.googleapis.com
### or use DEFAULT_GEMINI_ENDPOINT to select endpoint based on project and location automatically
# # LLM_BINDING_HOST=DEFAULT_GEMINI_ENDPOINT
# # LLM_MODEL=gemini-2.5-flash
# GOOGLE_GENAI_USE_VERTEXAI=true
# GOOGLE_CLOUD_PROJECT='your-project-id'
# GOOGLE_CLOUD_LOCATION='us-central1'
# GOOGLE_APPLICATION_CREDENTIALS='/Users/xxxxx/your-service-account-credentials-file.json'

### Ollama example
# # LLM_BINDING=ollama
# # LLM_BINDING_HOST=http://localhost:11434
# # LLM_MODEL=qwen3.5:9b

### use the following command to see all support options for Ollama LLM
### lightrag-server --llm-binding ollama --help
### Ollama Server Specific Parameters
### OLLAMA_LLM_NUM_CTX must be provided, and should at least larger than MAX_TOTAL_TOKENS + 2000
OLLAMA_LLM_NUM_CTX=32768
### Set the max_output_tokens to mitigate endless output of some LLM (less than LLM_TIMEOUT * llm_output_tokens/second, i.e. 9000 = 180s * 50 tokens/s)
OLLAMA_LLM_NUM_PREDICT=512
# OLLAMA_LLM_TEMPERATURE=0.85
### Stop sequences for Ollama LLM
# OLLAMA_LLM_STOP='["</s>", "<|EOT|>"]'

### Bedrock Specific Parameters
### Bedrock uses AWS credentials from the environment / AWS credential chain.
### It does not use LLM_BINDING_API_KEY.
# # LLM_BINDING=aws_bedrock
# # LLM_MODEL=anthropic.claude-3-5-sonnet-20241022-v2:0
# AWS_ACCESS_KEY_ID=your_aws_access_key_id
# AWS_SECRET_ACCESS_KEY=your_aws_secret_access_key
# AWS_SESSION_TOKEN=your_optional_aws_session_token
# AWS_REGION=us-east-1
# BEDROCK_LLM_TEMPERATURE=1.0

#######################################################################################
### Embedding Configuration (Should not be changed after the first file processed)
### EMBEDDING_BINDING: ollama, openai, azure_openai, jina, lollms, aws_bedrock
### EMBEDDING_BINDING_HOST: Service endpoint (left empty if using default endpoint provided by openai or gemini SDK)
### EMBEDDING_BINDING_API_KEY: api key
### If LightRAG deployed in Docker:
###    uses host.docker.internal instead of localhost in EMBEDDING_BINDING_HOST
### Control whether to send embedding_dim parameter to embedding API
###    For OpenAI: Set EMBEDDING_SEND_DIM=true to enable dynamic dimension adjustment
###    For OpenAI: Set EMBEDDING_SEND_DIM=false (default) to disable sending dimension parameter
###    For Gemini: Allways set EMBEDDING_SEND_DIM=true
### Control whether to use base64 encoding format for embeddings (improves performance for OpenAI)
###    For OpenAI: Set EMBEDDING_USE_BASE64=true (default) to use base64 encoding
###    For Yandex Cloud and other providers that don't support it: Set EMBEDDING_USE_BASE64=false
#######################################################################################
# EMBEDDING_TIMEOUT=30

### OpenAI compatible embedding
EMBEDDING_BINDING=ollama
# EMBEDDING_BINDING=openai
EMBEDDING_BINDING_HOST=http://localhost:11434
# EMBEDDING_BINDING_HOST=https://api.openai.com/v1
EMBEDDING_BINDING_API_KEY=your_api_key
EMBEDDING_MODEL=bge-m3:latest
# EMBEDDING_MODEL=text-embedding-3-large
EMBEDDING_DIM=1024
# EMBEDDING_DIM=3072
EMBEDDING_TOKEN_LIMIT=8192
EMBEDDING_SEND_DIM=false
# EMBEDDING_USE_BASE64=true

### Optional for Azure Embedding
### Use deployment name as model name or set AZURE_EMBEDDING_DEPLOYMENT instead
# # EMBEDDING_BINDING=azure_openai
# # EMBEDDING_BINDING_HOST=https://xxxx.openai.azure.com/
# # EMBEDDING_API_KEY=your_api_key
# # EMBEDDING_MODEL==my-text-embedding-3-large-deployment
# # EMBEDDING_DIM=3072
# AZURE_EMBEDDING_API_VERSION=2024-08-01-preview

### Gemini embedding
# # EMBEDDING_BINDING=gemini
# # EMBEDDING_MODEL=gemini-embedding-001
# # EMBEDDING_DIM=1536
# # EMBEDDING_TOKEN_LIMIT=2048
# # EMBEDDING_BINDING_HOST=https://generativelanguage.googleapis.com
# # EMBEDDING_BINDING_API_KEY=your_api_key
### Gemini embedding requires sending dimension to server
# # EMBEDDING_SEND_DIM=true

### Ollama embedding
# # EMBEDDING_BINDING=ollama
# # EMBEDDING_BINDING_HOST=http://localhost:11434
# # EMBEDDING_BINDING_API_KEY=your_api_key
# # EMBEDDING_MODEL=qwen3-embedding:4b
# # EMBEDDING_DIM=2560
### Optional for Ollama embedding
OLLAMA_EMBEDDING_NUM_CTX=8192
### use the following command to see all support options for Ollama embedding
### lightrag-server --embedding-binding ollama --help

### Bedrock embedding
### Bedrock uses AWS credentials from the environment / AWS credential chain.
### It does not use EMBEDDING_BINDING_API_KEY.
# # EMBEDDING_BINDING=aws_bedrock
# # EMBEDDING_MODEL=amazon.titan-embed-text-v2:0
# # EMBEDDING_DIM=3072
# AWS_ACCESS_KEY_ID=your_aws_access_key_id
# AWS_SECRET_ACCESS_KEY=your_aws_secret_access_key
# AWS_SESSION_TOKEN=your_optional_aws_session_token
# AWS_REGION=us-east-1

### Jina AI Embedding
# # EMBEDDING_BINDING=jina
# # EMBEDDING_BINDING_HOST=https://api.jina.ai/v1/embeddings
# # EMBEDDING_MODEL=jina-embeddings-v4
# # EMBEDDING_DIM=2048
# # EMBEDDING_BINDING_API_KEY=your_api_key

####################################################################
### WORKSPACE sets workspace name for all storage types
### for the purpose of isolating data from LightRAG instances.
### Valid workspace name constraints: a-z, A-Z, 0-9, and _
####################################################################
# WORKSPACE=

############################
### Data storage selection
############################
### Default storage: JSON/Nano/NetworkX (Recommended for test deployment)
LIGHTRAG_KV_STORAGE=JsonKVStorage
LIGHTRAG_DOC_STATUS_STORAGE=JsonDocStatusStorage
LIGHTRAG_GRAPH_STORAGE=NetworkXStorage
LIGHTRAG_VECTOR_STORAGE=NanoVectorDBStorage

### Wizard metadata used to preserve env-storage Docker deployment defaults across setup reruns
# LIGHTRAG_SETUP_POSTGRES_DEPLOYMENT=docker
# LIGHTRAG_SETUP_NEO4J_DEPLOYMENT=docker
# LIGHTRAG_SETUP_MONGODB_DEPLOYMENT=docker
# LIGHTRAG_SETUP_MONGODB_DEPLOYMENT=atlas-capable
# LIGHTRAG_SETUP_REDIS_DEPLOYMENT=docker
# LIGHTRAG_SETUP_MILVUS_DEPLOYMENT=docker
# LIGHTRAG_SETUP_QDRANT_DEPLOYMENT=docker
# LIGHTRAG_SETUP_MEMGRAPH_DEPLOYMENT=docker
# LIGHTRAG_SETUP_OPENSEARCH_DEPLOYMENT=docker

### PostgreSQL Configuration
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=your_username
POSTGRES_PASSWORD='your_password'
POSTGRES_DATABASE=rag
POSTGRES_MAX_CONNECTIONS=25
### DB specific workspace should not be set, keep for compatible only
# POSTGRES_WORKSPACE=forced_workspace_name


### Use HNSW_HALFVEC for large embeddings (2000+ dim).
### Requires pgvector extension >= 0.7.0.
### Vector storage type: HNSW, HNSW_HALFVEC, IVFFlat, VCHORDRQ
POSTGRES_VECTOR_INDEX_TYPE=HNSW
POSTGRES_HNSW_M=16
POSTGRES_HNSW_EF=200
POSTGRES_IVFFLAT_LISTS=100
POSTGRES_VCHORDRQ_BUILD_OPTIONS=
POSTGRES_VCHORDRQ_PROBES=
POSTGRES_VCHORDRQ_EPSILON=1.9

### PostgreSQL Connection Retry Configuration (Network Robustness)
### NEW DEFAULTS (v1.4.10+): Optimized for HA deployments with ~30s switchover time
### These defaults provide out-of-the-box support for PostgreSQL High Availability setups
###
### Number of retry attempts (1-100, default: 10)
###   - Default 10 attempts allows ~225s total retry time (sufficient for most HA scenarios)
###   - For extreme cases: increase up to 20-50
### Initial retry backoff in seconds (0.1-300.0, default: 3.0)
###   - Default 3.0s provides reasonable initial delay for switchover detection
###   - For faster recovery: decrease to 1.0-2.0
### Maximum retry backoff in seconds (must be >= backoff, max: 600.0, default: 30.0)
###   - Default 30.0s matches typical switchover completion time
###   - For longer switchovers: increase to 60-90
### Connection pool close timeout in seconds (1.0-30.0, default: 5.0)
# POSTGRES_CONNECTION_RETRIES=10
# POSTGRES_CONNECTION_RETRY_BACKOFF=3.0
# POSTGRES_CONNECTION_RETRY_BACKOFF_MAX=30.0
# POSTGRES_POOL_CLOSE_TIMEOUT=5.0

### PostgreSQL SSL Configuration (Optional)
# POSTGRES_SSL_MODE=require
# POSTGRES_SSL_CERT=/path/to/client-cert.pem
# POSTGRES_SSL_KEY=/path/to/client-key.pem
# POSTGRES_SSL_ROOT_CERT=/path/to/ca-cert.pem
# POSTGRES_SSL_CRL=/path/to/crl.pem

### PostgreSQL Server Settings (for Supabase Supavisor)
# Use this to pass extra options to the PostgreSQL connection string.
# For Supabase, you might need to set it like this:
# POSTGRES_SERVER_SETTINGS='options=reference%3D[project-ref]'

# Default is 100 set to 0 to disable
# POSTGRES_STATEMENT_CACHE_SIZE=100

### Neo4j Configuration
NEO4J_URI=neo4j+s://xxxxxxxx.databases.neo4j.io
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD='your_password'
NEO4J_DATABASE=neo4j
NEO4J_MAX_CONNECTION_POOL_SIZE=100
NEO4J_CONNECTION_TIMEOUT=30
NEO4J_CONNECTION_ACQUISITION_TIMEOUT=30
NEO4J_MAX_TRANSACTION_RETRY_TIME=30
NEO4J_MAX_CONNECTION_LIFETIME=300
NEO4J_LIVENESS_CHECK_TIMEOUT=30
NEO4J_KEEP_ALIVE=true
### DB specific workspace should not be set, keep for compatible only
# NEO4J_WORKSPACE=forced_workspace_name

### MongoDB Configuration
# For MongoVectorDBStorage, MONGO_URI must point to a MongoDB endpoint with
# Atlas Search / Vector Search support, such as MongoDB Atlas or Atlas local.
MONGO_URI=mongodb://localhost:27017/
MONGO_DATABASE=LightRAG
### DB specific workspace should not be set, keep for compatible only
# MONGODB_WORKSPACE=forced_workspace_name

# Community/local Docker MongoDB example for KV, graph, or doc-status storage only:
# MONGO_URI=mongodb://localhost:27017/

### OpenSearch Configuration
### OpenSearch can be used for all storage types: KV, Vector, Graph, DocStatus
### Connection settings (comma-separated host:port entries; do not include http:// or https://)
### This setup wizard supports authenticated OpenSearch clusters only.
### OPENSEARCH_USE_SSL controls whether those hosts are reached over TLS.
OPENSEARCH_HOSTS=localhost:9200
OPENSEARCH_USER=admin
OPENSEARCH_PASSWORD=LightRAG2026_!@
OPENSEARCH_USE_SSL=true
OPENSEARCH_VERIFY_CERTS=false
# OPENSEARCH_TIMEOUT=30
# OPENSEARCH_MAX_RETRIES=3
### Index Settings (for 3-AZ Amazon OpenSearch Service, set replicas to 2)
# OPENSEARCH_NUMBER_OF_SHARDS=1
# OPENSEARCH_NUMBER_OF_REPLICAS=0
### k-NN Settings for Vector Storage (HNSW algorithm)
# OPENSEARCH_KNN_EF_CONSTRUCTION=200
# OPENSEARCH_KNN_M=16
# OPENSEARCH_KNN_EF_SEARCH=100
### PPL graphlookup for server-side graph traversal (auto-detected if not set)
# OPENSEARCH_USE_PPL_GRAPHLOOKUP=true
### DB specific workspace should not be set, keep for compatible only
# OPENSEARCH_WORKSPACE=forced_workspace_name

### Milvus Configuration
MILVUS_URI=http://localhost:19530
MILVUS_DB_NAME=lightrag
# MILVUS_DEVICE=cpu
# MILVUS_USER=root
# MILVUS_PASSWORD=your_password
# MILVUS_TOKEN=your_token
# Required for the bundled Docker Milvus stack; may come from .env or exported shell variables.
# MINIO_ACCESS_KEY_ID=minioadmin
# MINIO_SECRET_ACCESS_KEY=minioadmin
### DB specific workspace should not be set, keep for compatible only
# MILVUS_WORKSPACE=forced_workspace_name

### Milvus Vector Index Configuration
### Index type: AUTOINDEX (default), HNSW, HNSW_SQ, HNSW_PQ, IVF_FLAT, IVF_SQ8, DISKANN
# MILVUS_INDEX_TYPE=AUTOINDEX

### Metric type: COSINE (default), L2, IP
# MILVUS_METRIC_TYPE=COSINE

### HNSW / HNSW_SQ / HNSW_PQ Parameters (aligned with Milvus 2.4+ defaults)
### M: Maximum number of connections per node [2-2048], default 16
# MILVUS_HNSW_M=16
### efConstruction: Size of dynamic candidate list during build [8-512], default 360
# MILVUS_HNSW_EF_CONSTRUCTION=360
### ef: Size of dynamic candidate list during search, default 200
# MILVUS_HNSW_EF=200

### HNSW_SQ Specific Parameters (requires Milvus 2.6.8+)
### sq_type: Scalar quantization type - SQ4U, SQ6, SQ8 (default), BF16, FP16
# MILVUS_HNSW_SQ_TYPE=SQ8
### refine: Enable refinement step for higher precision, default false
# MILVUS_HNSW_SQ_REFINE=false
### refine_type: Refinement precision (must be higher than sq_type) - SQ6, SQ8, BF16, FP16, FP32
# MILVUS_HNSW_SQ_REFINE_TYPE=FP32
### refine_k: Refinement expansion factor, default 10
# MILVUS_HNSW_SQ_REFINE_K=10

### IVF_FLAT / IVF_SQ8 Parameters
### nlist: Number of cluster units [1-65536], recommended sqrt(n) for n>1M, default 1024
# MILVUS_IVF_NLIST=1024
### nprobe: Number of units to query [1-nlist], default 16
# MILVUS_IVF_NPROBE=16

### Qdrant
QDRANT_URL=http://localhost:6333
# QDRANT_DEVICE=cpu
# QDRANT_API_KEY=your-api-key
### Qdrant upsert batching (enabled by default)
### Split large upserts by estimated JSON payload size and point count
### Default 16MB keeps safe headroom below common 32MB gateway/request limits
# QDRANT_UPSERT_MAX_PAYLOAD_BYTES=16777216
# QDRANT_UPSERT_MAX_POINTS_PER_BATCH=128
### DB specific workspace should not be set, keep for compatible only
# QDRANT_WORKSPACE=forced_workspace_name

### Redis
REDIS_URI=redis://localhost:6379
REDIS_SOCKET_TIMEOUT=30
REDIS_CONNECT_TIMEOUT=10
REDIS_MAX_CONNECTIONS=100
REDIS_RETRY_ATTEMPTS=3
### DB specific workspace should not be set, keep for compatible only
# REDIS_WORKSPACE=forced_workspace_name

### Memgraph Configuration
MEMGRAPH_URI=bolt://localhost:7687
MEMGRAPH_USERNAME=
MEMGRAPH_PASSWORD=
MEMGRAPH_DATABASE=memgraph
### DB specific workspace should not be set, keep for compatible only
# MEMGRAPH_WORKSPACE=forced_workspace_name

###########################################################
### Langfuse Observability Configuration
### Only works with LLM provided by OpenAI compatible API
### Install with: pip install lightrag-hku[observability]
### Sign up at: https://cloud.langfuse.com or self-host
###########################################################
# LANGFUSE_SECRET_KEY=''
# LANGFUSE_PUBLIC_KEY=''
# LANGFUSE_HOST='https://cloud.langfuse.com'
# LANGFUSE_ENABLE_TRACE=true

############################
### Evaluation Configuration
############################
### RAGAS evaluation models (used for RAG quality assessment)
### ⚠️ IMPORTANT: Both LLM and Embedding endpoints MUST be OpenAI-compatible
### Default uses OpenAI models for evaluation

### LLM Configuration for Evaluation
# EVAL_LLM_MODEL=gpt-4o-mini
### API key for LLM evaluation (fallback to OPENAI_API_KEY if not set)
# EVAL_LLM_BINDING_API_KEY=your_api_key
### Custom OpenAI-compatible endpoint for LLM evaluation (optional)
# EVAL_LLM_BINDING_HOST=https://api.openai.com/v1

### Embedding Configuration for Evaluation
# EVAL_EMBEDDING_MODEL=text-embedding-3-large
### API key for embeddings (fallback: EVAL_LLM_BINDING_API_KEY -> OPENAI_API_KEY)
# EVAL_EMBEDDING_BINDING_API_KEY=your_embedding_api_key
### Custom OpenAI-compatible endpoint for embeddings (fallback: EVAL_LLM_BINDING_HOST)
# EVAL_EMBEDDING_BINDING_HOST=https://api.openai.com/v1

### Performance Tuning
### Number of concurrent test case evaluations
### Lower values reduce API rate limit issues but increase evaluation time
# EVAL_MAX_CONCURRENT=2
### TOP_K query parameter of LightRAG (default: 10)
### Number of entities or relations retrieved from KG
# EVAL_QUERY_TOP_K=10
### LLM request retry and timeout settings for evaluation
# EVAL_LLM_MAX_RETRIES=5
# EVAL_LLM_TIMEOUT=180

##########################################################################
### ----- Preserved custom environment variables from previous .env  -----
### ----- Comments in this session will persist across regenerations -----
### (This must be the final session; ensure the preceding lines unchanged)
##########################################################################

### Default Storage (Recommended for test deployment)
# LIGHTRAG_KV_STORAGE=JsonKVStorage
# LIGHTRAG_DOC_STATUS_STORAGE=JsonDocStatusStorage
# LIGHTRAG_GRAPH_STORAGE=NetworkXStorage
# LIGHTRAG_VECTOR_STORAGE=NanoVectorDBStorage

### Production Storage
# LIGHTRAG_KV_STORAGE=RedisKVStorage
# LIGHTRAG_DOC_STATUS_STORAGE=RedisDocStatusStorage
# LIGHTRAG_VECTOR_STORAGE=QdrantVectorDBStorage
# LIGHTRAG_GRAPH_STORAGE=MemgraphStorage

### Select OpenSearch for all storages
# LIGHTRAG_KV_STORAGE=OpenSearchKVStorage
# LIGHTRAG_DOC_STATUS_STORAGE=OpenSearchDocStatusStorage
# LIGHTRAG_GRAPH_STORAGE=OpenSearchGraphStorage
# LIGHTRAG_VECTOR_STORAGE=OpenSearchVectorDBStorage

### Select PostgreSQL for all storages
# LIGHTRAG_KV_STORAGE=PGKVStorage
# LIGHTRAG_DOC_STATUS_STORAGE=PGDocStatusStorage
# LIGHTRAG_GRAPH_STORAGE=PGGraphStorage
# LIGHTRAG_VECTOR_STORAGE=PGVectorStorage

### Select MongoDB for all storage (Vector storage requires an Atlas-capable deployment)
# LIGHTRAG_KV_STORAGE=MongoKVStorage
# LIGHTRAG_DOC_STATUS_STORAGE=MongoDocStatusStorage
# LIGHTRAG_GRAPH_STORAGE=MongoGraphStorage
# LIGHTRAG_VECTOR_STORAGE=MongoVectorDBStorage

### ----- Extra setting from previous .env -----
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
-问答测试
![](/blogs/TencentOS-install-lightRAG/50249074dc9bc9fe.png)
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