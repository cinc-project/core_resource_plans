# 0024. Built-in `nftables_table` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

nftables groups live policy into tables and applies command batches through
netlink. Persistence is configured by the distribution's loader and site
includes. Cinc Client can install a fragment with `file` or invoke a
command with `execute` (`lib/chef/resource/file.rb:32` and
`lib/chef/resource/execute.rb:27`). These resources do not model table
ownership or a validated table-scoped update.

The shared Chef source has no built-in `nftables_table` resource.

On Ubuntu 24.04 with nftables 1.0.9 and a private network namespace,
`nft --check` accepted a candidate without installing it. Applying,
replacing and removing the marked table left a second test table intact.

## Decision

Add `nftables_table` to Cinc Client in the shared Chef codebase. Manage one dedicated `inet` table with an input drop list for IPv4 and IPv6 source CIDRs. Use a fixed input hook priority of 10 and an accept policy. The generated rules drop only the declared sources. Other tables retain their own policies.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Validate and install the persistent table fragment. |
| `:apply` |  | Apply the managed on-disk table to live state. |
| `:delete` |  | Remove the persistent fragment. |
| `:unload` |  | Remove only the marked table from live state. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens, with unique mapping to a valid identifier. | Table and fragment identity. Prefix the generated table with `chef_`. |
| `blocked_sources` | Array of String | Required for create | Nonempty IPv4/IPv6 CIDRs with valid prefixes. Canonicalise network bits and remove duplicates. Reject prefix length zero. | Source networks dropped by the managed table. |

### Current state and action steps

- Confirm the site's loader includes `/etc/nftables.d/chef-NAME.nft`.
  Render a deterministic fragment containing the table marker, input chain
  and separate address-family sets. Compare managed file bytes and
  metadata. Stage changes privately. Check the exact candidate table
  transaction with `nft --check` before installing the marked `0640`
  root-owned fragment. The fragment contains no global flush.
- `:apply` reads the installed managed fragment and reconstructs the
  expected table from the supported format. Reject additional commands.
  Read JSON for the live table and check its ownership marker. Ignore
  rule handles and transient counters when comparing state. If different,
  check and apply one batch that deletes the prior owned table when
  present and creates its replacement. Verify the named table afterwards.
- `:delete` removes only the marked file. `:unload` deletes only the
  marked live table, and is a no-op if absent. Use both for complete removal.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Unmanaged table, unsafe file or ambiguous identifier | Inspect exact destination and ownership marker. | Fail before mutation. |
| Invalid candidate or unavailable netlink | Failed check or capability preflight. | Report the diagnostic and retain the installed policy. |
| State changes concurrently or batch fails | Check current table before the batch and re-read afterwards. | Report current state and allow a fresh retry. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04, Debian 12 and RHEL 9 | Proposed with compatible nft and a site-owned loader include. Check coexistence with any firewall manager. |
| Other firewalls without table-level coordination | Deferred. |

### Target mode

Defer until SSH regression tests cover both file updates and applying policy without disconnecting the management route.

## Usage examples

### :create

```ruby
nftables_table 'worker-input' do
  blocked_sources ['192.0.2.0/24']
  action :create
end
```

### :apply

```ruby
nftables_table 'worker-input' do
  action :apply
end
```

### :delete

```ruby
nftables_table 'worker-input' do
  action :delete
end
```

### :unload

```ruby
nftables_table 'worker-input' do
  action :unload
end
```

## End-to-end testing

Use a private network namespace or guest. Verify staged and live states,
IPv4 and IPv6 drop behaviour, preservation of another table, rejection of
an invalid batch, repeat convergence and removal. Check boot persistence.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for files. Live checks require a private namespace with CAP_NET_ADMIN. |
| kitchen-exec | Possible in a private test namespace on a Linux runner. |
| Vagrant | Preferred for full traffic and boot-loader integration tests. |

Run render fixtures in Docker. Run traffic and batch checks in a private namespace and a guest reboot case for the loader.

## Consequences

### Positive

A recipe can own a small explicit firewall table without replacing the rest of the ruleset.

### Negative

The initial interface is a source drop list only. A bad source selection can block legitimate traffic, so activation is explicit.

### Follow-up

Verify check and replace semantics and table comments on all target versions. Test coexistence with distro firewall managers before supporting mixed ownership.

## Alternatives considered

Templating and reloading the full ruleset is appropriate on a dedicated firewall with one owner. A table-scoped transaction gives other components separate policy ownership.

## References

- [nftables upstream manual](https://netfilter.org/projects/nftables/manpage.html)
- [Netfilter configuration and batching](https://wiki.nftables.org/wiki-nftables/index.php/Atomic_rule_replacement)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
