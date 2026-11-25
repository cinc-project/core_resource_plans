# Core resource plans

Design records for new built-in Cinc Client resources. The goal is that a person
can configure every core part of a modern Linux system without `execute` blocks
or community cookbooks.

The work is done as Cinc. Resources are written in the shared Chef codebase so
they can be contributed upstream to Chef Infra Client later.

## ADR format

Each resource gets one ADR in `docs/adr/`, named `ADR-YYYYMMDDHHMM-short-title.md`.
Take the timestamp once, when you create the file, with `date -u +%Y%m%d%H%M`.

Each ADR has these sections:

- Status block: Status, Date, Deciders, Consulted / Informed. Consulted / Informed is always "Cinc maintainers".
- Context
- Decision: actions, properties with accepted values, the steps of each action,
  error handling, platform support, target mode
- Usage examples: at least one example per action, plus any helpers
- End-to-end testing: whether the resource can be tested in CI with Test Kitchen
- Consequences: Positive, Negative, Follow-up
- Alternatives considered
- References

## Index

| ADR | Resource | Status |
|---|---|---|
| [ADR-202611250052](docs/adr/ADR-202611250052-apparmor-profile-resource.md) | `apparmor_profile` | Proposed |
