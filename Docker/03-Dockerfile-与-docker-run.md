---
title: "Dockerfile 与 docker run"
published: 2026-09-04
description: "先搞清楚镜像、容器、仓库三个概念，再看如何用 Dockerfile 描述构建过程、用 docker run 启动容器。"
category: Docker
tags: ["Dockerfile", "docker run", "镜像"]
draft: false
---
## 一、镜像、容器与仓库

- **镜像**：包含文件系统层和运行配置的只读模板。
- **容器**：镜像的运行实例，拥有自己的可写层与进程生命周期。
- **仓库**：保存和分发镜像的服务。

## 一、镜像、容器与仓库

- **镜像**：包含文件系统层和运行配置的只读模板。
- **容器**：镜像的运行实例，拥有自己的可写层与进程生命周期。
- **仓库**：保存和分发镜像的服务。

## 二、Dockerfile 指令速查

### 2.1 基础指令

| 指令 | 作用 | 示例 |
|------|------|------|
| `FROM` | 开始构建阶段；前面可以有全局 `ARG`、注释和解析器指令 | `FROM ubuntu:22.04` |
| `MAINTAINER` | 声明作者（已过时，推荐用 `LABEL`） | `MAINTAINER CloudStudio author` |
| `LABEL` | 以键值对形式写元数据 | `LABEL env=production` |
| `WORKDIR` | 设置后续指令的工作目录（不存在会自动创建） | `WORKDIR /app` |
| `USER` | 指定运行用户，**避免用 root** | `USER www-data` |
| `ARG` | 构建时参数，唯一可出现在 `FROM` 之前的指令 | `ARG VERSION=latest` |

`ARG` 与 `ENV` 的区别要记住：

```
ARG  → 构建参数，不自动成为容器环境变量；值仍可能出现在构建历史或证明信息里，不能用于传递密钥
ENV  → 构建期和运行期都在，会持久化到容器里
```

### 2.2 文件操作：COPY 还是 ADD？

| 指令 | 能力 | 建议 |
|------|------|------|
| `COPY` | 单纯复制本地文件/目录 | **默认就用它** |
| `ADD` | 额外支持 URL 下载、tar 自动解压 | 只有需要这两个特性时才用 |

```dockerfile
# 本地目录
COPY site/ /var/www/html/
# 远程下载
ADD https://ftp.xxx/jenkins/war/2.397/jenkins.war /data/app
# 自动解压
ADD project.tar.gz /install/
```

### 2.3 执行指令：最容易混淆的三个

| 指令 | 执行时机 | 能否被覆盖 | 典型用途 |
|------|----------|-----------|----------|
| `RUN` | `docker build` 时 | 不适用 | 装软件、建目录、编译 |
| `CMD` | `docker run` 时 | ✅ 可覆盖 | 容器默认启动命令 |
| `ENTRYPOINT` | `docker run` 时 | 可用 `--entrypoint` 覆盖 | 固定默认入口，exec 形式可搭配 CMD 提供默认参数 |

```dockerfile
# 构建时装软件
RUN apt-get install -y nginx
# 启动时跑服务
CMD ["nginx", "-g", "daemon off;"]
```

覆盖示例：

```bash
docker run myimage nginx -g "daemon off;"   # 覆盖 CMD
docker run myimage -v                       # exec ENTRYPOINT 下替换 CMD 参数
docker run --entrypoint sh myimage          # 覆盖 ENTRYPOINT
```

> **推荐 `CMD` 用 exec 形式（JSON 数组）**：`CMD ["nginx","-g","daemon off;"]`。
> 写 `CMD nginx -g "daemon off;"`（shell 形式）会额外起一个 shell 进程，导致信号无法正确传递给主进程，`docker stop` 时容器不会优雅退出。

### 2.4 配置指令

| 指令 | 作用 | 示例 |
|------|------|------|
| `ENV` | 设置环境变量 | `ENV DEBIAN_FRONTEND=noninteractive` |
| `EXPOSE` | **声明**容器监听端口 | `EXPOSE 80` |
| `VOLUME` | 声明数据卷 | `VOLUME ["/data"]` |

`EXPOSE` 只是**声明/文档**，不会真的开放端口。发布到宿主机端口通常用运行时的 `-p`；同网容器通信无需先发布端口：

```bash
docker run -p 8080:80 myimage    # 宿主机8080 → 容器80
```

---

## 三、从 Dockerfile 到容器的四个阶段

```bash
# 1. 默认读取名为 Dockerfile 的文件；其他文件名可通过 -f 指定
vim Dockerfile

# 2. 编写指令（见下文实战）

# 3. 构建镜像，-t 打标签，末尾的 . 是构建上下文（当前目录）
docker build -t mywebserver:1.0 .

# 4. 运行容器
docker run -d --name myweb -p 80:80 mywebserver:1.0
```

**关于构建上下文**：`docker build` 末尾的 `.` 不是"Dockerfile 所在目录"这么简单，而是**要把哪些文件打包发送给 Docker 守护进程**。所以：

- `COPY ./nginx.conf ...` 的源路径必须在上下文内；
- 上下文里塞一堆无用文件会拖慢构建 → 用 `.dockerignore` 排除。

---

## 四、实战：把 Web 服务器装进容器

### 4.1 版本 A：Apache（教学内容示例）

```dockerfile
FROM ubuntu
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update
RUN apt-get install apache2 -y
RUN apt-get install apache2-utils -y
RUN apt-get clean
EXPOSE 80
CMD ["apache2ctl","-D","FOREGROUND"]
```

逐句说明：

| 行 | 作用 |
|----|------|
| `FROM ubuntu` | 以官方 Ubuntu 为基础镜像 |
| `ENV DEBIAN_FRONTEND=noninteractive` | 装软件时弹出交互式对话框会让构建**卡死**，这条让它全部用默认值 |
| `RUN apt-get update` | 刷新软件源索引，不做的话常报 `Unable to locate package` |
| `RUN apt-get install apache2 -y` | 安装 Apache；`-y` 自动回答 yes |
| `RUN apt-get install apache2-utils -y` | 安装配套工具（`ab` 压测、`htpasswd` 等） |
| `RUN apt-get clean` | 清理下载缓存，减小镜像 |
| `EXPOSE 80` | 声明监听 80 端口 |
| `CMD ["apache2ctl","-D","FOREGROUND"]` | **前台**启动 Apache |

### 4.2 版本 B：nginx（优化版，推荐照这个写）

```dockerfile
FROM ubuntu:22.04
ENV DEBIAN_FRONTEND=noninteractive

# 合并 RUN，减少层数；同一层内清理缓存，镜像才真的变小
RUN apt-get update \
 && apt-get install -y nginx \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

# 放入自定义配置
COPY nginx.conf /etc/nginx/nginx.conf
# 放入站点内容
COPY site/ /var/www/html/

EXPOSE 80
# daemon off = 前台运行
CMD ["nginx", "-g", "daemon off;"]
```

配套的 `nginx.conf`（最小可用）：

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 768;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    sendfile on;
    keepalive_timeout 65;

    server {
        listen 80 default_server;
        server_name _;
        root /var/www/html;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

### 4.3 三个必踩的坑

**坑 1：服务后台运行，容器秒退**

容器的生命周期 = 主进程的生命周期。Apache/nginx 默认以守护进程方式启动，主进程 fork 完就退出，容器随之 Exited。

```dockerfile
# ✅ 前台
CMD ["apache2ctl","-D","FOREGROUND"]
# ✅ 前台
CMD ["nginx","-g","daemon off;"]
# ❌ 后台，容器立刻退出
CMD ["service","nginx","start"]
```

**坑 2：`COPY` 的源文件不在构建上下文里**

```
COPY ./nginx.conf /etc/nginx/nginx.conf
ERROR: "/nginx.conf": not found
```

解决：确认宿主机当前目录下真有 `nginx.conf`，且没被 `.dockerignore` 排除。

**坑 3：镜像名当成了占位符**

```bash
docker run -d -p 80:80 tag_name     # ❌ tag_name 是文档占位符
docker run -d --name myweb -p 80:80 mywebserver:1.0   # ✅ 换成真实镜像名
```

报错特征（本地没有 + 远程也没有）：

```
Unable to find image 'tag_name:latest' locally
pull access denied for tag_name, repository does not exist
```

### 4.4 验证

```bash
docker ps                      # STATUS 应为 Up
docker logs myweb              # 启动失败时看这里
curl http://localhost:80       # 或平台「端口管理」打开 80 端口
docker exec -it myweb bash     # 进容器内部排查
```

---

## 五、写出好 Dockerfile 的六个习惯

1. **按兼容性和维护状态选基础镜像**。Alpine 通常更小，但使用 musl，部分原生依赖可能需要改用 Debian/Ubuntu slim 镜像。
2. **合并 `RUN`**，并在同一层内清理缓存：
   ```dockerfile
   RUN apt-get update && apt-get install -y nginx && apt-get clean && rm -rf /var/lib/apt/lists/*
   ```
   分开写多条 `RUN`，前一层已写进去的缓存在后一层删不掉，镜像不会变小。
3. **必须配 `.dockerignore`**，排除 `.git`、`node_modules`、`*.md`、日志等。
4. **善用构建缓存**：把变化少的指令（`FROM`、装依赖）放前面，变化频繁的（`COPY` 代码）放后面。
5. **用 `COPY` 而非 `ADD`**，用 `WORKDIR` 而非一堆 `cd`，避免以 root 运行。
6. **多阶段构建**瘦身，尤其适合编译型项目：
   ```dockerfile
   FROM maven:3.5-jdk-8-alpine AS build
   WORKDIR /app
   COPY . .
   RUN mvn install

   FROM openjdk:8-jre-alpine
   WORKDIR /app
   COPY --from=build /app/target/app.jar app.jar
   CMD ["java","-jar","app.jar"]
   ```
   第一阶段负责编译，只把产物捞到第二阶段，JDK、Maven 依赖全部不进最终镜像。

---

## 六、排错清单

| 现象 | 原因与处理 |
|------|-----------|
| `Unable to locate package xxx` | 缺 `apt-get update` |
| 构建卡在交互界面 | 加 `ENV DEBIAN_FRONTEND=noninteractive` 和 `-y` |
| 容器一启动就 Exited | 服务没前台运行（见坑 1） |
| `COPY xxx: not found` | 文件不在上下文或被 `.dockerignore` 排除 |
| `pull access denied` | 镜像名错误或不存在（见坑 3） |
| 端口访问不通 | 漏了 `-p`；或容器内服务只监听 `127.0.0.1` |
| `bind: address already in use` | 宿主机端口被占用，换一个：`-p 8080:80` |
| `mvn install` 报 no POM | 目录里没有 `pom.xml` |
| 镜像过大 | 多阶段构建 + 同层清理缓存 |
| 改了代码容器里没变 | `COPY` 是构建时复制，改完必须重新 `docker build` |

---

## 七、附录：可复用的脚本模板

把下面两个文件放进项目根目录，`chmod +x` 后即可用，避免每次手敲长命令。

### build.sh

```bash
#!/usr/bin/env bash
# 构建镜像：读取当前目录下的 Dockerfile
set -e

IMAGE_NAME=${IMAGE_NAME:-mywebserver}
IMAGE_TAG=${IMAGE_TAG:-1.0}

docker build -t "${IMAGE_NAME}:${IMAGE_TAG}" .

echo "✅ 镜像构建完成：${IMAGE_NAME}:${IMAGE_TAG}"
echo "   查看详情：docker images ${IMAGE_NAME}"
```

### run.sh

```bash
#!/usr/bin/env bash
# 运行容器：把容器端口映射到宿主机端口
set -e

IMAGE_NAME=${IMAGE_NAME:-mywebserver}
IMAGE_TAG=${IMAGE_TAG:-1.0}
CONTAINER_NAME=${CONTAINER_NAME:-myweb}
HOST_PORT=${HOST_PORT:-80}            # 宿主机端口
CONTAINER_PORT=${CONTAINER_PORT:-80}  # 容器内应用实际监听的端口；EXPOSE 不会强制设置它

# 同名容器已存在则先移除，避免名称/端口冲突
if docker ps -a --format '{{.Names}}' | grep -q "^${CONTAINER_NAME}\$"; then
  echo "⚠️  容器 ${CONTAINER_NAME} 已存在，先移除"
  docker rm -f "${CONTAINER_NAME}" >/dev/null
fi

docker run -d \
  --name "${CONTAINER_NAME}" \
  -p "${HOST_PORT}:${CONTAINER_PORT}" \
  "${IMAGE_NAME}:${IMAGE_TAG}"

echo "✅ 容器已启动：${CONTAINER_NAME}"
echo "   访问地址：http://localhost:${HOST_PORT}"
echo "   查看日志：docker logs -f ${CONTAINER_NAME}"
echo "   停止容器：docker stop ${CONTAINER_NAME}"
```

### 用法

```bash
chmod +x build.sh run.sh

./build.sh                                        # 默认构建 mywebserver:1.0
./run.sh                                          # 默认 -p 80:80

IMAGE_NAME=blog IMAGE_TAG=2.0 ./build.sh          # 自定义镜像名与版本
HOST_PORT=8080 CONTAINER_PORT=80 ./run.sh         # 80 被占用时换宿主机端口
docker build --build-arg VERSION=2.0 -t app:2.0 . # 传 ARG 构建参数
```

### 配套 .dockerignore

```
.git
.gitignore
.vscode
.idea
.tutorial
*.md
Dockerfile*
.dockerignore
*.sh
node_modules
target
dist
*.log
```

---

## 写在最后

Dockerfile 语法本身不多，真正决定水平的是这几个判断：

- 服务要**前台**跑，容器才不会秒退；
- 清理缓存要和安装在**同一层**，镜像才真的会变小；
- `COPY` 是构建时快照，改代码就要重新构建；
- 报错先看 `docker logs`，再看 `docker ps -a` 的状态。

## 核对资料

- [Dockerfile 语法及 ENTRYPOINT 覆盖](https://docs.docker.com/reference/dockerfile/)
