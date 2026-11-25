# 0025. Built-in `ca_trust_certificate` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Linux distributions generate system trust stores from packaged and local
CA certificates. Cinc Client has `chef_client_trusted_certificate` for the
client's own certificate directory
(`lib/chef/resource/chef_client_trusted_certificate.rb:24`). System trust
is a separate destination used by applications that consume the distro store.

The shared Chef source has no built-in `ca_trust_certificate` resource.

On Ubuntu 24.04 with ca-certificates 20260816ubuntu0.24.04.2, a refresh
added a disposable root CA to the system bundle. After removing its
anchor and refreshing again, OpenSSL rejected that certificate.

## Decision

Add `ca_trust_certificate` to Cinc Client in the shared Chef codebase. Manage one explicitly supplied root CA certificate in the distro's local anchor directory. Refresh the system-generated store with a separate action so callers can batch anchor changes.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:create` | Yes | Validate and install a system trust anchor. |
| `:delete` |  | Remove the resource-owned local anchor. |
| `:refresh` |  | Regenerate the distribution's system trust store. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `name` | String | Resource name | ASCII letters, digits, underscores and hyphens. | Anchor identity in `chef-NAME.crt`. |
| `certificate` | String | Required for create | Exactly one PEM root certificate with CA basic constraints, valid dates and a verified self-signature. Reject private keys and chained inputs. | Certificate to trust. |

### Current state and action steps

- Select the distro adapter. On Debian-family systems target
  `/usr/local/share/ca-certificates` and use `update-ca-certificates`. On
  RHEL-family systems target `/etc/pki/ca-trust/source/anchors` and use
  `update-ca-trust extract`.
- `:create` validates and normalises the PEM, compares the fingerprint and
  metadata, then atomically installs a root-owned `0644` file. Because an
  anchor file must contain a clean PEM, keep ownership and fingerprint in
  an adjacent resource record outside the loader's search path. Check both
  files under a lock. Recover an interrupted two-file update from a pending
  ownership record and a matching on-disk fingerprint.
- `:delete` removes only the owned matching anchor and releases the
  ownership record. `:refresh` invokes the distro tool, checks its status
  and reports changes to the representative generated bundle by digest.
  Other anchors can still trust a removed certificate.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Certificate invalid or expired | Strict PEM, certificate and date validation. | Fail before installing. |
| Unmanaged or unexpectedly changed anchor | Ownership record and fingerprint differ, or destination is unsafe. | Refuse to overwrite or delete it. |
| Adapter or refresh unavailable | Distro and tool checks or command failure. | Report the unmet dependency or failed refresh. Retain valid edits for retry. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 24.04 and Debian 12 | Proposed with ca-certificates and the standard bundle. |
| RHEL 9 | Proposed with the packaged update-ca-trust adapter. |
| Embedded application stores | Outside this system trust interface. |

### Target mode

Defer until privileged remote writes, refresh and bundle verification have SSH coverage.

## Usage examples

### :create

```ruby
ca_trust_certificate 'site-root' do
  certificate lazy { ::File.read('/etc/chef/ca/site-root.pem') }
  action :create
end
```

### :delete

```ruby
ca_trust_certificate 'site-root' do
  action :delete
end
```

### :refresh

```ruby
ca_trust_certificate 'site-root' do
  action :refresh
end
```

## End-to-end testing

Use a disposable CA and a fixture certificate. Verify trust before adding,
after install and refresh, and after deletion and refresh. Check second
converge, unrelated roots and interrupted anchor-record updates.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for generated bundle and OpenSSL verification. |
| kitchen-exec | Good on a disposable Linux runner. |
| Vagrant | Useful for distro trust consumers and reboot persistence. |

Run trust-bundle tests in Ubuntu, Debian and EL containers. Add a guest test for one representative system trust consumer.

## Consequences

### Positive

Cookbooks can manage a system anchor without taking ownership of the generated store or other local certificates.

### Negative

An installed root CA changes trust for a broad class of applications. Applications with independent trust stores require their own configuration.

### Follow-up

Verify anchor refresh and removal on each distro, including duplicate fingerprint reporting and interrupted local-record updates.

## Alternatives considered

Writing the system bundle directly would overwrite packaged certificates and be lost on regeneration. Managing a local anchor lets the distro continue to own extraction and packaging.

## References

- [Debian update-ca-certificates manual](https://manpages.debian.org/bookworm/ca-certificates/update-ca-certificates.8.en.html)
- [Red Hat system-wide trust store](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/securing_networks/using-shared-system-certificates_securing-networks)
- [Chef client-specific trusted certificate resource](https://docs.chef.io/client/19/resources/bundled/chef_client_trusted_certificate/)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
