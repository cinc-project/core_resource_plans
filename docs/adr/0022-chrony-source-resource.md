# 0022. Built-in `chrony_source` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

chrony reads `server` and `pool` declarations from its main configuration
and from `.sources` files in configured `sourcedir` directories. Cinc Client
can distribute files and control services (`lib/chef/resource/file.rb:32`
and `lib/chef/resource/service.rb:31`). Recipes coordinate the source
fragment and chrony's runtime source reload themselves.

The shared Chef source has no built-in `chrony_source` resource.

On Ubuntu 24.04 with chrony 4.5, `chronyd -p` accepted a private
configuration containing a `sourcedir` directive. Live source reload
and selection remain guest tests.

## Decision

Add `chrony_source` to Cinc Client in the shared Chef codebase. Manage one site clock source in an existing chrony `sourcedir`. Require a configured source directory and keep source reload explicit.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Write or reconcile the source fragment. |
| `:delete` |  | Remove the managed source fragment. |
| `:reload` |  | Ask chronyd to reload configured source directories. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Fragment ID in `chef-NAME.sources`. |
| `address` | String | Required for create | Valid ASCII hostname or IPv4 or IPv6 literal without a scope identifier. A pool requires a hostname. | Source address. |
| `source_type` | Symbol | `:server` | `:server` or `:pool`. | chrony source directive. |
| `iburst` | Boolean | `true` | Boolean. | Enable initial burst sampling. |
| `sources_dir` | String | Distro default | Absolute existing administrator-owned directory declared by the effective chrony configuration. Required when selection is ambiguous. | Fragment destination. |

### Current state and action steps

- Discover the installed chrony configuration and source directories.
  Validate the chosen directory and render one deterministic source line.
  Validate it with `chronyd -p` as a top-level directive in a private
  candidate configuration.
- `:create` compares file contents and metadata, then installs the marked
  fragment atomically as `root:root` and `0644`. `:delete` removes only the
  marked file after the same destination checks.
- `:reload` runs `chronyc reload sources` with a timeout. It is intended
  for coalesced notifications. Tool acknowledgement confirms reload, while
  source reachability and clock synchronisation remain runtime concerns.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Unknown source directory | No matching `sourcedir` in the effective configuration or conflicting discovery. | Fail before editing. |
| Validation failure or path collision | Invalid directive, symlink, special file or absent marker on an existing fragment. | Keep the existing file. |
| Reload fails | chronyc timeout or nonzero status. | Report the failure and keep persisted edits for retry. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04, Debian 12 and RHEL 9 | Proposed with chrony versions that support `sourcedir` and an explicitly configured directory. |
| Other time daemons and chrony without `sourcedir` | Deferred. |

### Target mode

Defer until remote config discovery, private staging and privileged chronyc reload are proven.

## Usage examples

### :create

```ruby
chrony_source 'worker-time' do
  address 'time.example.net'
  source_type :server
  action :create
end
```

### :delete

```ruby
chrony_source 'worker-time' do
  action :delete
end
```

### :reload

```ruby
chrony_source 'worker-time' do
  action :reload
end
```

## End-to-end testing

Check parsing of server and pool fragments, repeated converge, removal
and an unsupported directory. Use an isolated NTP fixture to show that
reload adds and removes an actual chrony source without restarting chronyd.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for file checks and private configuration parsing. |
| kitchen-exec | Possible with private daemon configuration and a reachable fixture. |
| Vagrant | Preferred for live chronyd and clock integration in an isolated guest. |

Run parser and file tests in distro containers. Use a disposable guest and a controlled time source for the reload suite.

## Consequences

### Positive

Recipes can stage clock sources independently of packaged chrony configuration and reload them in a single notification.

### Negative

The site must configure a source directory first on distributions that omit it. Reload processes all configured source directories.

### Follow-up

Confirm distro default directories, duplicate source behaviour and live reload semantics against each packaged version.

## Alternatives considered

Templating the main configuration gives Chef control over all time settings. A sourcedir fragment keeps changes confined to the requested source.

## References

- [chrony configuration and sourcedir](https://chrony-project.org/doc/4.5/chrony.conf.html)
- [chronyc reload sources](https://chrony-project.org/doc/4.5/chronyc.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
