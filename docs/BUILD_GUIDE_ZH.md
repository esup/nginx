# NGINX 编译构建详细指南

## 目录
- [环境准备](#环境准备)
- [依赖库安装](#依赖库安装)
- [获取源码](#获取源码)
- [配置选项详解](#配置选项详解)
- [编译过程](#编译过程)
- [安装部署](#安装部署)
- [编译第三方模块](#编译第三方模块)
- [交叉编译](#交叉编译)
- [故障排除](#故障排除)

---

## 环境准备

### 支持的操作系统

NGINX 支持以下操作系统：

| 操作系统 | 版本要求 | 推荐度 |
|---------|---------|--------|
| Linux | 2.6+ | ⭐⭐⭐⭐⭐ |
| FreeBSD | 10.0+ | ⭐⭐⭐⭐⭐ |
| macOS | 10.12+ | ⭐⭐⭐⭐ |
| Solaris | 10+ | ⭐⭐⭐ |
| Windows | 7+ | ⭐⭐ (仅用于开发) |

### 硬件要求

**最低要求**:
- CPU: 单核 1GHz
- 内存: 512MB
- 磁盘: 100MB

**推荐配置**:
- CPU: 多核 2GHz+
- 内存: 2GB+
- 磁盘: 1GB+（包含日志空间）

---

## 依赖库安装

### Ubuntu/Debian 系统

```bash
# 更新软件包列表
sudo apt update

# 安装基础编译工具
sudo apt install -y \
    build-essential \
    gcc \
    g++ \
    make

# 安装必需的依赖库
sudo apt install -y \
    libpcre3 \
    libpcre3-dev \
    zlib1g \
    zlib1g-dev

# 安装可选依赖库
sudo apt install -y \
    libssl-dev \          # OpenSSL - HTTPS支持
    libgd-dev \           # GD库 - 图像过滤
    libgeoip-dev \        # GeoIP - IP地理位置
    libxml2-dev \         # XML - XSLT支持
    libxslt1-dev \        # XSLT转换
    libperl-dev           # Perl - 嵌入Perl
```

### CentOS/RHEL/Fedora 系统

```bash
# 安装基础编译工具
sudo yum groupinstall -y "Development Tools"
sudo yum install -y gcc gcc-c++ make

# 安装必需的依赖库
sudo yum install -y \
    pcre pcre-devel \
    zlib zlib-devel

# 安装可选依赖库
sudo yum install -y \
    openssl openssl-devel \
    gd gd-devel \
    GeoIP GeoIP-devel \
    libxml2 libxml2-devel \
    libxslt libxslt-devel \
    perl-ExtUtils-Embed
```

### macOS 系统

```bash
# 安装 Homebrew（如果未安装）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 安装依赖
brew install pcre zlib openssl
```

### 依赖库说明

| 库 | 用途 | 是否必需 |
|---|------|---------|
| PCRE | 正则表达式支持（rewrite模块） | 是 |
| zlib | gzip压缩支持 | 是 |
| OpenSSL | HTTPS/TLS支持 | 否（但强烈推荐） |
| GD | 图像处理（image_filter模块） | 否 |
| GeoIP | IP地理位置查询 | 否 |
| libxml2/libxslt | XML/XSLT处理 | 否 |
| Perl | Perl嵌入支持 | 否 |

---

## 获取源码

### 方法1: 从 GitHub 克隆（推荐用于开发）

```bash
# 克隆主分支
git clone https://github.com/nginx/nginx.git
cd nginx

# 查看所有分支
git branch -a

# 切换到稳定分支（如果需要）
git checkout stable-1.24

# 查看当前版本
git describe --tags
```

### 方法2: 下载发布版本

```bash
# 从官网下载（稳定版）
wget http://nginx.org/download/nginx-1.24.0.tar.gz

# 或下载主线版本
wget http://nginx.org/download/nginx-1.25.0.tar.gz

# 验证下载（可选）
wget http://nginx.org/download/nginx-1.24.0.tar.gz.asc
gpg --verify nginx-1.24.0.tar.gz.asc nginx-1.24.0.tar.gz

# 解压
tar -xzf nginx-1.24.0.tar.gz
cd nginx-1.24.0
```

### 方法3: 使用发行版包管理器获取源码

```bash
# Ubuntu/Debian
apt source nginx

# CentOS/RHEL
yumdownloader --source nginx
rpm -ivh nginx-*.src.rpm
```

---

## 配置选项详解

### 运行 configure 脚本

```bash
cd /path/to/nginx
./auto/configure [options]
```

### 路径相关选项

```bash
--prefix=/usr/local/nginx              # 安装根目录
--sbin-path=/usr/sbin/nginx            # nginx可执行文件路径
--conf-path=/etc/nginx/nginx.conf     # 配置文件路径
--error-log-path=/var/log/nginx/error.log  # 错误日志路径
--http-log-path=/var/log/nginx/access.log  # 访问日志路径
--pid-path=/var/run/nginx.pid         # PID文件路径
--lock-path=/var/lock/nginx.lock      # 锁文件路径
--user=nginx                           # worker进程运行用户
--group=nginx                          # worker进程运行组
```

### HTTP 模块选项

```bash
# 启用 HTTP 模块（默认启用的可以用 --without- 禁用）
--with-http_ssl_module                 # SSL/TLS支持
--with-http_v2_module                  # HTTP/2支持
--with-http_v3_module                  # HTTP/3支持（实验性）
--with-http_realip_module              # 真实IP获取
--with-http_addition_module            # 响应前后添加内容
--with-http_sub_module                 # 字符串替换
--with-http_dav_module                 # WebDAV支持
--with-http_flv_module                 # FLV流媒体
--with-http_mp4_module                 # MP4流媒体
--with-http_gunzip_module              # gunzip支持
--with-http_gzip_static_module         # 预压缩文件支持
--with-http_auth_request_module        # 认证请求
--with-http_random_index_module        # 随机首页
--with-http_secure_link_module         # 安全链接
--with-http_degradation_module         # 降级处理
--with-http_slice_module               # 切片
--with-http_stub_status_module         # 状态页面
--with-http_perl_module                # Perl支持

# 禁用默认启用的模块
--without-http_charset_module          # 字符集
--without-http_gzip_module             # gzip压缩
--without-http_ssi_module              # SSI
--without-http_userid_module           # 用户ID
--without-http_access_module           # 访问控制
--without-http_auth_basic_module       # 基本认证
--without-http_mirror_module           # 镜像
--without-http_autoindex_module        # 自动索引
--without-http_geo_module              # Geo
--without-http_map_module              # Map
--without-http_split_clients_module    # 分流
--without-http_referer_module          # Referer
--without-http_rewrite_module          # 重写
--without-http_proxy_module            # 代理
--without-http_fastcgi_module          # FastCGI
--without-http_uwsgi_module            # uWSGI
--without-http_scgi_module             # SCGI
--without-http_grpc_module             # gRPC
--without-http_memcached_module        # Memcached
--without-http_limit_conn_module       # 连接限制
--without-http_limit_req_module        # 请求限制
--without-http_empty_gif_module        # 空GIF
--without-http_browser_module          # 浏览器
--without-http_upstream_hash_module    # Hash负载均衡
--without-http_upstream_ip_hash_module # IP Hash负载均衡
--without-http_upstream_least_conn_module  # 最少连接负载均衡
--without-http_upstream_random_module  # 随机负载均衡
--without-http_upstream_keepalive_module   # 上游keepalive
--without-http_upstream_zone_module    # 上游共享内存区域
```

### Mail 模块选项

```bash
--with-mail                            # 启用Mail代理
--with-mail_ssl_module                 # Mail SSL支持
--without-mail_pop3_module             # 禁用POP3
--without-mail_imap_module             # 禁用IMAP
--without-mail_smtp_module             # 禁用SMTP
```

### Stream 模块选项

```bash
--with-stream                          # TCP/UDP代理
--with-stream_ssl_module               # Stream SSL支持
--with-stream_realip_module            # Stream真实IP
--with-stream_geoip_module             # Stream GeoIP
--with-stream_ssl_preread_module       # SSL预读
--without-stream_limit_conn_module     # 禁用连接限制
--without-stream_access_module         # 禁用访问控制
--without-stream_geo_module            # 禁用Geo
--without-stream_map_module            # 禁用Map
--without-stream_split_clients_module  # 禁用分流
--without-stream_return_module         # 禁用Return
--without-stream_set_module            # 禁用Set
--without-stream_upstream_hash_module  # 禁用Hash负载均衡
--without-stream_upstream_least_conn_module  # 禁用最少连接
--without-stream_upstream_random_module      # 禁用随机
--without-stream_upstream_zone_module  # 禁用上游区域
```

### 第三方模块选项

```bash
# 添加静态模块
--add-module=/path/to/module

# 添加动态模块
--add-dynamic-module=/path/to/module

# 启用动态模块兼容性
--with-compat
```

### 编译器和优化选项

```bash
--with-cc=/usr/bin/gcc                 # 指定C编译器
--with-cpp=/usr/bin/cpp                # 指定C预处理器
--with-cc-opt='-O2 -g'                 # C编译器选项
--with-ld-opt='-L/usr/local/lib'       # 链接器选项

# 常用优化选项
--with-cc-opt='-O2 -pipe -march=native -mtune=native'
```

### 调试选项

```bash
--with-debug                           # 启用调试日志
--with-dtrace-probes                   # DTrace探针
--with-libatomic                       # 原子操作库
```

### 其他选项

```bash
--with-threads                         # 线程池支持
--with-file-aio                        # 文件AIO
--with-http_image_filter_module        # 图像过滤（需要GD库）
--with-http_geoip_module               # GeoIP（需要GeoIP库）
--with-http_xslt_module                # XSLT（需要libxml2/libxslt）
--with-pcre                            # 使用内置PCRE
--with-pcre=/path/to/pcre              # 使用指定PCRE源码
--with-pcre-jit                        # PCRE JIT编译
--with-zlib=/path/to/zlib              # 使用指定zlib源码
--with-openssl=/path/to/openssl        # 使用指定OpenSSL源码
--with-openssl-opt=options             # OpenSSL编译选项
```

### 配置示例

#### 最小化配置

```bash
./auto/configure \
    --prefix=/usr/local/nginx \
    --with-http_ssl_module
```

#### 标准Web服务器配置

```bash
./auto/configure \
    --prefix=/usr/local/nginx \
    --sbin-path=/usr/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --lock-path=/var/lock/nginx.lock \
    --user=nginx \
    --group=nginx \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_realip_module \
    --with-http_addition_module \
    --with-http_sub_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_mp4_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_random_index_module \
    --with-http_secure_link_module \
    --with-http_stub_status_module \
    --with-http_auth_request_module \
    --with-threads \
    --with-file-aio \
    --with-http_image_filter_module \
    --with-http_geoip_module \
    --with-pcre-jit
```

#### 完整功能配置

```bash
./auto/configure \
    --prefix=/usr/local/nginx \
    --sbin-path=/usr/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --lock-path=/var/lock/nginx.lock \
    --user=nginx \
    --group=nginx \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_realip_module \
    --with-http_addition_module \
    --with-http_sub_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_mp4_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_random_index_module \
    --with-http_secure_link_module \
    --with-http_stub_status_module \
    --with-http_auth_request_module \
    --with-http_xslt_module \
    --with-http_image_filter_module \
    --with-http_geoip_module \
    --with-http_perl_module \
    --with-threads \
    --with-stream \
    --with-stream_ssl_module \
    --with-stream_realip_module \
    --with-stream_geoip_module \
    --with-stream_ssl_preread_module \
    --with-mail \
    --with-mail_ssl_module \
    --with-file-aio \
    --with-pcre-jit \
    --with-debug
```

---

## 编译过程

### 执行编译

```bash
# 使用所有CPU核心编译
make -j$(nproc)

# 或指定核心数
make -j4

# 仅编译nginx二进制文件
make build

# 仅编译模块
make modules
```

### 编译输出

```
编译过程会生成：
nginx/
├── objs/                    # 编译输出目录
│   ├── nginx               # 主程序二进制文件
│   ├── ngx_modules.c       # 模块列表
│   ├── ngx_auto_config.h   # 自动配置头文件
│   ├── ngx_auto_headers.h  # 自动头文件
│   ├── autoconf.err        # 配置错误日志
│   ├── Makefile            # 生成的Makefile
│   └── src/                # 编译的对象文件
│       ├── core/
│       ├── event/
│       ├── http/
│       └── ...
└── Makefile                # 主Makefile
```

### 编译验证

```bash
# 查看编译的二进制文件
ls -lh objs/nginx

# 查看编译选项
objs/nginx -V

# 测试二进制文件
objs/nginx -t
```

---

## 安装部署

### 安装到系统

```bash
# 安装（需要root权限）
sudo make install

# 安装的文件位置（取决于--prefix配置）
/usr/local/nginx/
├── sbin/
│   └── nginx               # 可执行文件
├── conf/
│   ├── nginx.conf          # 配置文件
│   ├── mime.types
│   ├── fastcgi_params
│   ├── scgi_params
│   └── uwsgi_params
├── html/
│   ├── index.html          # 默认首页
│   └── 50x.html            # 错误页面
└── logs/                   # 日志目录（创建后）
```

### 创建系统用户

```bash
# 创建nginx用户和组
sudo groupadd nginx
sudo useradd -g nginx -s /sbin/nologin -M nginx
```

### 创建systemd服务文件

```bash
sudo vim /lib/systemd/system/nginx.service
```

```ini
[Unit]
Description=NGINX HTTP Server
Documentation=http://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t
ExecStart=/usr/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID
PrivateTmp=true
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
# 重新加载systemd
sudo systemctl daemon-reload

# 启用开机自启
sudo systemctl enable nginx

# 启动服务
sudo systemctl start nginx

# 查看状态
sudo systemctl status nginx
```

### 配置防火墙

```bash
# Ubuntu/Debian (ufw)
sudo ufw allow 'Nginx Full'
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# CentOS/RHEL (firewalld)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# 或直接添加端口
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

---

## 编译第三方模块

### 下载第三方模块

```bash
# 示例：编译nginx-module-vts（流量统计模块）
cd /usr/local/src
git clone https://github.com/vozlt/nginx-module-vts.git
```

### 作为静态模块编译

```bash
cd /path/to/nginx/source
./auto/configure \
    --prefix=/usr/local/nginx \
    --add-module=/usr/local/src/nginx-module-vts
make -j$(nproc)
sudo make install
```

### 作为动态模块编译

```bash
./auto/configure \
    --prefix=/usr/local/nginx \
    --with-compat \
    --add-dynamic-module=/usr/local/src/nginx-module-vts
make modules
sudo cp objs/*.so /usr/local/nginx/modules/
```

在 nginx.conf 中加载：

```nginx
load_module modules/ngx_http_vhost_traffic_status_module.so;
```

### 常用第三方模块

| 模块 | 功能 | GitHub |
|-----|------|--------|
| nginx-module-vts | 流量统计 | vozlt/nginx-module-vts |
| ngx_cache_purge | 缓存清除 | FRiCKLE/ngx_cache_purge |
| headers-more-nginx-module | 更多头部操作 | openresty/headers-more-nginx-module |
| nginx-rtmp-module | RTMP流媒体 | arut/nginx-rtmp-module |
| ngx_http_geoip2_module | GeoIP2支持 | leev/ngx_http_geoip2_module |
| nginx-module-sysguard | 系统保护 | vozlt/nginx-module-sysguard |

---

## 交叉编译

### 为ARM平台编译

```bash
# 安装交叉编译工具链
sudo apt install gcc-arm-linux-gnueabi

# 配置
./auto/configure \
    --prefix=/usr/local/nginx \
    --with-cc=arm-linux-gnueabi-gcc \
    --with-cpp=arm-linux-gnueabi-cpp \
    --with-cc-opt='-static -static-libgcc' \
    --with-ld-opt='-static' \
    --without-http_rewrite_module \
    --without-http_gzip_module

# 编译
make -j$(nproc)
```

---

## 故障排除

### 常见编译错误

#### 错误1: PCRE库未找到

```
./auto/configure: error: the HTTP rewrite module requires the PCRE library.
```

**解决方案**:
```bash
# Ubuntu/Debian
sudo apt install libpcre3-dev

# CentOS/RHEL
sudo yum install pcre-devel

# 或使用NGINX自带的PCRE
./auto/configure --with-pcre=/path/to/pcre/source
```

#### 错误2: zlib库未找到

```
./auto/configure: error: the HTTP gzip module requires the zlib library.
```

**解决方案**:
```bash
# Ubuntu/Debian
sudo apt install zlib1g-dev

# CentOS/RHEL
sudo yum install zlib-devel

# 或禁用gzip模块
./auto/configure --without-http_gzip_module
```

#### 错误3: OpenSSL库未找到

```
./auto/configure: error: SSL modules require the OpenSSL library.
```

**解决方案**:
```bash
# Ubuntu/Debian
sudo apt install libssl-dev

# CentOS/RHEL
sudo yum install openssl-devel

# macOS
brew install openssl
./auto/configure --with-cc-opt="-I/usr/local/opt/openssl/include" \
                 --with-ld-opt="-L/usr/local/opt/openssl/lib"
```

#### 错误4: 编译器不支持

```
./auto/configure: error: C compiler cc is not found
```

**解决方案**:
```bash
# Ubuntu/Debian
sudo apt install build-essential

# CentOS/RHEL
sudo yum groupinstall "Development Tools"
```

### 调试编译问题

```bash
# 查看配置错误日志
cat objs/autoconf.err

# 详细的配置输出
./auto/configure --with-debug 2>&1 | tee config.log

# 查看生成的配置
cat objs/ngx_auto_config.h
```

### 清理编译文件

```bash
# 清理所有编译文件
make clean

# 重新配置需要删除objs目录
rm -rf objs/
```

---

## 编译优化建议

### 性能优化编译选项

```bash
./auto/configure \
    --with-cc-opt='-O3 -march=native -mtune=native -flto' \
    --with-ld-opt='-Wl,-O1 -Wl,--as-needed -flto' \
    --with-pcre-jit \
    --with-file-aio \
    --with-threads
```

### 最小化二进制大小

```bash
./auto/configure \
    --with-cc-opt='-Os -ffunction-sections -fdata-sections' \
    --with-ld-opt='-Wl,--gc-sections -Wl,-s' \
    --without-http_charset_module \
    --without-http_ssi_module \
    --without-http_userid_module \
    --without-http_autoindex_module \
    --without-http_geo_module \
    --without-http_map_module \
    --without-http_split_clients_module \
    --without-http_referer_module \
    --without-http_fastcgi_module \
    --without-http_uwsgi_module \
    --without-http_scgi_module \
    --without-http_memcached_module
```

---

## 总结

NGINX的编译构建系统设计精良，支持：

1. **灵活的模块选择**: 可根据需求定制功能
2. **跨平台支持**: 适配多种操作系统
3. **优化选项**: 支持各种编译器优化
4. **第三方扩展**: 易于集成第三方模块
5. **动态模块**: 支持运行时加载模块

通过合理配置，可以构建出最适合您需求的NGINX版本。
