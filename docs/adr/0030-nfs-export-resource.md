# 0030. Built-in `nfs_export` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Linux NFS servers read `/etc/exports` and their supported export fragments.
The Sous Chefs `nfs` cookbook already provides `nfs_export`. It uses
`line` to edit the configured exports file and calls `exportfs -ar` on
changes (`resources/nfs_export.rb:36` and `:44` in that cookbook).
Cinc Client ships `mount` for clients (`lib/chef/resource/mount.rb:27`).

The shared Chef source has no built-in `nfs_export` resource.

Ubuntu 26.04 LTS arm64 provides `exportfs` in nfs-kernel-server 2.8.5.
The live export and cookbook migration checks require an NFS server guest.

## Decision

Add `nfs_export` to Cinc Client in the shared Chef codebase. Promote the cookbook's export declaration into Cinc Client. Manage one export with an explicit client set, read-only and root-squash defaults, and a separate activation action. Server installation and service management stay in `nfs_server` or the site's provisioning.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Stage or reconcile a persistent export. |
| `:delete` |  | Remove the resource-owned export from disk. |
| `:reload` |  | Reconcile the live export table from the site's configuration. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `directory` | String | Resource name for file actions | Absolute existing local directory. Reject ambiguous escapes and unsafe or unsupported paths. | Export path. |
| `network` | String, Array of String | Required for create | Explicit hostnames, IP literals or CIDRs. Reject wildcards, netgroups and duplicates. | Authorised clients sharing the options. |
| `writeable`, `sync` | Boolean | `false`, `true` | Boolean. | Preserve the cookbook spelling and map to ro/rw and sync/async. |
| `options` | Array of String | `['root_squash']` | Validated supported export options. Reject contradictions and options that disable root squashing. | Additional per-client flags. |
| `anonuser`, `anongroup` | String | Unset | Existing local account or group. | Resolve to numeric `anonuid` and `anongid`. |
| `fsid` | String | Unset | Explicit numeric ID or `root` when the server policy supports it. Refuse a conflicting ID. | Export identifier override. |
| `exports_file` | String | Generated file in `/etc/exports.d` | `/etc/exports` or an absolute `.exports` file under `/etc/exports.d`. | Loaded exports file, including an in-place migration path. |
| `adopt_existing` | Boolean | `false` | Explicit opt-in. Requires one exact, unambiguous equivalent row. | Take ownership of a legacy row without adding a second export. |

### Current state and action steps

- Inspect the effective exports configuration and parse path, client and
  option tuples. Compare the resource-owned row and check for overlapping
  or contradictory unmanaged exports. Own the row with a separate adjacent
  marker. For new exports, use a collision-resistant fragment name derived
  from the directory.
- `:create` validates the supported fields and stages the row atomically.
  Preserve other rows and metadata in an existing file. Exact adoption
  adds the marker without changing effective export text. `:delete` removes
  only the owned row and marker. Repeated file actions are idempotent.
- `:reload` runs `exportfs -ra` with a timeout and checks the effective
  export table. It reconciles all server exports, including those owned by
  other components. Use one delayed notification for a set of changes.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Unsupported or ambiguous clients or options | Check parsed tuples, overlapping scopes and export identity. | Fail before publishing. |
| Unmanaged, duplicate or mismatched legacy row | Compare loaded rows and adoption criteria. | Fail with the path and conflicting entry. |
| Server or reload unavailable | Verify prerequisites and command result. | Preserve staged configuration and report the error for retry. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 family | Proposed with the local NFS server utilities and supported exports loader. |
| Ubuntu 26.10 and Debian 12 LTS | Smoke and compatibility checks for parser and loader behaviour. |

See the [platform baseline](../../README.md#platform-baseline) for images and client availability.

### Target mode

Defer until target-side export discovery, atomic edits and server activation are verified.

### Cookbook migration

Start with the cookbook's `:create` and `:delete` actions and its
`directory`, `network`, `writeable`, `sync`, `options`, `anonuser`,
`anongroup` and `fsid` inputs. Set `fsid` explicitly if the old cookbook
inserted a root ID implicitly on the previous platform. Translate a supported
legacy `exports_file` to the same loaded file and set `adopt_existing true`
for the first converge. Compare `exportfs -v` before switching recipe
ownership. Remove the opt-in after the row is marked.

Replace the old immediate reload with a notification to `:reload`. Retain
`nfs_server` if the cookbook provisions the host. Verify resource resolution
before cutover because both definitions use `nfs_export`. Use a cookbook
version that retires the older declaration while retaining server resources,
or move server setup into independent provisioning before dropping the
cookbook dependency. Keep using the cookbook for unsupported
wildcards or options until those policies are redesigned. Review recipes using
`unique true` or broad deletion and consolidate or remove conflicting legacy
rows explicitly before adopting them.

## Usage examples

### :create

```ruby
nfs_export '/srv/projects' do
  network %w(192.0.2.0/24 2001:db8:42::/64)
  writeable false
  action :create
end
```

### :delete

```ruby
nfs_export '/srv/projects' do
  action :delete
end
```

### :reload

```ruby
nfs_export '/srv/projects' do
  action :reload
end
```

## End-to-end testing

Migrate a cookbook-managed row in place and compare exported clients,
flags and access before and after cutover. Cover a denied client, read-only
access, duplicate detection, repeated converge, activation and removal.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for row parsing and migration fixtures. |
| kitchen-exec | Possible with an isolated guest kernel that supports NFS server tests. |
| Vagrant | Preferred for real NFS client/server access and reboot coverage. |

Use the baseline containers for parser fixtures. Run a two-guest client and server suite for active exports and migration equivalence.

## Consequences

### Positive

Existing recipes have a migration path to built-in export lifecycle management without giving up other exports in the same file.

### Negative

A mixed cookbook and built-in deployment needs careful resource resolution. Activation reconciles the complete export table.

### Follow-up

Verify supported export options, server start ordering and overlap diagnostics on each packaged server. Confirm action equivalence on live NFSv4 fixtures.

## Alternatives considered

Keeping the cookbook works where recipes already depend on it. Promotion removes the requirement to bring the line-editing dependency for a standalone export and gives recipes explicit activation.

## References

- [Sous Chefs nfs_export documentation](https://github.com/sous-chefs/nfs/blob/54145f2/documentation/nfs_export.md)
- [Sous Chefs nfs_export implementation](https://github.com/sous-chefs/nfs/blob/54145f2/resources/nfs_export.rb)
- [Linux exports format](https://man7.org/linux/man-pages/man5/exports.5.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
