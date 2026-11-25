# 0014. Built-in `nsswitch_entry` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

On glibc systems the Name Service Switch selects databases and lookup
order through `/etc/nsswitch.conf`. The `hosts` row can contain service
names and bracketed status actions. Cinc Client currently supplies generic
file resources such as `template` (`lib/chef/resource/template.rb:38`).

The shared Chef source has no built-in `nsswitch_entry` resource.

On Ubuntu 24.04 with glibc 2.39, `getent hosts` resolved a test mapping
after setting the hosts row to `files dns`. Precedence against competing
answers and failure cases still require isolated guest tests.

## Decision

Add `nsswitch_entry` to Cinc Client in the shared Chef codebase. Manage the lookup order for the glibc `hosts` database. Own the full policy for that one row. Other databases keep their existing policy.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:set` | Yes | Set the lookup policy for the named database. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | `'hosts'` in the initial release. | Database key. |
| `sources` | Array of String | Required | Unique service identifiers with `files` first, then optional `resolve` or `dns`. Reject embedded status clauses. | Ordered lookup sources with glibc's default status actions. |
| `replace_status_actions` | Boolean | `false` | Boolean. | Allow replacing an existing row that contains explicit status clauses. |

### Current state and action steps

- Read and lock the target configuration. Parse active rows and bracketed
  clauses so current lookup order, status actions and comments can be
  distinguished. Reject duplicate active `hosts` rows.
- `:set` checks the chosen sources against the node's supported resolver
  setup, then replaces or appends one row. Changing a row containing custom
  status clauses requires `replace_status_actions true`. Preserve other
  rows, comments and file metadata.
  Compare the parsed policy for convergence before writing.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| System manages the file | Symlink or known generated profile such as authselect. | Fail and identify the owner. |
| Named resolver unavailable | Backend capability check. | Fail before writing. |
| Existing row contains status clauses | Parsed clauses are present and `replace_status_actions` is false. | Keep the row and report its existing policy. |
| Ambiguous or malformed row | Duplicate key or invalid bracket structure. | Fail and report the affected row. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS and Debian 13 with glibc | Proposed for regular administrator-owned files. |
| RHEL authselect-managed systems and musl systems | Deferred. |

Use the [platform baseline](../../README.md#platform-baseline) for Ubuntu 26.10 smoke tests and
Debian 12 LTS compatibility checks when the required tools are available.

### Target mode

Defer until target-side source availability checks and atomic file replacement are proven.

## Usage examples

### :set

```ruby
nsswitch_entry 'hosts' do
  sources %w(files dns)
  action :set
end
```

## End-to-end testing

Use a disposable guest with a hosts-file answer and a DNS fixture giving a
different answer for the same name. Verify source precedence and recovery
from an unavailable resolver. Cover comments, existing status clauses,
managed-file refusal and repeated converge.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for parser and file cases. |
| kitchen-exec | Possible for disposable resolver smoke tests. |
| Vagrant | Preferred for conflicting answers and distro resolver services. |

Run parser fixtures in Ubuntu and Debian. Gate through a guest with controlled DNS answers and assertions using a fresh process.

## Consequences

### Positive

A cookbook can express lookup order while preserving policies for accounts, groups and other databases.

### Negative

Options such as mDNS and custom status actions are outside the first interface. The resource takes ownership of the selected row.

### Follow-up

Validate systemd-resolved module discovery and glibc resolution behaviour, including process lifetime and fallback.

## Alternatives considered

An entire template makes sense for an appliance with one configuration owner. Managing a single parsed row limits changes to the database the cookbook owns.

## References

- [glibc Name Service Switch](https://www.gnu.org/software/libc/manual/html_node/Name-Service-Switch.html)
- [nsswitch.conf syntax](https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
