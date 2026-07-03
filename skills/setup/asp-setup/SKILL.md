---
name: asp-setup
description: "配置 ASP CLI 认证，并验证 ASP skills 可以访问平台。"
argument-hint: "配置 ASP CLI"
compatibility: requires asp-cli >= 0.1.0
disable-model-invocation: true
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [asp, setup, cli, authentication]
  documentation: https://asp.viperrtp.com/
---

# ASP Setup

当用户首次准备使用 ASP skills、切换 ASP 服务地址，或排查认证问题时，使用这个 skill。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 设置流程

1. 检查本机是否已安装 `asp` 命令：

   ```bash
   asp --help
   ```

2. 如果 `asp` 命令不存在，提示用户先安装 CLI：

   ```bash
   pipx install asp-cli
   ```

3. 引导用户在本机完成认证。不要要求用户把 API key 粘贴到对话中，除非他们无法直接执行命令：

   ```bash
   asp auth login --api-url https://asp.example.com --api-key asp_xxx
   ```

4. 验证连接和认证状态：

   ```bash
   asp doctor --output json
   ```

5. 如果 `doctor` 成功，说明 ASP skills 已可使用。

## 规则

- 不要把 API key 写入仓库文件、skill 文件或 markdown 记录。
- 不要在最终回复中输出完整 API key。
- 如果 `asp doctor --output json` 失败，停止继续操作，并根据 JSON error 或 CLI 错误说明失败原因。
- 所有 ASP 操作类 skill 都应使用 `asp ... --output json`。
