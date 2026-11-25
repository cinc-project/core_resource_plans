# 0026. Built-in `systemd_resolved` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

systemd-resolved combines system-wide DNS policy with per-link settings.
ADR-0014 proposes glibc lookup order management. The resolved daemon's
DNS servers and fallback policy live in `resolved.conf.d`, and Cinc Client
currently supplies generic `file` and `service` resources for those changes
(`lib/chef/resource/file.rb:32` and `lib/chef/resource/service.rb:31`).

The shared Chef source has no built-in `systemd_resolved` resource.

On Ubuntu 24.04 with systemd 255.4-1ubuntu8.19, `systemd-analyze
cat-config` showed a test `FallbackDNS` setting in the merged
configuration. Resolver selection and actual lookups remain guest tests.

## Decision

Add `systemd_resolved` to Cinc Client in the shared Chef codebase. Manage one global systemd-resolved DNS fragment. Per-link DNS, routing domains and `/etc/resolv.conf` ownership stay with the active resolver and network manager.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Install the global resolver fragment. |
| `:delete` |  | Remove the managed resolver fragment. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Basename in `90-chef-NAME.conf`. |
| `dns`, `fallback_dns` | Array of String | Unset | Nonempty ordered list of unique IPv4 or IPv6 literals. Reject scoped addresses, ports and TLS names. | System DNS endpoints and separate last-resort DNS endpoints. |

### Current state and action steps

- Confirm the host uses systemd-resolved and require at least one list.
  Render a deterministic `[Resolve]` fragment with the configured
  assignments. The first release contributes settings without emitting
  an empty assignment that resets other fragments' lists.
- `:create` compares content and metadata and atomically installs a marked
  `0644` root-owned fragment. `:delete` removes only the marked fragment.
- Keep activation explicit through the existing `service` resource.
  A restart can interrupt lookups. Test effective global and per-link
  configuration after activating in a disposable guest.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Unsupported resolver | Service and distro discovery show a different implementation. | Fail before changing the destination. |
| Invalid server or unmanaged destination | Address validation, symlink, special file or absent marker. | Keep the existing fragment. |
| Configuration shadowed or reset | Effective fragment inspection identifies a later reset. | Report the competing path when detected. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04 and Debian 12 | Proposed where systemd-resolved is installed and selected. |
| Network managers with different resolvers | Deferred. |

### Target mode

Defer until the fragment checks work under SSH and restart tests cover resolver continuity on the target.

## Usage examples

### :create

```ruby
systemd_resolved 'site-resolver' do
  fallback_dns ['192.0.2.53']
  action :create
end
```

### :delete

```ruby
systemd_resolved 'site-resolver' do
  action :delete
end
```

## End-to-end testing

Check fragment composition and no-op second converge. In a disposable
guest, compare `resolvectl status` before and after activation and removal.
Use a controlled DNS fixture to check fallback and per-link settings.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for rendering and fragment ordering. |
| kitchen-exec | Possible with a dedicated test resolver on a disposable runner. |
| Vagrant | Preferred for live lookup and coexistence with network-provided DNS. |

Run file and ordering checks in Ubuntu and Debian containers, then test effective settings in a guest with a private DNS fixture.

## Consequences

### Positive

Cookbooks can express global DNS policy without owning per-link configuration or the resolver symlink.

### Negative

A fallback server is only used under resolved's fallback conditions. The resource does not promise that an endpoint is reachable.

### Follow-up

Verify list accumulation and resets on supported versions and cover fallback selection with controlled per-link inputs.

## Alternatives considered

Templating `/etc/resolv.conf` works for a traditional resolver. On resolved systems, a drop-in preserves the daemon's routing and manager integration.

## References

- [systemd resolved.conf and precedence](https://www.freedesktop.org/software/systemd/man/latest/resolved.conf.html)
- [resolvectl output and diagnostics](https://www.freedesktop.org/software/systemd/man/latest/resolvectl.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
