# AGENTS.md

This repo holds design records (ADRs) for new built-in Cinc Client resources
for core Linux configuration. It contains Markdown only. There is no code to
build or test.

## Framing

- This work is done as **Cinc**. Call the product "Cinc Client".
- Resources are written in the shared Chef codebase so they can be contributed
  upstream to Chef Infra Client later. Mention upstream contribution only as a
  possible later step.
- Code paths such as `lib/chef/resource/...` and `chef-utils/...` refer to that
  shared codebase. Cite them as `path:line`.

## ADR files

- One ADR per resource, or per small group of resources that only make sense
  together, in `docs/adr/`.
- Name it `NNNN-short-title.md`. Number sequentially from `0001`: take the
  highest number in `docs/adr/` plus one. Cite it as `ADR-0001`.
- Add every new ADR to the index table in `README.md`.

## ADR structure

Start with this status block:

```markdown
# 0002. Built-in `example_resource` resource

- **Status:** Proposed
- **Date:** YYYY-MM-DD
- **Deciders:** @tas50
- **Consulted / Informed:** Cinc maintainers
```

`Consulted / Informed` is always `Cinc maintainers`.

Then these sections, in this order:

1. **Context.** Why the resource is needed, what the client ships today, and
   how the underlying Linux subsystem works.
2. **Decision.** Actions table, properties table (type, default, accepted
   values and validation, description), current-state detection, the steps of
   each action, error handling table (condition, detection, behavior), platform
   support table, and target mode support. Put helpers here as well.
3. **Usage examples.** At least one example per action, and one for each
   helper. Group them under `### :action_name` headings. Write `action`
   explicitly, even for the default action.
4. **End-to-end testing.** What only a real system can prove, and which Test
   Kitchen drivers can prove it in CI (Dokken, `kitchen-exec` on GitHub
   runners, Vagrant). Give a feasibility rating for each driver and a
   proposed CI layout.
5. **Consequences.** `### Positive`, `### Negative`, `### Follow-up`.
6. **Alternatives considered.** Name the obvious alternative in particular,
   and say why it lost.
7. **References.**

Do **not** include:

- a Performance implications section
- a Security implications section. Put any security-relevant behavior, such as
  logging or input validation, in the Decision.
- a code-level resource declaration (`class ... < Chef::Resource`, `provides`,
  `unified_mode`). An ADR describes behavior, not implementation.

## Examples must pass cookstyle

Every Ruby block in an ADR must pass `cookstyle` with no offenses. In practice
that means single-quoted strings unless you interpolate. Cookstyle applies its
Chef cops only inside a cookbook layout, so run the examples as recipes:

```bash
adr=docs/adr/0001-apparmor-profile-resource.md
tmp=$(mktemp -d) && mkdir -p "$tmp/example/recipes"
printf "name 'example'\nversion '0.1.0'\n" > "$tmp/example/metadata.rb"
python3 - "$adr" "$tmp/example/recipes" <<'EOF'
import re, sys
src = open(sys.argv[1]).read()
for i, b in enumerate(re.findall(r"^```ruby\n(.*?)^```", src, re.S | re.M), 1):
    open(f"{sys.argv[2]}/ex{i:02d}.rb", "w").write(b)
EOF
cookstyle "$tmp/example"
```

## Evidence

- Verify claims about tool behavior before you write them down. Run the tool in
  a container of the target distribution (`docker run --rm ubuntu:24.04 ...`)
  and say which version you checked.
- Verify claims about the client against the source, and cite `path:line`.
- If you could not verify something, say so in the text, and list it under
  Follow-up. Do not present a guess as fact.
- Docker Desktop on macOS runs a LinuxKit kernel. It lacks some kernel
  features, AppArmor among them. Note it when a local check is impossible for
  that reason.

## Writing style

- Short sentences, active voice, one idea per sentence.
- Lead with the design. Put the reasons and evidence after it.
- Prefer tables for actions, properties, errors and platforms.
- No filler, jokes or narration of how the document was written.

## Public repository

This repo is public. Do not name customers, private repositories, or private
issues and pull requests anywhere in it, including commit messages.

## Commits

- Sign off every commit: `git commit -s`.
- Credit only the human author. Add no other trailers or footers.
