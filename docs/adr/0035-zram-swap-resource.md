# 0035. Built-in `zram_swap` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Linux zram exposes compressed RAM-backed block devices. The systemd
zram generator configures a device and emits swap units from declarative
configuration. Cinc Client has `swap_file` for ordinary backing files
(`lib/chef/resource/swap_file.rb:24`).

The shared Chef source has no built-in `zram_swap` resource.

Ubuntu 26.04 LTS arm64 installs the generator from
`systemd-zram-generator` 1.2.1. Generated units and swap activation
remain guest tests.

## Decision

Add `zram_swap` to Cinc Client in the shared Chef codebase. Manage persistent configuration for one zram swap device on Ubuntu and Debian. Stage changes for the next boot and leave live swap lifecycle with the operator.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Validate and stage the generator fragment. |
| `:delete` |  | Remove the managed generator fragment. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Basename in `90-chef-NAME.conf`. |
| `device` | String | `'zram0'` | `zram` followed by a canonical decimal index. Refuse another generator section for the same device. | Device to configure as swap. |
| `size_mib` | Integer | Required for create | Positive MiB size within the generator's accepted range. | Logical device size, distinct from compressed RAM consumption. |
| `swap_priority` | Integer | `100` | 0 to 32767. | Linux swap selection priority. |

### Current state and action steps

- Verify a packaged zram generator and check its effective configuration
  for competing ownership of the selected device. Confirm zram is available
  on the native guest kernel before claiming runtime support.
- `:create` renders the selected device section with a numeric size and
  priority, then atomically installs a marked `0644` fragment under
  `/etc/systemd/zram-generator.conf.d` only when bytes or metadata differ.
- `:delete` removes only the marked fragment. Both actions leave current
  swap devices active. The boot test verifies which units the generator
  creates and whether removing policy suppresses future activation.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Generator or zram unavailable | Package and native kernel capability checks. | Report the missing prerequisite. |
| Competing zram policy | Check effective device sections and another active manager. | Fail before editing. |
| Unmanaged destination | Symlink, special file or missing marker. | Keep the existing path. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS and Debian 13 | Proposed with systemd-zram-generator and a zram-capable kernel. Ubuntu requires Universe. |
| Ubuntu 26.10 and Debian 12 LTS | Utility and generator smoke or compatibility checks where packaged. |
| RHEL 10 family | Deferred pending a supported packaged generator path. |

See the [platform baseline](../../README.md#platform-baseline) for images and client availability.

### Target mode

Defer until remote kernel checks and generator discovery have SSH coverage.

## Usage examples

### :create

```ruby
zram_swap 'worker-swap' do
  device 'zram0'
  size_mib 512
  swap_priority 100
  action :create
end
```

### :delete

```ruby
zram_swap 'worker-swap' do
  device 'zram0'
  action :delete
end
```

## End-to-end testing

Generate units on each distro and boot a disposable guest. Verify the
zram swap device, requested logical size and priority, repeat converge,
coexistence with disk swap, and removal after a later reboot.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for configuration and generator userspace checks. |
| kitchen-exec | Limited by runner kernel module availability. |
| Vagrant | Preferred for boot, zram kernel and swap lifecycle tests. |

Run userspace fixtures on Ubuntu and Debian. Use native guests for boot and swap assertions.

## Consequences

### Positive

Recipes can stage compressed swap without writing kernel commands or configuring a swap file.

### Negative

Changing the fragment defers live swap changes to a boot or maintenance workflow.

### Follow-up

Verify size-unit handling and supported generator drop-in paths on both distro families. Test memory-constrained guest behaviour.

## Alternatives considered

A direct zramctl and swapon recipe can configure a current session. The generator handles persistent boot integration and service ordering.

## References

- [systemd zram-generator](https://github.com/systemd/zram-generator)
- [Ubuntu 26.04 package](https://packages.ubuntu.com/resolute/systemd-zram-generator)
- [Debian 13 package](https://packages.debian.org/trixie/systemd-zram-generator)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
