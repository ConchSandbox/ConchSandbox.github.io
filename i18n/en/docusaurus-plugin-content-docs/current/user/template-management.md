---
title: Template Management
---

# Template Management

A Template is the boot object that Conch uses to create a Sandbox. Each Template has a mutable `Template Name` and an immutable `Template ID`:

| Concept | Example | Purpose |
| --- | --- | --- |
| Template Name | `localhost/conch/python:latest` | Creates, queries, publishes, unpacks, and deletes a Template. Writing the same name updates its target. |
| Template ID | `sha256:...` | The OCI Boot Index digest. It identifies immutable content precisely and can also create a Sandbox directly. |

A Template Boot Index contains an EROFS rootfs, guest kernel, and initrd. A resume Template created by a checkpoint also contains VMM memory state.

This page aligns with the `dev-openEuler-24.03-LTS_SP4` branch of the Conch source repository.

## Create a Template from an OCI Image

```bash
sudo ./bin/conch template create \
  --config config/config.local.yaml \
  --name localhost/conch/python:latest \
  --source docker.io/library/python:3.12 \
  --kernel /var/lib/conch/kernel \
  --initrd /var/lib/conch/conch.initrd
```

`--name`, `--source`, `--kernel`, and `--initrd` are all required. Conch performs these steps in order:

1. Looks for `--source` in the containerd embedded in conchd and pulls it from the registry if absent.
2. Verifies that the source is a regular OCI Image rather than a Boot Index.
3. Converts the rootfs to EROFS.
4. Combines it with the kernel and initrd into a cold Boot Index.
5. Points `--name` to the generated Template ID.

Example success output:

```text
Template Name: localhost/conch/python:latest
Template ID: sha256:0123456789abcdef...
```

For a private registry, use `--user username:password` or pass `--username` and `--password` separately. Pass `--plain-http` only for a registry that explicitly uses HTTP.

## Query and filter Templates

List every Template:

```bash
sudo ./bin/conch template ls --config config/config.local.yaml
```

Filter by origin or boot mode:

```bash
sudo ./bin/conch template ls \
  --config config/config.local.yaml \
  --origin checkpoint \
  --boot-mode resume
```

Allowed filter values:

- `origin`: `image` or `checkpoint`.
- `boot-mode`: `cold` or `resume`.

Inspect a name:

```bash
sudo ./bin/conch template inspect \
  --config config/config.local.yaml \
  localhost/conch/python:latest
```

The output columns are `NAME`, `TEMPLATE_ID`, `ORIGIN`, `BOOT_MODE`, `SOURCE_REF`, and `SOURCE_SANDBOX`.

## Template Name update semantics

A Template Name maps to an internal containerd image record. Running `template create`, `template pull`, or checkpoint again with the same name updates the Boot Index targeted by that record. An old Template ID that has already been obtained does not change.

Therefore:

- Use a Template Name for routine Sandbox creation when it should follow the name's latest content.
- Store and use a Template ID for reproducible workloads.
- Updating a name does not modify running Sandboxes.

## Publish and pull

When publishing a local Template, the first positional argument is the local Template Name and the second is the remote registry reference:

```bash
sudo ./bin/conch template push \
  --config config/config.local.yaml \
  localhost/conch/python:latest \
  registry.example.com/conch/python:latest
```

Pull a remote Boot Index:

```bash
sudo ./bin/conch template pull \
  --config config/config.local.yaml \
  registry.example.com/conch/python:latest
```

After a successful pull, the local Template Name uses the normalized source name returned by the registry, and the command also prints the Template ID. `template pull` validates the Boot Index structure. Use `conch image pull` for a regular OCI Image.

## Move Templates offline

Without a registry, move Templates offline with `save` / `load`. As with `docker save` / `docker load`, the archive contains the complete Boot Index, not a flat export of the container filesystem:

```bash
sudo ./bin/conch template save \
  --config config/config.local.yaml \
  -o openeuler.tar localhost/conch/openeuler:latest

sudo ./bin/conch template load \
  --config config/config.local.yaml \
  -i openeuler.tar
```

### save

The command takes the form `conch template save [options] <template-name> [<template-name>...]` and exports one or more Templates at a time. The archive contains the Boot Index and every blob needed to boot it:

```bash
sudo ./bin/conch template save \
  --config config/config.local.yaml \
  -o bundle.tar \
  localhost/conch/openeuler:latest \
  localhost/conch/python:latest
```

| Option | Description |
| --- | --- |
| `-o, --output` | Required. The archive file to write; the path may be relative. |
| `--timeout` | Caps this archive transfer, for example `30m`; unset means no limit. |
| `--config` | Config file path; common paths are auto-detected by default. |

`save` writes the archive to a temporary file in the same directory first and only replaces the file named by `-o` after success; on failure the original file is kept, and parent directories are not created.

### load

The command takes the form `conch template load [options]`; `-i` is required and no positional arguments are accepted:

| Option | Description |
| --- | --- |
| `-i, --input` | Required. The archive file to load. |
| `--timeout` | Caps this archive transfer, for example `30m`; unset means no limit. |
| `--config` | Config file path; common paths are auto-detected by default. |

`load` accepts gzip- or zstd-compressed archives, creates or updates local Templates by the names stored in the archive, and prints `Loaded image: <name>` for each entry. An existing Template with the same name is pointed at the archived Template ID. `load` does not unpack and does not reject archives by host architecture.

## Unpack a Template

Sandbox creation prepares Boot Index components on demand. To unpack all components of a Template into the local snapshotter in advance, run:

```bash
sudo ./bin/conch template unpack \
  --config config/config.local.yaml \
  localhost/conch/python:latest
```

This command accepts a Template Name, not a Template ID.

## Delete a Template

```bash
sudo ./bin/conch template rm \
  --config config/config.local.yaml \
  localhost/conch/python:latest
```

Deletion uses the Template Name and is idempotent: a missing name is also considered successful. The operation removes only the Template's internal image record. Content no longer referenced by another image record or GC root is reclaimed by containerd GC.

## Manage Images

Use `conch image` for regular OCI Images:

```bash
sudo ./bin/conch image pull docker.io/library/python:3.12
sudo ./bin/conch image ls
sudo ./bin/conch image rm docker.io/library/python:3.12
```

By default, `image ls` hides Templates and the internal records used by Boot Index components. Use `conch image ls --all` for diagnostics. `image rm` does not delete Templates; use `template rm` instead.

## Request timeouts

Except for `template save` and `template load`, CLI requests to conchd have a default timeout of 2 minutes. For a large image pull, conversion, or publication, set a positive Go duration for the current command:

```bash
CONCH_API_TIMEOUT=30m sudo --preserve-env=CONCH_API_TIMEOUT \
  ./bin/conch template create \
  --config config/config.local.yaml \
  --name localhost/conch/python:latest \
  --source docker.io/library/python:3.12 \
  --kernel /var/lib/conch/kernel \
  --initrd /var/lib/conch/conch.initrd
```

`template push --timeout 30m` and `image push --timeout 30m` override only that push request and take precedence over `CONCH_API_TIMEOUT`. `template save` and `template load` have no default time limit; use `--timeout` when a cap is needed, which also takes precedence over `CONCH_API_TIMEOUT`.

## Related documentation

- [Create and Delete Sandboxes](/docs/user/sandbox-lifecycle)
- [Snapshot Startup](/docs/user/snapshot-start)
- [Template Reference](/docs/reference/template)
