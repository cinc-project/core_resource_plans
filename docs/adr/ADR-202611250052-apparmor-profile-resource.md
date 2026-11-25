# 202611250052. Built-in `apparmor_profile` resource

- **Status:** Proposed
- **Date:** 2026-11-25
- **Deciders:** @tas50
- **Consulted / Informed:** Cinc maintainers

## Context

This work is done as part of Cinc Client, the community build of Chef Infra
Client. The resource is written in the shared Chef codebase so it can be
contributed upstream to Chef Infra Client later. File paths below refer to that
codebase.

The client ships a complete set of SELinux resources (`selinux_state`,
`selinux_module`, `selinux_boolean`, `selinux_fcontext`, `selinux_port`,
`selinux_permissive`, `selinux_login`, `selinux_user`, `selinux_install`). It ships
nothing for AppArmor. A search of `lib/` in chef and ohai for `apparmor` returns no
matches.

AppArmor is the mandatory access control (MAC) system that is on by default on:

- Ubuntu (all supported releases)
- Debian 10 and later
- SUSE Linux Enterprise 15 and openSUSE Leap 15.x

Arch Linux supports it as an opt-in. SLES 16 switched to SELinux: its
documentation describes SELinux as present by default on 16.0 images and in
enforcing mode by default on 16.1. I have not confirmed whether SLES 16 still
ships AppArmor as an option, or what openSUSE Tumbleweed defaults to on a new
install.

CIS benchmarks for Ubuntu and Debian require every AppArmor profile to be in
enforce or complain mode. Today a cookbook author has two ways to meet that, and
both are fragile:

1. A `cookbook_file` plus an `execute "apparmor_parser -r ..."` with a
   hand-written guard.
2. `execute "aa-enforce ..."`. This needs `apparmor-utils`, a Python package that
   minimal images do not install.

Neither approach validates a profile before it lands in `/etc/apparmor.d`. A bad
file there does not change the running policy, but it fails at the next boot, and
the service it was meant to confine then runs unconfined.

### How AppArmor stores and loads policy

I verified these points in an `ubuntu:24.04` container with AppArmor parser
4.0.1:

- Profiles are files in `/etc/apparmor.d/`. One file can define several profiles
  and child profiles. `apparmor_parser -N <file>` prints their names, for example
  `demo` and `demo//child`.
- `apparmor_parser -Q -K <file>` parses and compiles a file without loading it,
  and exits 1 on a syntax error. A trivial profile compiles in about 10 ms.
- Quoted relative includes (`include "rel"`) resolve against the file's own
  directory. A copy of the file in a temporary directory therefore fails to parse
  even when the real file is valid.
- `apparmor_parser` itself honors the `disable/` and `force-complain/` symlink
  directories. Its strings include `Skipping profile in %s/disable: %s` and
  `Warning: found %s in %s/force-complain, forcing complain mode`. That means
  `apparmor.service` at boot, package postinst scripts, and this resource all
  agree on a profile's mode without anyone rewriting the profile file.
- Packaged profiles end with `include if exists <local/NAME>`. The
  `/etc/apparmor.d/local/` directory is where site-specific additions go, so that
  package upgrades do not overwrite them.
- The `apparmor` package ships `apparmor_parser`, `aa-enabled`, `aa-exec` and
  `aa-status`. `aa-enforce` and `aa-complain` come from `apparmor-utils`, which is
  not installed by default.
- The kernel lists loaded profiles in `/sys/kernel/security/apparmor/profiles`,
  one per line, as `NAME (MODE)`.

One fact about Chef itself constrains the design. A `file` resource's `verify`
command always passes in target mode (`lib/chef/resource/file/verification.rb:120`,
`return true if Chef::Config.target_mode?`). The resource therefore cannot use
`verify` as its only safety check.

## Decision

We will add two things to Cinc Client, written so that they can be contributed
upstream to Chef Infra Client unchanged:

- an `apparmor_profile` resource
- an `apparmor_enabled?` helper that tells a recipe whether AppArmor can be
  used on the node (see "`apparmor_enabled?` helper")

The resource manages one
file in `/etc/apparmor.d/`, an optional site-local addition in
`/etc/apparmor.d/local/`, and the kernel state of the profiles that the file
defines.

The resource works through `apparmor_parser` and the standard symlink
directories only. It does not depend on `apparmor-utils`.

It covers three use cases:

1. **Ship your own profile.** Set `content`, `source` or `template`.
2. **Tune a vendor profile without replacing it.** Set `local_content` or
   `local_source`, and leave the main file to its package.
3. **Set the mode of a vendor profile.** Set no content properties at all. The
   file must already exist.

### Actions

| Action | Default | What it does |
|---|---|---|
| `:create` | yes | Writes the managed files, if any. Makes sure the profile is not disabled. Loads or replaces every profile in the file in the requested `mode`. Converges only when something differs. |
| `:disable` | | Unloads the file's profiles from the kernel, then adds a `disable/` symlink so they stay unloaded after a reboot. Keeps the profile file. Does nothing if the file does not exist. |
| `:delete` | | Unloads the profiles. Removes the `disable/` and `force-complain/` symlinks, the profile file, and `local/<file_name>`. |
| `:reload` | | Replaces the file's profiles in the kernel unconditionally. Use it in a notification, for example after a change to an abstraction or tunable that the profile includes. Does nothing if the profile is disabled. |

Mode is a property and not an action (`:enforce`, `:complain`). Mode is desired
state, and Chef models desired state as properties. `selinux_state` follows the
same rule.

### Properties

| Property | Type | Default | Accepted values / validation | Description |
|---|---|---|---|---|
| `file_name` | String | name property | `/\A[A-Za-z0-9][A-Za-z0-9._+-]*\z/`. It must not be one of `abi`, `abstractions`, `tunables`, `local`, `disable`, `force-complain`, `cache`. | The basename of the file in `base_dir`, for example `usr.sbin.nginx` or `myapp`. The pattern excludes `/`, `..` and leading dots. A file with a leading dot is skipped by the parser. |
| `content` | String | | Exclusive with `source` and `template` | The full profile text. |
| `source` | String, Array | | Exclusive with `content` and `template` | A cookbook file under `files/`, used as in `cookbook_file`. |
| `template` | String | | Exclusive with `content` and `source` | An ERB template under `templates/`. |
| `variables` | Hash | `{}` | Only valid with `template` | Template variables. |
| `cookbook` | String | the current cookbook | | The cookbook to read `source`, `template` or `local_source` from. `desired_state: false`. |
| `local_content` | String | | Exclusive with `local_source` | The text for `local/<file_name>`. |
| `local_source` | String, Array | | Exclusive with `local_content` | A cookbook file for `local/<file_name>`. |
| `mode` | Symbol, String | `:enforce` | `:enforce`, `:complain`. Strings are coerced to symbols. | The mode to load the profiles in. |
| `profile_names` | Array of String | `lazy { apparmor_parser -N output }` | Must be non-empty | The kernel profile names to check. Set it only to skip the `-N` call, or when `:delete` must unload a profile whose file is already gone. `desired_state: false`. |
| `write_cache` | true, false | `true` | | Passes `-W` so the boot cache is updated. `false` passes `-K`. |
| `base_dir` | String | `/etc/apparmor.d` | Absolute path | The policy directory. It exists mostly for tests. `desired_state: false`. |

Modes deliberately left out:

- **`kill`, `unconfined`, `prompt`, `default_allow`.** These are profile flags
  that AppArmor 3.1 and 4.x added. No symlink directory exists for them, so the
  profile text owns them. A user who wants them writes them in the profile header
  and leaves `mode` at `:enforce`.
- **`:disabled`.** This is the `:disable` action, because it also changes which
  files the boot loads.

### Current state

`load_current_value` reads:

- the main file and `local/` file contents, if they exist
- whether `disable/<file_name>` and `force-complain/<file_name>` exist
- the mode of each name in `profile_names`, from
  `/sys/kernel/security/apparmor/profiles`. A name that is not listed is
  "not loaded".

All reads go through `TargetIO::File`, and all commands run through
`shell_out` with argument arrays, never a shell string.

### `:create` step by step

1. **Preflight.** See "Error handling". Stop here if AppArmor is not usable.
2. **Write the files.** Write `local/<file_name>` first, then the main file. The
   main file usually includes the local file, so this order means the main file
   is never checked against a stale local file. The resource keeps the previous
   bytes of both files, or the fact that they did not exist, in memory.
3. **Validate in place.** If either file changed, run
   `apparmor_parser -Q -K -I <base_dir> <base_dir>/<file_name>`. If it fails,
   restore both files to their previous state, then raise. The check runs on the
   real path so that quoted relative includes resolve. It does not use the file
   resource's `verify`, which target mode skips.
4. **Enable.** Remove `disable/<file_name>` if it exists, and log it at info
   level. `:create` means enabled.
5. **Mode.** For `:complain`, create `force-complain/<file_name>` as a symlink to
   `../<file_name>`. For `:enforce`, remove that symlink if it exists.
6. **Load.** If a file or symlink changed, if a profile is not loaded, or if a
   loaded mode differs from `mode`, run
   `apparmor_parser -r -W <base_dir>/<file_name>` (or `-K` when `write_cache` is
   false). The parser applies `force-complain` itself.
7. **Confirm.** Read the kernel state again. If a profile is still in the wrong
   mode, raise. This happens when the profile header has `flags=(complain)` and
   `mode` is `:enforce`. The message tells the user to remove the flag or set
   `mode :complain`.

With no content properties set, steps 2 and 3 are skipped. The main file must
already exist, or the resource raises.

### `:disable` and `:delete` ordering

The resource runs `apparmor_parser -R` **before** it creates the `disable/`
symlink. The parser skips files that have a `disable/` symlink, and I have not yet
confirmed whether that skip also applies to `-R`. Unloading first is correct
either way.

If the main file is already gone, `:delete` unloads each name in `profile_names`
by writing it to `/sys/kernel/security/apparmor/.remove`. Without `profile_names`,
it logs a warning that the loaded profiles cannot be identified, and leaves them.

`:delete` does not check whether a package owns the file. A deleted file that
belongs to a package comes back on the next package upgrade. The documentation
tells users to choose `:disable` for vendor profiles. A package-ownership check
would need `dpkg -S` or `rpm -qf` on every run, plus a target-mode code path, for
a warning.

### Error handling

| Condition | Detection | Behavior |
|---|---|---|
| Not a Linux node | The resource is registered for Linux only | The client raises its usual "no resource found" error. |
| Kernel has no AppArmor, or it is off | First check of `apparmor_enabled?`: `/sys/module/apparmor/parameters/enabled` is missing or is not `Y` | Raise `Chef::Exceptions::UnsupportedAction`: "AppArmor is not enabled in the running kernel." On RHEL family nodes, add "This platform uses SELinux; see the selinux_* resources." |
| securityfs is not mounted (usual in containers) | Second check of `apparmor_enabled?`: `/sys/kernel/security/apparmor` is missing | Raise. The message names the missing path and says that containers need a privileged run with securityfs mounted. |
| Userspace tools missing | `which("apparmor_parser")` returns nil | Raise: "Install the apparmor package." The resource does not install it. A separate `apparmor_install` resource can do that, as `selinux_install` does for SELinux. |
| More than one of `content` / `source` / `template` set, or `variables` without `template`, or `local_content` together with `local_source` | `after_created` | Raise `Chef::Exceptions::ValidationFailed` at compile time, before any change. |
| Invalid `file_name` or `mode` | property `callbacks` and `equal_to` | Raise `Chef::Exceptions::ValidationFailed` at compile time. |
| `:create` with no content properties and no file on disk | step 2 | Raise: "<path> does not exist; set content, source or template, or install the package that provides it." |
| The profile does not parse | `apparmor_parser -Q` exits non-zero | Restore both files, then raise `Mixlib::ShellOut::ShellCommandFailed`. The message includes the parser's stderr, which names the file and line. |
| The file defines no profiles | `-N` output is empty | Raise: "<path> contains no profile definitions." |
| The profile was loaded but is in the wrong mode | step 7 | Raise, with the `flags=` explanation. |
| The kernel rejects the load | `apparmor_parser -r` exits non-zero | Restore the files. The kernel keeps the previous policy, because a failed replace is atomic. Raise with stderr. |
| The profile's `abi` is newer than the kernel's | The parser writes a warning and still loads | Forward each parser warning with `logger.warn`. Do not fail. |
| Cache directory not writable | The parser warns | Forward the warning. Do not fail. |

**Why-run.** The preflight checks run. When AppArmor is unavailable, why-run
reports that the action would fail and continues, using Chef's standard
requirement and `whyrun_assumption` pattern. Every change is wrapped in
`converge_by` with a message such as
`load profile myapp (2 profiles) in enforce mode`.

**Logging.** A switch to complain mode and a disable both take confinement away,
so they must be easy to find afterwards. The resource logs each mode change and
each disable at info level, with the resource name and the profile names.

### Platform support

| Platform | Support |
|---|---|
| Ubuntu 20.04 and later | Supported, tested |
| Debian 11 and later | Supported, tested |
| SLES 15, openSUSE Leap 15.x | Supported, tested |
| SLES 16 | Not a target. SELinux is the default. If AppArmor is turned on, the resource should work, but this is not tested. |
| Arch Linux, other distros with AppArmor turned on | Expected to work, not tested |
| RHEL family, Fedora, Amazon Linux | Not supported. The preflight check raises. |

### Target mode

The resource declares `target_mode support: :full`. Every file operation uses the
`file`, `cookbook_file`, `template` and `link` resources, which already support
target mode fully, or `TargetIO::File`. Every command uses `shell_out`, which runs
on the target. Validation (step 3) does not use `verify`, so it runs in target
mode as well.

### `apparmor_enabled?` helper

We will add `apparmor_enabled?` to `ChefUtils::DSL::Introspection` in
chef-utils. That module holds the other live-system checks, such as `systemd?`
and `docker?`. Through `Chef::DSL::PlatformIntrospection` the helper is
available in recipes, in resource guards, in custom resources and in libraries.

**Question it answers.** "Can policy be managed on this node right now?" A
kernel with AppArmor turned on is not enough. A container on an AppArmor host
has the kernel feature but usually has no policy interface. Returning true there
would let a guard pass, and the resource would then fail. The helper therefore
returns true only when both checks pass:

1. `/sys/module/apparmor/parameters/enabled` exists and its content, stripped,
   is `Y`. sysfs is always mounted, and this file is world-readable.
2. The directory `/sys/kernel/security/apparmor` exists, which means securityfs
   is mounted and the policy interface is present.

These are the same two checks `aa-enabled` makes. On a container without
AppArmor I saw `aa-enabled` print `No - not available on this system` and exit 1.
The helper does not run `aa-enabled`, because a node can have AppArmor in the
kernel without the `apparmor` package installed.

**Signature and behavior.**

| Item | Value |
|---|---|
| Name | `apparmor_enabled?(node = __getnode)` |
| Returns | `true` or `false`, never `nil` |
| `node` | Accepted to match the other introspection helpers. Not used. |
| Non-Linux nodes | `false`. The sysfs paths do not exist there, so no platform check is needed. |
| Read errors (`Errno::EACCES`, `Errno::ENOENT`, `IOError`) | `false`. A guard must never raise. |
| Cost | Two file checks and one small read. No command runs. No caching, because the result can change during a run, for example when a recipe mounts securityfs. |
| Target mode | Works. It uses `file_exist?` and `file_read` from `ChefUtils::DSL::TrainHelpers`, which go through the Train connection in target mode. |
| Userspace tools | Not checked. If the kernel side works but `apparmor_parser` is missing, the helper returns true and the resource raises "Install the apparmor package". Installing a package is a fix the user should make, not a reason to skip the resource. |

The resource's preflight runs the same two checks itself, so that it can give a
specific error message for each one. The helper only returns a boolean.

A related bug turned up while checking `TrainHelpers`. In target mode,
`file_directory?` calls `__transport_connection.file(filename)` but its argument
is named `path` (`chef-utils/lib/chef-utils/dsl/train_helpers.rb`), so it raises
`NameError`. The helper avoids `file_directory?` and uses `file_exist?` on the
directory. The bug should still be fixed on its own.

## Usage examples

### `:create`

`:create` is the default action. The examples set it explicitly to show which
action runs.

**Ship and enforce a profile for your own service.**

```ruby
apparmor_profile "myapp" do
  content <<~PROFILE
    abi <abi/3.0>,
    include <tunables/global>

    profile myapp /opt/myapp/bin/myapp {
      include <abstractions/base>
      include <abstractions/nameservice>

      /opt/myapp/** r,
      /opt/myapp/bin/myapp ix,
      /var/lib/myapp/** rwk,
      /var/log/myapp/*.log w,
      network inet stream,

      include if exists <local/myapp>
    }
  PROFILE
  action :create
  notifies :restart, "service[myapp]"
end
```

**Roll out a new profile in complain mode, then switch it to enforce.** A
valid profile can still be too strict for its workload. Load it in complain
mode, read the denials with `journalctl -k | grep apparmor=`, adjust the
profile, and then change `mode` to `:enforce`.

```ruby
apparmor_profile "myapp" do
  template "myapp.apparmor.erb"
  variables data_dir: node["myapp"]["data_dir"]
  mode node["myapp"]["apparmor_mode"] # "complain" first, "enforce" later
  action :create
end
```

**Add site rules to a packaged profile without replacing it.**

```ruby
apparmor_profile "usr.sbin.nginx" do
  local_content <<~RULES
    /srv/www/** r,
    /run/myapp/upstream.sock rw,
  RULES
  action :create
end
```

**Put packaged profiles into enforce mode (CIS 1.3.x).** No content
properties are set, so the resource manages only the mode.

```ruby
%w(usr.sbin.cupsd usr.sbin.rsyslogd usr.bin.man).each do |p|
  apparmor_profile p do
    mode :enforce
    action :create
    only_if { ::File.exist?("/etc/apparmor.d/#{p}") }
  end
end
```

**Turn a disabled profile back on.** `:create` removes the `disable/` symlink
and loads the profile again.

```ruby
apparmor_profile "usr.sbin.mysqld" do
  action :create
end
```

### `:disable`

**Disable a profile that breaks a workload, and keep it disabled after a
reboot.** The profile file stays in place, so a package upgrade does not bring
the profile back into force.

```ruby
apparmor_profile "usr.sbin.mysqld" do
  action :disable
end
```

### `:delete`

**Remove a profile that you shipped earlier.** The resource unloads the
profile and removes the file, its symlinks and its `local/` file.

```ruby
apparmor_profile "myapp" do
  action :delete
end
```

**Remove a profile whose file is already gone.** `profile_names` tells the
resource which loaded profiles to unload, because it cannot read the names
from the file.

```ruby
apparmor_profile "legacy-agent" do
  profile_names %w(legacy-agent legacy-agent//helper)
  action :delete
end
```

### `:reload`

**Reload a profile after a shared abstraction changes.** The profile file did
not change, so `:create` would converge nothing. `:reload` replaces the
profile in the kernel so that it picks up the new abstraction.

```ruby
apparmor_profile "myapp" do
  source "myapp.apparmor"
  action :create
end

cookbook_file "/etc/apparmor.d/abstractions/myorg-base" do
  source "myorg-base"
  notifies :reload, "apparmor_profile[myapp]", :delayed
end
```

### `apparmor_enabled?`

**Skip nodes where AppArmor cannot be used, such as containers and RHEL.**

```ruby
apparmor_profile "myapp" do
  source "myapp.apparmor"
  only_if { apparmor_enabled? }
end
```

**Branch between AppArmor and SELinux in one cookbook.**

```ruby
if apparmor_enabled?
  apparmor_profile "myapp" do
    source "myapp.apparmor"
  end
else # RHEL family nodes
  selinux_module "myapp" do
    source "myapp.te"
  end
end
```

## End-to-end testing

### What has to be proven

A unit spec can mock `shell_out`. Only a real kernel can prove that:

1. a profile loads, and shows up in `/sys/kernel/security/apparmor/profiles` in
   the requested mode
2. switching between enforce and complain works, and a second run converges
   nothing
3. a profile that does not parse is rejected, and the previous file and kernel
   state are restored
4. `:disable` unloads the profile and survives `systemctl restart apparmor`
5. the profile actually confines a process. `aa-exec -p myapp cat /etc/denied`
   must fail in enforce mode and succeed, with an audit entry, in complain mode.
   `aa-exec` is in the base `apparmor` package.
6. `apparmor_enabled?` returns true exactly when `aa-enabled` exits 0. In
   Dokken it returns false before the securityfs mount and true after it.
   Unit specs in `chef-utils/spec/unit/dsl/introspection_spec.rb` cover the
   other cases with stubbed files: module missing, `N`, `Y` without securityfs,
   `Y` with securityfs, and a read error.

### Feasibility per Kitchen driver

| Driver | Feasible? | Notes |
|---|---|---|
| Dokken on a local Mac | **No** | I checked: the Docker VM kernel (`7.0.14-linuxkit`) has no `/sys/module/apparmor` and no `/sys/kernel/security/lsm`. Unit specs are the only local option. |
| Dokken on GitHub `ubuntu-latest` (current `kitchen.dokken.yml`, `privileged: true`) | **Partly** | The container shares the runner's Ubuntu kernel, which has AppArmor turned on. A `pre_converge` step must `mount -t securityfs securityfs /sys/kernel/security`. A privileged container loads profiles into the **host** kernel's policy, because Docker gives a container no AppArmor namespace of its own. That is acceptable on an ephemeral runner, but tests 1 to 3 and 6 run against the host kernel with the container's parser version. Test 4 needs systemd in the container, which the dokken images provide. This is good enough for Debian and SUSE userspace coverage. It is not proof that a real Debian or SUSE kernel behaves the same. Not yet verified on a runner. |
| `kitchen-exec` on GitHub `ubuntu-24.04` / `ubuntu-24.04-arm` | **Yes, recommended** | Same pattern as the existing `kitchen.exec.macos.yml` and `kitchen.exec.windows.yml`. The client converges the runner VM itself, with its real kernel, real systemd and real `apparmor.service`. All six tests run. The runner is discarded afterwards, so loading policy into it is harmless. |
| `kitchen-vagrant` with libvirt, on runners that have `/dev/kvm` | Yes, but costly | Real Debian 12 and SLES 15 kernels. Gives full coverage at the cost of VM boot time and box maintenance. Defer unless Dokken coverage turns out to be wrong. |

### Proposed CI layout

The paths below are in the upstream chef repository, which is where the Kitchen
suites live and where the resource lands when it is contributed. Whether Cinc's
own pipeline should also run these suites is still open.

1. Add a `kitchen.exec.linux.yml` and an `end-to-end-ubuntu-24.04` matrix entry
   in `.github/workflows/kitchen.yml`, for amd64 and arm64. This is the release
   gate for the resource.
2. Add an `apparmor` recipe to the `end_to_end` test cookbook. It runs every
   usage example above, plus a deliberately broken profile wrapped in
   `ignore_failure true` followed by a `ruby_block` that asserts the restore.
3. Run the Dokken suites for Debian 12 and openSUSE Leap 15 with the securityfs
   `pre_converge` mount. Mark them non-blocking until a few runs have shown they
   are stable.
4. Write the verification as InSpec controls that read
   `/sys/kernel/security/apparmor/profiles` and run `aa-exec`. The workflow
   currently has the `kitchen verify` step commented out
   (`.github/workflows/kitchen.yml:355-357`), so until verify comes back, the recipe
   asserts state in `ruby_block` resources that raise on a mismatch.
5. Make the second converge an idempotence check: `kitchen converge` twice, and
   fail if the second run reports any updated resources.

First task: run a one-off spike on an `ubuntu-latest` runner. It should confirm
that a privileged Dokken container can mount securityfs and load a profile. That
result decides whether item 3 is worth building.

## Consequences

### Positive

- AppArmor reaches parity with the SELinux resources. Ubuntu, Debian and SUSE
  users can meet the CIS AppArmor controls without `execute` blocks.
- Mode stays consistent across Chef, boot and package scripts, because all three
  use the parser's own symlink directories.
- A bad profile can no longer land on disk and break the next boot.
- `apparmor-utils` and Python are not needed.
- Cookbooks that support both AppArmor and SELinux platforms can guard
  resources with `apparmor_enabled?` instead of copying sysfs checks around.

### Negative

- Without `profile_names`, every run calls `apparmor_parser -N` once per
  resource. That costs a few milliseconds for a small profile, more for profiles
  with many includes.
- The resource cannot detect drift where the kernel holds an edited policy under
  the same profile name and mode while the file is unchanged. Running `:reload` on
  a schedule covers this.
- Cookbooks that define their own `apparmor_profile` resource will clash with the
  built-in one. Search Supermarket and GitHub before choosing the name.

### Follow-up

- Fix `file_directory?` in `ChefUtils::DSL::TrainHelpers` for target mode.
- `apparmor_install` resource: install the `apparmor` package (plus
  `apparmor-profiles` and `apparmor-profiles-extra` as options), and turn on the
  boot parameter.
- Possibly an `apparmor_state` resource that turns AppArmor on or off at boot,
  mirroring `selinux_state`. It needs its own ADR, because it edits the kernel
  command line, which overlaps with the planned `kernel_args` resource.
- An Ohai `apparmor` plugin that reports whether AppArmor is enabled and which
  profiles are loaded in which mode.
- Confirm whether `-R` honors `disable/` (see "`:disable` and `:delete`
  ordering").
- Confirm that the parser skips Chef's staging tempfiles, which start with
  `.chef-`, if a run crashes between write and rename.

## Alternatives considered

### Shell out to `aa-enforce`, `aa-complain` and `aa-disable`

These tools do what the actions need. They are in `apparmor-utils`, which pulls
in Python and is missing from minimal and cloud images. `aa-complain` also
rewrites the profile's `flags=`, so the client would fight its own `content` on every
run. Rejected.

### `:enforce` and `:complain` as actions instead of a `mode` property

This mirrors the `aa-*` command names. But `action [:create, :enforce]` reads
badly, and an action cannot be the target of `property_is_set?` or show up in
resource reporting as a desired-state change. Rejected, to stay consistent with
`selinux_state`.

### Separate resources per concern

One resource for the profile file, one for `local/` additions
(`apparmor_local_profile`), one for mode (`apparmor_profile_mode`). Each would be
simpler, but they all depend on the same reload. Three resources would need
notifications between them to reload once, and users would have to wire that up
correctly. Rejected. Use case 2 (local additions only) and use case 3 (mode
only) are covered by leaving the other properties unset.

### Rely on the file resource's `verify` for validation

This is the idiomatic Chef way. It always passes in target mode
(`verification.rb:120`), and it checks a temporary copy, which breaks quoted
relative includes. Rejected in favor of in-place validation with restore.

### A generic `mac_profile` resource for both AppArmor and SELinux

The two systems have no common model. AppArmor uses path-based profiles.
SELinux uses type enforcement with labels, modules and booleans. A shared
interface would expose the overlap only, which is almost nothing. The SELinux
resources already exist with their own names. Rejected.

### Put the AppArmor check in an Ohai attribute instead of a helper

Ohai collects data once, at the start of a run. A recipe that mounts securityfs
or installs AppArmor would then read a stale value for the rest of the run.
The helper costs two file checks, so a live check is cheap. An Ohai plugin that
reports loaded profiles is still useful, but for reporting, and it stays a
follow-up.

### Check only the kernel parameter in `apparmor_enabled?`

This is simpler, and it matches the name more literally. It returns true in
containers that cannot load policy, so `only_if { apparmor_enabled? }` would
pass and the resource would then fail. Rejected: a guard helper must answer
the question the guard actually asks.

### Read `/sys/kernel/security/lsm` in `apparmor_enabled?`

This file lists the active security modules. It lives in securityfs, so it is
missing in the same cases as the policy interface, and it adds no information
beyond the two chosen checks. Rejected.

## References

- AppArmor core policy reference: <https://gitlab.com/apparmor/apparmor/-/wikis/AppArmor_Core_Policy_Reference>
- `apparmor_parser(8)`, `apparmor.d(5)`, `aa-exec(1)` man pages
- SUSE Linux Enterprise Server 16.1 documentation, SELinux: <https://documentation.suse.com/sles/16.1/html/SLES-SELinux/>
- Ubuntu Server documentation, AppArmor: <https://documentation.ubuntu.com/server/how-to/security/apparmor/>
- CIS Ubuntu Linux 24.04 LTS Benchmark, section 1.3 (Mandatory Access Control)
- Chef introspection helpers: `chef-utils/lib/chef-utils/dsl/introspection.rb`
- Chef SELinux resources: `lib/chef/resource/selinux_*.rb`
- Chef file verification in target mode: `lib/chef/resource/file/verification.rb:120`
- Chef Kitchen workflow: `.github/workflows/kitchen.yml`, `kitchen-tests/kitchen.dokken.yml`
