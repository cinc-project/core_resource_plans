# 0021. Built-in `subordinate_id` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Rootless workloads use subordinate UID and GID delegations to map user
namespace identities. shadow-utils can store those ranges in `/etc/subuid`
and `/etc/subgid` with the files backend. Cinc Client's `user` handles
local account creation (`lib/chef/resource/user.rb:28`). Delegated ranges
are separate from the account's own UID and GID.

The shared Chef source has no built-in `subordinate_id` resource.

On Ubuntu 24.04 with shadow-utils 4.13, adding the inclusive range
`300000–365535` produced records with start `300000` and count `65536`
in both databases. The matching delete operations removed those records.

## Decision

Add `subordinate_id` to Cinc Client in the shared Chef codebase. Manage one explicit local subordinate UID or GID range for an existing non-root account. Use the distro's shadow-utils commands and preserve other ranges.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Reserve the exact range if absent. |
| `:delete` |  | Revoke the exact range if present. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `user` | String | Required | Existing local non-root account. Resolve numeric and named file records to the same identity. | Delegated account. |
| `kind` | Symbol | Required | `:uid` or `:gid`. | Subordinate ID database. |
| `start_id` | Integer | Required | Positive start. Reject values reserved by the local policy. | First delegated ID. |
| `count` | Integer | Required | Positive count. The inclusive end must fit in the supported ID range. | Number of IDs to delegate. |

### Current state and action steps

- Confirm that NSS selects the files backend. Under a Chef-local lock,
  parse the relevant database and compare intervals for the account and
  other owners. Reject overlapping delegations or IDs used by local
  accounts or groups, as appropriate for the database.
- `:create` treats the exact matching range as a no-op. Convert start and
  count to an inclusive end and run the relevant `usermod --add-subuids`
  or `--add-subgids` operation. Verify the result with a fresh file read.
- `:delete` requires an exact match before using the corresponding
  `--del-subuids` or `--del-subgids` operation. A partially overlapping
  range is an error. Operators should stop dependent workloads before
  revoking mappings they may need on restart.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Range collision | Compare intervals and local UID or GID records. | Fail and identify the conflicting owner or ID. |
| Unsupported provider or malformed records | NSS configuration and strict parser. | Fail before invoking usermod. |
| Write fails or races another writer | Command status and new parsed snapshot. | Report the observed state. Avoid automatic reversal of concurrent updates. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04, Debian 12 and RHEL 9 | Proposed with shadow-utils and the local files backend. |
| External subid plugins and centrally assigned accounts | Deferred. |

### Target mode

Defer until remote account lookup, range comparison and command privileges have parity tests.

## Usage examples

### :create

```ruby
subordinate_id 'worker-subuids' do
  user 'worker'
  kind :uid
  start_id 300_000
  count 65_536
  action :create
end
```

### :delete

```ruby
subordinate_id 'worker-subuids' do
  user 'worker'
  kind :uid
  start_id 300_000
  count 65_536
  action :delete
end
```

## End-to-end testing

Verify creation, exact deletion, overlap detection and repeated runs.
In a guest, use a throwaway subordinate mapping and confirm a rootless
process sees the expected IDs. Check that removing a range affects new
namespace allocation and leaves other ranges in place.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for local files and usermod fixtures. |
| kitchen-exec | Possible in a disposable VM with enabled user namespaces. |
| Vagrant | Preferred for rootless workload and namespace integration. |

Run file cases in multiple distro containers. Use a VM for namespace tests with a separate local test account.

## Consequences

### Positive

Cookbooks can allocate predictable ranges for rootless services, with collision reporting and independent UID and GID actions.

### Negative

Changing a delegation can change which host IDs a workload can use. The resource has no workload or filesystem migration action.

### Follow-up

Test both file record forms, shadow-utils range semantics and conflicts with directory-backed identities before broadening support.

## Alternatives considered

Writing the databases directly would require duplicating shadow-utils locking. Automatic allocation is useful when stable values are unnecessary, but explicit ranges suit coordinated storage and workload policies.

## References

- [shadow-utils subuid file and NSS provider](https://man7.org/linux/man-pages/man5/subuid.5.html)
- [shadow-utils usermod options](https://man7.org/linux/man-pages/man8/usermod.8.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
