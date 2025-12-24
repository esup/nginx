# NGINX 中文文档

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://github.com/user-attachments/assets/9335b488-ffcc-4157-8364-2370a0b70ad0">
  <source media="(prefers-color-scheme: light)" srcset="https://github.com/user-attachments/assets/3a7eeb08-1133-47f5-859c-fad4f5a6a013">
  <img alt="NGINX Banner">
</picture>

[![Project Status: Active](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)
[![Community Forum](https://img.shields.io/badge/community-forum-009639?logo=discourse&link=https%3A%2F%2Fcommunity.nginx.org)](https://community.nginx.org)
[![License](https://img.shields.io/badge/License-BSD%202--Clause-blue.svg)](/LICENSE)

> **📚 详细文档**: 
> - [架构设计详解](docs/ARCHITECTURE_ZH.md) - 深入了解NGINX的内部架构和设计原理
> - [编译构建指南](docs/BUILD_GUIDE_ZH.md) - 完整的源码编译和构建说明
> - [部署运维指南](docs/DEPLOYMENT_ZH.md) - 生产环境部署、配置和运维最佳实践

## 目录
- [项目简介](#项目简介)
- [背景知识](#背景知识)
- [架构设计](#架构设计)
- [核心原理](#核心原理)
- [编译构建](#编译构建)
- [部署安装](#部署安装)
- [使用指南](#使用指南)
- [贡献指南](#贡献指南)

---

## 项目简介

**NGINX**（发音为"engine x"或"en-jin-eks"）是世界上最流行的Web服务器、高性能负载均衡器、反向代理、API网关和内容缓存系统。

NGINX 是自由开源软件，采用简化的[2-clause BSD许可证](LICENSE)发布。

企业版本、商业支持和培训服务由 [F5, Inc](https://www.f5.com/products/nginx) 提供。

### 主要特性

- **高性能**: 采用异步事件驱动架构，可处理数万并发连接
- **低资源消耗**: 相比传统Web服务器，内存占用更少
- **高可扩展性**: 模块化设计，支持静态和动态模块
- **负载均衡**: 支持多种负载均衡算法
- **反向代理**: 支持HTTP、HTTPS、TCP、UDP等协议
- **内容缓存**: 高效的静态内容缓存机制
- **TLS/SSL**: 完整的加密传输支持

---

## 背景知识

### Web 服务器的演进

#### 传统 Web 服务器（Apache）
传统的Web服务器（如Apache）采用**进程/线程模型**：
- 每个连接对应一个进程或线程
- 在高并发场景下，大量进程/线程导致上下文切换开销巨大
- 内存消耗随连接数线性增长
- C10K问题（无法有效处理1万个并发连接）

#### NGINX 的创新
NGINX 由 Igor Sysoev 于2004年创建，旨在解决C10K问题：
- 采用**异步事件驱动架构**
- 单个worker进程可处理数千个并发连接
- 非阻塞I/O操作
- 固定数量的worker进程（通常等于CPU核心数）

### 核心概念

#### 1. 异步非阻塞I/O
- 使用 epoll（Linux）、kqueue（FreeBSD）等高效事件通知机制
- worker进程不会因等待I/O而阻塞
- 事件循环模型处理所有连接

#### 2. Master-Worker 进程模型
```
┌─────────────────┐
│  Master Process │  ← 管理进程，读取配置，管理worker
└────────┬────────┘
         │
    ┌────┴────┬────────┬────────┐
    ▼         ▼        ▼        ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│Worker 1│ │Worker 2│ │Worker 3│ │Worker N│  ← 工作进程，处理请求
└────────┘ └────────┘ └────────┘ └────────┘
```

#### 3. 共享内存机制
- worker进程间通过共享内存通信
- 用于限流、会话状态、缓存等
- 无需进程间锁，提高性能

#### 4. 模块化设计
- 核心模块：基础功能
- 事件模块：事件处理机制
- HTTP模块：HTTP协议支持
- Mail模块：邮件代理
- Stream模块：TCP/UDP代理

---

## 架构设计

### 整体架构

```
                    ┌─────────────────────────────────┐
                    │     NGINX 核心架构              │
                    └─────────────────────────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
    ┌────▼────┐              ┌────▼────┐              ┌────▼────┐
    │  核心层  │              │ 事件层   │              │ 模块层   │
    │  Core   │              │  Event  │              │ Modules │
    └────┬────┘              └────┬────┘              └────┬────┘
         │                         │                         │
         │                         │                         │
    ┌────▼─────────────────────────▼─────────────────────────▼────┐
    │                        操作系统层                             │
    │            (epoll/kqueue/select/poll)                       │
    └─────────────────────────────────────────────────────────────┘
```

### 目录结构

```
nginx/
├── src/                    # 源代码目录
│   ├── core/              # 核心模块（数据结构、内存管理等）
│   ├── event/             # 事件处理模块
│   │   └── modules/       # 事件驱动模块（epoll、kqueue等）
│   ├── http/              # HTTP模块
│   │   ├── modules/       # HTTP功能模块
│   │   └── v2/            # HTTP/2支持
│   ├── mail/              # 邮件代理模块
│   ├── stream/            # TCP/UDP代理模块
│   └── os/                # 操作系统相关代码
├── auto/                   # 自动检测脚本
│   ├── configure          # 主配置脚本
│   ├── cc/                # 编译器检测
│   ├── lib/               # 库依赖检测
│   ├── os/                # 操作系统检测
│   └── modules            # 模块配置
├── conf/                   # 配置文件示例
│   ├── nginx.conf         # 主配置文件
│   ├── mime.types         # MIME类型
│   └── fastcgi_params     # FastCGI参数
├── contrib/                # 第三方贡献
└── docs/                   # 文档
```

### 核心数据结构

#### 1. ngx_cycle_t（核心周期结构）
存储NGINX运行时的全局配置和状态信息。

#### 2. ngx_connection_t（连接结构）
表示一个客户端连接，包含：
- socket文件描述符
- 读写事件
- 缓冲区
- 连接状态

#### 3. ngx_event_t（事件结构）
表示一个I/O事件，包含：
- 事件处理函数
- 事件数据
- 定时器

#### 4. ngx_buf_t（缓冲区结构）
管理内存和文件缓冲区。

#### 5. ngx_pool_t（内存池）
高效的内存管理，减少内存碎片。

### 请求处理流程

```
客户端请求
    │
    ▼
┌─────────────────┐
│  接收连接        │
│  (accept)       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  读取请求        │
│  (read event)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  解析请求        │
│  (parse)        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  处理请求        │
│  (handler)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  生成响应        │
│  (generate)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  发送响应        │
│  (write event)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  关闭连接        │
│  (close)        │
└─────────────────┘
```

---

## 核心原理

### 1. 事件驱动模型

#### epoll 工作原理（Linux）
```c
// 简化的事件循环伪代码
while (1) {
    // 等待事件发生（非阻塞或超时等待）
    n = epoll_wait(epfd, events, maxevents, timeout);
    
    for (i = 0; i < n; i++) {
        // 处理每个就绪事件
        if (events[i].data.ptr) {
            handler = events[i].data.ptr;
            handler->handler(handler);
        }
    }
    
    // 处理定时器事件
    ngx_process_timers();
}
```

### 2. 内存管理

NGINX 使用内存池机制：
- **分配快**: 从预分配的内存块中分配
- **释放快**: 整个池一次性释放
- **无碎片**: 减少内存碎片
- **生命周期管理**: 与请求生命周期绑定

```c
// 内存池分配示例
ngx_pool_t *pool = ngx_create_pool(size, log);
void *ptr = ngx_palloc(pool, size);
// 使用完后销毁整个池
ngx_destroy_pool(pool);
```

### 3. 配置解析

NGINX 的配置系统：
- 基于指令（directives）的声明式配置
- 支持上下文（context）嵌套
- 配置继承和合并机制
- 模块化的配置处理

```nginx
# 配置示例
http {                          # HTTP 上下文
    server {                    # Server 上下文
        listen 80;
        server_name example.com;
        
        location / {            # Location 上下文
            root /var/www;
            index index.html;
        }
    }
}
```

### 4. 过滤器链（Filter Chain）

HTTP响应处理采用过滤器链模式：
```
原始响应
    │
    ▼
┌─────────────┐
│ gzip filter │  ← 压缩
└──────┬──────┘
       │
       ▼
┌─────────────┐
│charset filter│ ← 字符集转换
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ header filter│ ← 添加响应头
└──────┬──────┘
       │
       ▼
    发送给客户端
```

### 5. 负载均衡算法

NGINX 支持多种负载均衡算法：

- **轮询（round-robin）**: 默认算法，依次分配
- **加权轮询（weighted）**: 根据权重分配
- **最少连接（least_conn）**: 分配给连接数最少的后端
- **IP哈希（ip_hash）**: 根据客户端IP哈希，保证会话一致性
- **一致性哈希（hash）**: 自定义哈希键

---

## 编译构建

### 系统要求

- **操作系统**: Linux、FreeBSD、Solaris、macOS、Windows
- **编译器**: GCC 或兼容编译器
- **必需库**:
  - PCRE (Perl Compatible Regular Expressions) - 正则表达式支持
  - zlib - gzip压缩支持
- **可选库**:
  - OpenSSL - HTTPS支持
  - libatomic_ops - 原子操作支持

### 安装依赖（Ubuntu/Debian）

```bash
# 更新包管理器
sudo apt update

# 安装编译工具
sudo apt install gcc make

# 安装必需库
sudo apt install libpcre3-dev zlib1g-dev

# 安装可选库（推荐）
sudo apt install libssl-dev
```

### 配置构建选项

```bash
# 克隆仓库
git clone https://github.com/nginx/nginx.git
cd nginx

# 运行配置脚本
./auto/configure \
    --prefix=/usr/local/nginx \           # 安装路径
    --with-http_ssl_module \              # 启用SSL模块
    --with-http_v2_module \               # 启用HTTP/2模块
    --with-http_realip_module \           # 启用真实IP模块
    --with-http_gzip_static_module \      # 启用预压缩模块
    --with-http_stub_status_module \      # 启用状态监控模块
    --with-stream \                       # 启用TCP/UDP代理
    --with-stream_ssl_module              # 启用Stream SSL
```

### 常用配置选项

| 选项 | 说明 |
|------|------|
| `--prefix=PATH` | 设置安装路径 |
| `--sbin-path=PATH` | 设置nginx二进制文件路径 |
| `--conf-path=PATH` | 设置配置文件路径 |
| `--error-log-path=PATH` | 设置错误日志路径 |
| `--pid-path=PATH` | 设置PID文件路径 |
| `--with-http_ssl_module` | 启用HTTPS支持 |
| `--with-http_v2_module` | 启用HTTP/2支持 |
| `--with-http_gzip_static_module` | 启用gzip压缩 |
| `--with-stream` | 启用TCP/UDP代理 |
| `--without-http_rewrite_module` | 禁用重写模块 |

### 编译和安装

```bash
# 编译（使用多核心加速）
make -j$(nproc)

# 安装（需要root权限）
sudo make install
```

### 验证安装

```bash
# 查看版本和编译选项
/usr/local/nginx/sbin/nginx -V

# 测试配置文件
/usr/local/nginx/sbin/nginx -t

# 启动NGINX
sudo /usr/local/nginx/sbin/nginx

# 测试是否运行
curl http://localhost
```

### 构建动态模块

```bash
# 配置时指定动态模块
./auto/configure \
    --with-compat \
    --add-dynamic-module=/path/to/module

# 编译
make modules

# 安装模块
sudo cp objs/*.so /usr/local/nginx/modules/
```

---

## 部署安装

### 使用包管理器安装（推荐）

#### Ubuntu/Debian

```bash
# 添加NGINX官方仓库
sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring

curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list

# 安装
sudo apt update
sudo apt install nginx
```

#### CentOS/RHEL

```bash
# 添加NGINX官方仓库
sudo yum install yum-utils
sudo yum-config-manager --add-repo \
    https://nginx.org/packages/centos/nginx.repo

# 安装
sudo yum install nginx
```

### 目录结构（包管理器安装）

```
/etc/nginx/                 # 配置文件目录
├── nginx.conf             # 主配置文件
├── conf.d/                # 站点配置目录
├── sites-available/       # 可用站点配置
└── sites-enabled/         # 启用站点配置

/usr/sbin/nginx            # 可执行文件

/var/log/nginx/            # 日志目录
├── access.log            # 访问日志
└── error.log             # 错误日志

/var/www/html/             # 默认Web根目录

/var/cache/nginx/          # 缓存目录
```

### 服务管理（systemd）

```bash
# 启动NGINX
sudo systemctl start nginx

# 停止NGINX
sudo systemctl stop nginx

# 重启NGINX
sudo systemctl restart nginx

# 重新加载配置（不中断服务）
sudo systemctl reload nginx

# 开机自启
sudo systemctl enable nginx

# 查看状态
sudo systemctl status nginx
```

### 信号控制

NGINX支持多种控制信号：

```bash
# 快速关闭
nginx -s stop

# 优雅关闭（处理完当前请求）
nginx -s quit

# 重新加载配置
nginx -s reload

# 重新打开日志文件
nginx -s reopen
```

### Docker 部署

```bash
# 拉取官方镜像
docker pull nginx:latest

# 运行容器
docker run -d \
    --name my-nginx \
    -p 80:80 \
    -p 443:443 \
    -v /path/to/nginx.conf:/etc/nginx/nginx.conf:ro \
    -v /path/to/html:/usr/share/nginx/html:ro \
    nginx:latest

# 查看日志
docker logs my-nginx

# 重新加载配置
docker exec my-nginx nginx -s reload
```

### Dockerfile 示例

```dockerfile
FROM nginx:alpine

# 复制自定义配置
COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d/ /etc/nginx/conf.d/

# 复制静态文件
COPY html/ /usr/share/nginx/html/

# 暴露端口
EXPOSE 80 443

# 启动NGINX
CMD ["nginx", "-g", "daemon off;"]
```

---

## 使用指南

### 基础配置

#### 1. 静态Web服务器

```nginx
http {
    server {
        listen 80;
        server_name example.com www.example.com;
        
        root /var/www/html;
        index index.html index.htm;
        
        location / {
            try_files $uri $uri/ =404;
        }
        
        # 启用gzip压缩
        gzip on;
        gzip_types text/plain text/css application/json;
    }
}
```

#### 2. 反向代理

```nginx
http {
    upstream backend {
        server backend1.example.com:8080;
        server backend2.example.com:8080;
        server backend3.example.com:8080;
    }
    
    server {
        listen 80;
        server_name example.com;
        
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

#### 3. 负载均衡配置

```nginx
http {
    # 定义后端服务器组
    upstream myapp {
        # 负载均衡算法选择
        least_conn;  # 最少连接算法
        # ip_hash;   # IP哈希算法
        
        # 后端服务器列表
        server app1.example.com:8080 weight=3;  # 权重为3
        server app2.example.com:8080 weight=2;  # 权重为2
        server app3.example.com:8080 weight=1;  # 权重为1
        server app4.example.com:8080 backup;    # 备份服务器
        
        # 健康检查
        server app5.example.com:8080 max_fails=3 fail_timeout=30s;
    }
    
    server {
        listen 80;
        
        location / {
            proxy_pass http://myapp;
        }
    }
}
```

#### 4. HTTPS/TLS 配置

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    
    # SSL证书配置
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # SSL协议和加密套件
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # SSL会话缓存
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # HSTS (HTTP Strict Transport Security)
    add_header Strict-Transport-Security "max-age=31536000" always;
    
    location / {
        root /var/www/html;
        index index.html;
    }
}

# HTTP重定向到HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

#### 5. 缓存配置

```nginx
http {
    # 定义缓存路径和参数
    proxy_cache_path /var/cache/nginx/proxy
                     levels=1:2
                     keys_zone=my_cache:10m
                     max_size=1g
                     inactive=60m
                     use_temp_path=off;
    
    server {
        listen 80;
        
        location / {
            proxy_cache my_cache;
            proxy_cache_valid 200 60m;
            proxy_cache_valid 404 1m;
            proxy_cache_use_stale error timeout updating;
            proxy_cache_background_update on;
            proxy_cache_lock on;
            
            # 缓存键
            proxy_cache_key "$scheme$request_method$host$request_uri";
            
            # 添加缓存状态头
            add_header X-Cache-Status $upstream_cache_status;
            
            proxy_pass http://backend;
        }
    }
}
```

#### 6. 限流配置

```nginx
http {
    # 定义限流区域
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
    
    # 定义连接数限制
    limit_conn_zone $binary_remote_addr zone=addr:10m;
    
    server {
        listen 80;
        
        location /api/ {
            # 应用限流规则
            limit_req zone=mylimit burst=20 nodelay;
            
            # 限制每个IP的并发连接数
            limit_conn addr 10;
            
            proxy_pass http://backend;
        }
    }
}
```

### 高级配置

#### 1. URL重写

```nginx
location / {
    # 重写规则
    rewrite ^/old-path/(.*)$ /new-path/$1 permanent;
    rewrite ^/product/(\d+)$ /item?id=$1 last;
    
    # 条件判断
    if ($request_method = POST) {
        return 405;
    }
}
```

#### 2. 访问控制

```nginx
location /admin {
    # IP白名单
    allow 192.168.1.0/24;
    allow 10.0.0.1;
    deny all;
    
    # 基本认证
    auth_basic "Restricted Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

#### 3. 性能优化

```nginx
http {
    # 连接优化
    keepalive_timeout 65;
    keepalive_requests 100;
    
    # 文件传输优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    
    # 客户端缓冲区
    client_body_buffer_size 128k;
    client_max_body_size 10m;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 4k;
    
    # 输出缓冲区
    output_buffers 1 32k;
    postpone_output 1460;
    
    # 文件描述符缓存
    open_file_cache max=1000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
}
```

#### 4. 日志配置

```nginx
http {
    # 自定义日志格式
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '$request_time';
    
    # JSON格式日志
    log_format json escape=json '{'
        '"time": "$time_iso8601",'
        '"remote_addr": "$remote_addr",'
        '"request": "$request",'
        '"status": $status,'
        '"body_bytes_sent": $body_bytes_sent,'
        '"request_time": $request_time,'
        '"upstream_response_time": "$upstream_response_time"'
    '}';
    
    server {
        # 访问日志
        access_log /var/log/nginx/access.log main;
        
        # 错误日志
        error_log /var/log/nginx/error.log warn;
        
        # 禁用特定location的日志
        location /health {
            access_log off;
        }
    }
}
```

### 常用命令

```bash
# 测试配置文件语法
nginx -t

# 查看编译选项
nginx -V

# 指定配置文件启动
nginx -c /path/to/nginx.conf

# 发送信号
nginx -s reload   # 重新加载配置
nginx -s stop     # 快速停止
nginx -s quit     # 优雅停止
nginx -s reopen   # 重新打开日志

# 查看master和worker进程
ps aux | grep nginx

# 查看监听端口
netstat -tulpn | grep nginx
# 或
ss -tulpn | grep nginx
```

### 故障排查

#### 1. 查看错误日志

```bash
# 实时查看错误日志
tail -f /var/log/nginx/error.log

# 查看最近的错误
tail -100 /var/log/nginx/error.log
```

#### 2. 调试模式

```nginx
# 在nginx.conf中启用调试日志
error_log /var/log/nginx/error.log debug;
```

#### 3. 常见问题

**问题1: 403 Forbidden**
```nginx
# 检查文件权限
ls -la /var/www/html

# 确保worker进程有权限访问
user www-data;
```

**问题2: 502 Bad Gateway**
```nginx
# 检查后端服务是否运行
curl http://backend:8080

# 增加超时时间
proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
```

**问题3: 504 Gateway Timeout**
```nginx
# 增加超时限制
proxy_connect_timeout 300s;
proxy_send_timeout 300s;
proxy_read_timeout 300s;
```

---

## 贡献指南

### 如何贡献

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 创建 Pull Request

### 代码规范

- 遵循现有代码风格
- 添加适当的注释
- 确保代码通过所有测试
- 更新相关文档

### 报告问题

请访问 [Issues](https://github.com/nginx/nginx/issues) 页面报告问题。

---

## 其他资源

- [官方文档](https://nginx.org/en/docs/)
- [官方博客](https://blog.nginx.org/)
- [社区论坛](https://community.nginx.org)
- [GitHub仓库](https://github.com/nginx/nginx)

---

## 许可证

本项目采用 [2-clause BSD 许可证](LICENSE)。

---

**注**: 更多详细信息请参考官方文档: https://nginx.org/en/docs/
