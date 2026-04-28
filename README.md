# Xray POC 转换器

将互联网上的 POC 内容（HTTP 请求包、漏洞描述、Nuclei/Goby YAML 等）转换为长亭 Xray 可用的 YAML POC 规则。

## 功能特性

- **智能转换**：HTTP 请求包 → Xray POC YAML
- **多格式支持**：Nuclei/Goby YAML → Xray 格式
- **交互式确认**：生成前询问关键信息（状态码、文件路径、验证特征等）
- **YAML 语法校验**：使用 Python yaml.safe_load() 自动检测语法错误
- **质量优化**：自动符合 Xray 高质量 POC 规范
- **实战验证**：所有示例均经过 Xray 1.9.11 实际测试

## 支持的漏洞类型

文件读取、SQL 注入、命令执行、反序列化、XXE、SSRF、未授权访问等。

## 使用方法

在 Claude Code 中输入：

```
帮我把这个请求包转成 Xray POC：
POST /cgi/webcgi?syscore=checklogin HTTP/1.1
Host: target.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary
...
```

系统会自动询问关键信息：
- 第一个请求的预期状态码
- 验证请求的路径和匹配特征
- 是否需要文件清理步骤
- FOFA 搜索语法（可选）

生成后会自动进行 YAML 语法校验，确保格式正确。

## 安装

```bash
# 克隆到本地 skills 目录
git clone https://github.com/1156494497/xray-poc-converter.git ~/.claude/skills/xray-poc-converter
```

## POC 质量规范

转换时遵循以下原则：

- **无害性**：不执行破坏性操作（删除文件、修改密码等）
- **随机性**：使用 `randomInt()` 生成随机验证值，避免固定值冲突
- **确定性**：匹配规则唯一确定，避免误报
- **通用性**：支持多平台（Linux/Windows）和多版本
- **语法正确**：自动进行 YAML 语法校验，确保 Xray 可加载

## 语法规范（Xray 1.9.x）

- Body 必须使用单行字符串，换行用 `\r\n`
- Headers 值必须用引号包裹
- Expression 必须在 request 同级
- 最终 expression 必须单行
- 状态码必须精确匹配（如 502 就写 502）

## 目录结构

```
xray-poc-converter/
├── SKILL.md                          # 主技能文件
└── references/
    ├── poc_template.md              # 各类型 POC 模板
    ├── expression_functions.md       # Xray 表达式函数参考
    └── best_practices.md            # 高质量编写规范
```

## 贡献

欢迎提交 Issue 和 PR。
