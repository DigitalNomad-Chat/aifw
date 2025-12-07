# OneAIFW部署和集成完整教程

## 目录

1. [项目概述](#项目概述)
2. [系统架构](#系统架构)
3. [服务器配置要求](#服务器配置要求)
4. [部署方案](#部署方案)
5. [API集成方式](#api集成方式)
6. [Redis集成方案](#redis集成方案)
7. [工作流集成](#工作流集成)
8. [监控和管理](#监控和管理)
9. [最佳实践](#最佳实践)
10. [故障排除](#故障排除)

## 项目概述

OneAIFW是一个本地轻量级的"AI防火墙"，专门用于在使用大型语言模型（LLM）时保护隐私数据。它通过匿名化敏感信息，确保数据不会暴露给第三方AI服务。

### 核心功能
- **隐私保护**：物理地址、邮箱、姓名、电话、银行账户、支付信息
- **机密信息保护**：验证码、密码
- **加密货币信息保护**：种子、私钥、地址

### 工作流程
```
原始文本 → 匿名化处理 → LLM处理 → 恢复处理 → 最终结果
```

## 系统架构

### 分层架构设计
```
┌─────────────────┐
│   应用层        │ ← Web应用、浏览器扩展、Python服务
├─────────────────┤
│   语言绑定层    │ ← aifw-js (JavaScript)、aifw-py (Python)
├─────────────────┤
│   核心引擎层    │ ← Zig + Rust (WASM + 原生库)
└─────────────────┘
```

### 技术栈
- **核心引擎**：Zig + Rust（支持WASM和原生编译）
- **识别组件**：正则表达式 + NER模型（Transformers.js）
- **语言绑定**：JavaScript、Python
- **服务框架**：FastAPI + Uvicorn

## 服务器配置要求

### 最低配置（开发/测试环境）
```
CPU: 2核心
内存: 2GB RAM
存储: 10GB SSD
网络: 10Mbps
```

### 推荐配置（生产环境）
```
CPU: 4核心 (支持AVX指令集)
内存: 4-8GB RAM
存储: 20-50GB SSD
网络: 100Mbps+
```

### 高性能配置（高并发场景）
```
CPU: 8核心+ (支持AVX2/AVX512)
内存: 16-32GB RAM
存储: 100GB+ NVMe SSD
网络: 1Gbps+
```

### 资源消耗分析

#### 磁盘空间需求
- **Docker镜像**：~750MB
- **模型文件**：~50MB
- **日志和缓存**：~500MB
- **总需求**：建议预留2GB

#### 内存消耗
- **空闲状态**：~500MB
- **单请求处理**：+50-100MB
- **并发处理**：每请求+30-50MB
- **推荐配置**：轻负载2GB，中负载4GB，高负载8GB+

## 部署方案

### 1. Docker容器化部署（推荐）

#### docker-compose.yml配置
```yaml
version: '3.8'

services:
  # OneAIFW服务
  aifw:
    build:
      context: .
      dockerfile: cli/python/Dockerfile
    ports:
      - "8844:8844"
    environment:
      - AIFW_HTTP_API_KEY=${AIFW_HTTP_API_KEY}
      - AIFW_API_KEY_FILE=/data/aifw/keys.json
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - METADATA_TTL=3600
    volumes:
      - ./config:/data/aifw
      - ./logs:/var/log/aifw
    depends_on:
      - redis
    restart: unless-stopped
    networks:
      - aifw-network
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 1G
          cpus: '0.5'

  # Redis服务
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
    volumes:
      - redis-data:/data
    restart: unless-stopped
    networks:
      - aifw-network

volumes:
  redis-data:

networks:
  aifw-network:
    driver: bridge
```

#### 环境变量配置 (.env)
```bash
# OneAIFW配置
AIFW_HTTP_API_KEY=your-secret-api-key-here
REDIS_PASSWORD=your-redis-password-here

# LLM API配置
OPENAI_API_KEY=your-openai-api-key
OPENAI_BASE_URL=https://api.openai.com/v1

# Redis配置
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0
REDIS_MAX_CONNECTIONS=100

# 元数据TTL配置（秒）
METADATA_TTL_SHORT=300     # 5分钟 - 实时翻译
METADATA_TTL_MEDIUM=3600   # 1小时 - 对话会话
METADATA_TTL_LONG=86400    # 24小时 - 批量处理
```

#### 部署脚本 (deploy.sh)
```bash
#!/bin/bash

echo "🚀 部署OneAIFW + Redis..."

# 创建必要的目录
mkdir -p config logs redis

# 生成随机密码
REDIS_PASSWORD=$(openssl rand -base64 32)
AIFW_API_KEY=$(openssl rand -base64 32)

# 保存密码到.env
cat > .env << EOF
REDIS_PASSWORD=${REDIS_PASSWORD}
AIFW_HTTP_API_KEY=${AIFW_API_KEY}
OPENAI_API_KEY=your-openai-api-key
EOF

echo "🔑 密码已生成并保存到.env"

# 构建镜像
docker-compose build

# 启动服务
docker-compose up -d

echo "⏳ 等待服务启动..."
sleep 10

# 检查服务状态
echo "📊 检查服务状态:"
docker-compose ps

# 测试服务
echo "🧪 测试服务健康状态:"
curl -f http://localhost:8844/api/health || echo "❌ 服务未就绪"

echo "✅ 部署完成!"
```

### 2. Kubernetes部署

#### k8s-deployment.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: aifw-service
spec:
  selector:
    app: aifw
  ports:
  - port: 8844
    targetPort: 8844
  type: ClusterIP

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aifw
spec:
  replicas: 3
  selector:
    matchLabels:
      app: aifw
  template:
    metadata:
      labels:
        app: aifw
    spec:
      containers:
      - name: aifw
        image: oneaifw:latest
        ports:
        - containerPort: 8844
        env:
        - name: AIFW_HTTP_API_KEY
          valueFrom:
            secretKeyRef:
              name: aifw-secrets
              key: api-key
        - name: REDIS_HOST
          value: "redis-service"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /api/health
            port: 8844
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/health
            port: 8844
          initialDelaySeconds: 5
          periodSeconds: 5
```

## API集成方式

OneAIFW主要通过HTTP REST API集成到工作流中。

### API端点概览

```yaml
Base URL: http://your-server:8844
Authentication: Bearer Token (可选)
Content-Type: application/json
```

### 核心API接口

#### 1. 健康检查
```bash
GET /api/health
```

#### 2. 一键式处理（推荐）
```bash
POST /api/call
{
  "text": "请翻译：My email is user@example.com",
  "apiKeyFile": "/path/to/llm-keys.json",
  "model": "gpt-4o-mini",
  "temperature": 0.0
}
```

#### 3. 分步处理
```bash
# 步骤1: 匿名化
POST /api/mask_text
{
  "text": "My email is user@example.com",
  "language": "en"
}

# 响应
{
  "output": {
    "text": "My email is __PII_EMAIL_ADDRESS_00000001__",
    "maskMeta": "base64编码的元数据"
  }
}

# 步骤2: 恢复
POST /api/restore_text
{
  "text": "My email is __PII_EMAIL_ADDRESS_00000001__",
  "maskMeta": "上一步返回的base64元数据"
}
```

#### 4. 批量处理
```bash
POST /api/mask_text_batch
POST /api/restore_text_batch
```

#### 5. 动态配置
```bash
POST /api/config
{
  "maskConfig": {
    "maskEmail": true,
    "maskPhoneNumber": true,
    "maskAddress": false,
    "maskUserName": true
  }
}
```

### Python客户端集成示例

```python
import requests
import time
from typing import Optional

class AIFWClient:
    def __init__(self, base_url="http://localhost:8844", api_key=None):
        self.base_url = base_url
        self.headers = {"Content-Type": "application/json"}
        if api_key:
            self.headers["Authorization"] = f"Bearer {api_key}"

    def protect_and_call(self, text, model="gpt-4o-mini"):
        """一键式处理：最简单的方式"""
        response = requests.post(
            f"{self.base_url}/api/call",
            headers=self.headers,
            json={
                "text": text,
                "model": model,
                "temperature": 0.0
            }
        )
        return response.json()["output"]["text"]

    def mask_text(self, text, language=None):
        """仅匿名化，用于需要中间处理的场景"""
        response = requests.post(
            f"{self.base_url}/api/mask_text",
            headers=self.headers,
            json={"text": text, "language": language}
        )
        result = response.json()["output"]
        return result["text"], result["maskMeta"]

    def restore_text(self, masked_text, mask_meta):
        """恢复文本"""
        response = requests.post(
            f"{self.base_url}/api/restore_text",
            headers=self.headers,
            json={"text": masked_text, "maskMeta": mask_meta}
        )
        return response.json()["output"]["text"]

# 使用示例
def demo_integration():
    client = AIFWClient(api_key="your-secret-key")

    # 方式1: 一键式处理
    result = client.protect_and_call("请翻译：My email is user@example.com")
    print("一键式结果:", result)

    # 方式2: 分步处理（适合复杂工作流）
    original = "Contact John at john@company.com for project details"
    masked, meta = client.mask_text(original)
    print(f"匿名化: {masked}")

    # 在这里可以添加其他处理步骤
    # 比如发送给不同的LLM、进行翻译、分析等

    # 模拟LLM处理
    llm_response = f"I received: {masked}"

    # 恢复文本
    restored = client.restore_text(llm_response, meta)
    print(f"恢复: {restored}")

if __name__ == "__main__":
    demo_integration()
```

## Redis集成方案

在分布式场景下，使用Redis存储元数据是最佳实践。

### Redis配置和集成

#### Redis连接配置
```python
# metadata_manager.py
import redis
import json
import uuid
import time
from typing import Optional, Dict, Any
from dataclasses import dataclass
import os

@dataclass
class MetadataConfig:
    host: str = "localhost"
    port: int = 6379
    password: Optional[str] = None
    db: int = 0
    default_ttl: int = 3600
    max_connections: int = 100

class RedisMetadataManager:
    """基于Redis的元数据管理器"""

    def __init__(self, config: MetadataConfig):
        self.config = config
        self.redis_client = None
        self._connect_redis()

    def _connect_redis(self):
        """连接Redis"""
        try:
            self.redis_client = redis.Redis(
                host=self.config.host,
                port=self.config.port,
                password=self.config.password,
                db=self.config.db,
                max_connections=self.config.max_connections,
                decode_responses=True,
                socket_timeout=5,
                socket_connect_timeout=5,
                retry_on_timeout=True
            )
            # 测试连接
            self.redis_client.ping()
            print(f"✅ Redis连接成功: {self.config.host}:{self.config.port}")
        except Exception as e:
            print(f"❌ Redis连接失败: {e}")
            raise

    def store_metadata(self, mask_meta: str, ttl: Optional[int] = None) -> str:
        """存储元数据，返回request_id"""
        request_id = str(uuid.uuid4())
        ttl = ttl or self.config.default_ttl

        try:
            # 存储到Redis
            key = f"aifw:metadata:{request_id}"
            self.redis_client.setex(key, ttl, mask_meta)

            # 存储元数据信息（用于监控和管理）
            info_key = f"aifw:info:{request_id}"
            info = {
                "created_at": time.time(),
                "ttl": ttl,
                "expires_at": time.time() + ttl,
                "size_bytes": len(mask_meta.encode('utf-8'))
            }
            self.redis_client.setex(info_key, ttl, json.dumps(info))

            print(f"📦 元数据已存储: {request_id}, TTL: {ttl}秒")
            return request_id

        except Exception as e:
            print(f"❌ 元数据存储失败: {e}")
            raise

    def get_metadata(self, request_id: str) -> Optional[str]:
        """获取元数据"""
        try:
            key = f"aifw:metadata:{request_id}"
            mask_meta = self.redis_client.get(key)

            if mask_meta:
                # 更新访问时间
                info_key = f"aifw:info:{request_id}"
                self.redis_client.hincrby(info_key, "access_count", 1)
                return mask_meta
            else:
                print(f"⚠️ 元数据不存在或已过期: {request_id}")
                return None

        except Exception as e:
            print(f"❌ 元数据获取失败: {e}")
            return None

    def get_stats(self) -> Dict[str, Any]:
        """获取统计信息"""
        try:
            # 获取所有元数据键
            metadata_keys = self.redis_client.keys("aifw:metadata:*")
            memory_info = self.redis_client.info("memory")

            stats = {
                "total_metadata_count": len(metadata_keys),
                "used_memory_mb": memory_info.get("used_memory", 0) / 1024 / 1024,
                "used_memory_human": memory_info.get("used_memory_human", "0B"),
                "connected_clients": self.redis_client.info("clients").get("connected_clients", 0)
            }

            return stats

        except Exception as e:
            print(f"❌ 统计信息获取失败: {e}")
            return {}

# 全局实例
metadata_manager = None

def get_metadata_manager() -> RedisMetadataManager:
    """获取元数据管理器单例"""
    global metadata_manager
    if metadata_manager is None:
        config = MetadataConfig(
            host=os.getenv("REDIS_HOST", "localhost"),
            port=int(os.getenv("REDIS_PORT", 6379)),
            password=os.getenv("REDIS_PASSWORD"),
            db=int(os.getenv("REDIS_DB", 0)),
            default_ttl=int(os.getenv("METADATA_TTL", 3600))
        )
        metadata_manager = RedisMetadataManager(config)
    return metadata_manager
```

#### 基于Redis的API接口
```python
# 在main.py中添加Redis版本的API
from .metadata_manager import get_metadata_manager

class MaskInV2(BaseModel):
    text: str
    language: Optional[str] = None
    ttl: Optional[int] = None  # 自定义TTL

class RestoreInV2(BaseModel):
    text: str
    requestId: str  # 使用request_id替代直接的mask_meta

@app.post("/api/mask_text_v2")
async def api_mask_text_v2(inp: MaskInV2, authorization: Optional[str] = Header(None)):
    """基于Redis的匿名化API"""
    check_api_key(authorization)

    try:
        metadata_manager = get_metadata_manager()

        # 执行匿名化
        res = api.mask_text(text=inp.text, language=inp.language)

        # 存储元数据到Redis
        request_id = metadata_manager.store_metadata(
            mask_meta=res["maskMeta"],
            ttl=inp.ttl
        )

        return {
            "output": {
                "requestId": request_id,
                "maskedText": res["text"]
            },
            "error": None
        }
    except Exception as e:
        logger.exception("/api/mask_text_v2 failed")
        return {"output": None, "error": {"message": str(e), "code": None}}

@app.post("/api/restore_text_v2")
async def api_restore_text_v2(inp: RestoreInV2, authorization: Optional[str] = Header(None)):
    """基于Redis的恢复API"""
    check_api_key(authorization)

    try:
        metadata_manager = get_metadata_manager()

        # 从Redis获取元数据
        mask_meta = metadata_manager.get_metadata(inp.requestId)
        if not mask_meta:
            return {
                "output": None,
                "error": {"message": "元数据不存在或已过期", "code": "METADATA_EXPIRED"}
            }

        # 执行恢复
        restored = api.restore_text(text=inp.text, mask_meta=mask_meta)

        return {"output": {"text": restored}, "error": None}
    except Exception as e:
        logger.exception("/api/restore_text_v2 failed")
        return {"output": None, "error": {"message": str(e), "code": None}}

@app.get("/api/metadata/stats")
async def get_metadata_stats(authorization: Optional[str] = Header(None)):
    """获取元数据统计信息"""
    check_api_key(authorization)

    try:
        metadata_manager = get_metadata_manager()
        stats = metadata_manager.get_stats()
        return {"output": stats, "error": None}
    except Exception as e:
        logger.exception("/api/metadata/stats failed")
        return {"output": None, "error": {"message": str(e), "code": None}}
```

### Redis集成客户端示例

```python
class AIFWRedisClient:
    def __init__(self, base_url="http://localhost:8844", api_key=None):
        self.base_url = base_url
        self.headers = {"Content-Type": "application/json"}
        if api_key:
            self.headers["Authorization"] = f"Bearer {api_key}"

    def mask_text_with_redis(self, text: str, language: str = None, ttl: int = 3600) -> tuple:
        """匿名化文本，返回(masked_text, request_id)"""
        response = requests.post(
            f"{self.base_url}/api/mask_text_v2",
            headers=self.headers,
            json={"text": text, "language": language, "ttl": ttl}
        )

        if response.status_code == 200:
            result = response.json()["output"]
            return result["maskedText"], result["requestId"]
        else:
            raise Exception(f"匿名化失败: {response.text}")

    def restore_text_with_redis(self, masked_text: str, request_id: str) -> str:
        """使用request_id恢复文本"""
        response = requests.post(
            f"{self.base_url}/api/restore_text_v2",
            headers=self.headers,
            json={"text": masked_text, "requestId": request_id}
        )

        if response.status_code == 200:
            return response.json()["output"]["text"]
        else:
            error_info = response.json()
            if error_info.get("error", {}).get("code") == "METADATA_EXPIRED":
                raise Exception("元数据已过期，无法恢复文本")
            else:
                raise Exception(f"恢复失败: {response.text}")

# 使用示例
def demo_redis_workflow():
    client = AIFWRedisClient(
        base_url="http://your-cloud-server:8844",
        api_key="your-api-key"
    )

    try:
        # 步骤1: 匿名化
        original_text = "请联系张三，邮箱zhangsan@company.com"
        masked_text, request_id = client.mask_text_with_redis(
            text=original_text,
            ttl=1800  # 30分钟
        )
        print(f"匿名化: {masked_text}")
        print(f"Request ID: {request_id}")

        # 模拟异步处理...
        print("等待5秒模拟异步处理...")
        time.sleep(5)

        # 步骤2: 在另一个进程中恢复
        restored_text = client.restore_text_with_redis(masked_text, request_id)
        print(f"恢复: {restored_text}")

    except Exception as e:
        print(f"处理失败: {e}")
```

## 工作流集成

### 1. 完整的工作流程

#### 单次请求流程
```
用户输入 → 匿名化 → 存储元数据 → LLM处理 → 恢复 → 返回结果
```

#### 分布式场景流程
```
用户输入 → 匿名化 → 存储元数据到Redis → 异步LLM处理 → 从Redis获取元数据 → 恢复 → 返回结果
```

### 2. 实际集成场景

#### ChatGPT集成示例
```python
def secure_chatgpt_integration(user_input):
    """安全的ChatGPT集成示例"""

    client = AIFWRedisClient(
        base_url="http://aifw-server:8844",
        api_key="your-api-key"
    )

    try:
        # 1. 匿名化用户输入，设置较长的TTL
        masked_text, request_id = client.mask_text_with_redis(
            user_input,
            ttl=1800  # 30分钟
        )

        # 2. 发送匿名化文本给ChatGPT
        chatgpt_response = call_chatgpt_api(masked_text)

        # 3. 恢复ChatGPT的回复
        secure_answer = client.restore_text_with_redis(
            chatgpt_response,
            request_id
        )

        return secure_answer

    except Exception as e:
        print(f"处理失败: {e}")
        return "抱歉，处理您的请求时遇到错误"

# 使用示例
def chat_interface():
    print("安全ChatGPT助手（输入'退出'结束）")

    while True:
        user_question = input("\n您: ")
        if user_question.lower() == '退出':
            break

        try:
            ai_response = secure_chatgpt_integration(user_question)
            print(f"助手: {ai_response}")
        except Exception as e:
            print(f"错误: {e}")
```

#### 翻译服务集成示例
```python
class SecureTranslationService:
    def __init__(self, aifw_client):
        self.aifw_client = aifw_client
        self.supported_languages = ['en', 'zh', 'ja', 'ko', 'fr', 'de']

    def translate(self, text, target_language, source_language='auto'):
        """安全翻译服务"""

        try:
            # 1. 匿名化原文
            masked_text, request_id = self.aifw_client.mask_text_with_redis(
                text,
                ttl=3600  # 1小时
            )

            # 2. 调用翻译API
            translated_masked = self.call_translation_api(
                masked_text,
                target_language,
                source_language
            )

            # 3. 恢复翻译结果
            final_result = self.aifw_client.restore_text_with_redis(
                translated_masked,
                request_id
            )

            return final_result

        except Exception as e:
            print(f"翻译失败: {e}")
            return text  # 返回原文

    def call_translation_api(self, text, target_lang, source_lang):
        """调用实际的翻译API（示例）"""
        # 这里可以是Google Translate、DeepL等
        # 为了演示，我们简单返回处理后的文本
        return f"[Translated to {target_lang}]: {text}"

# 使用示例
def demo_translation():
    client = AIFWRedisClient()
    translator = SecureTranslationService(client)

    text_to_translate = "请联系王经理，电话13812345678，邮箱wang@company.com，讨论合同事宜"

    result = translator.translate(text_to_translate, 'en', 'zh')
    print(f"翻译结果: {result}")
```

#### 批量处理示例
```python
class BatchProcessor:
    def __init__(self, aifw_client, batch_size=10):
        self.aifw_client = aifw_client
        self.batch_size = batch_size

    def process_batch(self, text_list, operation_type='translate'):
        """批量处理文本"""

        results = []
        batch_requests = []

        # 阶段1: 批量匿名化
        for i, text in enumerate(text_list):
            try:
                masked_text, request_id = self.aifw_client.mask_text_with_redis(
                    text,
                    ttl=7200  # 2小时
                )
                batch_requests.append({
                    'index': i,
                    'original_text': text,
                    'masked_text': masked_text,
                    'request_id': request_id
                })
            except Exception as e:
                print(f"文本 {i} 匿名化失败: {e}")
                results.append({'index': i, 'error': str(e)})

        # 阶段2: 批量处理（这里可以并发处理）
        for request in batch_requests:
            try:
                if operation_type == 'translate':
                    processed_text = self._translate(request['masked_text'])
                elif operation_type == 'analyze':
                    processed_text = self._analyze(request['masked_text'])
                else:
                    processed_text = request['masked_text']

                # 阶段3: 恢复文本
                restored_text = self.aifw_client.restore_text_with_redis(
                    processed_text,
                    request['request_id']
                )

                results.append({
                    'index': request['index'],
                    'original': request['original_text'],
                    'processed': restored_text
                })

            except Exception as e:
                print(f"文本 {request['index']} 处理失败: {e}")
                results.append({
                    'index': request['index'],
                    'error': str(e)
                })

        # 按原始顺序排序结果
        results.sort(key=lambda x: x['index'])
        return results

    def _translate(self, text):
        """翻译处理"""
        return f"[Translated]: {text}"

    def _analyze(self, text):
        """分析处理"""
        return f"[Analysis]: {text}"

# 使用示例
def demo_batch_processing():
    client = AIFWRedisClient()
    processor = BatchProcessor(client)

    texts = [
        "请联系张三，邮箱zhangsan@example.com",
        "李四的电话是13812345678，地址是北京市朝阳区",
        "王五的银行卡号是6222021234567890",
        "请联系赵六，邮箱zhaoliu@company.com"
    ]

    results = processor.process_batch(texts, 'translate')

    for result in results:
        if 'error' in result:
            print(f"文本 {result['index']} 处理失败: {result['error']}")
        else:
            print(f"原文: {result['original']}")
            print(f"处理结果: {result['processed']}")
            print("-" * 50)
```

## 监控和管理

### 1. 系统监控

#### 监控脚本 (monitor.py)
```python
import requests
import time
import json
from datetime import datetime

class AIFWMonitor:
    def __init__(self, base_url="http://localhost:8844", api_key=None):
        self.base_url = base_url
        self.headers = {"Content-Type": "application/json"}
        if api_key:
            self.headers["Authorization"] = f"Bearer {api_key}"

    def get_stats(self):
        """获取系统统计信息"""
        try:
            response = requests.get(
                f"{self.base_url}/api/metadata/stats",
                headers=self.headers
            )
            if response.status_code == 200:
                return response.json()["output"]
        except:
            pass
        return {}

    def health_check(self):
        """健康检查"""
        try:
            response = requests.get(
                f"{self.base_url}/api/health",
                timeout=5
            )
            return response.status_code == 200
        except:
            return False

    def monitor_loop(self, interval=60):
        """监控循环"""
        print(f"🔍 开始监控 OneAIFW 服务 ({self.base_url})")
        print(f"⏰ 监控间隔: {interval}秒")
        print("-" * 60)

        while True:
            try:
                timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')

                # 健康检查
                is_healthy = self.health_check()

                if is_healthy:
                    stats = self.get_stats()

                    if stats:
                        print(f"✅ [{timestamp}] 系统正常")
                        print(f"   元数据数量: {stats.get('total_metadata_count', 0)}")
                        print(f"   内存使用: {stats.get('used_memory_human', 'N/A')}")
                        print(f"   连接客户端: {stats.get('connected_clients', 0)}")

                        # 告警检查
                        warnings = []

                        if stats.get('total_metadata_count', 0) > 10000:
                            warnings.append("元数据数量过多")

                        if stats.get('used_memory_mb', 0) > 500:
                            warnings.append("内存使用过高")

                        if warnings:
                            print(f"⚠️ 警告: {', '.join(warnings)}")

                        # 自动清理建议
                        if stats.get('total_metadata_count', 0) > 5000:
                            print(f"💡 建议: 考虑清理过期元数据")
                    else:
                        print(f"⚠️ [{timestamp}] 服务正常但无法获取统计信息")
                else:
                    print(f"❌ [{timestamp}] 服务不可用")

            except Exception as e:
                print(f"❌ [{timestamp}] 监控失败: {e}")

            print("-" * 60)
            time.sleep(interval)

    def generate_report(self):
        """生成监控报告"""
        stats = self.get_stats()
        is_healthy = self.health_check()

        report = {
            "timestamp": datetime.now().isoformat(),
            "health_status": "healthy" if is_healthy else "unhealthy",
            "statistics": stats
        }

        return report

if __name__ == "__main__":
    import sys

    # 命令行参数
    if len(sys.argv) > 1:
        api_key = sys.argv[1]
    else:
        api_key = None

    monitor = AIFWMonitor(api_key=api_key)

    if len(sys.argv) > 2 and sys.argv[2] == "report":
        # 生成单次报告
        report = monitor.generate_report()
        print(json.dumps(report, indent=2, ensure_ascii=False))
    else:
        # 持续监控
        monitor.monitor_loop()
```

#### Docker健康检查
```yaml
# 在docker-compose.yml中添加健康检查
aifw:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8844/api/health"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s
```

### 2. 日志管理

#### 日志配置 (logging.yaml)
```yaml
version: 1
disable_existing_loggers: false

formatters:
  standard:
    format: '%(asctime)s [%(levelname)s] %(name)s: %(message)s'
  json:
    format: '{"timestamp": "%(asctime)s", "level": "%(levelname)s", "logger": "%(name)s", "message": "%(message)s"}'

handlers:
  console:
    class: logging.StreamHandler
    level: INFO
    formatter: standard
    stream: ext://sys.stdout

  file:
    class: logging.handlers.RotatingFileHandler
    level: INFO
    formatter: json
    filename: /var/log/aifw/aifw.log
    maxBytes: 104857600  # 100MB
    backupCount: 10

  error_file:
    class: logging.handlers.RotatingFileHandler
    level: ERROR
    formatter: json
    filename: /var/log/aifw/aifw-error.log
    maxBytes: 104857600
    backupCount: 10

loggers:
  aifw:
    level: INFO
    handlers: [console, file, error_file]
    propagate: false

root:
  level: INFO
  handlers: [console, file]
```

### 3. 性能优化

#### 连接池配置
```python
# Redis连接池配置
redis_pool = redis.ConnectionPool(
    host='redis',
    port=6379,
    password='your-password',
    db=0,
    max_connections=100,
    retry_on_timeout=True,
    socket_timeout=5,
    socket_connect_timeout=5
)

redis_client = redis.Redis(connection_pool=redis_pool)
```

#### 缓存策略
```python
class CachedMetadataManager:
    def __init__(self, redis_manager, cache_size=1000):
        self.redis_manager = redis_manager
        self.cache = {}
        self.cache_size = cache_size

    def get_metadata(self, request_id):
        # 1. 先查本地缓存
        if request_id in self.cache:
            return self.cache[request_id]

        # 2. 查Redis
        metadata = self.redis_manager.get_metadata(request_id)

        if metadata:
            # 3. 存入本地缓存（LRU策略）
            if len(self.cache) >= self.cache_size:
                # 删除最旧的条目
                oldest_key = next(iter(self.cache))
                del self.cache[oldest_key]

            self.cache[request_id] = metadata

        return metadata
```

## 最佳实践

### 1. 安全配置

#### API密钥管理
```bash
# 使用环境变量，不要硬编码
export AIFW_HTTP_API_KEY="your-secret-key"
export REDIS_PASSWORD="your-redis-password"
export OPENAI_API_KEY="your-openai-key"
```

#### 网络安全
```yaml
# 使用HTTPS和防火墙
services:
  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - aifw
```

### 2. 性能优化

#### 资源限制
```yaml
# 合理设置资源限制
deploy:
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
```

#### TTL策略
```python
# 根据业务场景设置合适的TTL
TTL_CONFIGS = {
    'real_time': 300,      # 5分钟 - 实时翻译
    'chat_session': 1800,   # 30分钟 - 对话会话
    'batch_processing': 3600,  # 1小时 - 批量处理
    'audit_required': 86400,   # 24小时 - 需要审计
}
```

### 3. 错误处理

#### 重试机制
```python
import time
from functools import wraps

def retry_on_failure(max_retries=3, delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    time.sleep(delay * (2 ** attempt))  # 指数退避
            return None
        return wrapper
    return decorator

# 使用示例
@retry_on_failure(max_retries=3, delay=1)
def safe_api_call(url, data):
    response = requests.post(url, json=data)
    response.raise_for_status()
    return response.json()
```

### 4. 监控告警

#### 告警配置
```python
# 监控阈值配置
ALERT_THRESHOLDS = {
    'metadata_count_warning': 10000,
    'metadata_count_critical': 50000,
    'memory_usage_warning': 80,  # 百分比
    'memory_usage_critical': 95,
    'response_time_warning': 1000,  # 毫秒
    'response_time_critical': 5000
}

def check_alerts(stats):
    alerts = []

    if stats.get('total_metadata_count', 0) > ALERT_THRESHOLDS['metadata_count_warning']:
        alerts.append({
            'level': 'warning' if stats['total_metadata_count'] < ALERT_THRESHOLDS['metadata_count_critical'] else 'critical',
            'message': f"元数据数量过多: {stats['total_metadata_count']}"
        })

    return alerts
```

## 故障排除

### 常见问题和解决方案

#### 1. Redis连接问题
```bash
# 检查Redis连接
docker-compose exec redis redis-cli ping

# 检查Redis配置
docker-compose exec redis redis-cli config get "*"

# 查看Redis日志
docker-compose logs redis
```

#### 2. OneAIFW服务问题
```bash
# 检查服务状态
curl -f http://localhost:8844/api/health

# 查看服务日志
docker-compose logs aifw

# 检查资源使用
docker stats
```

#### 3. 内存使用过高
```python
# 清理过期元数据
def cleanup_expired_metadata():
    response = requests.post(
        "http://localhost:8844/api/metadata/cleanup",
        headers={"Authorization": "Bearer your-api-key"}
    )
    return response.json()

# 监控内存使用
def monitor_memory():
    stats = requests.get(
        "http://localhost:8844/api/metadata/stats",
        headers={"Authorization": "Bearer your-api-key"}
    ).json()

    memory_mb = stats['output'].get('used_memory_mb', 0)
    if memory_mb > 500:  # 500MB警告阈值
        print(f"⚠️ 内存使用过高: {memory_mb}MB")
        cleanup_expired_metadata()
```

#### 4. 元数据过期问题
```python
# 检查TTL设置
def check_ttl_settings():
    config = {
        'default_ttl': int(os.getenv('METADATA_TTL', 3600)),
        'short_ttl': int(os.getenv('METADATA_TTL_SHORT', 300)),
        'long_ttl': int(os.getenv('METADATA_TTL_LONG', 86400))
    }
    return config

# 延长TTL
def extend_ttl(request_id, additional_time=3600):
    response = requests.post(
        f"http://localhost:8844/api/metadata/extend_ttl",
        json={"requestId": request_id, "additionalTime": additional_time},
        headers={"Authorization": "Bearer your-api-key"}
    )
    return response.status_code == 200
```

### 调试工具

#### 调试脚本 (debug.py)
```python
#!/usr/bin/env python3

import requests
import json
import sys

class AIFWDebugger:
    def __init__(self, base_url="http://localhost:8844", api_key=None):
        self.base_url = base_url
        self.headers = {"Content-Type": "application/json"}
        if api_key:
            self.headers["Authorization"] = f"Bearer {api_key}"

    def test_connection(self):
        """测试连接"""
        try:
            response = requests.get(f"{self.base_url}/api/health", timeout=5)
            if response.status_code == 200:
                print("✅ 服务连接正常")
                return True
            else:
                print(f"❌ 服务响应异常: {response.status_code}")
                return False
        except Exception as e:
            print(f"❌ 连接失败: {e}")
            return False

    def test_mask_restore(self, text="测试文本，邮箱test@example.com"):
        """测试匿名化和恢复"""
        print(f"🧪 测试文本: {text}")

        try:
            # 匿名化
            mask_response = requests.post(
                f"{self.base_url}/api/mask_text",
                headers=self.headers,
                json={"text": text}
            )

            if mask_response.status_code != 200:
                print(f"❌ 匿名化失败: {mask_response.text}")
                return False

            mask_result = mask_response.json()["output"]
            masked_text = mask_result["text"]
            mask_meta = mask_result["maskMeta"]

            print(f"📦 匿名化结果: {masked_text}")

            # 恢复
            restore_response = requests.post(
                f"{self.base_url}/api/restore_text",
                headers=self.headers,
                json={"text": masked_text, "maskMeta": mask_meta}
            )

            if restore_response.status_code != 200:
                print(f"❌ 恢复失败: {restore_response.text}")
                return False

            restored_text = restore_response.json()["output"]["text"]
            print(f"🔄 恢复结果: {restored_text}")

            # 验证
            if text == restored_text:
                print("✅ 匿名化-恢复流程测试通过")
                return True
            else:
                print("❌ 恢复结果与原文不匹配")
                return False

        except Exception as e:
            print(f"❌ 测试失败: {e}")
            return False

    def check_redis_status(self):
        """检查Redis状态"""
        try:
            response = requests.get(
                f"{self.base_url}/api/metadata/stats",
                headers=self.headers
            )

            if response.status_code == 200:
                stats = response.json()["output"]
                print("✅ Redis连接正常")
                print(f"   元数据数量: {stats.get('total_metadata_count', 0)}")
                print(f"   内存使用: {stats.get('used_memory_human', 'N/A')}")
                return True
            else:
                print("❌ Redis状态检查失败")
                return False

        except Exception as e:
            print(f"❌ Redis检查异常: {e}")
            return False

    def run_full_diagnosis(self):
        """运行完整诊断"""
        print("🔍 OneAIFW 服务诊断")
        print("=" * 40)

        all_passed = True

        # 1. 连接测试
        print("1. 测试服务连接...")
        if not self.test_connection():
            all_passed = False

        print()

        # 2. Redis状态检查
        print("2. 检查Redis状态...")
        if not self.check_redis_status():
            all_passed = False

        print()

        # 3. 匿名化-恢复流程测试
        print("3. 测试匿名化-恢复流程...")
        if not self.test_mask_restore():
            all_passed = False

        print()
        print("=" * 40)

        if all_passed:
            print("🎉 所有测试通过！服务运行正常。")
        else:
            print("⚠️ 发现问题，请检查服务配置和日志。")

        return all_passed

if __name__ == "__main__":
    import argparse

    parser = argparse.ArgumentParser(description='OneAIFW调试工具')
    parser.add_argument('--url', default='http://localhost:8844', help='OneAIFW服务地址')
    parser.add_argument('--api-key', help='API密钥')
    parser.add_argument('--test', choices=['connection', 'redis', 'mask_restore'], help='指定测试项目')

    args = parser.parse_args()

    debugger = AIFWDebugger(args.url, args.api_key)

    if args.test:
        if args.test == 'connection':
            debugger.test_connection()
        elif args.test == 'redis':
            debugger.check_redis_status()
        elif args.test == 'mask_restore':
            debugger.test_mask_restore()
    else:
        debugger.run_full_diagnosis()
```

## 总结

本教程涵盖了OneAIFW项目的完整部署和集成流程，包括：

1. **系统架构理解**：掌握分层架构和技术栈
2. **服务器配置**：根据需求选择合适的硬件配置
3. **部署方案**：Docker和Kubernetes部署方法
4. **API集成**：REST API的使用方式和最佳实践
5. **Redis集成**：分布式场景下的元数据管理
6. **工作流集成**：实际业务场景中的集成示例
7. **监控管理**：系统监控、日志管理和性能优化
8. **故障排除**：常见问题的诊断和解决方案

通过本教程，你应该能够：
- 成功部署OneAIFW服务到云服务器
- 理解元数据的生命周期管理
- 实现安全的AI工作流集成
- 进行系统监控和故障排除

OneAIFW为企业使用外部LLM服务提供了一个安全、可靠的隐私保护解决方案，让企业可以在保护敏感数据的前提下，充分利用大语言模型的能力。