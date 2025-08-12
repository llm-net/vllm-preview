# vLLM Preview Repository

## 概述
本仓库是 vllm-project/vllm 的 fork，用于快速创建支持新模型的 vLLM Docker 镜像。

## 工作流程

每次根据需求，我将执行以下步骤：

### 1. 拉取最新代码
```bash
git pull origin main
```
确保本地代码与上游仓库保持同步

### 2. 创建 Dockerfile
- 在 `previews/` 目录下创建新的 Dockerfile
- 基于 `docker/Dockerfile` 文件作为模板
- 根据要求添加新的 transformers 版本或其他必要模块
- 镜像 tag 由用户指定

### 3. 构建镜像
```bash
docker build -f previews/[Dockerfile名称] -t [镜像tag] .
```

### 4. 测试镜像
运行测试以验证新镜像功能正常

### 5. 推送镜像
将构建好的镜像推送到 Docker Hub：
```bash
docker push [镜像tag]
```

## 目录结构
- `docker/Dockerfile` - 基础 Dockerfile 模板
- `previews/` - 存放新创建的 Dockerfile 文件

## 注意事项
- 每个新的 Dockerfile 都基于 docker/Dockerfile 进行修改
- 根据具体需求调整 transformers 版本和依赖
- 确保测试通过后再推送到 Docker Hub