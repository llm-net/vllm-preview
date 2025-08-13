# vLLM Preview Repository

## 概述
本仓库是 vllm-project/vllm 的 fork，用于快速创建支持新模型的 vLLM Docker 镜像。

## 构建策略

### 简化构建方案
直接从 CUDA 12.8 基础镜像开始，通过 pip 安装 vLLM 和依赖，避免复杂的编译过程：

1. **基础镜像**：使用 `nvidia/cuda:12.8.1-runtime-ubuntu22.04` 作为基础
2. **安装方式**：通过 pip 直接安装预编译的 vLLM wheel 包
3. **兼容性**：确保与标准 vllm-openai 镜像使用方式一致（相同的入口点和环境变量）

## 工作流程

每次根据需求，我将执行以下步骤：

### 1. 同步上游代码（可选）
如果需要最新的配置或示例文件：
```bash
# 添加上游仓库（如果尚未添加）
git remote add upstream https://github.com/vllm-project/vllm.git

# 获取上游最新代码
git fetch upstream

# 合并上游主分支到本地
git merge upstream/main
```

### 2. 创建简化 Dockerfile
在 `previews/` 目录下创建新的 Dockerfile，内容结构：
```dockerfile
# 基础镜像
FROM nvidia/cuda:12.8.1-runtime-ubuntu22.04

# 安装 Python 和基础依赖
RUN apt-get update && apt-get install -y python3 python3-pip ...

# 安装 vLLM 和依赖
RUN pip install -U vllm --pre --extra-index-url https://wheels.vllm.ai/nightly

# 安装特定版本的包（如新模型支持）
RUN pip install [特定包名称和版本]

# 设置工作目录和环境变量
WORKDIR /workspace
ENV VLLM_USAGE_SOURCE=production-docker-image

# 设置入口点（与标准 vllm-openai 镜像一致）
ENTRYPOINT ["python3", "-m", "vllm.entrypoints.openai.api_server"]
```

### 3. 构建镜像
**重要**：构建需要启用 Docker BuildKit（因为 Dockerfile 使用了 --mount 缓存选项）

#### 安装 Docker Buildx（如果未安装）
```bash
# 创建 Docker CLI 插件目录
mkdir -p ~/.docker/cli-plugins

# 下载 buildx 二进制文件
curl -L "https://github.com/docker/buildx/releases/download/v0.13.1/buildx-v0.13.1.linux-amd64" \
  -o ~/.docker/cli-plugins/docker-buildx

# 添加执行权限
chmod +x ~/.docker/cli-plugins/docker-buildx

# 验证安装
docker buildx version

# 创建并使用 buildx builder
docker buildx create --name mybuilder --use
docker buildx inspect --bootstrap
```

#### 执行构建
使用 buildx 构建镜像：
```bash
docker buildx build -f previews/[Dockerfile名称] -t [镜像tag] --load . 2>&1 | tee preview_build.log
```

或使用传统方式（需要 BuildKit）：
```bash
DOCKER_BUILDKIT=1 docker build -f previews/[Dockerfile名称] -t [镜像tag] . 2>&1 | tee preview_build.log
```

注意：
- 必须安装 docker-buildx 插件或设置 `DOCKER_BUILDKIT=1` 环境变量
- BuildKit 提供更高效的缓存机制，加速构建过程
- 使用 buildx 时需要加 `--load` 参数将镜像加载到本地 Docker
- 如果遇到 "--mount option requires BuildKit" 错误，请确保正确安装了 buildx

### 4. 测试镜像
运行测试以验证新镜像功能正常

### 5. 推送镜像
将构建好的镜像推送到 Docker Hub：
```bash
# 先给镜像打标签
docker tag [镜像tag] llmnet/vllm-preview:[镜像tag]

# 推送到 Docker Hub（使用账号 llmnet）
docker push llmnet/vllm-preview:[镜像tag]
```

## 目录结构
- `docker/Dockerfile` - 基础 Dockerfile 模板
- `previews/` - 存放新创建的 Dockerfile 文件

## 已发布镜像

### GLM-4.5V 支持镜像
- **镜像名称**: `llmnet/vllm-preview:glm-4.5v-20250812`
- **构建日期**: 2025-08-13
- **镜像大小**: 16.1GB
- **Digest**: `sha256:0b0e612f6762c40ec0314c23f971d4c590ef4e28b5d1f70b280b4fb36f9d6093`
- **特性**: 
  - 支持 GLM-4.5V 多模态模型
  - 包含 transformers-v4.55.0-GLM-4.5V-preview (4.56.0.dev0)
  - 支持 FP8 量化 (compressed-tensors)
  - 支持 tensor parallelism 多GPU推理
- **使用示例**:
```bash
docker run --rm --gpus all \
    -v /path/to/models:/models \
    -p 8000:8000 \
    -e VLLM_ALLOW_LONG_MAX_MODEL_LEN=1 \
    llmnet/vllm-preview:glm-4.5v-20250812 \
    --model /models/zai-org/GLM-4.5V-FP8 \
    --max-model-len 131072 \
    --tensor-parallel-size 4 \
    --tool-call-parser glm45 \
    --reasoning-parser glm45 \
    --enable-auto-tool-choice \
    --served-model-name zai-org/GLM-4.5V \
    --allowed-local-media-path / \
    --media-io-kwargs '{"video": {"num_frames": -1}}'
```

## 注意事项
- 每个新的 Dockerfile 都基于 docker/Dockerfile 进行修改
- 根据具体需求调整 transformers 版本和依赖
- 确保测试通过后再推送到 Docker Hub
- GLM-4.5V 模型需要多GPU支持（单GPU会OOM）
- 使用 tensor-parallel-size 参数时确保 NCCL 配置正确