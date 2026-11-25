# 0006. Built-in `kernel_args` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Cinc Client ships `sysctl` for live kernel tunables and their persistent
`sysctl.d` files (see `lib/chef/resource/sysctl.rb:24` and
`lib/chef/resource/sysctl.rb:135`). Boot parameters come from the boot
loader and take effect when an entry boots. The shared Chef resource
inventory has no boot-parameter resource. ADR-0001 identifies
`kernel_args` as the expected owner of kernel command-line edits.

RHEL 9 documents using `grubby` to update the kernel parameters stored in
GRUB Boot Loader Specification (BLS) entries. `/proc/cmdline` reports the
arguments used for the current boot. A staged change can be correct while
the running command line still shows the previous value.

In an AlmaLinux 9 arm64 container with grubby 8.40-69.el9, a synthetic BLS
entry changed from `loglevel=4` to `loglevel=5`. Removing the key left
its unrelated arguments in place. This exercised file edits without
booting the entry.

## Decision

Add `kernel_args` to Cinc Client in the shared Chef codebase. Manage one
single-valued kernel parameter across the installed BLS entries on
explicitly supported GRUB systems. Use `grubby` for discovery and editing.
Leave reboot scheduling to recipes.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:set` | Yes | Ensure all managed BLS entries contain the requested flag or value. |
| `:remove` | | Ensure the named parameter is absent from all managed BLS entries. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | Key starts with an ASCII letter or digit. Other characters may be letters, digits, period, underscore or hyphen. Exclude `root`, which grubby reports separately. | Parameter key, matched at the token boundary. |
| `value` | String, nil | `nil` | A nonempty token with simple ASCII letters, digits or `_.,:+/=-`. Reject whitespace, quotes, variable references and backslashes. | Desired value. `nil` means a bare flag. |

### Current state and action steps

1. Verify the OS, active boot setup and path to `grubby`. Under a
   node-local resource lock, read `grubby --info=ALL` and record every BLS
   entry's identity and argument list. Reject ambiguous or unsupported
   layouts, unresolved substitutions and repeated instances of the
   managed key. Defer parameters that rely on an ordered list of values.
2. For `:set`, compare each entry against the exact expected token. If
   every entry matches, succeed without an update. Otherwise pass one
   `--args=KEY[=VALUE]` argument to `grubby --update-kernel=ALL`.
3. For `:remove`, return without updates if the key is absent everywhere.
   Otherwise pass `--remove-args=KEY` to the same update command.
4. Re-read the entries. Verify each original entry still exists, the
   requested state is present and unrelated arguments, including the
   separately reported root device, are preserved.
   Report an incomplete update with the affected entry IDs. The next run
   re-reads each entry to reconcile an interrupted change.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Unrecognised boot setup | Missing utility, absent BLS entries, unsupported boot loader or ambiguous output. | Fail before edits. |
| Ambiguous parameter | Duplicate managed key or unresolvable option expression. | Fail and identify the affected entry. |
| Update fails partway | Nonzero exit or failed postcondition. | Report affected entries and captured diagnostics. Leave retry to a fresh read of disk state. |
| Concurrent boot changes | Entry set or unrelated values changed during editing. | Fail with the observed change. Do not overwrite another writer's changes to attempt rollback. |

### Platform support

| Platform | Support |
|---|---|
| RHEL 9 and compatible distributions with GRUB BLS on x86-64 | Proposed initial support after fixture and reboot validation. |
| RHEL 10, Debian, Ubuntu, SUSE, systemd-boot and zipl | Deferred until their boot setup and backend contracts are tested. |

### Target mode

Defer support at launch. Remote use requires reliable discovery of the
target's boot volumes and privilege to edit them, plus SSH parity tests.

## Usage examples

### :set

```ruby
kernel_args 'loglevel' do
  value '5'
  action :set
end
```

### :remove

```ruby
kernel_args 'loglevel' do
  action :remove
end
```

## End-to-end testing

Use fixtures with multiple entries, mixed existing values, ties in
kernel paths and unrelated flags. Verify exact key matching and no-op
converges. On a guest with two installed kernels, verify staged edits,
boot both entries and inspect `/proc/cmdline`. Remove the argument and
repeat. Test whether newly installed kernels inherit the setting.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for grubby and BLS fixtures on a private container filesystem. |
| kitchen-exec | Limited. Standard hosted Ubuntu GitHub runners lack the proposed RHEL boot stack. |
| Vagrant | Required for BIOS and UEFI boot assertions on disposable RHEL-family guests. |

Run BLS fixture checks in a RHEL-family Dokken image. Use rebootable
guests for the release gate, including one UEFI guest and a second boot
entry. Record the boot arguments only after each guest restarts.

## Consequences

### Positive

Recipes can converge a boot parameter and detect when installed kernel
entries disagree without replacing their whole command lines.

### Negative

The first backend covers one boot layout and single-valued parameters.
An edit can be pending until reboot. grubby may update several entries
before reporting a failure.

### Follow-up

Verify actual BIOS and UEFI boots, rescue-entry handling and package
inheritance. Check whether the backend updates ancillary kernel-install
configuration on the supported package versions. Add other backends
after verifying their update and discovery contracts.

## Alternatives considered

Templating `/etc/default/grub` and regenerating configuration gives the
cookbook ownership of unrelated settings and does not model each
existing BLS entry. Editing `/proc/cmdline` cannot stage a boot change.
Using `grubby` follows the distro's supported interface.

## References

- [ADR-0001](0001-apparmor-profile-resource.md)
- [RHEL 9 kernel command-line parameters](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_monitoring_and_updating_the_kernel/configuring-kernel-command-line-parameters_managing-monitoring-and-updating-the-kernel)
- [Boot Loader Specification](https://uapi-group.org/specifications/specs/boot_loader_specification/)
- [Shared Chef resource inventory](https://github.com/chef/chef/tree/819adc4979af2e92980ae856a48e67c3f380111f/lib/chef/resource)
