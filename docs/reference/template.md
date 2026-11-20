---
title: Template
---

# Template 模块参考

Template 模块是可启动 OCI Boot Index 的命名目录。实现位于 `internal/template`，containerd 适配器位于 `internal/adapters/containerd/template`。

本页内容与 Conch 源码仓的 `dev-openEuler-24.03-LTS_SP4` 分支对齐。

## 数据模型

```go
type Entry struct {
    Name                  string
    Origin                Origin
    BootMode              BootMode
    BootIndexDigest       string
    ParentBootIndexDigest string
    SourceSandboxID       string
    SourceRef             string
    Labels                map[string]string
    CreatedAt             int64
}
```

| 字段 | 说明 | 约束 |
| --- | --- | --- |
| `Name` | 用户可见的 Template Name。 | 必填，首尾空白会被删除。 |
| `Origin` | Template 的产生方式。 | `image` 或 `checkpoint`。 |
| `BootMode` | Sandbox 启动方式。 | `cold` 或 `resume`，必须与 Boot Index 内容一致。 |
| `BootIndexDigest` | 不可变 Template ID 和 Boot Index digest。 | 必填，必须是有效 OCI digest。 |
| `ParentBootIndexDigest` | checkpoint 的父 Template ID。 | 可选。 |
| `SourceSandboxID` | 产生 checkpoint 的 Sandbox ID。 | 可选。 |
| `SourceRef` | 创建或拉取时的来源 registry reference。 | 可选。 |
| `Labels` | 调用方标签。 | 可选；键必须能编码为 containerd label。 |
| `CreatedAt` | 内部 image record 的创建时间。 | Unix 纳秒。 |

`Origin` 与 `BootMode` 的常见组合：

| Origin | BootMode | 来源 |
| --- | --- | --- |
| `image` | `cold` | 从普通 OCI rootfs Image、kernel 和 initrd 构建。 |
| `checkpoint` | `resume` | 从 Sandbox checkpoint 生成，或从包含内存状态的 Boot Index 拉取。 |

## Name 与 ID

Template Name 是可变引用，Template ID 是不可变内容标识：

```text
localhost/conch/python:latest  ──>  sha256:012345...
          Template Name                 Template ID
```

Store 为每个 Name 创建一个内部 containerd image record。record 名称由 `TemplateRecordName(name)` 确定性生成；对同一个 Name 再次执行 `Put` 会更新 record 的 target 和 metadata。因此旧 ID 仍然标识旧内容，但 Name 会指向新的 ID。

Sandbox 创建支持两种解析路径：

- `template_name`：通过 Store 查找当前 ID，同时记录解析后的 Name 和 ID。
- `template_id`：直接校验 containerd 中对应的 Boot Index，不要求存在 Template Name record。

请求必须恰好选择其中一种；两者都传或都未配置时返回无效参数。conchd 可以通过 `sandbox.default_spec` 提供默认选择器。

## Store 接口

```go
type Store interface {
    Put(context.Context, Entry, ocispec.Descriptor) (Entry, error)
    Get(context.Context, string) (Entry, error)
    List(context.Context, Filter) ([]Entry, error)
    Delete(context.Context, string) error
}
```

### Put

```go
Put(ctx context.Context, entry Entry, target ocispec.Descriptor) (Entry, error)
```

`Put` 执行以下校验和持久化步骤：

1. 规范化 `Entry`，校验 Name、Origin、BootMode 和 digest。
2. 要求 `target.Digest` 与 `BootIndexDigest` 相同。
3. 读取 Boot Index 内容，确认 cold/resume 模式与 `BootMode` 相同。
4. 为 descriptor closure 写入 containerd GC children labels。
5. 创建或更新由 Name 派生的内部 image record。
6. 从 record 的 `CreatedAt` 填充返回值。

同名 record 已存在且属于 Template schema 时会被更新；如果该内部 record 被非 Template 数据占用，则返回 already-exists 类错误。

### Get

```go
Get(ctx context.Context, name string) (Entry, error)
```

`Get` 按 Name 查找内部 image record，重新检查 schema label、record 名称、Boot Index 内容、image kind 和 metadata，然后返回规范化 `Entry`。接口不按 Template ID 查询。

### List

```go
type Filter struct {
    Origin   Origin
    BootMode BootMode
}

List(ctx context.Context, filter Filter) ([]Entry, error)
```

空字段匹配所有值；非空字段只接受已定义的 Origin 或 BootMode。Store 只返回带当前 Template schema label、且 record 名称能够反解出 Template Name 的记录。

### Delete

```go
Delete(ctx context.Context, name string) error
```

`Delete` 按 Name 删除内部 image record，并使用条件 target 删除防止误删已并发更新的记录。不存在的 Name 视为成功；内容何时真正回收由 containerd GC 与其它引用共同决定。

## containerd 持久化

Template metadata 保存在内部 image record labels：

- `io.conch.template.schema`
- `io.conch.template.origin`
- `io.conch.template.parent`
- `io.conch.template.source-sandbox`
- `io.conch.template.source-ref`
- `io.conch.template.user.<key>`

Boot Index 类型同时通过 `io.conch.kind` 区分 cold 与 resume。用户不应直接修改这些 record 或 labels；使用 `conch template` 命令管理 Template。

## 对外记录

控制面将内部 `Entry` 转换为以下 JSON：

```json
{
  "name": "localhost/conch/python:latest",
  "template_id": "sha256:0123456789abcdef...",
  "origin": "image",
  "boot_mode": "cold",
  "parent_template_id": "",
  "source_sandbox_id": "",
  "source_ref": "docker.io/library/python:3.12",
  "labels": {"purpose": "agent"},
  "created_at": 1788249600000000000
}
```

## HTTP 接口

Template 管理接口与 [Sandbox HTTP API](/docs/reference/sandbox) 共用同一个 Unix socket、错误响应格式和 JSON 请求限制。除 `create` 使用 multipart 外，其余接口均为 `POST` + JSON：

| 路径 | 说明 |
| --- | --- |
| `/api/template/create` | 从 OCI Image 创建 Template（multipart）。 |
| `/api/template/pull` | 从 registry 拉取 Boot Index Template。 |
| `/api/template/push` | 将 Template 发布到 registry。 |
| `/api/template/save` | 将一个或多个 Template 导出为离线 tar 归档。 |
| `/api/template/load` | 导入离线 tar 归档。 |
| `/api/template/unpack` | 预解包指定 Template 的全部组件。 |
| `/api/template/list` | 列出 Template。 |
| `/api/template/inspect` | 查询单个 Template。 |
| `/api/template/remove` | 删除 Template record。 |

### create

`POST /api/template/create` 使用 `multipart/form-data`，包含一个 JSON 表单字段和两个文件字段：

| 部分 | 类型 | 说明 |
| --- | --- | --- |
| `metadata` | JSON 表单字段 | `name`、`source` 必填；可选 `plain_http`、`username`、`password`、`labels`。 |
| `kernel` | 文件 | guest kernel 文件，最大 64 MiB。 |
| `initrd` | 文件 | initrd 文件，最大 64 MiB。 |

请求体总大小上限为 129 MiB。成功返回：

```json
{
  "status": "ok",
  "template_name": "localhost/conch/python:latest",
  "template_id": "sha256:0123456789abcdef..."
}
```

### pull

```json
{
  "reference": "registry.example.com/conch/python:latest",
  "plain_http": false,
  "username": "",
  "password": "",
  "labels": {}
}
```

`reference` 必填，其余可选。成功返回与 `create` 相同的 `status`、`template_name` 和 `template_id`。拉取后本地 Template Name 使用 registry 返回的规范化来源名称；`pull` 校验 Boot Index 结构，普通 OCI Image 不走该接口。

### push

```json
{
  "name": "localhost/conch/python:latest",
  "remote_reference": "registry.example.com/conch/python:latest",
  "plain_http": false,
  "username": "",
  "password": ""
}
```

`name` 和 `remote_reference` 必填，其余可选。成功返回 `{"status":"ok"}`。

### save

```json
{
  "names": ["localhost/conch/openeuler:latest"]
}
```

`names` 至少包含一个非空且不重复的 Template Name。成功时响应体为 `application/x-tar` 归档流，包含每个 Template 的完整 Boot Index closure。归档开始输出前出错返回普通 JSON 错误；开始输出后出错则直接中断连接，避免调用方保留截断的归档。

### load

请求体为 `save` 导出的 tar 归档，也接受 gzip 或 zstd 压缩格式。成功返回：

```json
{
  "items": [
    {
      "template_name": "localhost/conch/openeuler:latest",
      "template_id": "sha256:0123456789abcdef..."
    }
  ]
}
```

`load` 按归档中的 Name 创建或更新本地 Template；已有同名 Template 指向归档中的 Template ID。`load` 不会 unpack，也不按本机架构拒绝归档。

### unpack、list、inspect 与 remove

`unpack`、`inspect` 和 `remove` 请求为 `{"name": "<Template Name>"}`，只接受 Template Name。`list` 请求为 `{"origin": "...", "boot_mode": "..."}`，两个过滤字段均可省略，取值见数据模型；返回 `{"items": [TemplateRecord]}`。`inspect` 直接返回单个 TemplateRecord；`unpack` 和 `remove` 返回 `{"status":"ok"}`。`remove` 具有幂等语义：名称不存在时也视为成功。

## 离线归档

`template save` 导出 Boot Index 完整 closure 的 OCI image layout tar 归档。Template Name、Origin、血缘和 Labels 保存在 layout index 的 annotations 中，不属于 Boot Index blob，因此 Template ID 不变。`template load` 导入该归档并创建或更新同名 record。现有 pull、push、create 和 Sandbox 接口不受离线归档影响。

CLI 操作及 Template Name 更新语义见[模板管理](/docs/user/template-management)。
