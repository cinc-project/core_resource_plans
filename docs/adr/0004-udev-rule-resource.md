# 0004. Built-in `udev_rule` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

udev applies rules to device events. Cookbooks use local rules to assign
stable symlinks, permissions and device properties. udev processes the
effective `.rules` files in lexical order. A file under
`/etc/udev/rules.d` wins over a lower-priority file with the same basename.

Cinc Client has `file` and `execute` for writing fragments and invoking
commands (see `lib/chef/resource/file.rb:32` and
`lib/chef/resource/execute.rb:27`). The shared Chef resource inventory has no
udev resource. A cookbook has to arrange validation and reloads itself.

On Ubuntu 24.04 with udev 255.4-1ubuntu8.19, `udevadm verify` accepted a rule
assigning a serial-device symlink and rejected a rule with an invalid key.
A test serial-device rule passed on Ubuntu 26.04 LTS with udev 259.9,
Ubuntu 26.10 with udev 261.3, and Debian 13 with udev 257.13. AlmaLinux
10's systemd-udev 257 exposes the verifier. These checks used arm64
containers.

## Decision

Add `udev_rule` to Cinc Client in the shared Chef codebase. Own one
fragment in `/etc/udev/rules.d/NN-chef-NAME.rules`. Validate candidate
content before publishing it. Make daemon reload explicit so recipes can
batch fragments. Reload affects subsequent events. Existing devices need
a new event to pick up changes.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Verify and install the rule fragment on disk. |
| `:delete` | | Remove the managed fragment. |
| `:reload` | | Ask the running udev daemon to reload its configuration. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | Letters, digits, underscores or hyphens. Starts with a letter or digit. | Fragment identifier. |
| `priority` | Integer | `70` | `0` to `99`, formatted with two digits. | Lexical order and part of fragment identity. |
| `content` | String | Required for `:create` | Nonempty UTF-8 rule text with no NUL characters. Must pass udev validation. | Complete rule content. |

### Current state and action steps

- `:create` compares desired content and root ownership and mode against
  the installed regular file. Normalise the final newline and add a
  managed-file marker. Validate changed text with
  `udevadm verify --no-style` on a staged `.rules` file outside udev's
  search path. After validation, atomically publish it as `0644`.
- `:delete` removes the marked file at the named priority if present.
  Changing priority requires deleting the old declaration explicitly.
- `:reload` runs `udevadm control --reload` with a finite timeout on
  each invocation. This action applies globally and is intended for
  coalesced notifications. It performs no device replay.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Verifier is unavailable | `udevadm` lacks the required verifier. | Fail before publishing a fragment. |
| Candidate fails validation | Verifier returns nonzero. | Keep the current file and include the diagnostic. |
| Unexpected destination or namespace collision | Symlink, special file, unmarked local file or same basename at a lower priority. | Fail and report the path. |
| Reload cannot reach daemon | Timeout or nonzero status. | Fail the explicit action. Keep edits on disk so the caller can retry `:reload`. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 family | Proposed with `udevadm verify` from systemd-udev 254 or later and a daemon for `:reload`. |
| Ubuntu 26.10 | Utility smoke tests with the distro's verifier and a source-built client when needed. |
| Debian 12 LTS and EL9 | Compatibility fixtures require a build that exposes `udevadm verify`. |
| Older udev, eudev or daemon-free containers | Older tools require a separate validation strategy. A daemon-free image can stage verified files. |

See the [platform baseline](../../README.md#platform-baseline) for image and
client availability.

### Target mode

Defer support at launch. Remote staging, validation and reload must all
act on the same node and preserve local rule diagnostics.

## Usage examples

### :create

```ruby
udev_rule 'reload-rules' do
  action :nothing
end

udev_rule 'telemetry-serial' do
  content 'SUBSYSTEM=="tty", KERNEL=="ttyUSB0", SYMLINK+="telemetry-serial"'
  notifies :reload, 'udev_rule[reload-rules]', :delayed
  action :create
end
```

### :delete

```ruby
udev_rule 'telemetry-serial' do
  action :delete
end
```

### :reload

```ruby
udev_rule 'reload-rules' do
  action :reload
end
```

## End-to-end testing

Check valid and invalid fragments, first and second converges, unmanaged
paths and reload when the daemon is unavailable. Use a known test device
or virtual device in a disposable guest. Prove a new device event uses the
new rule and that repeated converge sends no reload notification.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for syntax and file tests with the required udev package. |
| kitchen-exec | Good for daemon smoke tests with a disposable Ubuntu runner. |
| Vagrant | Good for reproducible virtual-device events and boot coverage. |

Run syntax and file checks on Ubuntu, Debian 13 and an EL10 image. Run
daemon and event tests on Ubuntu 26.04 LTS, with a guest for controlled
virtual device setup.

## Consequences

### Positive

Invalid staged rules leave the installed rules intact. Recipes can reload
once after editing several fragments.

### Negative

Older distributions may lack the verifier. Reload acknowledgement alone
does not establish which rules matched a device.

### Follow-up

Verify reload and event processing on an actual VM. Evaluate a
device-scoped replay action with an explicit device selector.

## Alternatives considered

A `cookbook_file` and an `execute` notification allow the same file format.
The built-in resource gives recipes a standard verification step and
consistent ownership and reload semantics. Automatically triggering all
devices on every update could rerun unrelated device actions.

## References

- [udev rule ordering and syntax](https://www.freedesktop.org/software/systemd/man/latest/udev.html)
- [udevadm verify and daemon control](https://www.freedesktop.org/software/systemd/man/latest/udevadm.html)
- [Shared Chef resource inventory](https://github.com/chef/chef/tree/819adc4979af2e92980ae856a48e67c3f380111f/lib/chef/resource)
