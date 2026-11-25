# 0032. Built-in `systemd_link` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

systemd-udevd selects the first matching `.link` file in lexical order
when a device appears. This controls link attributes before higher-level
network management. Cinc Client's `systemd_unit` manages units
(`lib/chef/resource/systemd_unit.rb:27`). ADR-0016 manages a NetworkManager
connection profile and ADR-0004 manages general udev rules.

The shared Chef source has no built-in `systemd_link` resource.

The Ubuntu 26.04 LTS arm64 image provided udev 259.9. Link selection
and NIC discovery remain guest tests.

## Decision

Add `systemd_link` to Cinc Client in the shared Chef codebase. Manage one per-device `.link` file matched by permanent MAC address. Configure MTU and explicit naming policy. Leave device replay and activation to existing network workflows.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Validate and write the persistent link settings. |
| `:delete` |  | Remove the resource-owned link file. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Basename in `70-chef-NAME.link`. |
| `permanent_mac` | String | Required for create | Six colon-delimited hexadecimal octets for a unicast address. Reject zero and broadcast. | Permanent address used for the match. |
| `mtu_bytes` | Integer | Required for create | Positive value within the target device's bounds when it is present. | Desired MTU for the selected link. |
| `name_policy` | Array of String | Required for create | Ordered unique supported `NamePolicy` values. | Explicit naming policy because the first matching file supplies link settings. |

### Current state and action steps

- Inspect the effective `.link` search paths and compare the candidate's
  match against higher-priority files. Render deterministic `[Match]` and
  `[Link]` sections. Report known shadowing, another declaration for the
  same permanent address or conflicting manager configuration.
- `:create` compares content and metadata, then installs a marked
  root-owned `0644` file atomically. Typed fields restrict generated syntax.
  Apply the settings at a later device event. Leave active links untouched.
- `:delete` removes only the marked file. A later discovery uses the next
  matching file. Report live verification separately from file convergence.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Invalid link properties or ambiguous device | Parse inputs and compare available stable device identity. | Fail before publishing. |
| Earlier match overrides the resource | Check effective paths and matching settings. | Report the competing file and refuse a detected conflict. |
| Unexpected destination | Symlink, special file or missing ownership marker. | Preserve the file. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 family | Proposed where systemd-udevd provides net_setup_link. |
| Ubuntu 26.10 and Debian 12 LTS | Smoke and compatibility fixtures where the match and link settings are supported. |

See the [platform baseline](../../README.md#platform-baseline) for images and client availability.

### Target mode

Defer until remote search-path and NIC identity tests cover the expected target layout.

## Usage examples

### :create

```ruby
systemd_link 'worker-uplink' do
  permanent_mac '02:00:00:00:00:32'
  mtu_bytes 1500
  name_policy %w(keep kernel database onboard slot path)
  action :create
end
```

### :delete

```ruby
systemd_link 'worker-uplink' do
  action :delete
end
```

## End-to-end testing

Use a guest with an extra test NIC. Prove the file matches only that NIC,
does not change an active link until a new event, and applies the expected
MTU and naming on discovery. Check shadowing, deletion and repeat convergence.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for generating fixtures and inspecting packaged udev tooling. |
| kitchen-exec | Possible with a separate virtual interface on an ephemeral runner. |
| Vagrant | Preferred for predictable NIC discovery and reboot tests. |

Run file fixtures on the baseline. Use a guest with an extra NIC for discovery and naming assertions.

## Consequences

### Positive

Cookbooks can describe early link settings for a particular NIC without owning the network manager's connection profiles.

### Negative

An earlier match replaces the vendor link file for that device. Changes to naming and link attributes may need coordinated downtime.

### Follow-up

Verify effective match detection and behaviour when the network manager also sets MTU. Compare vendor default naming before adopting a .link file.

## Alternatives considered

NetworkManager profiles are useful when the manager owns the device lifecycle. A .link file applies early link policy during device discovery.

## References

- [systemd.link match and precedence](https://www.freedesktop.org/software/systemd/man/latest/systemd.link.html)
- [NetworkManager connection proposal](0016-networkmanager-connection-resource.md)
- [udev rules proposal](0004-udev-rule-resource.md)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
