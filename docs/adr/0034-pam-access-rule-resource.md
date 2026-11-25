# 0034. Built-in `pam_access_rule` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Linux-PAM's `pam_access` module evaluates ordered access rules for the
account phase of a PAM service. Cinc Client's `user_ulimit` manages limits
files (`lib/chef/resource/user_ulimit.rb:27`). Access policy uses a separate
stack module and configuration.

The shared Chef source has no built-in `pam_access_rule` resource.

Ubuntu 26.04 LTS arm64 packages `pam_access.so` in Linux-PAM 1.7.0.
The effective stack, origin matching and login behaviour remain guest tests.

## Decision

Add `pam_access_rule` to Cinc Client in the shared Chef codebase. Manage one explicit deny rule for named local accounts and IP networks in a managed access.d fragment. Require the chosen PAM service to already use pam_access and its default file search.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Stage the named access restriction. |
| `:delete` |  | Remove the marked access fragment. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Basename in `40-chef-NAME.conf`. |
| `pam_service` | String | Required | Literal service name whose effective account stack uses the supported module options. | Stack used to validate prerequisites. The default access files can affect other services that read them. |
| `accounts` | Array of String | Required for create | Nonempty unique literal local account names. Reject PAM keywords, grouplists and delimiters. | Login names to restrict. |
| `origins` | Array of String | Required for create | Nonempty explicit IPv4 or IPv6 CIDRs with nonzero prefix. | Matching remote origins. |

### Current state and action steps

- Inspect the effective account stack for the declared service. Require
  `pam_access` with default file discovery and supported list semantics.
  Refuse an explicit `accessfile`, incompatible separators or ambiguity.
  Check the named accounts against the group database. A group with the
  same name requires `nodefgroup` in each service that reads the shared
  rule. Refuse unresolved group-matching behaviour.
- `:create` renders a single typed denial with a managed marker and
  compares file contents and metadata. Inspect loaded rule order and
  refuse a detected prior match that makes the rule ineffective. Publish
  atomically as `root:root` with mode `0644`.
- `:delete` removes only the owned fragment. Rules affect new PAM account
  checks when the module reads its policy. The resource leaves PAM stack
  files and the service lifecycle in place.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Unsupported or inactive module | Effective PAM stack or module options differ. | Fail before writing. |
| Unparseable or shadowing rules | Check supported grammar and precedence. | Refuse an ambiguous result and identify the competing rule. |
| Unexpected destination | Symlink, special file or missing marker. | Preserve the path. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 family | Proposed with packaged Linux-PAM and access.d support. |
| Ubuntu 26.10 and Debian 12 LTS | Smoke and compatibility checks where the chosen stack supports the same access file semantics. |

See the [platform baseline](../../README.md#platform-baseline) for images and client availability.

### Target mode

Defer until stack discovery and test login continuity are verified over SSH.

## Usage examples

### :create

```ruby
pam_access_rule 'worker-login-policy' do
  pam_service 'sshd'
  accounts %w(worker worker-ci)
  origins ['192.0.2.0/24']
  action :create
end
```

### :delete

```ruby
pam_access_rule 'worker-login-policy' do
  pam_service 'sshd'
  action :delete
end
```

## End-to-end testing

Use an isolated PAM service on a test guest and mock login origins.
Check allowed and denied account checks, first-match precedence, a
missing module, removal and preservation of the management login.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for static parser and file fixtures. |
| kitchen-exec | Possible for a separate test PAM service on an ephemeral runner. |
| Vagrant | Preferred for real network login and lockout-recovery fixtures. |

Run order and detection fixtures on the baseline. Test the selected module and shared file effects on a disposable guest.

## Consequences

### Positive

Cookbooks can control an explicit login restriction while keeping stack configuration with its provisioning owner.

### Negative

All PAM consumers using the same default files can observe the restriction. Policy ordering and service authentication flows require guest tests.

### Follow-up

Confirm access.d ordering, module flags and named account matching on each packaged PAM version.

## Alternatives considered

Templating access.conf suits a site that owns its entire login policy. A marked fragment gives independently managed restrictions an explicit identity.

## References

- [Linux-PAM pam_access module](https://man7.org/linux/man-pages/man8/pam_access.8.html)
- [Linux-PAM access.conf grammar](https://man7.org/linux/man-pages/man5/access.conf.5.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
