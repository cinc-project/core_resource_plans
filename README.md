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

## Index

| ADR | Resource | Status |
|---|---|---|
| [ADR-0001](docs/adr/0001-apparmor-profile-resource.md) | `apparmor_profile` | Proposed |
| [ADR-0002](docs/adr/0002-systemd-tmpfiles-resource.md) | `systemd_tmpfiles` | Proposed |
| [ADR-0003](docs/adr/0003-systemd-sysusers-resource.md) | `systemd_sysusers` | Proposed |
| [ADR-0004](docs/adr/0004-udev-rule-resource.md) | `udev_rule` | Proposed |
| [ADR-0005](docs/adr/0005-auditd-rule-resource.md) | `auditd_rule` | Proposed |
| [ADR-0006](docs/adr/0006-kernel-args-resource.md) | `kernel_args` | Proposed |
