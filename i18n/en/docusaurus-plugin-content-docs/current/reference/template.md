---
title: Template
---

# Template Module Reference

The Template module is a named catalog of bootable OCI Boot Indexes. Its implementation is in `internal/template`, and its containerd adapter is in `internal/adapters/containerd/template`.

This page aligns with the `dev-openEuler-24.03-LTS_SP4` branch of the Conch source repository.

## Data model

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

| Field | Description | Constraint |
| --- | --- | --- |
| `Name` | User-visible Template Name. | Required; leading and trailing whitespace is removed. |
| `Origin` | How the Template was produced. | `image` or `checkpoint`. |
| `BootMode` | How a Sandbox boots. | `cold` or `resume`; must match the Boot Index content. |
| `BootIndexDigest` | Immutable Template ID and Boot Index digest. | Required; must be a valid OCI digest. |
| `ParentBootIndexDigest` | Parent Template ID of a checkpoint. | Optional. |
| `SourceSandboxID` | ID of the Sandbox that produced a checkpoint. | Optional. |
| `SourceRef` | Source registry reference used during creation or pull. | Optional. |
| `Labels` | Caller-provided labels. | Optional; keys must be encodable as containerd labels. |
| `CreatedAt` | Creation time of the internal image record. | Unix nanoseconds. |

Common `Origin` and `BootMode` combinations:

| Origin | BootMode | Source |
| --- | --- | --- |
| `image` | `cold` | Built from a regular OCI rootfs Image, kernel, and initrd. |
| `checkpoint` | `resume` | Generated from a Sandbox checkpoint or pulled from a Boot Index that contains memory state. |

## Name and ID

A Template Name is a mutable reference; a Template ID is an immutable content identifier:

```text
localhost/conch/python:latest  ──>  sha256:012345...
          Template Name                 Template ID
```

The Store creates an internal containerd image record for each Name. `TemplateRecordName(name)` derives the record name deterministically. Calling `Put` again for the same Name updates the record target and metadata. The old ID still identifies the old content, while the Name points to the new ID.

Sandbox creation supports two resolution paths:

- `template_name`: the Store resolves the current ID and records both the resolved Name and ID.
- `template_id`: validates the corresponding Boot Index in containerd directly; a Template Name record is not required.

A request must select exactly one. Supplying both or leaving both unconfigured returns an invalid-argument error. conchd can provide a default selector through `sandbox.default_spec`.

## Store interface

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

`Put` performs these validation and persistence steps:

1. Normalizes `Entry` and validates its Name, Origin, BootMode, and digest.
2. Requires `target.Digest` to match `BootIndexDigest`.
3. Reads the Boot Index and verifies that its cold/resume mode matches `BootMode`.
4. Writes containerd GC children labels for the descriptor closure.
5. Creates or updates the internal image record derived from the Name.
6. Populates the returned value from the record's `CreatedAt`.

An existing record with the same name is updated if it belongs to the Template schema. If non-Template data occupies that internal record, `Put` returns an already-exists error.

### Get

```go
Get(ctx context.Context, name string) (Entry, error)
```

`Get` looks up an internal image record by Name, revalidates the schema label, record name, Boot Index content, image kind, and metadata, then returns a normalized `Entry`. It does not query by Template ID.

### List

```go
type Filter struct {
    Origin   Origin
    BootMode BootMode
}

List(ctx context.Context, filter Filter) ([]Entry, error)
```

Empty fields match all values. Non-empty fields accept only a defined Origin or BootMode. The Store returns only records with the current Template schema label whose record names can be decoded into Template Names.

### Delete

```go
Delete(ctx context.Context, name string) error
```

`Delete` removes the internal image record by Name and uses a conditional target deletion to avoid removing a concurrently updated record. A missing Name is treated as success. Actual content reclamation depends on containerd GC and other references.

## containerd persistence

Template metadata is stored in internal image-record labels:

- `io.conch.template.schema`
- `io.conch.template.origin`
- `io.conch.template.parent`
- `io.conch.template.source-sandbox`
- `io.conch.template.source-ref`
- `io.conch.template.user.<key>`

The Boot Index type also uses `io.conch.kind` to distinguish cold and resume content. Users should not edit these records or labels directly; manage Templates with `conch template` commands.

## External record

The control plane converts the internal `Entry` to this JSON structure:

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

## HTTP API

Template management endpoints share the same Unix socket, error response format, and JSON request limits as the [Sandbox HTTP API](/docs/reference/sandbox). Except for `create`, which uses multipart, every endpoint accepts `POST` with a JSON body:

| Path | Description |
| --- | --- |
| `/api/template/create` | Create a Template from an OCI Image (multipart). |
| `/api/template/pull` | Pull a Boot Index Template from a registry. |
| `/api/template/push` | Publish a Template to a registry. |
| `/api/template/save` | Export one or more Templates as an offline tar archive. |
| `/api/template/load` | Import an offline tar archive. |
| `/api/template/unpack` | Unpack all components of a Template in advance. |
| `/api/template/list` | List Templates. |
| `/api/template/inspect` | Inspect a single Template. |
| `/api/template/remove` | Delete a Template record. |

### create

`POST /api/template/create` uses `multipart/form-data` with one JSON form field and two file fields:

| Part | Type | Description |
| --- | --- | --- |
| `metadata` | JSON form field | `name` and `source` are required; `plain_http`, `username`, `password`, and `labels` are optional. |
| `kernel` | file | Guest kernel file, at most 64 MiB. |
| `initrd` | file | Initrd file, at most 64 MiB. |

The total request body is limited to 129 MiB. A successful response is:

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

`reference` is required; the rest are optional. A successful response returns the same `status`, `template_name`, and `template_id` as `create`. After a pull, the local Template Name uses the normalized source name returned by the registry. `pull` validates the Boot Index structure; a regular OCI Image does not go through this endpoint.

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

`name` and `remote_reference` are required; the rest are optional. Success returns `{"status":"ok"}`.

### save

```json
{
  "names": ["localhost/conch/openeuler:latest"]
}
```

`names` must contain at least one non-empty, non-duplicate Template Name. On success the response body is an `application/x-tar` archive stream containing the complete Boot Index closure of each Template. A failure before the archive starts returns a regular JSON error; a failure after output has started aborts the connection so the caller does not keep a truncated archive.

### load

The request body is the tar archive produced by `save`; gzip- and zstd-compressed archives are also accepted. A successful response is:

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

`load` creates or updates local Templates by the names stored in the archive; an existing Template with the same name is pointed at the archived Template ID. `load` does not unpack and does not reject archives by host architecture.

### unpack, list, inspect, and remove

`unpack`, `inspect`, and `remove` take `{"name": "<Template Name>"}` and accept only a Template Name. `list` takes `{"origin": "...", "boot_mode": "..."}`; both filter fields are optional and use the values from the data model, and the response is `{"items": [TemplateRecord]}`. `inspect` returns a single TemplateRecord directly; `unpack` and `remove` return `{"status":"ok"}`. `remove` is idempotent: a missing name is also treated as success.

## Offline archive

`template save` exports the Boot Index closure as an OCI image layout tar archive. Template name, origin, lineage, and labels are annotations on the layout index, not part of the Boot Index blob, so the Template ID is unchanged. `template load` imports that archive and creates or updates the named records. Existing pull, push, create, and Sandbox APIs are unchanged by the offline archive.

For CLI operations and Template Name update semantics, see [Template Management](/docs/user/template-management).
