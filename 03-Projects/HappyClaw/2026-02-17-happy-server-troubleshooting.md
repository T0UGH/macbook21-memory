# Happy Server 自托管问题排查总结

## 问题描述

用户在 AWS 上自托管 Happy Server，连接移动端和桌面客户端时遇到多个问题。

## 问题 1: Docker 镜像不存在

**错误**: `slopus/happy-server:latest` 镜像不存在

**解决**: 从 GitHub 克隆源码并本地构建

```bash
git clone https://github.com/happy-hacking/happy.git
cd happy/packages/happy-server
docker build -t happy-server:latest .
```

## 问题 2: 缺少环境变量

**错误**: 数据库连接失败、存储配置缺失

**解决**: 在 docker-compose.yml 中添加必要环境变量

```yaml
environment:
  - NODE_ENV=production
  - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/happy-server
  - REDIS_URL=redis://redis:6379
  - HANDY_MASTER_SECRET=<生成密钥>
  - S3_HOST=minio
  - S3_PORT=9000
  - S3_USE_SSL=false
  - S3_ACCESS_KEY=minioadmin
  - S3_SECRET_KEY=minioadmin
  - S3_BUCKET=happy-server
  - S3_PUBLIC_URL=http://13.192.204.28:9000/happy-server
  - SEED=<生成密钥>
  - PORT=3000
```

## 问题 3: 客户端连接错误服务器

**错误**: 客户端一直连接官方服务器，而非自托管服务器

**解决**:
1. 设置环境变量
```bash
export HAPPY_SERVER_URL="https://t0u9h.online"
```
2. 重新认证
```bash
happy auth login --force
```

## 问题 4: WebSocket 连接失败

**错误**: 桌面客户端 WebSocket 连接一直失败，错误信息 `TransportError`

**排查过程**:
1. 发现是 ShadowsocksX-NG 代理在 global 模式干扰连接
2. 关闭代理后仍然失败
3. 发现是 TLS 证书链问题：`unable to get local issuer certificate`

**解决**: 在 Mac 上添加环境变量指定根证书

```bash
echo 'export NODE_EXTRA_CA_CERTS=/etc/ssl/cert.pem' >> ~/.zshrc
source ~/.zshrc
```

然后重启 daemon

## 问题 5: Caddy HTTP/2 导致 WebSocket 失败

**错误**: Caddy 默认使用 HTTP/2，Socket.io WebSocket 升级失败

**解决**: 修改 Caddyfile 强制使用 HTTP/1.1

```
https://t0u9h.online {
    reverse_proxy localhost:3000 {
        transport http {
            versions 1.1
        }
    }
}
```

## 最终配置

### docker-compose.yml
```yaml
version: '3.8'
services:
  happy-server:
    image: happy-server:latest
    ports:
      - "3000:3000"
    restart: unless-stopped
    volumes:
      - ./prisma:/app/prisma
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/happy-server
      - REDIS_URL=redis://redis:6379
      - HANDY_MASTER_SECRET=<密钥>
      - S3_HOST=minio
      - S3_PORT=9000
      - S3_USE_SSL=false
      - S3_ACCESS_KEY=minioadmin
      - S3_SECRET_KEY=minioadmin
      - S3_BUCKET=happy-server
      - S3_PUBLIC_URL=http://13.192.204.28:9000/happy-server
      - SEED=<密钥>
      - PORT=3000

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=happy-server

  redis:
    image: redis:7-alpine

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    volumes:
      - minio-data:/data

volumes:
  minio-data:
```

### Caddyfile
```
https://t0u9h.online {
    reverse_proxy localhost:3000 {
        transport http {
            versions 1.1
        }
    }
}
```

### ~/.zshrc
```bash
export HAPPY_SERVER_URL="https://t0u9h.online"
export NODE_EXTRA_CA_CERTS=/etc/ssl/cert.pem
```

## 关键命令

```bash
# 启动服务器
cd ~/happy-server && sudo docker-compose up -d

# 查看日志
sudo docker-compose logs happy-server --tail 30

# 重启 daemon
pkill -f "happy-coder.*daemon"
source ~/.zshrc && happy daemon start &

# 强制重新认证
happy auth login --force
```

## 经验总结

1. **代理问题**: ShadowsocksX-NG global 模式会拦截所有流量，需要关闭或切换到 PAC 模式
2. **TLS 证书链**: Node.js 需要完整的证书链，Mac 上可能需要设置 `NODE_EXTRA_CA_CERTS`
3. **HTTP/2 兼容**: Socket.io WebSocket 需要 HTTP/1.1，Caddy 需要配置 transport
4. **环境变量**: 修改环境变量后需要重启 daemon 才能生效
5. **账号一致性**: 移动端和桌面端必须登录同一个账号才能收发消息
