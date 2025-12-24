# NGINX 架构设计详解

## 目录
- [整体架构](#整体架构)
- [进程模型](#进程模型)
- [事件驱动机制](#事件驱动机制)
- [内存管理](#内存管理)
- [模块系统](#模块系统)
- [HTTP处理流程](#http处理流程)
- [配置系统](#配置系统)

---

## 整体架构

### 分层架构

NGINX 采用分层架构设计，从下到上分为以下几层：

```
┌─────────────────────────────────────────────────────────┐
│                     应用层 (Application)                 │
│  HTTP模块、Mail模块、Stream模块、第三方模块               │
└─────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────┐
│                     核心层 (Core)                        │
│  配置解析、模块管理、事件驱动、内存管理、日志系统         │
└─────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────┐
│                  操作系统抽象层 (OS)                      │
│  网络I/O、文件I/O、进程管理、信号处理                     │
└─────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────┐
│                  操作系统 (Linux/BSD/etc)                 │
└─────────────────────────────────────────────────────────┘
```

### 核心组件

```
                    ┌────────────────┐
                    │  NGINX Master  │
                    └────────┬───────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼────────┐  ┌────────▼───────┐  ┌────────▼───────┐
│  Configuration │  │  Module System │  │  Event System  │
│     Parser     │  │                │  │                │
└────────────────┘  └────────────────┘  └────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼────────┐  ┌────────▼───────┐  ┌────────▼───────┐
│   Worker 1     │  │   Worker 2     │  │   Worker N     │
│                │  │                │  │                │
│  ┌──────────┐  │  │  ┌──────────┐  │  │  ┌──────────┐  │
│  │Event Loop│  │  │  │Event Loop│  │  │  │Event Loop│  │
│  └──────────┘  │  │  └──────────┘  │  │  └──────────┘  │
└────────────────┘  └────────────────┘  └────────────────┘
```

---

## 进程模型

### Master-Worker 架构

#### Master 进程职责

1. **读取和验证配置文件**
   - 解析 nginx.conf
   - 验证配置语法
   - 加载模块

2. **管理 Worker 进程**
   - 创建 Worker 进程
   - 监控 Worker 状态
   - 重启崩溃的 Worker

3. **处理信号**
   - 接收管理命令（reload、stop、quit等）
   - 优雅重启
   - 日志轮转

4. **绑定监听端口**
   - 创建监听 socket
   - 传递给 Worker 进程

#### Worker 进程职责

1. **处理客户端请求**
   - 接受连接
   - 读取请求
   - 处理请求
   - 发送响应

2. **事件处理**
   - 网络事件
   - 定时器事件
   - 信号事件

3. **缓存管理**
   - 内存缓存
   - 文件缓存

#### 进程数量配置

```nginx
# 自动设置为CPU核心数（推荐）
worker_processes auto;

# 或手动指定
worker_processes 4;

# 绑定Worker到特定CPU核心
worker_cpu_affinity 0001 0010 0100 1000;
```

### 进程间通信

#### 1. 共享内存

Worker 进程通过共享内存共享数据：

```
┌────────────┐
│  Worker 1  │──────┐
└────────────┘      │
                    ▼
┌────────────┐  ┌──────────────────┐
│  Worker 2  │──│  Shared Memory   │
└────────────┘  │                  │
                │  - 限流数据       │
┌────────────┐  │  - 会话信息       │
│  Worker 3  │──│  - 缓存索引       │
└────────────┘  │  - 统计数据       │
                └──────────────────┘
```

#### 2. Socket Pair

Master 和 Worker 之间通过 socket pair 通信：
- 传递文件描述符
- 发送控制命令
- 同步状态信息

#### 3. 信号

使用Unix信号进行进程控制：

| 信号 | 作用 |
|------|------|
| TERM, INT | 快速停止 |
| QUIT | 优雅停止 |
| HUP | 重新加载配置 |
| USR1 | 重新打开日志文件 |
| USR2 | 平滑升级可执行文件 |
| WINCH | 优雅关闭Worker进程 |

---

## 事件驱动机制

### 事件模型

NGINX 使用异步非阻塞事件驱动模型：

```c
// 事件循环伪代码
void ngx_process_events_and_timers(ngx_cycle_t *cycle)
{
    ngx_msec_t  timer;
    
    // 计算最近定时器的超时时间
    timer = ngx_event_find_timer();
    
    // 等待事件（可能超时返回）
    (void) ngx_process_events(cycle, timer, NGX_UPDATE_TIME);
    
    // 处理已到期的定时器
    ngx_event_expire_timers();
    
    // 处理已发布的事件（延迟处理）
    ngx_event_process_posted(cycle, &ngx_posted_accept_events);
    ngx_event_process_posted(cycle, &ngx_posted_events);
}
```

### 事件类型

#### 1. 网络事件

```c
typedef struct ngx_event_s {
    void            *data;          // 关联数据（通常是连接）
    
    unsigned         write:1;       // 写事件
    unsigned         accept:1;      // 接受连接事件
    
    ngx_event_handler_pt  handler;  // 事件处理函数
    
    ngx_log_t       *log;
    
    ngx_rbtree_node_t   timer;      // 定时器节点
    
    unsigned         timedout:1;    // 超时标志
    unsigned         ready:1;       // 就绪标志
    unsigned         active:1;      // 激活标志
} ngx_event_t;
```

#### 2. 定时器事件

NGINX 使用红黑树管理定时器：

```
红黑树（按超时时间排序）
        ┌─────────┐
        │  300ms  │
        └────┬────┘
             │
      ┌──────┴──────┐
      │             │
  ┌───▼───┐     ┌───▼───┐
  │ 100ms │     │ 500ms │
  └───────┘     └───┬───┘
                    │
                ┌───▼───┐
                │ 600ms │
                └───────┘
```

### 事件驱动模块

NGINX 支持多种事件驱动机制，根据操作系统自动选择：

| 模块 | 操作系统 | 特点 |
|------|---------|------|
| epoll | Linux 2.6+ | 高效，支持大量连接 |
| kqueue | FreeBSD, macOS | 高效，性能优秀 |
| event ports | Solaris 10+ | Solaris专用 |
| /dev/poll | Solaris | 较老的机制 |
| select | 所有Unix | 通用但效率低 |
| poll | 所有Unix | 比select稍好 |

#### epoll 实现（Linux）

```c
// 创建epoll实例
epfd = epoll_create(nevents);

// 添加事件
struct epoll_event ee;
ee.events = EPOLLIN | EPOLLET;  // 边缘触发
ee.data.ptr = c;                 // 连接对象
epoll_ctl(epfd, EPOLL_CTL_ADD, c->fd, &ee);

// 等待事件
n = epoll_wait(epfd, events, nevents, timer);

// 处理事件
for (i = 0; i < n; i++) {
    c = events[i].data.ptr;
    
    if (events[i].events & EPOLLIN) {
        c->read->handler(c->read);
    }
    
    if (events[i].events & EPOLLOUT) {
        c->write->handler(c->write);
    }
}
```

### 边缘触发 vs 水平触发

NGINX 默认使用**边缘触发（ET）**模式：

**边缘触发（Edge Triggered）**:
- 只在状态变化时通知
- 必须一次性读完所有数据
- 更高效，但编程复杂

**水平触发（Level Triggered）**:
- 只要有数据就通知
- 可以慢慢读取数据
- 简单但可能产生大量事件

---

## 内存管理

### 内存池（Memory Pool）

NGINX 使用内存池减少内存分配开销：

```c
struct ngx_pool_s {
    ngx_pool_data_t       d;           // 数据区
    size_t                max;         // 最大分配大小
    ngx_pool_t           *current;     // 当前内存池
    ngx_chain_t          *chain;       // 缓冲区链
    ngx_pool_large_t     *large;       // 大块内存链表
    ngx_pool_cleanup_t   *cleanup;     // 清理函数链表
    ngx_log_t            *log;         // 日志对象
};
```

#### 内存分配策略

```
小内存（< max）:
┌─────────────────────────────────┐
│  Pool Header                    │
├─────────────────────────────────┤
│  已分配内存 1                    │
├─────────────────────────────────┤
│  已分配内存 2                    │
├─────────────────────────────────┤
│  空闲空间                        │
└─────────────────────────────────┘

大内存（>= max）:
Pool ──→ Large Block 1 ──→ Large Block 2 ──→ NULL
```

#### 内存池优势

1. **快速分配**: 从预分配块中分配，无需系统调用
2. **批量释放**: 整个池一次性释放
3. **减少碎片**: 固定大小的内存块
4. **自动清理**: 与请求生命周期绑定

### 缓冲区（Buffer）

```c
typedef struct ngx_buf_s {
    u_char          *pos;      // 当前读位置
    u_char          *last;     // 当前写位置
    
    off_t            file_pos; // 文件读位置
    off_t            file_last;// 文件写位置
    
    u_char          *start;    // 缓冲区起始
    u_char          *end;      // 缓冲区结束
    
    ngx_buf_tag_t    tag;      // 缓冲区标记
    ngx_file_t      *file;     // 关联文件
    
    unsigned         temporary:1;  // 临时缓冲区
    unsigned         memory:1;     // 内存缓冲区
    unsigned         mmap:1;       // mmap映射
    unsigned         recycled:1;   // 可回收
    unsigned         in_file:1;    // 数据在文件中
    unsigned         flush:1;      // 需要刷新
    unsigned         last_buf:1;   // 最后一个缓冲区
} ngx_buf_t;
```

### 缓冲区链（Buffer Chain）

```
请求 ──→ Chain 1 ──→ Chain 2 ──→ Chain 3 ──→ NULL
          │           │           │
          ▼           ▼           ▼
        Buffer 1    Buffer 2    Buffer 3
        (Header)    (Body)      (File)
```

---

## 模块系统

### 模块分类

```
NGINX 模块
│
├── 核心模块 (Core Modules)
│   ├── ngx_core_module        - 核心模块
│   ├── ngx_errlog_module      - 错误日志
│   ├── ngx_regex_module       - 正则表达式
│   └── ngx_thread_pool_module - 线程池
│
├── 事件模块 (Event Modules)
│   ├── ngx_event_core_module  - 事件核心
│   ├── ngx_epoll_module       - epoll支持
│   └── ngx_kqueue_module      - kqueue支持
│
├── HTTP 模块 (HTTP Modules)
│   ├── ngx_http_core_module   - HTTP核心
│   ├── ngx_http_access_module - 访问控制
│   ├── ngx_http_gzip_module   - gzip压缩
│   ├── ngx_http_proxy_module  - 反向代理
│   └── ngx_http_ssl_module    - SSL/TLS
│
├── Mail 模块 (Mail Modules)
│   ├── ngx_mail_core_module   - Mail核心
│   ├── ngx_mail_pop3_module   - POP3支持
│   └── ngx_mail_smtp_module   - SMTP支持
│
└── Stream 模块 (Stream Modules)
    ├── ngx_stream_core_module - Stream核心
    └── ngx_stream_ssl_module  - Stream SSL
```

### 模块结构

```c
typedef struct ngx_module_s {
    ngx_uint_t            ctx_index;  // 模块索引
    ngx_uint_t            index;      // 模块在所有模块中的索引
    
    char                 *name;       // 模块名称
    
    ngx_uint_t            spare0;
    ngx_uint_t            spare1;
    
    ngx_uint_t            version;    // 模块版本
    const char           *signature;  // 签名
    
    void                 *ctx;        // 模块上下文
    ngx_command_t        *commands;   // 模块指令
    
    ngx_uint_t            type;       // 模块类型
    
    // 模块初始化钩子
    ngx_int_t           (*init_master)(ngx_log_t *log);
    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    
    // 模块退出钩子
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);
    void                (*exit_master)(ngx_cycle_t *cycle);
} ngx_module_t;
```

### 模块加载顺序

```
1. 核心模块初始化
   ↓
2. 事件模块初始化
   ↓
3. HTTP/Mail/Stream 模块初始化
   ↓
4. 第三方模块初始化
```

---

## HTTP处理流程

### 请求处理阶段

NGINX HTTP 请求处理分为 11 个阶段：

```
NGX_HTTP_POST_READ_PHASE          (0) 读取请求后
    ↓
NGX_HTTP_SERVER_REWRITE_PHASE     (1) server块重写
    ↓
NGX_HTTP_FIND_CONFIG_PHASE        (2) 查找location配置
    ↓
NGX_HTTP_REWRITE_PHASE            (3) location块重写
    ↓
NGX_HTTP_POST_REWRITE_PHASE       (4) 重写后处理
    ↓
NGX_HTTP_PREACCESS_PHASE          (5) 访问前处理
    ↓
NGX_HTTP_ACCESS_PHASE             (6) 访问控制
    ↓
NGX_HTTP_POST_ACCESS_PHASE        (7) 访问后处理
    ↓
NGX_HTTP_PRECONTENT_PHASE         (8) 内容前处理
    ↓
NGX_HTTP_CONTENT_PHASE            (9) 内容生成
    ↓
NGX_HTTP_LOG_PHASE                (10) 日志记录
```

### 处理阶段详解

#### 1. POST_READ_PHASE
- 读取请求头完成后的处理
- 可以获取客户端真实IP（realip模块）

#### 2. SERVER_REWRITE_PHASE
- server块中的rewrite指令处理
- 可以修改请求URI

#### 3. FIND_CONFIG_PHASE
- 根据URI查找匹配的location
- 系统自动处理，无法注册处理函数

#### 4. REWRITE_PHASE
- location块中的rewrite指令处理

#### 5. POST_REWRITE_PHASE
- 检查是否需要重新查找location
- 系统自动处理

#### 6. PREACCESS_PHASE
- 访问控制前的限制检查
- 限流（limit_req）
- 连接数限制（limit_conn）

#### 7. ACCESS_PHASE
- 访问权限检查
- IP访问控制（allow/deny）
- 认证（auth_basic）

#### 8. POST_ACCESS_PHASE
- 访问控制结果处理
- 系统自动处理

#### 9. PRECONTENT_PHASE
- 内容生成前的处理
- try_files指令处理

#### 10. CONTENT_PHASE
- 内容生成阶段
- 静态文件处理
- 反向代理
- FastCGI/uWSGI

#### 11. LOG_PHASE
- 访问日志记录

### 过滤器链

响应数据通过过滤器链处理：

```
Header Filter Chain:
    ngx_http_not_modified_filter
        ↓
    ngx_http_range_filter
        ↓
    ngx_http_gzip_filter
        ↓
    ngx_http_header_filter

Body Filter Chain:
    ngx_http_range_body_filter
        ↓
    ngx_http_gzip_body_filter
        ↓
    ngx_http_postpone_filter
        ↓
    ngx_http_write_filter
```

---

## 配置系统

### 配置文件结构

```nginx
# 全局块
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;

# events 块
events {
    worker_connections 1024;
    use epoll;
}

# http 块
http {
    # http 全局配置
    include mime.types;
    default_type application/octet-stream;
    
    # upstream 块
    upstream backend {
        server 127.0.0.1:8080;
    }
    
    # server 块
    server {
        listen 80;
        server_name example.com;
        
        # location 块
        location / {
            root /var/www;
            index index.html;
        }
        
        location /api {
            proxy_pass http://backend;
        }
    }
}
```

### 配置上下文

```
main                        全局上下文
 │
 ├─ events                  事件上下文
 │
 ├─ http                    HTTP上下文
 │   │
 │   ├─ upstream            上游服务器组
 │   │
 │   └─ server              虚拟主机
 │       │
 │       └─ location        位置匹配
 │
 ├─ mail                    Mail上下文
 │   └─ server
 │
 └─ stream                  Stream上下文
     └─ server
```

### 配置指令

```c
typedef struct ngx_command_s {
    ngx_str_t             name;      // 指令名称
    ngx_uint_t            type;      // 指令类型和参数
    char               *(*set)(ngx_conf_t *cf, 
                               ngx_command_t *cmd, 
                               void *conf);  // 设置函数
    ngx_uint_t            conf;      // 配置偏移
    ngx_uint_t            offset;    // 字段偏移
    void                 *post;      // 后处理
} ngx_command_t;
```

### 配置合并

NGINX 的配置具有继承和合并机制：

```
http {
    gzip on;              # HTTP级别配置
    
    server {
        # 继承: gzip on
        gzip_comp_level 6; # Server级别配置
        
        location / {
            # 继承: gzip on, gzip_comp_level 6
            gzip_types text/plain;  # Location级别配置
        }
        
        location /api {
            gzip off;      # 覆盖继承的配置
        }
    }
}
```

---

## 总结

NGINX 的架构设计体现了以下核心思想：

1. **异步非阻塞**: 高并发处理能力
2. **事件驱动**: 高效的I/O处理
3. **内存池**: 快速内存管理
4. **模块化**: 灵活扩展
5. **分层设计**: 清晰的职责划分
6. **进程隔离**: 稳定性和安全性

这些设计使 NGINX 成为高性能 Web 服务器的标杆。
