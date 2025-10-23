# JoyAgent-JDGenie Docker 部署指南

## 📋 目录
- [环境要求](#环境要求)
- [快速部署](#快速部署)
- [配置说明](#配置说明)
- [常见问题](#常见问题)

---

## 环境要求

### 硬件要求
- CPU: 2核及以上
- 内存: 4GB及以上
- 磁盘: 10GB可用空间

### 软件要求
- Docker: 20.10+
- Docker Compose: 1.29+ (可选)

---

## 快速部署

### 1. 克隆项目

```bash
git clone https://github.com/jd-opensource/joyagent-jdgenie.git
cd joyagent-jdgenie
```

### 2. 配置LLM服务

#### 2.1 配置后端LLM

编辑 `genie-backend/src/main/resources/application.yml`:

```yaml
llm:
  default:
    base_url: 'https://api.openai.com/v1'  # 修改为你的LLM服务地址
    apikey: '<input llm key>'               # 替换为你的API密钥
    model: gpt-4.1                          # 修改为你的模型名称
    max_tokens: 16384                       # 根据模型调整
```

**使用DeepSeek示例:**
```yaml
llm:
  default:
    base_url: 'https://api.deepseek.com/v1'
    apikey: 'sk-xxx'
    model: deepseek-chat
    max_tokens: 8192  # DeepSeek限制
```

#### 2.2 配置genie-tool环境变量

编辑 `genie-tool/.env_template`:

```bash
# 基础LLM配置
OPENAI_API_KEY=<input llm key>
OPENAI_BASE_URL=https://api.openai.com/v1

# 如果使用DeepSeek
# DEEPSEEK_API_KEY=sk-xxx
# DEEPSEEK_API_BASE=https://api.deepseek.com/v1
# DEFAULT_MODEL=deepseek/deepseek-chat

# 搜索API配置(可选但推荐)
SERPER_SEARCH_API_KEY=<input serper search api key>
```

**重要提示:**
- 如果使用DeepSeek,请确保将所有`${DEFAULT_MODEL}`替换为`deepseek/deepseek-chat`
- 如果不配置`SERPER_SEARCH_API_KEY`,网络搜索功能将无法使用

### 3. 构建Docker镜像

```bash
docker build -t genie:latest .
```

**注意事项:**
- 构建过程可能需要10-20分钟,取决于网络速度
- Dockerfile已配置国内镜像源(npm和pip),加速构建
- 如遇到网络问题,可多次重试

### 4. 启动容器

```bash
docker run -d \
  -p 3000:3000 \
  -p 8080:8080 \
  -p 1601:1601 \
  --name genie-app \
  genie:latest
```

### 5. 访问服务

打开浏览器访问: **http://localhost:3000**

### 6. 检查服务状态

```bash
# 查看容器日志
docker logs -f genie-app

# 检查服务端口
docker ps | grep genie-app

# 停止服务
docker stop genie-app

# 重启服务
docker restart genie-app

# 删除容器
docker rm -f genie-app
```

---

## 配置说明

### 核心配置文件

#### 1. `genie-backend/src/main/resources/application.yml`

**关键配置项说明:**

```yaml
llm:
  default:
    base_url: '<LLM服务地址>'
    apikey: '<LLM API密钥>'
    model: '<模型名称,如gpt-4.1或deepseek-chat>'
    max_tokens: 16384  # 根据模型调整

autobots:
  genie:
    # 工具服务地址配置
    code_interpreter_url: "https://joyagent-llm.datamunger.io"
    deep_search_url: "https://joyagent-llm.datamunger.io"
    knowledge_url: "https://joyagent-llm.datamunger.io"
    data_analysis_url: "https://joyagent-llm.datamunger.io"

    # 如果自建genie-tool服务,改为:
    # code_interpreter_url: "http://127.0.0.1:1601"
    # deep_search_url: "http://127.0.0.1:1601"
    # ...
```

**重要变更:**
- 默认配置使用远程服务`https://joyagent-llm.datamunger.io`
- 如果需要本地运行genie-tool,需要将所有URL改回`http://127.0.0.1:1601`

#### 2. `genie-tool/.env_template`

**必填配置:**

```bash
# LLM服务配置
OPENAI_API_KEY=<必填>
OPENAI_BASE_URL=https://api.openai.com/v1
DEFAULT_MODEL=gpt-4.1

# 搜索引擎配置(可选)
SERPER_SEARCH_API_KEY=<可选,用于Google搜索>

# 文件服务配置
FILE_SERVER_URL=https://joyagent-llm.datamunger.io/v1/file_tool
# 本地运行改为: http://127.0.0.1:1601/v1/file_tool
```

### 使用不同LLM提供商

#### OpenAI

```yaml
# application.yml
llm:
  default:
    base_url: 'https://api.openai.com/v1'
    apikey: 'sk-xxx'
    model: 'gpt-4-turbo'
```

```bash
# .env_template
OPENAI_API_KEY=sk-xxx
OPENAI_BASE_URL=https://api.openai.com/v1
DEFAULT_MODEL=gpt-4-turbo
```

#### DeepSeek

```yaml
# application.yml
llm:
  default:
    base_url: 'https://api.deepseek.com/v1'
    apikey: 'sk-xxx'
    model: 'deepseek-chat'
    max_tokens: 8192  # DeepSeek限制
```

```bash
# .env_template
DEEPSEEK_API_KEY=sk-xxx
DEEPSEEK_API_BASE=https://api.deepseek.com/v1
DEFAULT_MODEL=deepseek/deepseek-chat

# 重要:需要将配置文件中所有${DEFAULT_MODEL}替换为deepseek/deepseek-chat
```

#### Anthropic Claude

```bash
# .env_template
ANTHROPIC_API_KEY=sk-ant-xxx
ANTHROPIC_API_BASE=https://api.anthropic.com
DEFAULT_MODEL=anthropic/claude-3-5-sonnet-20241022
```

