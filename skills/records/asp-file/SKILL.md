---
name: asp-file
description: "管理 ASP 文件和评论附件。适用于上传、查看元数据、下载或读取文本文件。"
argument-hint: "upload <path> | info <file_key> | download <file_key> | read-text <file_key>"
compatibility: requires asp-cli >= 0.1.0
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [file, attachment, upload, download]
  documentation: https://asp.viperrtp.com/
---

# ASP File

当用户要根据 `file_key` 获取 ASP 文件信息、下载文件、读取受支持的文本文件，或上传文件后作为评论附件使用时，使用这个 skill。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户从评论附件或其他 ASP 响应中拿到了 `file_key`，希望查看文件元数据。
- 用户希望下载附件或证据文件。
- 用户希望读取文本类附件内容。
- 用户要把本地文件上传到 ASP，并把返回的 `file_key` 用于后续评论。

## 运行规则

- 不要编造 `file_key`、文件名或下载地址。
- `asp file info` 只返回元数据和下载地址，不返回文件正文。
- `asp file read-text` 只适用于受支持的文本类内容，并受 `--max-bytes` 限制。
- 不要把大段文件内容直接倾倒到回复中；需要时只总结与用户问题相关的部分。
- 如果用户要把文件附加到评论，先上传文件，再转交 `asp-comment`。

## 命令契约

### 上传本地文件

```bash
asp file upload ./evidence.txt --output json
```

上传成功后返回的 `file_key` 可用于评论附件。

### 查看文件信息

```bash
asp file info <file_key> --output json
```

返回字段通常包括 `file_key`、`filename`、`size`、`content_type`、`download_url` 和 `uploaded_at`。

### 下载文件

```bash
asp file download <file_key> --output-path ./downloaded-evidence.txt --output json
```

如果用户未指定输出路径，CLI 会使用文件名或 `file_key` 作为默认文件名。

### 读取文本文件

```bash
asp file read-text <file_key> --max-bytes 65536 --output json
```

- `--max-bytes` 用于限制读取大小。
- 如果文件不是支持的文本类型，命令会失败；不要伪装成已读取。

## 决策流程

1. 用户要上传本地文件时，确认路径，然后执行 `asp file upload <path> --output json`。
2. 用户要查看文件元数据时，确认 `file_key`，然后执行 `asp file info <file_key> --output json`。
3. 用户要下载文件时，确认 `file_key` 和可选输出路径，然后执行 `asp file download`。
4. 用户要读取内容时，优先确认是否是文本类文件；执行 `asp file read-text` 并限制 `--max-bytes`。
5. 用户要把文件附加到评论时，先拿到 `file_key`，再转交 `asp-comment`。

## SOP

### 获取文件信息

1. 确认用户提供了 `file_key`。
2. 执行 `asp file info <file_key> --output json`。
3. 返回文件名、类型、大小、上传时间和下载地址。
4. 如果用户要求读取内容，说明需要使用 `asp file read-text`，并受内容类型和大小限制。

### 上传文件供评论使用

1. 确认本地文件路径存在且是要上传的文件。
2. 执行 `asp file upload <path> --output json`。
3. 记录返回的 `file_key`。
4. 如果用户要添加评论，转交 `asp-comment`，通过 `asp comment add <target_id> --file-key <file_key> --output json` 关联附件。

### 读取文本内容

1. 确认 `file_key`。
2. 选择合理的 `--max-bytes`，默认不要超过用户需要的范围。
3. 执行 `asp file read-text <file_key> --max-bytes <n> --output json`。
4. 如果返回 `truncated=true`，明确说明内容被截断。
5. 只总结和用户问题相关的文本，不输出无关大段内容。

## 澄清规则

- 缺少 `file_key` 时，询问。
- 用户说“这个文件”但没有给本地路径或 `file_key` 时，询问文件来源。
- 用户要下载但没有指定输出路径时，可使用 CLI 默认路径；如果可能覆盖重要文件，先确认。
- 用户要读取非文本内容时，说明 `read-text` 不适用，可改为下载文件。

## 输出规则

- 保持简洁。
- 不要声称已经读取文件内容，除非实际执行了 `asp file read-text` 或用户提供了内容。
- 文件类型不受默认假设约束；不要假设一定是文本、图片或 JSON。
- 对下载结果给出输出路径；对上传结果给出 `file_key`。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果 `file_key` 无效或文件不存在，直接说明。
- 如果文件不是支持的文本类型，说明不能用 `read-text`，不要尝试伪造内容。
