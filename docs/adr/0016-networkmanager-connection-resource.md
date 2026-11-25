# 0016. Built-in `networkmanager_connection` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

On NetworkManager systems, a keyfile stores a connection profile and
its UUID separately from current device state. Cinc Client ships
`ifconfig` and `route` for interface and route resources
(`lib/chef/resource/ifconfig.rb:26` and `lib/chef/resource/route.rb:26`).

The shared Chef source has no built-in `networkmanager_connection` resource.

On Ubuntu 24.04 with NetworkManager 1.46.0, `nmcli --offline connection
add` emitted an Ethernet DHCP keyfile with a supplied UUID and
autoconnect disabled, without a running manager.

## Decision

Add `networkmanager_connection` to Cinc Client in the shared Chef codebase. Manage one persistent Ethernet DHCP profile under `/etc/NetworkManager/system-connections`. Stage the profile with autoconnect disabled. Loading a profile is explicit, and interface activation stays with the operator or existing network workflow.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Generate and install the managed keyfile. |
| `:delete` |  | Remove an inactive managed keyfile from disk. |
| `:reload` |  | Tell NetworkManager to re-read its on-disk connection profiles. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Profile ID and filename component. |
| `interface_name` | String | Required for create | Linux interface name of 1–15 ASCII letters, digits, underscores, periods or hyphens. | Device binding when the profile is activated. |
| `uuid` | String | Required for file actions | Canonical UUID. Uniqueness checked before mutation. | Stable persistent profile identity. |

### Current state and action steps

- `:create` checks disk and the local daemon, if available, for connection
  ID and UUID collisions. Generate a DHCP Ethernet profile with the explicit
  UUID, bound interface and autoconnect disabled using `nmcli --offline`.
  Compare parsed settings and metadata, then atomically install the marked
  `0600` keyfile owned by root.
- `:delete` checks that the UUID is inactive and the persistent file matches
  the marker and UUID, then removes it if present. Require an available
  manager when the file exists so activity can be checked.
- `:reload` runs `nmcli connection reload` with a timeout. It re-reads the
  full profile set, including removals, without explicitly activating a
  connection or reapplying settings to an active device.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| UUID or ID collision | File and manager discovery show a separate profile. | Fail and identify the existing profile. |
| Active connection or unknown activity | Manager status cannot confirm the profile is inactive. | Refuse to delete it. |
| Bad keyfile or unavailable manager | Offline command or explicit reload fails. | Keep prior state when generation fails. Report reload failure and keep the persisted edit for retry. |
| Unsafe path | Symlink, special file or missing marker. | Refuse file mutation. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS and RHEL 10 with supported nmcli | Proposed after offline feature and manager checks. |
| systemd-networkd or other backends | Deferred. |

Use the [platform baseline](../../README.md#platform-baseline) for Ubuntu 26.10 smoke tests.

### Target mode

Defer until remote manager discovery and profile generation are covered. Live activation stays outside this interface.

## Usage examples

### :create

```ruby
networkmanager_connection 'worker-uplink' do
  interface_name 'eth1'
  uuid '8e8dd25c-6ebc-4f3e-a7ca-9724bc093ccf'
  action :create
end
```

### :delete

```ruby
networkmanager_connection 'worker-uplink' do
  uuid '8e8dd25c-6ebc-4f3e-a7ca-9724bc093ccf'
  action :delete
end
```

### :reload

```ruby
networkmanager_connection 'worker-uplink' do
  action :reload
end
```

## End-to-end testing

Verify keyfile settings, root-only mode, stable UUID and repeated converge.
Use a secondary virtual NIC and DHCP fixture to load and activate the profile
explicitly. Check that the management NIC stays connected. Cover inactive
deletion, active deletion refusal, collisions and restart persistence.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for offline generation and file cases. |
| kitchen-exec | Possible with a private virtual NIC on a disposable runner. |
| Vagrant | Preferred for live manager, DHCP and reboot tests. |

Run offline fixtures on Ubuntu and EL with compatible nmcli. Run live checks on a disposable guest with an isolated second interface.

## Consequences

### Positive

Recipes can stage or review a connection profile without disrupting active interfaces.

### Negative

The initial resource covers Ethernet DHCP only. Static addressing, Wi-Fi credentials and activation are outside this design.

### Follow-up

Verify accepted offline arguments and manager reload semantics on each proposed distro. Check daemon collision reporting and profile import on boot.

## Alternatives considered

Editing the active interface couples persistent policy with live network changes. A native keyfile gives the recipe a separate on-disk state to converge.

## References

- [NetworkManager nmcli reference](https://networkmanager.dev/docs/api/latest/nmcli.html)
- [NetworkManager keyfile format](https://networkmanager.dev/docs/api/latest/nm-settings-keyfile.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
