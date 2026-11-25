# 0013. Built-in `hosts_entry` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

The local hosts table supplies static name and address mappings to the
NSS files backend. Cinc Client ships `hostname` for the node's name
(`lib/chef/resource/hostname.rb:27`). Managing a mapping today requires
editing or templating `/etc/hosts`.

The shared Chef source has no built-in `hosts_entry` resource.

On Ubuntu 24.04 with glibc 2.39, `getent -s files hosts` resolved a test
name from a new entry with an inline comment.

## Decision

Add `hosts_entry` to Cinc Client in the shared Chef codebase. Own one marked static mapping in `/etc/hosts`. Preserve entries supplied by the installer, cloud tooling and other declarations.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Create or reconcile the marked mapping. |
| `:delete` |  | Remove the marked mapping for the canonical name. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII hostname, validated per label. Compare case-insensitively. | Canonical hostname and entry identity. |
| `address` | String | Required for create | IPv4 or IPv6 literal without prefix or scope. | Static address. |
| `aliases` | Array of String | Empty | Unique valid names distinct from the canonical name. | Alternate names on the same line. |

### Current state and action steps

- Lock and parse the file as address, canonical name, aliases and optional
  comment. Locate a row using the Chef marker with the canonical name and
  check for collisions with other canonical names or aliases. Existing rows
  can use the same address with different names.
- `:create` compares the normalised address and requested names. Add or
  replace only the marked row and preserve other bytes and file metadata.
- `:delete` removes only the marked row if present. Verify the file after
  atomic replacement. Resolver behaviour is covered by integration tests.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Name collision | Compare canonical names and aliases case-insensitively. | Fail and identify the conflicting row. |
| Unsafe file layout | Unexpected symlink, malformed content or bind mount that rejects rename. | Fail without in-place fallbacks. |
| Duplicate managed rows | More than one file row carries the marker. | Fail and report the conflicting line numbers. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 | Proposed for regular writable hosts files using glibc NSS. |
| Container-managed hosts file or cloud-init rewrite | Keep ownership with the system that regenerates the file. |

Use the [platform baseline](../../README.md#platform-baseline) for Ubuntu 26.10 smoke tests and
Debian 12 LTS compatibility checks when the required tools are available.

### Target mode

Defer until SSH coverage proves atomic read-modify-write and file metadata preservation.

## Usage examples

### :create

```ruby
hosts_entry 'worker.internal' do
  address '192.0.2.50'
  aliases ['worker']
  action :create
end
```

### :delete

```ruby
hosts_entry 'worker.internal' do
  action :delete
end
```

## End-to-end testing

Resolve test names through the files service with `getent`. Check IPv4,
IPv6, aliases, conflicting unmanaged rows, removal and second converge.
Verify that loopback, installer entries and comments match their input bytes.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for file fixtures. Mount a separate writable test filesystem for real hosts-file replacement. |
| kitchen-exec | Good for native NSS checks on a disposable runner. |
| Vagrant | Useful for cloud-init persistence and reboot cases. |

Run parser fixtures on Ubuntu and EL. Verify real lookup and atomic replacement on a disposable guest.

## Consequences

### Positive

Cookbooks can declare local mappings without recreating the rest of the hosts table.

### Negative

The first version owns a single address per canonical name. Dual-stack names with separate entries require a later identity model.

### Follow-up

Verify case and IPv6 handling and whether automated hosts-file managers can delegate specific entries safely.

## Alternatives considered

A whole-file template works where the recipe owns the node's complete hosts data. A line-oriented template replacement lacks structured conflict detection.

## References

- [Linux hosts file format](https://man7.org/linux/man-pages/man5/hosts.5.html)
- [glibc Name Service Switch](https://www.gnu.org/software/libc/manual/html_node/Name-Service-Switch.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
