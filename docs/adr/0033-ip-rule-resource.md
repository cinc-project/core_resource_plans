# 0033. Built-in `ip_rule` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @damacus
- **Consulted / Informed:** Cinc maintainers

## Context

Linux policy-routing rules choose a routing table based on packet selectors
and numerical priority. Cinc Client's `route` resource manages routing
entries (`lib/chef/resource/route.rb:26` and `:32`). Rule selection is a
separate part of route lookup.

The shared Chef source has no built-in `ip_rule` resource.

On Ubuntu 26.04 LTS arm64, iproute2 6.19 added and deleted the example
source rule in a private container network namespace. JSON represented
the host selector without `/32` and the table as a numeric string.

## Decision

Add `ip_rule` to Cinc Client in the shared Chef codebase. Manage one runtime source-based IPv4 or IPv6 table-lookup rule in the host network namespace. A recipe can converge it after the associated table has been provisioned.

### Actions

| Action | Default | Behaviour |
|---|---|---|
| `:add` | Yes | Ensure the exact rule exists at the specified priority. |
| `:delete` |  | Remove only the exact requested rule. |

### Properties

| Property | Type | Default | Accepted values and validation | Description |
|---|---|---|---|---|
| `from` | String | Required | Explicit IPv4 or IPv6 CIDR with nonzero prefix. Canonicalise the network. | Source selector and address family. |
| `priority` | Integer | Required | 1 to 32765. Refuse duplicates or conflicts within the same family. | Rule order. Lower numbers are evaluated earlier. |
| `table` | Integer | Required | Positive non-reserved routing table ID. | Table to look up when the selector matches. |

### Current state and action steps

- Read the relevant family with `ip -json rule show`. Compare selector,
  priority, lookup action, table and other fields that affect matching.
  Normalise numeric and named displays and implicit host prefixes. Refuse
  ambiguous or partial matches at the priority.
- `:add` returns if the exact rule is already present. Verify the target
  table's provisioning, add the full tuple with `ip rule add` and re-read
  the priority. Serialise cooperating writes to the same family and scope.
- `:delete` requires the same exact tuple and passes it to `ip rule delete`.
  Leave rules with other selectors, families or actions in place. A
  missing priority is a no-op.

### Error handling

| Condition | Detection | Behaviour |
|---|---|---|
| Priority collision | Read and normalise JSON rules by family and priority. | Report the observed tuple and refuse mutation. |
| Invalid or missing routing table | Numeric and provisioned-table checks. | Fail before adding the rule. |
| Command or read-back fails | Process status or unexpected fresh state. | Report the result for a new inspection. |

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 26.04 LTS, Debian 13 and RHEL 10 family | Proposed with compatible iproute2 JSON output and netlink privileges. |
| Ubuntu 26.10 and Debian 12 LTS | Smoke and compatibility checks in an isolated namespace. |

See the [platform baseline](../../README.md#platform-baseline) for images and client availability.

### Target mode

Defer until remote namespace identity and control-connection continuity are tested.

## Usage examples

### :add

```ruby
ip_rule 'worker-source-route' do
  from '192.0.2.50/32'
  priority 19_427
  table 427
  action :add
end
```

### :delete

```ruby
ip_rule 'worker-source-route' do
  from '192.0.2.50/32'
  priority 19_427
  table 427
  action :delete
end
```

## End-to-end testing

Use a private namespace and populate an alternate table. Verify selected
route from the scoped source, conflicts, an unchanged second converge,
exact deletion, IPv6 handling and preservation of default rules.

| Kitchen driver | Feasibility |
|---|---|
| Dokken | Good for isolated namespaces with CAP_NET_ADMIN. |
| kitchen-exec | Good on an ephemeral runner with a private namespace. |
| Vagrant | Useful for multiple interfaces and native network-manager interaction. |

Run parser tests across the baseline and scope routing mutations to a private test namespace or VM.

## Consequences

### Positive

Cookbooks can converge table selection while leaving route management to the existing resource or network manager.

### Negative

Runtime rules disappear when the namespace is recreated. The chosen priority and selector can affect active traffic.

### Follow-up

Verify alias and default JSON representations on each iproute2 version. Evaluate persistence in a separate backend only after clear ownership rules.

## Alternatives considered

Using `route` configures a table's entries. A policy rule makes the table eligible for the traffic selectors declared by the cookbook.

## References

- [iproute2 policy rule manual](https://man7.org/linux/man-pages/man8/ip-rule.8.html)
- [Chef Infra Client 19.5.19 resource inventory](https://github.com/chef/chef/tree/v19.5.19/lib/chef/resource)
