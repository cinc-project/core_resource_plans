# Core resource plans

Design records for new built-in Cinc Client resources. The goal is that a person
can configure every core part of a modern Linux system without `execute` blocks
or community cookbooks.

The work is done as Cinc. Resources are written in the shared Chef codebase so
they can be contributed upstream to Chef Infra Client later.

## ADR format

Each resource gets one ADR in `docs/adr/`, named `NNNN-short-title.md`. Numbers
are sequential, starting at `0001`: a new ADR takes the highest number in
`docs/adr/` plus one. Cite an ADR as `ADR-0001`. The heading is `# 0001. Title`.

Each ADR has these sections:

- Status block: Status, Date, Deciders, Consulted / Informed. Consulted / Informed is always "Cinc maintainers".
- Context
- Decision: actions, properties with accepted values, the steps of each action,
  error handling, platform support, target mode
- Usage examples: at least one example per action, plus any helpers. Every
  Ruby example must pass `cookstyle` with no offenses.
- End-to-end testing: whether the resource can be tested in CI with Test Kitchen
- Consequences: Positive, Negative, Follow-up
- Alternatives considered
- References

## Platform baseline

Use the latest stable release and the current LTS release where the
distribution publishes both. These are the default test targets as of
25 November 2026. Each ADR lists the applicable families, prerequisites
and proposed feature coverage.

| Family | Current targets | Test environment |
|---|---|---|
| Ubuntu | 26.10 (latest stable), 26.04 LTS | Use `ubuntu:26.10` for Linux utility and source-built client smoke tests. Use `ubuntu:26.04` or the GitHub-hosted `ubuntu-26.04` and `ubuntu-26.04-arm` runners for packaged-client tests. |
| Debian | 13 (stable), 12 (LTS) | Use `debian:13` for primary coverage and `debian:12` for compatibility checks where the required feature is available. |
| Enterprise Linux | RHEL 10 (current major) | Use `almalinux:10` for EL10 userspace fixtures. Use a bootable guest of the relevant distribution when the test depends on its kernel, boot loader or service integration. EL9 remains available for compatibility checks. |

Cinc publishes Ubuntu LTS builds. Omnitruck returns the Ubuntu 26.04 package
for a 26.10 request, so the interim-release smoke tests use distro tools or a
client built from the candidate source. Resource tests use the candidate
client build for the target. Docker images provide userspace coverage. Use
dedicated guests for kernel, boot and reboot tests. Record historical results
against the versions actually tested.

SUSE targets are resource-specific. The AppArmor proposal covers SLES 15 SP7
as a compatibility target. SLES 16.1 and openSUSE Leap 16.1 default to SELinux.
Leap 16.1 can be tested with AppArmor enabled explicitly. The published
`registry.suse.com/bci/bci-base:15.7` and `opensuse/leap:16.1` images
provide userspace fixtures.

Release and image references

- [Ubuntu lifecycle](https://endoflife.date/ubuntu) and [official Ubuntu images](https://hub.docker.com/_/ubuntu)
- [Debian lifecycle](https://endoflife.date/debian) and [official Debian images](https://hub.docker.com/_/debian)
- [RHEL lifecycle](https://endoflife.date/rhel) and [official AlmaLinux images](https://hub.docker.com/_/almalinux)
- [SLES lifecycle](https://endoflife.date/sles) and [openSUSE Leap 16.1 announcement](https://news.opensuse.org/2026/11/03/leap-unites-traditional-and-immutable)
- [Cinc build and release policy](https://cinc-project.gitlab.io/docs/platforms/) and [GitHub-hosted Ubuntu 26.04 runners](https://github.blog/changelog/2026-09-17-ubuntu-26-generally-available-and-latest-migration/)

## Index

| ADR | Resource | Status |
|---|---|---|
| [ADR-0001](docs/adr/0001-apparmor-profile-resource.md) | `apparmor_profile` | Proposed |
| [ADR-0002](docs/adr/0002-systemd-tmpfiles-resource.md) | `systemd_tmpfiles` | Proposed |
| [ADR-0003](docs/adr/0003-systemd-sysusers-resource.md) | `systemd_sysusers` | Proposed |
| [ADR-0004](docs/adr/0004-udev-rule-resource.md) | `udev_rule` | Proposed |
| [ADR-0005](docs/adr/0005-auditd-rule-resource.md) | `auditd_rule` | Proposed |
| [ADR-0006](docs/adr/0006-kernel-args-resource.md) | `kernel_args` | Proposed |
| [ADR-0007](docs/adr/0007-systemd-drop-in-resource.md) | `systemd_drop_in` | Proposed |
| [ADR-0008](docs/adr/0008-systemd-journald-resource.md) | `systemd_journald` | Proposed |
| [ADR-0009](docs/adr/0009-logrotate-rule-resource.md) | `logrotate_rule` | Proposed |
| [ADR-0010](docs/adr/0010-file-acl-resource.md) | `file_acl` | Proposed |
| [ADR-0011](docs/adr/0011-file-capability-resource.md) | `file_capability` | Proposed |
| [ADR-0012](docs/adr/0012-crypttab-entry-resource.md) | `crypttab_entry` | Proposed |
| [ADR-0013](docs/adr/0013-hosts-entry-resource.md) | `hosts_entry` | Proposed |
| [ADR-0014](docs/adr/0014-nsswitch-entry-resource.md) | `nsswitch_entry` | Proposed |
| [ADR-0015](docs/adr/0015-sshd-config-resource.md) | `sshd_config` | Proposed |
| [ADR-0016](docs/adr/0016-networkmanager-connection-resource.md) | `networkmanager_connection` | Proposed |
| [ADR-0017](docs/adr/0017-lvm-physical-volume-resource.md) | `lvm_physical_volume` | Proposed |
| [ADR-0018](docs/adr/0018-lvm-volume-group-resource.md) | `lvm_volume_group` | Proposed |
| [ADR-0019](docs/adr/0019-lvm-logical-volume-resource.md) | `lvm_logical_volume` | Proposed |
| [ADR-0020](docs/adr/0020-authorized-key-resource.md) | `authorized_key` | Proposed |
| [ADR-0021](docs/adr/0021-subordinate-id-resource.md) | `subordinate_id` | Proposed |
| [ADR-0022](docs/adr/0022-chrony-source-resource.md) | `chrony_source` | Proposed |
| [ADR-0023](docs/adr/0023-systemd-coredump-resource.md) | `systemd_coredump` | Proposed |
| [ADR-0024](docs/adr/0024-nftables-table-resource.md) | `nftables_table` | Proposed |
| [ADR-0025](docs/adr/0025-ca-trust-certificate-resource.md) | `ca_trust_certificate` | Proposed |
| [ADR-0026](docs/adr/0026-systemd-resolved-resource.md) | `systemd_resolved` | Proposed |
| [ADR-0027](docs/adr/0027-block-device-partition-resource.md) | `block_device_partition` | Proposed |
| [ADR-0028](docs/adr/0028-filesystem-format-resource.md) | `filesystem_format` | Proposed |
| [ADR-0029](docs/adr/0029-xfs-project-quota-resource.md) | `xfs_project_quota` | Proposed |
| [ADR-0030](docs/adr/0030-nfs-export-resource.md) | `nfs_export` | Proposed |
| [ADR-0031](docs/adr/0031-autofs-map-resource.md) | `autofs_map` | Proposed |
| [ADR-0032](docs/adr/0032-systemd-link-resource.md) | `systemd_link` | Proposed |
| [ADR-0033](docs/adr/0033-ip-rule-resource.md) | `ip_rule` | Proposed |
| [ADR-0034](docs/adr/0034-pam-access-rule-resource.md) | `pam_access_rule` | Proposed |
| [ADR-0035](docs/adr/0035-zram-swap-resource.md) | `zram_swap` | Proposed |
| [ADR-0036](docs/adr/0036-fapolicyd-rule-resource.md) | `fapolicyd_rule` | Proposed |
