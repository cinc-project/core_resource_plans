# 0023. Built-in `systemd_coredump` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Where systemd-coredump handles kernel core events, `coredump.conf.d`
drop-ins control how much dump data is processed and where it is stored.
Cinc Client can manage a resource limit with `user_ulimit` and write
configuration files with `file`. A session limit and host-wide dump
processing are different settings.

The shared Chef source has no built-in `systemd_coredump` resource.

On Ubuntu 24.04 with systemd 255.4-1ubuntu8.19, `systemd-analyze
cat-config` showed the test drop-in with `ProcessSizeMax=0`. The test
exercised drop-in discovery. Kernel-triggered handling remains a guest test.

## Decision

Add `systemd_coredump` to Cinc Client in the shared Chef codebase. Manage one systemd-coredump configuration fragment. The initial interface covers storage and maximum processed image size. Changes affect handling of future dumps.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Install the managed dump-handling policy. |
| `:delete` |  | Remove the managed policy fragment. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Fragment basename in `90-chef-NAME.conf`. |
| `storage` | Symbol | Required for create | `:none` or `:external`. | Persistent core image storage preference. |
| `process_size_max` | String | Required for create | `'0'` or a positive integer optionally followed by `K`, `M` or `G`. | Maximum size for processing a core image. |

### Current state and action steps

- Confirm systemd-coredump is installed and selected by the running
  kernel's core handler. Render a `[Coredump]` fragment with `Storage` and
  `ProcessSizeMax`. Compare contents and metadata before atomically
  installing the marked root-owned `0644` file.
- `:delete` removes only the marked fragment. Both file actions leave
  existing dumps and unrelated policy untouched.
- Report a competing later fragment when detected. Validate the effective
  policy in guests rather than inferring the handler's state from bytes.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Different core handler is active | Inspect the configured core pattern and installed handler. | Fail rather than staging policy for another handler. |
| Invalid requirement or unmanaged file | Field validation, symlink, special file or missing marker. | Leave the existing file in place. |
| Later fragment overrides the values | Merged configuration inspection. | Identify the conflicting path for reconciliation. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 | Proposed when systemd-coredump is installed and the active handler. |
| Apport, ABRT or traditional core-file handlers | Outside the initial scope. |

Use the [platform baseline](../../README.md#platform-baseline) for Ubuntu 26.10 smoke tests and
Debian 12 LTS compatibility checks when the required tools are available.

### Target mode

Defer until active-handler inspection and file writes can be verified on the remote node.

## Usage examples

### :create

```ruby
systemd_coredump 'site-dump-policy' do
  storage :none
  process_size_max '0'
  action :create
end
```

### :delete

```ruby
systemd_coredump 'site-dump-policy' do
  action :delete
end
```

## End-to-end testing

Check fragment precedence and repeated converge. Trigger synthetic faults
in a disposable guest and inspect journal metadata and stored images.
Cover both storage policies and deletion. Confirm earlier dumps stay intact.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for rendering and precedence fixtures. |
| kitchen-exec | Limited by the runner's active core handler and test isolation. |
| Vagrant | Preferred for kernel-triggered dumps and storage verification. |

Run fixture tests in Docker. Use an isolated guest with the handler selected for fault and dump assertions.

## Consequences

### Positive

Recipes can declare an explicit storage and processing policy without editing vendor settings.

### Negative

The resource does not select the kernel's core handler. Journal metadata can still be recorded when image storage and processing are disabled.

### Follow-up

Confirm temporary image handling, logging and precedence on each candidate distro.

## Alternatives considered

A session core limit operates at process launch and may require separate settings for system services. A coredump handler policy applies wherever the selected handler is invoked.

## References

- [systemd coredump configuration](https://www.freedesktop.org/software/systemd/man/latest/coredump.conf.html)
- [systemd-coredump handler lifecycle](https://www.freedesktop.org/software/systemd/man/latest/systemd-coredump.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
