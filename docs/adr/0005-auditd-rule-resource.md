# 0005. Built-in `auditd_rule` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Linux Audit policy can be split into fragments under `/etc/audit/rules.d`.
On distributions with an augenrules-based audit service, `augenrules`
merges the fragments into `/etc/audit/audit.rules` in natural sort order.
Some control directives are moved during the merge. In particular `-D`
comes first and `-e` last when present.

Cinc Client can distribute these files with `file` and run a command with
`execute` (see `lib/chef/resource/file.rb:32` and
`lib/chef/resource/execute.rb:27`). The shared Chef resource inventory has
no audit policy resource.

On Ubuntu 24.04 with auditd 3.1.2, `augenrules --check` reported pending
changes after adding a fragment. Running `augenrules` produced the merged
file, and a subsequent `--check` reported no change. This check exercised
the merge only.

## Decision

Add `auditd_rule` to Cinc Client in the shared Chef codebase. Manage one
file-access rule in `/etc/audit/rules.d/40-chef-NAME.rules`. Render an
architecture-qualified exit rule with path, permissions and audit key
filters. The initial design prepares persistent policy. Activation is
owned by the distribution's audit policy loader.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Write or reconcile a managed audit fragment. |
| `:delete` | | Remove the managed fragment. |
| `:compile` | | Reconcile the system-wide merged file using `augenrules` without `--load`. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | Letters, digits, underscores or hyphens. Starts with a letter or digit. | Fragment identifier. |
| `path` | String | Required for `:create` | Absolute file path with simple ASCII components. Reject whitespace, dot and parent components. | File accessed by audited processes. |
| `permissions` | String | `'wa'` | Nonempty subset of `r`, `w`, `x` and `a` without duplicates. | Access types audited. |
| `key` | String | Resource name | 1–31 ASCII letters, digits, underscores or hyphens. | Search key attached to matching audit events. |

### Current state and action steps

- `:create` renders a deterministic `-a always,exit` rule with `arch=b64`
  first, then the `path`, `perm` and `key` filters. Compare the managed
  fragment's content and metadata. Atomically install it as `root:root`
  and `0600`. The target file must exist by the time policy is loaded.
- `:delete` removes only the marked fragment if present. Use `:compile`
  afterwards when a current merged file is required before boot.
- `:compile` takes a node-local lock for this resource's compile
  operations. Run `augenrules` and compare the digest and existence of
  the generated file before and after. Report an update only if the
  artifact changes. Treat tool output as a diagnostic. A successful merge
  carries no guarantee that the kernel accepts every rule in the result.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Invalid rule inputs | Field validation before rendering. | Fail before opening the destination for writes. |
| Unmanaged destination | Fragment path is a symlink, special file or unmarked file. | Fail and identify the path. |
| Unsupported audit policy loader | Missing augenrules or unsupported distro service integration. | Fail with the unmet prerequisite. |
| Merge fails | Nonzero status or no generated regular file. | Report the merge failure. Keep the fragments available for diagnosis and retry. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04 and RHEL 9 on x86-64 | Initial targets, subject to tests of packaged rules loaders. |
| Other kernels and userspaces | Deferred pending checks of the audit ABI and distribution loader. |

### Target mode

Defer support at launch. Remote file privileges, merge locking and service
discovery require SSH coverage.

## Usage examples

### :create

```ruby
auditd_rule 'sudoers-changes' do
  path '/etc/sudoers'
  permissions 'wa'
  action :create
end
```

### :delete

```ruby
auditd_rule 'sudoers-changes' do
  action :delete
end
```

### :compile

```ruby
auditd_rule 'compile-site-audit-policy' do
  action :compile
end
```

## End-to-end testing

Run merge fixtures with vendor and site rules, check ordering and
permissions, and verify a second converge reports no changes. On a
disposable guest, boot the packaged audit service, verify the rule with
`auditctl -l` and check for the key after writing a dedicated test file.
Verify removal at the next boot. Include a guest with an immutable
policy to confirm that compile changes disk state only.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for fragment and merge tests. Shared host kernels make live audit-policy tests unsuitable. |
| kitchen-exec | Possible on an isolated GitHub runner VM with kernel audit enabled. |
| Vagrant | Preferred for the complete kernel, service and reboot test. |

Run Dokken merge coverage on Ubuntu and a RHEL family image. Gate live
audit coverage on a rebootable guest for each supported distro family.

## Consequences

### Positive

Cookbooks can manage individual file-access rules without taking
ownership of the site's complete audit policy.

### Negative

An existing policy can take precedence over an added rule. Runtime
activation follows the distro's loader, so disk changes can be pending
until the next boot. The initial resource covers one audit rule class.

### Follow-up

Verify kernel acceptance and logged events for the rendered rule on both
initial platforms. Evaluate compatibility ABIs, broader rule types and
an explicit live-load action with immutable and partial-load handling.

## Alternatives considered

Regenerating all of `audit.rules` from a template takes ownership away
from packaged and administrator-provided fragments. Automatically
running `augenrules --load` after every edit can change unrelated running
policy and risks a partial load. Use the existing boot-time policy loader
for the first version.

## References

- [Linux Audit rule ordering recommendations](https://github.com/linux-audit/audit-userspace/blob/master/rules/README-rules)
- [augenrules manual](https://github.com/linux-audit/audit-userspace/blob/master/docs/augenrules.8)
- [auditctl rule syntax](https://github.com/linux-audit/audit-userspace/blob/master/docs/auditctl.8)
- [Shared Chef resource inventory](https://github.com/chef/chef/tree/819adc4979af2e92980ae856a48e67c3f380111f/lib/chef/resource)
