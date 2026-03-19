# ApFree WiFiDog Captive Portal 核心逻辑与流程分析

## 目录
1. [项目概述](#项目概述)
2. [核心架构](#核心架构)
3. [主要组件](#主要组件)
4. [Captive Portal 认证流程](#captive-portal-认证流程)
5. [关键数据结构](#关键数据结构)
6. [防火墙机制](#防火墙机制)
7. [认证服务器通信](#认证服务器通信)
8. [特殊设备处理](#特殊设备处理)
9. [实时通信](#实时通信)
10. [代码流程图](#代码流程图)

---

## 项目概述

ApFree WiFiDog 是一个高性能的开源 Captive Portal 解决方案，专为 OpenWrt 平台设计。它通过拦截和重定向 HTTP/HTTPS 流量来实现用户认证，支持高并发、HTTPS 重定向、实时通信等高级特性。

### 主要特性
- **高性能**: 基于 libevent2 + epoll 事件驱动架构
- **安全**: 支持 HTTPS 重定向，使用 OpenSSL 加密
- **灵活**: 支持本地认证和云端认证模式
- **实时**: 支持 WebSocket 和 MQTT 长连接
- **兼容**: 支持 Apple、Android 等设备的 Captive Portal 检测
- **多防火墙**: 支持 iptables、nftables、VPP

---

## 核心架构

### 系统架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     ApFree WiFiDog                          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │ HTTP Server  │  │ HTTPS Server │  │  WebSocket   │        │
│  │  (libevent)  │  │  (OpenSSL)   │  │   Server     │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
│         │                 │                 │                 │
│         └─────────────────┼─────────────────┘                 │
│                           │                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Gateway Core (gateway.c)                │   │
│  │  - 信号处理                                          │   │
│  │  - 线程管理                                          │   │
│  │  - 配置初始化                                        │   │
│  └──────────────────────────────────────────────────────┘   │
│                           │                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │  Auth Module │  │ Firewall Mod │  │ Client List  │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
│         │                 │                 │                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │Central Server│  │  iptables/   │  │ Online/Offline│        │
│  │   Client     │  │  nftables    │  │   Clients    │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### 线程模型

ApFree WiFiDog 采用多线程架构，主要线程包括：

| 线程名称 | 功能描述 | 代码位置 |
|---------|---------|---------|
| 主线程 | 事件循环、信号处理 | [gateway.c](file:///e:/projects/github.com/apfree-wifidog/src/gateway.c) |
| HTTP 服务线程 | 处理 HTTP 请求 | [http.c](file:///e:/projects/github.com/apfree-wifidog/src/http.c) |
| HTTPS 服务线程 | 处理 HTTPS 请求 | [tls_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/tls_thread.c) |
| Ping 线程 | 向认证服务器发送心跳 | [ping_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/ping_thread.c) |
| 认证检查线程 | 检查客户端超时 | [auth.c](file:///e:/projects/github.com/apfree-wifidog/src/auth.c) |
| WebSocket 线程 | 处理 WebSocket 连接 | [ws_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/ws_thread.c) |
| MQTT 线程 | 处理 MQTT 连接 | [mqtt_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/mqtt_thread.c) |
| 控制线程 | 处理 wdctlx 命令 | [wdctlx_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/wdctlx_thread.c) |
| DNS 监控线程 | 监控 DNS 解析 | [dns_monitor.c](file:///e:/projects/github.com/apfree-wifidog/src/dns_monitor.c) |

---

## 主要组件

### 1. Gateway 模块 ([gateway.c](file:///e:/projects/github.com/apfree-wifidog/src/gateway.c))

**职责**:
- 程序入口和初始化
- 信号处理 (SIGTERM, SIGINT, SIGHUP, SIGCHLD, SIGUSR1, SIGPIPE)
- 线程创建和管理
- 资源初始化和清理

**关键函数**:
- `gw_main()`: 主函数入口
- `wd_init()`: 初始化所有组件
- `wd_signals_init()`: 初始化信号处理器
- `init_firewall()`: 初始化防火墙规则
- `init_resource_limits()`: 设置系统资源限制

### 2. HTTP 模块 ([http.c](file:///e:/projects/github.com/apfree-wifidog/src/http.c))

**职责**:
- 处理 HTTP/HTTPS 请求
- 实现 Captive Portal 重定向逻辑
- 处理特殊设备（Apple、Android）的 Captive Portal 检测
- 生成和发送认证页面

**关键函数**:
- `ev_http_callback_404()`: 404 处理器，实现重定向逻辑
- `ev_http_reply_client_error()`: 回复客户端错误页面
- `process_apple_wisper()`: 处理 Apple Whisper 协议
- `ev_http_send_redirect()`: 发送 HTTP 重定向
- `process_already_login_client()`: 处理已登录客户端

### 3. 认证模块 ([auth.c](file:///e:/projects/github.com/apfree-wifidog/src/auth.c))

**职责**:
- 处理客户端认证请求
- 与认证服务器通信
- 管理客户端登录/登出

**关键函数**:
- `ev_authenticate_client()`: 认证客户端
- `ev_logout_client()`: 登出客户端
- `thread_client_timeout_check()`: 检查客户端超时

### 4. 防火墙模块 ([firewall.c](file:///e:/projects/github.com/apfree-wifidog/src/firewall.c))

**职责**:
- 管理防火墙规则
- 控制客户端网络访问
- 支持多种防火墙后端（iptables、nftables、VPP）

**关键函数**:
- `fw_init()`: 初始化防火墙规则
- `fw_allow()`: 允许客户端访问
- `fw_deny()`: 拒绝客户端访问
- `fw_set_authservers()`: 设置认证服务器白名单
- `fw_set_authup()/fw_set_authdown()`: 处理认证服务器状态

### 5. 客户端列表模块 ([client_list.c](file:///e:/projects/github.com/apfree-wifidog/src/client_list.c))

**职责**:
- 管理在线客户端列表
- 管理离线客户端列表
- 客户端查找和操作

**关键函数**:
- `client_list_add()`: 添加客户端到在线列表
- `offline_client_list_add()`: 添加客户端到离线列表
- `client_list_find_by_mac()`: 根据 MAC 查找客户端
- `client_list_dup()`: 复制客户端列表（线程安全）

### 6. 认证服务器客户端模块 ([centralserver.c](file:///e:/projects/github.com/apfree-wifidog/src/centralserver.c))

**职责**:
- 与认证服务器通信
- 处理认证服务器响应
- 实现漫游功能

**关键函数**:
- `make_auth_request()`: 发送认证请求
- `make_roam_request()`: 发送漫游请求
- `process_auth_server_login_v2()`: 处理登录响应
- `process_auth_server_roam()`: 处理漫游响应

### 7. Ping 线程模块 ([ping_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/ping_thread.c))

**职责**:
- 定期向认证服务器发送心跳
- 报告网关状态
- 检查防火墙规则完整性
- 管理 Captive Portal 域名

**关键函数**:
- `thread_ping()`: Ping 线程主函数
- `ping_work_cb()`: 心跳回调
- `init_captive_domains()`: 初始化 Captive Portal 域名
- `check_wifidogx_firewall_rules()`: 检查防火墙规则

### 8. API 处理器模块 ([api_handlers.c](file:///e:/projects/github.com/apfree-wifidog/src/api_handlers.c))

**职责**:
- 处理 WebSocket 消息
- 处理 API 请求
- 实现实时通信

**关键函数**:
- `handle_heartbeat_request()`: 处理心跳请求
- `handle_auth_request()`: 处理认证请求
- `handle_kickoff_request()`: 处理踢下线请求

---

## Captive Portal 认证流程

### 完整认证流程图

```
┌─────────────┐
│  客户端连接  │
│  WiFi 网络   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 客户端尝试   │
│ 访问网络     │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 防火墙拦截   │
│ 未认证流量   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ HTTP Server │
│ 捕获请求     │
└──────┬──────┘
       │
       ├─────────────────────┐
       │                     │
       ▼                     ▼
┌─────────────┐     ┌─────────────┐
│ 特殊设备检测 │     │ 普通设备     │
│ (Apple等)   │     │ 处理         │
└──────┬──────┘     └──────┬──────┘
       │                     │
       └──────────┬──────────┘
                  │
                  ▼
         ┌─────────────┐
         │ 重定向到     │
         │ 认证页面     │
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
         │ 用户完成     │
         │ 认证         │
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
         │ 认证服务器   │
         │ 通知网关     │
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
         │ 更新防火墙   │
         │ 规则         │
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
         │ 客户端获得   │
         │ 网络访问     │
         └─────────────┘
```

### 详细流程说明

#### 1. 客户端连接阶段

当客户端连接到 WiFi 网络时：
- 客户端获得 IP 地址（DHCP）
- 客户端尝试访问网络（通常是访问某个 URL 或进行 DNS 查询）
- 防火墙规则拦截所有未认证的流量

#### 2. 流量拦截阶段

防火墙规则（在 [firewall.c](file:///e:/projects/github.com/apfree-wifidog/src/firewall.c) 中实现）：
```c
// 初始化防火墙规则
int fw_init(void) {
    // 清除现有规则
    fw_destroy();
    
    // 添加拦截规则
    // - 拦截所有未认证的 HTTP/HTTPS 流量
    // - 重定向到本地 HTTP 服务器
    
    // 添加白名单规则
    // - 允许访问认证服务器
    // - 允许访问 Captive Portal 检测域名
}
```

#### 3. HTTP 请求处理阶段

HTTP 服务器（在 [http.c](file:///e:/projects/github.com/apfree-wifidog/src/http.c) 中实现）捕获请求：

```c
void ev_http_callback_404(struct evhttp_request *req, void *arg) {
    // 1. 获取客户端信息
    char *remote_host = NULL;
    uint16_t port;
    int addr_type = ev_http_connection_get_peer(
        evhttp_request_get_connection(req), 
        &remote_host, 
        &port
    );
    
    // 2. 获取客户端 MAC 地址
    char *mac = arp_get(remote_host);
    
    // 3. 检查是否为已认证客户端
    if (process_already_login_client(req, mac, remote_host, addr_type, is_ssl)) {
        return; // 已认证，允许访问
    }
    
    // 4. 检查是否为特殊设备（Apple 等）
    if (process_apple_wisper(req, mac, remote_host, redir_url, mode)) {
        return; // 已处理
    }
    
    // 5. 检查是否为有线设备
    if (process_wired_device_pass(req, mac)) {
        return; // 已处理
    }
    
    // 6. 重定向到认证页面
    ev_http_send_redirect(req, redir_url, "Redirect to login page", 0);
}
```

#### 4. 特殊设备处理

**Apple 设备 Whisper 协议**（在 [http.c](file:///e:/projects/github.com/apfree-wifidog/src/http.c) 中实现）：

```c
static int process_apple_wisper(struct evhttp_request *req, 
                                const char *mac, 
                                const char *remote_host, 
                                const char *redir_url, 
                                const int mode) {
    // 检查是否为 Apple Captive Portal 域名
    if (!is_apple_captive(evhttp_request_get_host(req))) {
        return 0;
    }
    
    // 查找或创建离线客户端记录
    LOCK_OFFLINE_CLIENT_LIST();
    t_offline_client *o_client = offline_client_list_find_by_mac(mac);
    if (o_client == NULL) {
        o_client = offline_client_list_add(remote_host, mac);
    }
    
    // 根据客户端类型和命中次数决定如何处理
    if (o_client->client_type == 1) {
        // 已识别为 Apple 设备
        if (interval > 20 || o_client->hit_counts > 2) {
            ev_http_send_apple_redirect(req, redir_url);
        } else {
            ev_http_send_redirect(req, redir_url, "Redirect to login page", 0);
        }
    } else {
        // 首次检测到 Apple 设备
        o_client->client_type = 1;
        ev_http_replay_wisper(req); // 返回 Whisper 响应
    }
    
    return 1;
}
```

**Captive Portal 域名列表**（在 [ping_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/ping_thread.c) 中定义）：

```c
static struct captive_entry captive_entries[] = {
    {"captive.apple.com", "1.1.1.1"},
    {"www.apple.com", "1.1.1.1"},
    {"connect.rom.miui.com", "1.1.1.1"},
    {"www.msftconnecttest.com", "1.1.1.1"},
    {"www.gstatic.com", "1.1.1.1"},
    {"connectivitycheck.platform.hicloud.com", "1.1.1.1"},
    // ... 更多域名
};
```

#### 5. 认证阶段

**客户端认证流程**（在 [auth.c](file:///e:/projects/github.com/apfree-wifidog/src/auth.c) 中实现）：

```c
void ev_authenticate_client(struct evhttp_request *req, 
                           struct wd_request_context *context, 
                           t_client *client) {
    // 1. 生成认证 URI
    char *uri = get_auth_uri(REQUEST_TYPE_LOGIN, ONLINE_CLIENT, client);
    
    // 2. 设置请求上下文
    context->data = client;
    context->clt_req = req;
    
    // 3. 创建并发送认证请求
    struct evhttp_connection *wd_evcon = NULL;
    struct evhttp_request *wd_req = NULL;
    
    if (!wd_make_request(context, &wd_evcon, &wd_req, process_auth_server_login)) {
        evhttp_make_request(wd_evcon, wd_req, EVHTTP_REQ_GET, uri);
    }
    
    free(uri);
}
```

**认证服务器响应处理**（在 [centralserver.c](file:///e:/projects/github.com/apfree-wifidog/src/centralserver.c) 中实现）：

```c
static void process_auth_server_login_v2(struct evhttp_request *req, void *ctx) {
    auth_req_info *auth = ((struct wd_request_context *)ctx)->data;
    
    // 1. 读取响应数据
    char buffer[MAX_BUF] = {0};
    evbuffer_remove(evhttp_request_get_input_buffer(req), buffer, MAX_BUF-1);
    
    // 2. 解析 JSON 响应
    json_object *json_resp = json_tokener_parse(buffer);
    
    // 3. 检查返回码
    json_object *ret_code = NULL;
    json_object_object_get_ex(json_resp, "ret_code", &ret_code);
    int retCode = json_object_get_int(ret_code);
    
    if (retCode != 0) {
        // 认证失败
        return;
    }
    
    // 4. 处理客户端信息
    json_object *client = NULL;
    if (json_object_object_get_ex(json_resp, "client", &client)) {
        add_online_client(auth->ip, auth->mac, client);
    }
}
```

#### 6. 防火墙规则更新

**允许客户端访问**（在 [firewall.c](file:///e:/projects/github.com/apfree-wifidog/src/firewall.c) 中实现）：

```c
int fw_allow(t_client *client, int new_fw_connection_state) {
    int old_state = client->fw_connection_state;
    
    // 清除旧规则
    if (old_state != FW_MARK_NONE) {
        _fw_deny_raw(client->ip, client->mac, old_state);
    }
    
    // 设置新状态
    client->fw_connection_state = new_fw_connection_state;
    
    // 添加允许规则
#ifdef AW_FW3
    result = iptables_fw_access(FW_ACCESS_ALLOW, client->ip, client->mac, new_fw_connection_state);
#elif AW_FW4
    if (client->ip6) {
        result = nft_fw_access(FW_ACCESS_ALLOW, client->ip6, client->mac, new_fw_connection_state);
    }
    if (client->ip) {
        result = nft_fw_access(FW_ACCESS_ALLOW, client->ip, client->mac, new_fw_connection_state);
    }
#else
    result = vpp_fw_access(FW_ACCESS_ALLOW, client->ip, client->mac, new_fw_connection_state);
#endif
    
    return result;
}
```

#### 7. 客户端超时检查

**定期检查客户端超时**（在 [auth.c](file:///e:/projects/github.com/apfree-wifidog/src/auth.c) 中实现）：

```c
static void client_timeout_check_cb(evutil_socket_t fd, short event, void *arg) {
    struct wd_request_context *context = (struct wd_request_context *)arg;
    
    // 同步防火墙状态与认证服务器
#ifdef AUTHSERVER_V2
    ev_fw_sync_with_authserver_v2(context);
#else
    ev_fw_sync_with_authserver(context);
#endif
}
```

---

## 关键数据结构

### 客户端结构 (t_client)

```c
typedef struct _t_client {
    unsigned long long id;              // 客户端 ID
    char *ip;                           // IPv4 地址
    char *ip6;                          // IPv6 地址
    char *mac;                          // MAC 地址
    char *token;                        // 认证令牌
    char *name;                         // 客户端名称
    int fw_connection_state;             // 防火墙连接状态
    t_gateway_setting *gw_setting;      // 网关设置
    time_t first_login;                 // 首次登录时间
    int is_online;                      // 在线状态
    int wired;                          // 是否为有线设备
    
    // 流量统计
    struct {
        unsigned long long incoming_bytes;
        unsigned long long outgoing_bytes;
        unsigned long long incoming_packets;
        unsigned long long outgoing_packets;
        unsigned long long incoming_rate;
        unsigned long long outgoing_rate;
        time_t last_updated;
    } counters;
    
    // IPv6 流量统计
    struct {
        unsigned long long incoming_bytes;
        unsigned long long outgoing_bytes;
        unsigned long long incoming_packets;
        unsigned long long outgoing_packets;
        unsigned long long incoming_rate;
        unsigned long long outgoing_rate;
        time_t last_updated;
    } counters6;
    
    struct _t_client *next;            // 下一个客户端
} t_client;
```

### 离线客户端结构 (t_offline_client)

```c
typedef struct _t_offline_client {
    char *ip;                           // IP 地址
    char *mac;                          // MAC 地址
    time_t first_login;                 // 首次连接时间
    time_t last_login;                  // 最后连接时间
    int client_type;                    // 客户端类型（0=未知, 1=Apple）
    int hit_counts;                     // 命中次数
    int temp_passed;                    // 是否临时通过
    struct _t_offline_client *next;    // 下一个客户端
} t_offline_client;
```

### 网关设置结构 (t_gateway_setting)

```c
typedef struct _gateway_setting_t {
    char *gw_id;                        // 网关 ID
    char *gw_interface;                 // 网关接口
    char *gw_address_v4;                // IPv4 地址
    char *gw_address_v6;                // IPv6 地址
    char *gw_channel;                   // 网关信道
    struct _gateway_setting_t *next;   // 下一个网关设置
} t_gateway_setting;
```

### 认证服务器结构 (t_auth_serv)

```c
typedef struct _auth_serv_t {
    char *authserv_hostname;            // 认证服务器主机名
    char *authserv_path;                // 认证服务器路径
    int authserv_http_port;             // HTTP 端口
    int authserv_ssl_port;              // HTTPS 端口
    int authserv_use_ssl;               // 是否使用 SSL
    t_ip_trusted *ips_auth_server;       // 认证服务器 IP 列表
    struct _auth_serv_t *next;          // 下一个认证服务器
} t_auth_serv;
```

---

## 防火墙机制

### 防火墙规则集

ApFree WiFiDog 使用多个防火墙规则集来管理不同状态的客户端：

| 规则集名称 | 用途 | 标记值 |
|-----------|------|--------|
| `unknown-users` | 未认证用户 | FW_MARK_UNKNOWN |
| `validating-users` | 验证中用户 | FW_MARK_VALIDATING |
| `known-users` | 已认证用户 | FW_MARK_KNOWN |
| `locked-users` | 被锁定用户 | FW_MARK_LOCKED |
| `auth-is-down` | 认证服务器离线 | FW_MARK_AUTH_IS_DOWN |

### 防火墙初始化流程

```c
int fw_init(void) {
    // 1. 初始化 ICMP socket
    if (!init_icmp_socket()) {
        return 0;
    }
    
    // 2. 初始化防火墙规则
#ifdef AW_FW3
    result = iptables_fw_init();
#elif AW_FW4
    result = nft_fw_init();
    
    // 3. 如果是重启，恢复客户端规则
    if (restart_orig_pid) {
        nft_fw_reload_client();
        nft_fw_reload_trusted_maclist();
    } else {
        // 4. 加载绕过用户列表
        load_bypass_user_list();
    }
#endif
    
    return result;
}
```

### 防火墙操作

#### 允许客户端访问

```c
int fw_allow(t_client *client, int new_fw_connection_state) {
    // 1. 清除旧规则
    if (old_state != FW_MARK_NONE) {
        _fw_deny_raw(client->ip, client->mac, old_state);
    }
    
    // 2. 添加新规则
    client->fw_connection_state = new_fw_connection_state;
    
#ifdef AW_FW3
    result = iptables_fw_access(FW_ACCESS_ALLOW, client->ip, client->mac, new_fw_connection_state);
#elif AW_FW4
    if (client->ip6) {
        result = nft_fw_access(FW_ACCESS_ALLOW, client->ip6, client->mac, new_fw_connection_state);
    }
    if (client->ip) {
        result = nft_fw_access(FW_ACCESS_ALLOW, client->ip, client->mac, new_fw_connection_state);
    }
#endif
    
    return result;
}
```

#### 拒绝客户端访问

```c
int fw_deny(t_client *client) {
    int fw_connection_state = client->fw_connection_state;
    
    // 1. 清除状态
    client->fw_connection_state = FW_MARK_NONE;
    
    // 2. 删除规则
    if (client->ip6) {
        _fw_deny_raw(client->ip6, client->mac, fw_connection_state);
        conntrack_flush(client->ip6);
    }
    
    if (client->ip) {
        _fw_deny_raw(client->ip, client->mac, fw_connection_state);
        conntrack_flush(client->ip);
    }
    
    return nret;
}
```

### 认证服务器状态处理

```c
// 认证服务器离线时
int fw_set_authdown(void) {
#ifdef AW_FW3
    return iptables_fw_auth_unreachable(FW_MARK_AUTH_IS_DOWN);
#elif AW_FW4
    return nft_fw_auth_unreachable(FW_MARK_AUTH_IS_DOWN);
#endif
}

// 认证服务器在线时
int fw_set_authup(void) {
#ifdef AW_FW3
    return iptables_fw_auth_reachable();
#elif AW_FW4
    return nft_fw_auth_reachable();
#endif
}
```

---

## 认证服务器通信

### Ping (心跳) 接口

**请求参数**（在 [ping_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/ping_thread.c) 中实现）：

```c
// 每 60 秒发送一次心跳
void ping_work_cb(evutil_socket_t fd, short event, void *arg) {
    // 构建心跳请求
    char *uri = safe_strdup("/ping/?");
    
    // 添加参数
    // - device_id: 网关 ID
    // - sys_uptime: 系统运行时间
    // - sys_memfree: 系统空闲内存
    // - sys_load: 系统负载
    // - nf_conntrack_count: 连接跟踪计数
    // - cpu_usage: CPU 使用率
    // - wifidog_uptime: WiFiDog 运行时间
    // - online_clients: 在线客户端数
    // - offline_clients: 离线客户端数
    // - ssid: SSID
    // - aw_version: 版本
    
    // 发送请求
    evhttp_make_request(evcon, req, EVHTTP_REQ_GET, uri);
}
```

**服务器响应**：
- 成功：响应体包含 "Pong"
- 失败：标记认证服务器为离线

### Counters (计数器) 接口 (V2)

**请求格式**（JSON）：

```json
{
  "device_id": "字符串",
  "gateway": [
    {
      "gw_id": "字符串",
      "gw_channel": "字符串",
      "clients": [
        {
          "id": "整数",
          "ip": "字符串",
          "ip6": "字符串",
          "mac": "字符串",
          "token": "字符串",
          "name": "字符串",
          "incoming_bytes": "长长整型",
          "outgoing_bytes": "长长整型",
          "incoming_rate": "长长整型",
          "outgoing_rate": "长长整型",
          "incoming_packets": "长长整型",
          "outgoing_packets": "长长整型",
          "incoming_bytes_v6": "长长整型",
          "outgoing_bytes_v6": "长长整型",
          "incoming_rate_v6": "长长整型",
          "outgoing_rate_v6": "长长整型",
          "incoming_packets_v6": "长长整型",
          "outgoing_packets_v6": "长长整型",
          "first_login": "长长整型",
          "is_online": "布尔型",
          "wired": "布尔型"
        }
      ]
    }
  ]
}
```

**服务器响应**（JSON）：

```json
{
  "result": [
    {
      "gw_id": "字符串",
      "auth_op": [
        {
          "id": "整数",
          "auth_code": "整数"
        }
      ]
    }
  ]
}
```

**auth_code 值**：
- `0` (AUTH_ALLOWED): 允许客户端
- `1` (AUTH_DENIED): 拒绝客户端
- `2` (AUTH_VALIDATION): 验证中
- `5` (AUTH_VALIDATION_FAILED): 验证失败

### 漫游接口

**请求格式**：

```
/roam?gw_id={gw_id}&mac={mac}&gw_channel={gw_channel}
```

**服务器响应**（JSON）：

```json
{
  "roam": "yes|no",
  "client": {
    "token": "字符串",
    "first_login": "时间戳"
  }
}
```

---

## 特殊设备处理

### Apple 设备 Captive Portal 检测

Apple 设备使用 "Whisper" 协议检测 Captive Portal：

1. 设备连接到 WiFi
2. 设备访问 `captive.apple.com`
3. 服务器返回特殊的 HTML 响应
4. 设备显示 Captive Portal 页面

**处理流程**（在 [http.c](file:///e:/projects/github.com/apfree-wifidog/src/http.c) 中实现）：

```c
static int process_apple_wisper(struct evhttp_request *req, 
                                const char *mac, 
                                const char *remote_host, 
                                const char *redir_url, 
                                const int mode) {
    // 检查是否为 Apple Captive Portal 域名
    if (!is_apple_captive(evhttp_request_get_host(req))) {
        return 0;
    }
    
    // 首次检测：返回 Whisper 响应
    if (o_client->client_type != 1) {
        o_client->client_type = 1;
        ev_http_replay_wisper(req);
        return 1;
    }
    
    // 后续检测：重定向到认证页面
    if (interval > 20 || o_client->hit_counts > 2) {
        ev_http_send_apple_redirect(req, redir_url);
    } else {
        ev_http_send_redirect(req, redir_url, "Redirect to login page", 0);
    }
    
    return 1;
}
```

### Android 设备 Captive Portal 检测

Android 设备访问特定的检测域名：

- `connectivitycheck.gstatic.com`
- `www.google.cn`
- `clients1.google.com`
- `clients2.google.com`
- `clients3.google.com`
- `clients4.google.com`
- `clients5.google.com`

### 其他设备

- **小米**: `connect.rom.miui.com`
- **微软**: `www.msftconnecttest.com`
- **华为**: `connectivitycheck.platform.hicloud.com`
- **OPPO**: `conn1.oppomobile.com`, `conn2.oppomobile.com`
- **Vivo**: `wifi.vivo.com.cn`
- **Firefox**: `detectportal.firefox.com`

### Captive Portal 域名管理

**初始化 Captive Portal 域名**（在 [ping_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/ping_thread.c) 中实现）：

```c
static void init_captive_domains(void) {
    // 1. 创建自定义 hosts 文件
    FILE *fp = fopen(CUSTOM_HOSTS_FILE, "w");
    
    // 2. 写入所有 Captive Portal 域名
    for (int i = 0; i < (int)(sizeof(captive_entries)/sizeof(captive_entries[0])); i++) {
        fprintf(fp, "%s %s\n", captive_entries[i].ip, captive_entries[i].domain);
    }
    fclose(fp);
    
    // 3. 配置 dnsmasq 使用自定义 hosts 文件
    execute("uci -q add_list dhcp.@dnsmasq[0].addnhosts='" CUSTOM_HOSTS_FILE "'", 0);
    execute("uci commit dhcp && /etc/init.d/dnsmasq restart", 0);
}
```

**更新 Captive Portal 域名为真实 IP**：

```c
static void update_captive_domains_with_real_ips(void) {
    // 1. 解析每个域名的真实 IP
    for (int i = 0; i < (int)(sizeof(captive_entries)/sizeof(captive_entries[0])); i++) {
        struct hostent *he = gethostbyname(captive_entries[i].domain);
        if (he != NULL) {
            struct in_addr **addr_list = (struct in_addr **)he->h_addr_list;
            if (addr_list[0] != NULL) {
                char real_ip[16];
                inet_ntop(AF_INET, addr_list[0], real_ip, sizeof(real_ip));
                fprintf(fp, "%s %s\n", real_ip, captive_entries[i].domain);
            }
        }
    }
    
    // 2. 重启 dnsmasq
    execute("uci commit dhcp && /etc/init.d/dnsmasq restart", 0);
}
```

---

## 实时通信

### WebSocket 接口

ApFree WiFiDog 支持 WebSocket 实时通信，用于：

- 实时认证
- 客户端踢下线
- 临时访问授权
- 域名白名单同步
- 固件升级

**WebSocket 连接建立**（在 [ws_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/ws_thread.c) 中实现）：

```c
// 1. 发送 Upgrade 请求
GET /ws/wifidogx HTTP/1.1
Host: ws_server_hostname:ws_server_port
Upgrade: websocket
Connection: upgrade
Sec-WebSocket-Key: <随机密钥>
Sec-WebSocket-Version: 13

// 2. 服务器响应
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: <计算后的接受密钥>
```

**客户端到服务器消息**：

```json
// 连接消息
{
  "type": "connect",
  "device_id": "字符串",
  "gateway": [
    {
      "gw_id": "字符串",
      "gw_channel": "字符串",
      "gw_address_v4": "字符串",
      "auth_mode": "整数",
      "gw_interface": "字符串",
      "gw_address_v6": "字符串"
    }
  ]
}

// 心跳消息
{
  "type": "heartbeat",
  "device_id": "字符串",
  "gateway": [/* ... */]
}
```

**服务器到客户端消息**：

```json
// 认证消息
{
  "type": "auth",
  "token": "字符串",
  "client_ip": "字符串",
  "client_mac": "字符串",
  "client_name": "字符串",
  "gw_id": "字符串",
  "once_auth": "布尔型"
}

// 踢下线消息
{
  "type": "kickoff",
  "client_ip": "字符串",
  "client_mac": "字符串",
  "device_id": "字符串",
  "gw_id": "字符串"
}

// 临时访问消息
{
  "type": "tmp_pass",
  "client_mac": "字符串",
  "timeout": "整数"
}

// 域名同步消息
{
  "type": "sync_trusted_domain",
  "domains": ["domain1.com", "domain2.com"]
}

// 固件升级消息
{
  "type": "firmware_upgrade",
  "url": "<firmware_download_url>"
}
```

**消息处理**（在 [api_handlers.c](file:///e:/projects/github.com/apfree-wifidog/src/api_handlers.c) 中实现）：

```c
void handle_heartbeat_request(json_object *j_heartbeat) {
    // 1. 标记认证服务器为在线
    mark_auth_online();
    
    // 2. 提取网关数组
    json_object *gw_array = json_object_object_get(j_heartbeat, "gateway");
    
    // 3. 处理每个网关
    int gw_count = json_object_array_length(gw_array);
    for (int i = 0; i < gw_count; i++) {
        json_object *gw = json_object_array_get_idx(gw_array, i);
        // 更新网关状态
    }
}
```

---

## 代码流程图

### 主程序启动流程

```
main()
  │
  ├─> gw_main()
  │     │
  │     ├─> config_read()              // 读取配置
  │     ├─> wd_init()                 // 初始化
  │     │     │
  │     │     ├─> wd_msg_init()      // 初始化消息系统
  │     │     ├─> wd_redir_file_init() // 初始化重定向文件
  │     │     ├─> openssl_init()     // 初始化 OpenSSL
  │     │     ├─> init_resource_limits() // 设置资源限制
  │     │     ├─> init_started_time() // 设置启动时间
  │     │     ├─> init_firewall()    // 初始化防火墙
  │     │     └─> gateway_setting_init() // 初始化网关设置
  │     │
  │     ├─> wd_signals_init()         // 初始化信号处理
  │     ├─> init_service_threads()    // 初始化服务线程
  │     │     │
  │     │     ├─> thread_client_timeout_check() // 客户端超时检查
  │     │     ├─> thread_ping()       // Ping 线程
  │     │     ├─> thread_wdctl()     // 控制线程
  │     │     ├─> thread_https_server() // HTTPS 服务线程
  │     │     ├─> thread_mqtt_server() // MQTT 服务线程
  │     │     ├─> thread_websocket() // WebSocket 线程
  │     │     └─> thread_dns_monitor() // DNS 监控线程
  │     │
  │     └─> event_base_loop()        // 进入事件循环
```

### HTTP 请求处理流程

```
ev_http_callback_404()
  │
  ├─> ev_http_connection_get_peer()  // 获取客户端信息
  ├─> arp_get()                      // 获取 MAC 地址
  │
  ├─> process_already_login_client()  // 检查是否已登录
  │     │
  │     ├─> client_list_find_by_mac() // 查找客户端
  │     │
  │     ├─> 如果 IP 变更
  │     │     ├─> fw_allow_ip_mac()  // 允许新 IP
  │     │     └─> fw_deny_ip_mac()   // 拒绝旧 IP
  │     │
  │     └─> ev_http_wisper_success() // 返回成功
  │
  ├─> process_apple_wisper()         // 处理 Apple 设备
  │     │
  │     ├─> is_apple_captive()       // 检查是否为 Apple 域名
  │     ├─> offline_client_list_find_by_mac() // 查找离线客户端
  │     │
  │     ├─> 如果首次检测
  │     │     └─> ev_http_replay_wisper() // 返回 Whisper 响应
  │     │
  │     └─> 如果已检测
  │           └─> ev_http_send_apple_redirect() // 重定向
  │
  ├─> process_wired_device_pass()    // 处理有线设备
  │     │
  │     ├─> br_is_device_wired()     // 检查是否为有线设备
  │     ├─> add_trusted_maclist()    // 添加到信任列表
  │     └─> ev_http_resend()          // 重新发送请求
  │
  └─> ev_http_send_redirect()        // 重定向到认证页面
```

### 客户端认证流程

```
ev_authenticate_client()
  │
  ├─> get_auth_uri()                  // 生成认证 URI
  ├─> wd_make_request()               // 创建 HTTP 请求
  │     │
  │     ├─> evhttp_connection_new()   // 创建连接
  │     ├─> evhttp_request_new()      // 创建请求
  │     └─> evhttp_make_request()    // 发送请求
  │
  └─> process_auth_server_login_v2() // 处理响应
        │
        ├─> evbuffer_remove()         // 读取响应
        ├─> json_tokener_parse()      // 解析 JSON
        │
        ├─> 检查 ret_code
        │     │
        │     ├─> 如果为 0 (成功)
        │     │     └─> add_online_client() // 添加到在线列表
        │     │           │
        │     │           ├─> client_list_add() // 添加客户端
        │     │           └─> fw_allow()       // 更新防火墙规则
        │     │
        │     └─> 如果非 0 (失败)
        │           └─> safe_client_list_delete() // 删除客户端
        │
        └─> evhttp_send_reply()       // 回复客户端
```

### 防火墙规则更新流程

```
fw_allow()
  │
  ├─> 获取旧状态
  │
  ├─> 如果旧状态不为 NONE
  │     └─> _fw_deny_raw()           // 删除旧规则
  │
  ├─> 设置新状态
  │
  └─> 添加新规则
        │
        ├─> iptables_fw_access()      // iptables
        ├─> nft_fw_access()           // nftables
        └─> vpp_fw_access()           // VPP
```

---

## 总结

ApFree WiFiDog 的 Captive Portal 实现具有以下特点：

1. **高性能**: 采用事件驱动架构，支持高并发
2. **安全性**: 支持 HTTPS 重定向，使用 OpenSSL 加密
3. **灵活性**: 支持多种认证模式和防火墙后端
4. **兼容性**: 支持各种设备的 Captive Portal 检测
5. **实时性**: 支持 WebSocket 实时通信
6. **可扩展性**: 模块化设计，易于扩展

### 核心文件清单

| 文件 | 功能 |
|------|------|
| [gateway.c](file:///e:/projects/github.com/apfree-wifidog/src/gateway.c) | 主程序入口，初始化和管理 |
| [http.c](file:///e:/projects/github.com/apfree-wifidog/src/http.c) | HTTP 请求处理，重定向逻辑 |
| [auth.c](file:///e:/projects/github.com/apfree-wifidog/src/auth.c) | 客户端认证 |
| [firewall.c](file:///e:/projects/github.com/apfree-wifidog/src/firewall.c) | 防火墙规则管理 |
| [client_list.c](file:///e:/projects/github.com/apfree-wifidog/src/client_list.c) | 客户端列表管理 |
| [centralserver.c](file:///e:/projects/github.com/apfree-wifidog/src/centralserver.c) | 认证服务器通信 |
| [ping_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/ping_thread.c) | 心跳线程 |
| [api_handlers.c](file:///e:/projects/github.com/apfree-wifidog/src/api_handlers.c) | API 处理 |

### 参考文档

- [认证服务器 API 文档](file:///e:/projects/github.com/apfree-wifidog/AUTH_SERVER_API_ZH.md)
- [项目 README](file:///e:/projects/github.com/apfree-wifidog/README-zh.md)
- [配置文件示例](file:///e:/projects/github.com/apfree-wifidog/doc/wifidogx.conf)
