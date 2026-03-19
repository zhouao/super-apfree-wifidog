# ApFree WiFiDog MQTT 协议文档

本文档详细说明 ApFree WiFiDog 的 MQTT 通信协议，包括连接配置、消息格式、支持的操作命令及其实现细节。

---

## 目录

1. [概述](#概述)
2. [MQTT 配置](#mqtt-配置)
3. [连接建立](#连接建立)
4. [主题格式](#主题格式)
5. [消息格式](#消息格式)
6. [支持的命令](#支持的命令)
7. [响应格式](#响应格式)
8. [代码实现](#代码实现)
9. [使用示例](#使用示例)

---

## 概述

ApFree WiFiDog 使用 MQTT 协议实现与远程管理服务器的实时通信。通过 MQTT，服务器可以远程管理网关设备，包括：

- 管理信任列表（域名、IP、MAC 地址）
- 配置认证服务器
- 获取设备状态
- 执行设备操作（重启、重置等）

### 技术栈

- **MQTT 库**: Eclipse Mosquitto
- **TLS 支持**: 支持 TLS 加密连接
- **QoS 级别**: 0（最多一次）

---

## MQTT 配置

### 配置文件参数

在 `wifidogx.conf` 中配置 MQTT 服务器：

```conf
# MQTT 配置
MQTTServer      mqtt.example.com    # MQTT 服务器地址
serverport      8883                # MQTT 服务器端口（默认 1883）
mqttUsername    admin               # 用户名
mqttPassword    secret              # 密码
```

### 数据结构 ([conf.h](file:///e:/projects/github.com/apfree-wifidog/src/conf.h))

```c
typedef struct _mqtt_server_t {
    char *hostname;     // MQTT 服务器主机名
    char *username;     // 用户名
    char *password;     // 密码
    char *cafile;       // CA 证书文件路径
    char *crtfile;      // 客户端证书文件路径
    char *keyfile;      // 客户端密钥文件路径
    short port;         // 端口号
} t_mqtt_server;
```

---

## 连接建立

### 连接流程

```
┌─────────────────┐
│  thread_mqtt()  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ mosquitto_lib_init()  │  初始化 Mosquitto 库
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ mosquitto_new() │  创建客户端实例
│ (client_id = device_id)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ mosquitto_tls_set() │  配置 TLS（如果启用）
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 设置回调函数     │
│ - mqtt_connect_callback
│ - mqtt_message_callback
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ mosquitto_username_pw_set() │  设置认证信息
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ mosquitto_connect() │  连接到服务器
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ mosquitto_loop_forever() │  进入消息循环
└─────────────────┘
```

### 代码实现 ([mqtt_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/mqtt_thread.c))

```c
void thread_mqtt(void *arg)
{
    s_config *config = arg;
    struct mosquitto *mosq = NULL;
    char *host = config->mqtt_server->hostname;
    int port = config->mqtt_server->port;
    char *username = config->mqtt_server->username;
    char *password = config->mqtt_server->password;
    char *cafile = config->mqtt_server->cafile;
    int keepalive = 60;

    // 初始化 Mosquitto 库
    mosquitto_lib_init();

    // 创建客户端实例
    mosq = mosquitto_new(get_device_id(), true, config);
    if (mosq == NULL) {
        // 错误处理
        mosquitto_lib_cleanup();
        return;
    }

    // 配置 TLS
    if (mosquitto_tls_set(mosq, cafile, NULL, NULL, NULL, NULL)) {
        debug(LOG_INFO, "Error : Problem setting TLS option");
        mosquitto_destroy(mosq);
        mosquitto_lib_cleanup();
        return;
    }

    // 允许不安全的 TLS（简化部署）
    if (mosquitto_tls_insecure_set(mosq, true)) {
        debug(LOG_INFO, "Error : Problem setting TLS insecure option");
        mosquitto_destroy(mosq);
        mosquitto_lib_cleanup();
        return;
    }

    // 设置回调函数
    mosquitto_connect_callback_set(mosq, mqtt_connect_callback);
    mosquitto_message_callback_set(mosq, mqtt_message_callback);

    // 设置用户名密码
    if (username != NULL) {
        mosquitto_username_pw_set(mosq, username, password);
    }

    // 连接到服务器
    int retval = mosquitto_connect(mosq, host, port, keepalive);
    if (retval != MOSQ_ERR_SUCCESS) {
        debug(LOG_INFO, "Error : %s", mosquitto_strerror(retval));
        mosquitto_destroy(mosq);
        mosquitto_lib_cleanup();
        return;
    }

    // 进入消息循环
    mosquitto_loop_forever(mosq, -1, 1);

    mosquitto_destroy(mosq);
    mosquitto_lib_cleanup();
}
```

### 连接回调

```c
static void mqtt_connect_callback(struct mosquitto *mosq, void *obj, int rc)
{
    char *default_topic = NULL;
    safe_asprintf(&default_topic, "wifidogx/%s/request/+", get_device_id());
    mosquitto_subscribe(mosq, NULL, default_topic, 0); // QoS 0
    free(default_topic);
}
```

连接成功后，设备会自动订阅主题：`wifidogx/{device_id}/request/+`

---

## 主题格式

### 订阅主题（设备接收命令）

```
wifidogx/{device_id}/request/{req_id}
```

- `{device_id}`: 设备的唯一标识符
- `{req_id}`: 请求 ID（用于关联响应）

### 发布主题（设备发送响应）

```
wifidogx/{device_id}/response/{req_id}
```

---

## 消息格式

### 请求消息格式

所有请求消息使用 JSON 格式：

```json
{
  "op": "操作名称",
  "type": "操作类型",
  "value": "操作值"
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `op` | string | 是 | 操作名称，如 `set_trusted`、`del_trusted` 等 |
| `type` | string | 否 | 操作类型，如 `domain`、`ip`、`mac` 等 |
| `value` | string | 否 | 操作值，具体内容取决于操作类型 |

---

## 支持的命令

### 1. set_trusted - 添加信任列表

向信任列表中添加域名、IP 或 MAC 地址。

**请求格式：**

```json
{
  "op": "set_trusted",
  "type": "domain",
  "value": "example.com,example.org"
}
```

**支持的操作类型：**

| 类型 | 说明 | 示例值 |
|------|------|--------|
| `domain` | 添加信任域名 | `"example.com,google.com"` |
| `pdomain` | 添加通配符域名 | `".example.com,.google.com"` |
| `ip` | 添加信任 IP | `"192.168.1.1,10.0.0.1"` |
| `mac` | 添加信任 MAC | `"aa:bb:cc:dd:ee:ff"` |

**代码实现：**

```c
void set_trusted_op(void *mosq, const char *type, const char *value, 
                    const int req_id, const s_config *config)
{
    if (!type || !value) {
        send_mqtt_response(mosq, req_id, 400, "type or value is NULL", config);
        return;
    }

    for(int i = 0; mqtt_set_type[i].type != NULL; i++) {
        if (strcmp(mqtt_set_type[i].type, type) == 0) {
            mqtt_set_type[i].process_mqtt_set_type(value);
            send_mqtt_response(mosq, req_id, 200, "Ok", config);
            break;
        }	
    }
}
```

---

### 2. del_trusted - 删除信任列表

从信任列表中删除域名、IP 或 MAC 地址。

**请求格式：**

```json
{
  "op": "del_trusted",
  "type": "domain",
  "value": "example.com"
}
```

**支持的操作类型：** 与 `set_trusted` 相同

---

### 3. clear_trusted - 清空信任列表

清空指定类型的信任列表。

**请求格式：**

```json
{
  "op": "clear_trusted",
  "type": "domain"
}
```

**支持的操作类型：** 与 `set_trusted` 相同

---

### 4. show_trusted - 查看信任列表

查看指定类型的信任列表内容。

**请求格式：**

```json
{
  "op": "show_trusted",
  "type": "domain"
}
```

**响应示例：**

```json
{
  "response": "200",
  "msg": "example.com,google.com,baidu.com"
}
```

---

### 5. save_rule - 保存规则

将当前配置保存到持久化存储。

**请求格式：**

```json
{
  "op": "save_rule"
}
```

**代码实现：**

```c
void save_rule_op(void *mosq, const char *type, const char *value, 
                  const int req_id, const s_config *config)
{
    user_cfg_save();  // 保存用户配置
    send_mqtt_response(mosq, req_id, 200, "Ok", config);
}
```

---

### 6. get_status - 获取状态

获取设备当前状态信息。

**请求格式：**

```json
{
  "op": "get_status"
}
```

**说明：** 当前实现为空函数，需要扩展实现。

---

### 7. reboot - 重启设备

远程重启设备。

**请求格式：**

```json
{
  "op": "reboot"
}
```

**说明：** 当前实现为空函数，需要扩展实现。

---

### 8. reset - 重置设备

重置设备配置。

**请求格式：**

```json
{
  "op": "reset"
}
```

**说明：** 当前实现为空函数，需要扩展实现。

---

### 9. set_auth_serv - 设置认证服务器

动态配置认证服务器参数。

**请求格式：**

```json
{
  "op": "set_auth_serv",
  "value": {
    "hostname": "auth.example.com",
    "port": "80",
    "path": "/wifidog/"
  }
}
```

**字段说明：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `hostname` | string | 否 | 认证服务器主机名 |
| `port` | string | 否 | 认证服务器端口 |
| `path` | string | 否 | 认证服务器路径 |

**代码实现：**

```c
void set_auth_server_op(void *mosq, const char *type, const char *value, 
                        const int req_id, const s_config *config)
{
    json_object *json_request = json_tokener_parse(value);
    if (is_error(json_request)) {
        send_mqtt_response(mosq, req_id, 400, "Invalid JSON request", config);
        return;
    }

    json_object *jo_host_name = json_object_object_get(json_request, "hostname");
    json_object *jo_http_port = json_object_object_get(json_request, "port");
    json_object *jo_path = json_object_object_get(json_request, "path");

    LOCK_CONFIG();
    // 更新配置...
    UNLOCK_CONFIG();
    
    json_object_put(json_request);
    send_mqtt_response(mosq, req_id, 200, "Ok", config);
}
```

---

## 响应格式

### 成功响应

```json
{
  "response": "200",
  "msg": "Ok"
}
```

### 错误响应

```json
{
  "response": "400",
  "msg": "type or value is NULL"
}
```

### 响应代码

| 代码 | 含义 |
|------|------|
| 200 | 操作成功 |
| 400 | 请求参数错误 |

---

## 代码实现

### 命令处理表

```c
static struct wifidogx_mqtt_op {
    char *operation;
    void (*process_mqtt_op)(void *, const char *, const char *, const int, const s_config *);
} mqtt_op[] = {
    {"set_trusted", set_trusted_op},
    {"del_trusted", del_trusted_op},
    {"clear_trusted", clear_trusted_op},
    {"show_trusted", show_trusted_op},
    {"save_rule", save_rule_op},
    {"get_status", get_status_op},
    {"reboot", reboot_device_op},
    {"reset", reset_device_op},
    {"set_auth_serv", set_auth_server_op},
    {NULL, NULL}
};
```

### 类型处理表

**添加操作类型：**

```c
static struct wifidogx_mqtt_add_type {
    char *type;
    void (*process_mqtt_set_type)(const char *args);
} mqtt_set_type[] = {
    {"domain", add_trusted_domains},
    {"pdomain", add_trusted_pdomains},
    {"ip", add_trusted_iplist},
    {"mac", add_trusted_maclist},
    {NULL, NULL}
};
```

**删除操作类型：**

```c
static struct wifidogx_mqtt_del_type {
    char *type;
    void (*process_mqtt_del_type)(const char *args);
} mqtt_del_type[] = {
    {"domain", del_trusted_domains},
    {"pdomain", del_trusted_pdomains},
    {"ip", del_trusted_iplist},
    {"mac", del_trusted_maclist},
    {NULL, NULL}
};
```

**清空操作类型：**

```c
static struct wifidogx_mqtt_clear_type {
    char *type;
    void (*process_mqtt_clear_type)(void);
} mqtt_clear_type[] = {
    {"domain", clear_trusted_domains},
    {"pdomain", clear_trusted_pdomains},
    {"ip", clear_trusted_iplist},
    {"mac", clear_trusted_maclist},
    {NULL, NULL}
};
```

**查看操作类型：**

```c
static struct wifidogx_mqtt_show_type {
    char *type;
    char *(*process_mqtt_show_type)(void);
} mqtt_show_type[] = {
    {"domain", show_trusted_domains},
    {"pdomain", show_trusted_pdomains},
    {"ip", show_trusted_iplist},
    {"mac", show_trusted_maclist},
    {NULL, NULL}
};
```

### 消息处理流程

```c
static void mqtt_message_callback(struct mosquitto *mosq, void *obj, 
                                  const struct mosquitto_message *message)
{
    s_config *config = obj;

    if (message->payloadlen) {
        // 从主题中提取请求 ID
        unsigned int req_id = get_topic_req_id(message->topic);
        if (req_id) {
            // 处理 MQTT 请求
            process_mqtt_reqeust(mosq, req_id, message->payload, config);
        }
    }
}

static void process_mqtt_reqeust(struct mosquitto *mosq, const unsigned int req_id, 
                                 const char *data, s_config *config)
{
    // 解析 JSON 请求
    json_object *json_request = json_tokener_parse(data);
    if (is_error(json_request)) {
        debug(LOG_INFO, "user request is not valid");
        return;
    }

    // 获取操作名称
    const char *op = json_object_get_string(json_object_object_get(json_request, "op"));
    if (!op) {
        debug(LOG_INFO, "No op item get");
        return;
    }

    // 查找并执行对应的操作处理函数
    for (int i = 0; mqtt_op[i].operation != NULL; i++) {
        if (strcmp(op, mqtt_op[i].operation) == 0 && mqtt_op[i].process_mqtt_op) {
            const char *type = json_object_get_string(json_object_object_get(json_request, "type"));
            const char *value = json_object_get_string(json_object_object_get(json_request, "value"));
            mqtt_op[i].process_mqtt_op(mosq, type, value, req_id, config);
            break;
        }
    }

    json_object_put(json_request);
}
```

### 响应发送函数

```c
static void send_mqtt_response(struct mosquitto *mosq, const unsigned int req_id, 
                               int res_id, const char *msg, const s_config *config)
{
    char *topic = NULL;
    char *res_data = NULL;
    
    // 构建响应主题
    safe_asprintf(&topic, "wifidogx/%s/response/%d", get_device_id(), req_id);
    
    // 构建响应数据
    safe_asprintf(&res_data, "{\"response\":\"%d\",\"msg\":\"%s\"}", 
                  res_id, msg==NULL?"null":msg);
    
    debug(LOG_DEBUG, "send mqtt response: topic is %s msg is %s", topic, res_data);
    
    // 发布响应
    mosquitto_publish(mosq, NULL, topic, strlen(res_data), res_data, 0, false);
    
    free(topic);
    free(res_data);
}
```

---

## 使用示例

### 示例 1：添加信任域名

**请求：**

```bash
mosquitto_pub -h mqtt.example.com -p 8883 -u admin -P secret \
  -t "wifidogx/device001/request/1" \
  -m '{"op":"set_trusted","type":"domain","value":"example.com,google.com"}'
```

**响应：**

```
Topic: wifidogx/device001/response/1
Message: {"response":"200","msg":"Ok"}
```

---

### 示例 2：删除信任 MAC 地址

**请求：**

```bash
mosquitto_pub -h mqtt.example.com -p 8883 -u admin -P secret \
  -t "wifidogx/device001/request/2" \
  -m '{"op":"del_trusted","type":"mac","value":"aa:bb:cc:dd:ee:ff"}'
```

---

### 示例 3：查看信任 IP 列表

**请求：**

```bash
mosquitto_pub -h mqtt.example.com -p 8883 -u admin -P secret \
  -t "wifidogx/device001/request/3" \
  -m '{"op":"show_trusted","type":"ip"}'
```

**响应：**

```
Topic: wifidogx/device001/response/3
Message: {"response":"200","msg":"192.168.1.1,10.0.0.1,172.16.0.1"}
```

---

### 示例 4：清空信任域名列表

**请求：**

```bash
mosquitto_pub -h mqtt.example.com -p 8883 -u admin -P secret \
  -t "wifidogx/device001/request/4" \
  -m '{"op":"clear_trusted","type":"domain"}'
```

---

### 示例 5：设置认证服务器

**请求：**

```bash
mosquitto_pub -h mqtt.example.com -p 8883 -u admin -P secret \
  -t "wifidogx/device001/request/5" \
  -m '{"op":"set_auth_serv","value":"{\"hostname\":\"auth.example.com\",\"port\":\"80\",\"path\":\"/wifidog/\"}"}'
```

---

### 示例 6：保存配置

**请求：**

```bash
mosquitto_pub -h mqtt.example.com -p 8883 -u admin -P secret \
  -t "wifidogx/device001/request/6" \
  -m '{"op":"save_rule"}'
```

---

## 相关文件

| 文件 | 说明 |
|------|------|
| [mqtt_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/mqtt_thread.c) | MQTT 线程实现 |
| [mqtt_thread.h](file:///e:/projects/github.com/apfree-wifidog/src/mqtt_thread.h) | MQTT 头文件 |
| [wdctlx_thread.c](file:///e:/projects/github.com/apfree-wifidog/src/wdctlx_thread.c) | 控制线程（共享信任列表操作） |
| [wdctlx_thread.h](file:///e:/projects/github.com/apfree-wifidog/src/wdctlx_thread.h) | 控制线程头文件 |
| [conf.c](file:///e:/projects/github.com/apfree-wifidog/src/conf.c) | 配置管理 |
| [conf.h](file:///e:/projects/github.com/apfree-wifidog/src/conf.h) | 配置结构定义 |

---

## 注意事项

1. **TLS 配置**：虽然支持 TLS 加密，但代码中设置了 `mosquitto_tls_insecure_set(mosq, true)`，这会跳过证书验证，仅用于简化部署。生产环境建议启用完整的证书验证。

2. **QoS 级别**：当前使用 QoS 0（最多一次），消息可能会丢失。如需可靠传输，建议升级到 QoS 1 或 2。

3. **错误处理**：部分命令（如 `get_status`、`reboot`、`reset`）当前为空实现，需要根据实际需求扩展。

4. **线程安全**：配置操作使用了 `LOCK_CONFIG()` 和 `UNLOCK_CONFIG()` 保护，确保线程安全。

5. **持久化**：使用 `save_rule` 命令可以将配置保存到 UCI 配置系统，确保重启后配置不丢失。
