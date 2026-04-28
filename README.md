# Xray POC 转换器

将互联网上的 POC 内容（HTTP 请求包、漏洞描述、Nuclei/Goby YAML 等）转换为长亭 Xray 可用的 YAML POC 规则。

## 功能

- HTTP 请求包 → Xray POC YAML
- 漏洞描述 → 功能完整的 POC
- Nuclei/Goby YAML → Xray 格式
- 优化现有 POC 符合高质量规范

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

## 安装

```bash
# 克隆到本地 skills 目录
git clone https://github.com/1156494497/xray-poc-converter.git ~/.claude/skills/xray-poc-converter
```

## POC 质量规范

- **无害性**：不执行破坏性操作
- **随机性**：使用 `randomInt()` 生成随机验证值
- **确定性**：匹配规则唯一确定，避免误报
- **通用性**：支持多平台（Linux/Windows）

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
