# Xray POC 表达式函数参考

## 字符串函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `contains(str)` | 字符串包含 | `response.body_string.contains("test")` |
| `matches(str)` | 正则匹配（字符串） | `"root:.*?:[0-9]*:[0-9]*:".matches(response.body_string)` |
| `bmatches(bytes)` | 正则匹配（字节） | `"root:.*?:[0-9]*:[0-9]*:".bmatches(response.body)` |
| `bcontains(bytes)` | 字节包含 | `response.body.bcontains(b"test")` |
| `bstartsWith(bytes)` | 字节开头匹配 | `response.body.bstartsWith(bytes(string(s1 - s2)))` |
| `substr(str, start, length)` | 字符串截取 | `substr(md5(string(s1)), 2, 28)` |

## 数学/随机函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `randomInt(min, max)` | 生成随机整数 | `randomInt(100000, 200000)` |
| `md5(str)` | MD5 哈希 | `md5(string(s1))` |

## 编码/转换函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `string(val)` | 转字符串 | `string(s2 - s1)` |
| `bytes(str)` | 转字节 | `bytes(string(s1 - s2))` |
| `base64Decode(str)` | Base64 解码 | `base64Decode("...")` |
| `upper(str)` | 转大写 | `upper(rs)` |
| `lower(str)` | 转小写 | `lower(rs)` |

## 响应对象属性

| 属性 | 说明 | 示例 |
|------|------|------|
| `response.status` | HTTP 状态码 | `response.status == 200` |
| `response.body` | 响应体（字节） | `response.body.bcontains(b"test")` |
| `response.body_string` | 响应体（字符串） | `response.body_string.contains("test")` |
| `response.headers` | 响应头 | `response.headers["Content-Type"].contains("json")` |
| `response.latency` | 响应延迟（毫秒） | `response.latency - r0latency >= sleepSecond1 * 1000` |
| `response.raw` | 原始响应 | `response.raw.bcontains(b"STAT pid")` |

## 其他函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `reverse.wait(seconds)` | 反向连接等待 | `reverse.wait(5)` |
| `bsubmatch(regex)` | 正则子匹配 | `"(?P<tmp>.+?)".bsubmatch(response.body)` |

## 逻辑运算符

| 运算符 | 说明 |
|--------|------|
| `&&` | 逻辑与 |
| `||` | 逻辑或 |
| `==` | 等于 |
| `!=` | 不等于 |
| `>`, `<`, `>=`, `<=` | 比较运算符 |
| `!` | 逻辑非 |

## 使用示例

### 组合条件
```yaml
expression: |
  response.status == 200 && 
  response.body_string.contains("admin") && 
  "password.*?:.*".matches(response.body_string)
```

### 正则匹配
```yaml
expression: |
  "root:.*?:[0-9]*:[0-9]*:".matches(response.body_string)
```

### 字节匹配
```yaml
expression: |
  response.body.bcontains(b"\x89PNG")
```

### 时间盲注
```yaml
expression: |
  response.latency - r0latency >= sleepSecond1 * 1000
```

### 子匹配提取
```yaml
expression: |
  "(?P<tmp>.+?)".bsubmatch(response.body)
```
