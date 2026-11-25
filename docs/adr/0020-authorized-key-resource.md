# 0020. Built-in `authorized_key` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

OpenSSH reads authorised public keys from account-specific files to
decide which client identities may log in. Cinc Client's
`ssh_known_hosts_entry` manages client-side trust in server host keys
(`lib/chef/resource/ssh_known_hosts_entry.rb:27`). Server access policy uses
a separate file and key identity.

The shared Chef source has no built-in `authorized_key` resource.

On Ubuntu 24.04 with OpenSSH 9.6p1, `ssh-keygen -lf` parsed a generated
Ed25519 public key and returned its SHA-256 fingerprint. File semantics
and applying the key to a login remain to be verified.

## Decision

Add `authorized_key` to Cinc Client in the shared Chef codebase. Manage one Ed25519 public key for an existing local non-root account using the packaged OpenSSH default `.ssh/authorized_keys` path. Preserve existing user keys and restrictions.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Reconcile the labelled key in the account's file. |
| `:delete` |  | Remove the labelled key and retain all other entries. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. Unique per account. | Stable managed entry label. |
| `user` | String | Required | Existing local non-root account with an ordinary absolute home and effective default authorised-keys path. | Login account. |
| `public_key` | String | Required for create | Single valid `ssh-ed25519` public-key line. Reject key options, multiple lines and private-key material. | Identity to authorise. |

### Current state and action steps

- Verify the configured authorised-keys path for the supported global
  server configuration. Resolve the account and inspect its home and
  `.ssh` without following symlinks. Do file mutations with the account's
  privileges and lock the file update. Check for unsafe ownership or modes.
- `:create` parses the public key and compares its fingerprint against the
  labelled row and all unmanaged rows. Normalise the line and write it with
  a Chef-owned label as the comment. Create a private `.ssh` directory when
  absent and use atomic replacement for the `0600` file.
- `:delete` removes only the labelled row. Preserve other lines byte-for-byte
  and leave the directory in place. A missing label is a no-op.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Key already present under other ownership | Parsed key blob matches another row, including a restricted row. | Refuse to add an unrestricted duplicate. |
| Unsafe path or unsupported server layout | Target symlink, insecure home, or customised effective authorised-keys source. | Fail and leave existing policy in place. |
| Invalid key or duplicate marker | OpenSSH key validation or multiple rows for one label. | Report the reason and keep the file. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04, Debian 12 and RHEL 9 | Proposed for packaged OpenSSH and local non-root accounts. |
| External key commands, conflicting Match paths and shared home filesystems | Deferred. |

### Target mode

Defer until remote account privileges and file safety are verified. Resource use must account for the operator's own SSH connection.

## Usage examples

### :create

```ruby
authorized_key 'worker-deploy' do
  user 'deploy'
  public_key lazy { ::File.read('/etc/chef/keys/deploy.pub') }
  action :create
end
```

### :delete

```ruby
authorized_key 'worker-deploy' do
  user 'deploy'
  action :delete
end
```

## End-to-end testing

Use two disposable keys and an isolated SSH daemon. Verify the managed key
can log in, deletion removes that access, a pre-existing unmanaged key
still works, and a restricted copy is never duplicated without restrictions.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for key parsing and file operations. |
| kitchen-exec | Possible with an isolated test SSH daemon. |
| Vagrant | Preferred for packaged server settings and real remote-login assertions. |

Run file fixtures in Dokken and a secondary-daemon login suite on a disposable Ubuntu guest.

## Consequences

### Positive

Cookbooks can add or remove individual deploy keys without owning all authorised access for an account.

### Negative

The first interface handles plain Ed25519 keys for one local path. Key certificates and arbitrary per-key policy need separate design.

### Follow-up

Verify OpenSSH option tokenisation, path precedence and race protection on supported distributions.

## Alternatives considered

Templating the complete file is useful when Chef owns every key. A labelled record is a better fit for shared ownership and makes single-key replacement explicit.

## References

- [OpenSSH authorised key file format](https://man.openbsd.org/sshd.8#AUTHORIZED_KEYS_FILE_FORMAT)
- [OpenSSH AuthorizedKeysFile](https://man.openbsd.org/sshd_config#AuthorizedKeysFile)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
