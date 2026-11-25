# 0031. Built-in `autofs_map` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

autofs mounts filesystems on access using a master map and its referenced
maps. The Sous Chefs `autofs` cookbook exposes `map_entry`,
`automaster_entry` and an `nfs` convenience resource. Its `map_entry` uses
`line` to replace a map row and notifies autofs to reload
(`resources/map_entry.rb:33` in that cookbook). Cinc Client's `mount`
manages direct mounts and fstab (`lib/chef/resource/mount.rb:27`).

The shared Chef source has no built-in `autofs_map` resource.

The Ubuntu 26.04 LTS arm64 image runs automount 5.1.9. Map handling,
service reload and actual on-demand mounts remain guest tests.

## Decision

Add `autofs_map` to Cinc Client in the shared Chef codebase. Promote the cookbook's NFS map-entry API as `autofs_map`. Manage one row of an existing local direct or indirect map. Package, service and master-map provisioning remain separate.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Create or reconcile one persistent NFS map entry. |
| `:delete` |  | Remove the resource-owned map entry. |
| `:reload` |  | Tell the running autofs service to reload its maps. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `key` | String | Resource name for file actions | One literal key for an indirect map or an absolute path for a direct map. Reject wildcards and escapes. | Map entry identity. |
| `map` | String | Required for file actions | Existing regular local map under `/etc` with a verified master-map reference. | File to edit. |
| `mount_point` | String | Required for file actions | Exact master-map mount point. Use `'/-'` for a direct map. | Associated master-map entry. |
| `location` | String | Required for create | One hostname or IPv4 literal plus an absolute NFS export path. | Server and export. |
| `fstype` | String | `'nfs4'` | `'nfs'` or `'nfs4'`. | Preserve the cookbook's NFS type input. |
| `options` | String | Unset | Validated comma-separated NFS mount options. Reject contradictory flags and whitespace. | Per-entry mount options. |
| `adopt_existing` | Boolean | `false` | Explicit opt-in for one exact matching row. | Adopt a cookbook-created entry in place. |

### Current state and action steps

- Verify the effective master-map reference and map type. Parse the
  existing map, compare the key and effective NFS options, and reject
  duplicate or conflicting rows. Preserve other rows and file metadata.
- `:create` stages a validated row with an adjacent ownership marker.
  Adopt an exact existing row only with explicit consent. `:delete` removes
  the marked entry. File operations are atomic and leave active mounts in
  place.
- `:reload` verifies that the packaged service supports reload and calls
  it with a timeout. Multiple changed rows can notify one reload. Verify
  lookup on a fresh path separately from the service acknowledgement.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Master-map reference missing or unsupported | Inspect effective configuration and local map type. | Fail before writing. |
| Duplicate key, unmanaged row or unsafe path | Parse the map and verify ownership before touching the file. | Report the conflict. |
| Reload unsupported or timed out | Check service capability and command result. | Preserve file changes for a retry. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 family | Proposed for packaged autofs and local NFS map files. |
| Ubuntu 26.10 and Debian 12 LTS | Smoke and compatibility coverage for packaged map tools. |

See the [platform baseline](../../README.md#platform-baseline) for images and client availability.

### Target mode

Defer until the remote map, packaged service and validation checks use the target consistently.

### Cookbook migration

Translate `map_entry` declarations to `autofs_map` and retain their
`key`, `map`, `location`, `fstype` and `options`. Supply `mount_point`
explicitly because the existing cookbook can infer it from the filename.
For the cookbook's `nfs` wrapper, use the original mount path as the entry
key, set `mount_point '/-'` for its default direct map, and construct
`location` from the previous `server` and `export`. Preserve a custom
`location` as supplied.

Keep `automaster_entry` or equivalent provisioning for the master map,
package and service. Replace `map_entry`'s implicit reload with a delayed
notification to the built-in `:reload`. Adopt the exact existing map row
with `adopt_existing true` at cutover, then remove that property and the
old `map_entry` or `nfs` declaration. Compare the resolved map and a fresh
mount after switching. Keep the cookbook for unsupported map types.

## Usage examples

### :create

```ruby
autofs_map 'projects' do
  map '/etc/auto.projects'
  mount_point '/projects'
  location 'nfs.example.net:/srv/projects'
  options 'ro'
  action :create
end
```

### :delete

```ruby
autofs_map 'projects' do
  map '/etc/auto.projects'
  mount_point '/projects'
  action :delete
end
```

### :reload

```ruby
autofs_map 'projects' do
  action :reload
end
```

## End-to-end testing

Migrate both direct and indirect cookbook entries on disposable guests.
Verify first access mounts the same export, unrelated keys still resolve,
reload applies a changed entry, and deletion affects future lookups.
Check malformed rows, duplicate keys and repeats without changes.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for map parsing and adoption fixtures. |
| kitchen-exec | Limited to an isolated runner with kernel autofs and a test NFS server. |
| Vagrant | Preferred for real on-demand mounts and service reload. |

Run parser fixtures in the baseline containers. Use separate NFS server and client guests for lookup and live migration tests.

## Consequences

### Positive

Existing cookbook users can move individual NFS entries into core while keeping their master-map provisioning.

### Negative

Changes to an active mount can take effect on a later lookup or expiry. The initial backend covers local NFS maps only.

### Follow-up

Validate master-map discovery, map escape handling and service reload on each target. Check direct-map expiry behaviour during adoption.

## Alternatives considered

Continuing with the cookbook preserves its convenience wrapper and implicit provisioning. Promoting the row resource makes map ownership and activation explicit.

## References

- [Sous Chefs autofs cookbook API](https://github.com/sous-chefs/autofs/blob/674b793/README.md)
- [Sous Chefs map_entry implementation](https://github.com/sous-chefs/autofs/blob/674b793/resources/map_entry.rb)
- [Linux auto.master format](https://man7.org/linux/man-pages/man5/auto.master.5.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
