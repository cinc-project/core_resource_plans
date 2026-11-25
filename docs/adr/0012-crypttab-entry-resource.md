# 0012. Built-in `crypttab_entry` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

On systemd Linux, the cryptsetup generator reads `/etc/crypttab` and
creates units for encrypted mappings. Cinc Client's `mount` resource
manages filesystems (`lib/chef/resource/mount.rb:27`). Preparing the
mapping that a mount depends on is a separate step.

The shared Chef source has no built-in `crypttab_entry` resource.

On Ubuntu 24.04 with systemd 255.4-1ubuntu8.19 and cryptsetup 2.7.0,
a sample `luks,noauto` entry generated a cryptsetup service unit in a
private output directory.

## Decision

Add `crypttab_entry` to Cinc Client in the shared Chef codebase. Manage one non-root LUKS mapping declaration in `/etc/crypttab`. Leave disk formatting, key enrolment, initramfs regeneration and mapping activation to their existing workflows.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Ensure one matching declaration exists. |
| `:delete` |  | Remove the resource's marked declaration. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | Literal mapper name with ASCII letters, digits, hyphen or underscore. Reject root and swap mappings during platform checks. | Mapping identifier. |
| `device_uuid` | String | Required for create | Canonical UUID syntax. | LUKS source UUID. |
| `auto_open` | Boolean | `false` | Boolean. | Omitting `noauto` allows normal systemd boot processing. |

### Current state and action steps

- Parse existing active lines and comments under a local lock. Recognise a
  managed entry by its exact name and a separate adjacent Chef marker line.
  Preserve unrelated lines byte-for-byte.
- `:create` writes `NAME UUID=UUID none luks,noauto` by default. The `none`
  key field defers key acquisition to the normal boot environment. Compare
  parsed values, then atomically replace the file with its original metadata.
- `:delete` removes the marked pair if present. Replacing or removing a line
  changes future generator runs without closing a live mapping.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Duplicate mapping or unmanaged matching name | Parse active lines before rewriting. | Fail and report the conflict. |
| Unsupported target | Mapping backs root, swap or an initramfs-dependent volume. | Fail before editing. |
| Malformed file or unsafe destination | Parser error, symlink or failed atomic replacement. | Preserve the existing file and report the problem. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04, Debian 12 and RHEL 9 | Proposed for systemd with cryptsetup generator support and a regular local crypttab file. |
| Initramfs and non-systemd setups | Deferred. |

### Target mode

Defer until the remote boot-mapping checks and single-file transaction are covered.

## Usage examples

### :create

```ruby
crypttab_entry 'worker-data' do
  device_uuid '52b1d2e4-be56-469c-9c62-3b594ac7582b'
  auto_open false
  action :create
end
```

### :delete

```ruby
crypttab_entry 'worker-data' do
  action :delete
end
```

## End-to-end testing

Use a throwaway loopback LUKS device on a guest. Check generator output,
normal mount ordering, `noauto`, idempotence and removal. Keep a sentinel
entry and verify that editing one declaration preserves it.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for file parsing and generator fixtures. |
| kitchen-exec | Possible for isolated loopback smoke cases. |
| Vagrant | Preferred for reboot, encrypted device and mount-ordering assertions. |

Run parser and generator fixtures in Docker. Gate release on a disposable rebootable guest with an encrypted test volume.

## Consequences

### Positive

Cookbooks can manage one mapping without taking ownership of other crypttab entries or holding key material.

### Negative

Live mappings remain active until their owner changes them. The design covers only a simple LUKS configuration.

### Follow-up

Check UUID and ownership behaviour on all target distros and verify the root, swap and initramfs guard against real layouts.

## Alternatives considered

A whole-file template gives an image build complete control. Per-entry management fits nodes where packages or installers own other mappings.

## References

- [systemd crypttab syntax](https://www.freedesktop.org/software/systemd/man/latest/crypttab.html)
- [systemd cryptsetup generator](https://www.freedesktop.org/software/systemd/man/latest/systemd-cryptsetup-generator.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
