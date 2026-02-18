# ACL Base Image — Requested Changes for AKS Node Provisioning

---

## Background

**What is AgentBaker?** AgentBaker is the AKS (Azure Kubernetes Service) component that provisions new nodes. It generates configuration scripts that run on each VM to install Kubernetes components, configure networking, set up container runtimes, etc. It builds VHD (Virtual Hard Disk) base images using Packer, and also generates CSE (Custom Script Extension) scripts that run at provisioning time on those VHDs.

**What is the ACL VHD?** We are building a new VHD based on the ACL (Azure Container Linux) base image for use as AKS node images.

**What happened?** While integrating the ACL base image into the AgentBaker VHD pipeline, we hit several gaps — missing packages, configs, or files that upstream Flatcar provides but ACL doesn't yet. We've worked around them in AgentBaker for now, but each task below is something that would be better addressed in the base image. This would let us remove the workarounds.

**Relevant repos:**
| Repo | Purpose |
|------|---------|
| `acl-scripts` | ACL image build scripts |
| `AgentBaker` | AKS node provisioning ([Azure/AgentBaker](https://github.com/Azure/AgentBaker), branch [`aadagarwal/acl-v20260127`](https://github.com/Azure/AgentBaker/tree/aadagarwal/acl-v20260127)) |
| `mantle` | Flatcar/ACL platform tests |

---

## How to read this document

Each task is self-contained. For each one you'll find:

1. **What's wrong** — the symptom we see when running AKS on ACL
2. **Why it happens** — root cause traced to a specific file/line in `acl-scripts`
3. **What to change** — a concrete fix with file paths and code snippets
4. **How to verify** — how to confirm the fix works

---

## Task Summary

| # | Task | File to change |
|---|------|----------------|
| 1 | [Install `azure-vm-utils` package](#task-1-install-azure-vm-utils-package) | `package_catalog.sh` |
| 2 | [Ship Azure-optimized chrony config](#task-2-ship-azure-optimized-chrony-config) | `coreos-base/oem-azure/files/manglefs.sh` |
| 3 | [Add `tmpfiles.d/logrotate.conf`](#task-3-add-tmpfilesdlogrotateconf) | `build_library/rpm/build_image_util.sh` |
| 4 | [Add boot-time systemd preset processing](#task-4-add-boot-time-systemd-preset-processing) | New service file |
| 5 | [Fix `update-ssh-keys` cross-device mv bug](#task-5-fix-update-ssh-keys-cross-device-mv-bug) | `additional_files/update-ssh-keys` |
| 6 | [Remove or fix `/etc/profile.d/umask.sh`](#task-6-remove-or-fix-etcprofiledumasksh) | `build_image_util.sh` |
| 7 | [Evaluate cloud-init distro detection](#task-7-evaluate-cloud-init-distro-detection) | Investigation |

---

## Task 1: Install `azure-vm-utils` package

### What's wrong

Azure VMs rely on udev rules from `azure-vm-utils` to create `/dev/disk/azure/{root,os,resource}` symlinks. These symlinks are required by multiple AKS components (e.g., `disk_queue.service`) and are fundamental to any Azure VM operation. Without them, disk identification breaks — particularly on NVMe-capable VM sizes where the `/usr/sbin/azure-nvme-id` binary (also from `azure-vm-utils`) is needed.

### Why it happens

In `build_library/rpm/package_catalog.sh` (line 248):
```bash
["sys-apps/azure-vm-utils"]="SKIP"
```

This causes both the udev rules file (`80-azure-disk.rules`) and the `azure-nvme-id` binary to be absent from the ACL image. Upstream Flatcar installs these via the `azure-vm-utils` Gentoo ebuild.

### What to change

In `build_library/rpm/package_catalog.sh`, replace:
```bash
["sys-apps/azure-vm-utils"]="SKIP"
```
with the Azure Linux RPM equivalent:
```bash
["sys-apps/azure-vm-utils"]="azure-vm-utils"
```

If the RPM name differs in Azure Linux 3, check with:
```bash
tdnf search azure-vm-utils
# or
tdnf search azure-disk
```

### How to verify

After building the image, check:
```bash
# The udev rules file exists
test -f /usr/lib/udev/rules.d/80-azure-disk.rules && echo "OK: udev rules present"

# The NVMe ID binary exists
test -f /usr/sbin/azure-nvme-id && echo "OK: azure-nvme-id present"

# On an Azure VM, symlinks are created at boot
ls -la /dev/disk/azure/
# Expected: root, os, resource symlinks pointing to real devices
```

### What we had to do as a workaround

We added ~65 lines of inline udev rules in AgentBaker's `vhdbuilder/packer/pre-install-dependencies.sh` (a full copy of the `azure-vm-utils` v0.7.0 rules). This works for basic SCSI disk identification but cannot handle NVMe disks (the `azure-nvme-id` binary is still missing). Once the package is installed in the base image, we can remove this workaround entirely.

**AgentBaker commit**: [`04075c8`](https://github.com/Azure/AgentBaker/commit/04075c8463) — _fix: install azure-vm-utils udev rules on ACL before disk_queue_

---

## Task 2: Ship Azure-optimized chrony config

### What's wrong

ACL ships chrony with the **default Azure Linux RPM config** (`makestep 1.0 3`), which only corrects the clock during the first 3 NTP updates, then switches to gradual slewing. In Azure, VMs can wake from hibernation or migration with large time offsets that need immediate correction. The correct Azure config uses `makestep 1.0 -1` (always step) and `refclock PHC /dev/ptp_hyperv` (use Hyper-V PTP clock for sub-microsecond accuracy).

The correct config files **already exist in the acl-scripts repo** — they're just not making it into the final image.

### Why it happens

The Flatcar overlay at `sdk_container/src/third_party/coreos-overlay/coreos-base/oem-azure/files/` contains the correct Azure-optimized files:
- `chrony.conf` — with `makestep 1.0 -1` and `refclock PHC /dev/ptp_hyperv`
- `chrony-hyperv.conf` — systemd drop-in for `chronyd.service` that adds `Wants=dev-ptp_hyperv.device` and `After=dev-ptp_hyperv.device` (ensures the PTP device is available before chronyd starts)
- `var-chrony.conf` — tmpfiles config to create `/var/lib/chrony` at boot
- `etc-chrony.conf` — tmpfiles config that creates `/etc/chrony/` and a symlink `/etc/chrony/chrony.conf -> ../../usr/share/oem-azure/chrony.conf` (this symlink target is specific to the Flatcar ebuild path)

However, `coreos-base/oem-azure` is marked `SKIP` in `package_catalog.sh` (line 246). In RPM mode, the ebuild's `src_install()` never runs, so none of these files reach the final image.

Meanwhile, the oem-azure `manglefs.sh` (`sdk_container/src/third_party/coreos-overlay/coreos-base/oem-azure/files/manglefs.sh`) already handles moving chrony configs from `/etc` to `/usr/lib/chrony/` and patching the `chronyd.service` to use `-f /usr/lib/chrony/chrony.conf`. But it's moving the **wrong** config (the Azure Linux RPM default) instead of the Azure-optimized one.

### What to change

In the oem-azure `manglefs.sh` (`sdk_container/src/third_party/coreos-overlay/coreos-base/oem-azure/files/manglefs.sh`), add explicit copies of the companion files into the image. Since these files live in the same directory as `manglefs.sh` itself, you can reference them with `$(dirname "${BASH_SOURCE[0]}")`.

Add after the existing chrony handling (after the `chronyd.service` sed block, around line 88):
```bash
# Copy Azure-optimized chrony.conf from this directory
# (replaces the default Azure Linux RPM config with Azure VM-tuned settings:
#  makestep 1.0 -1 for always-step, PTP refclock for Hyper-V clock)
script_dir="$(dirname "${BASH_SOURCE[0]}")"

# 1. Azure-optimized chrony.conf → /usr/lib/chrony/chrony.conf
#    This overwrites the RPM default that manglefs already moved from /etc.
if [[ -f "${script_dir}/chrony.conf" ]]; then
    cp "${script_dir}/chrony.conf" "${rootfs}/usr/lib/chrony/chrony.conf"
fi

# 2. chronyd.service drop-in (Wants/After dev-ptp_hyperv.device)
if [[ -f "${script_dir}/chrony-hyperv.conf" ]]; then
    mkdir -p "${rootfs}/usr/lib/systemd/system/chronyd.service.d"
    cp "${script_dir}/chrony-hyperv.conf" \
       "${rootfs}/usr/lib/systemd/system/chronyd.service.d/"
fi

# 3. var-chrony.conf tmpfiles (creates /var/lib/chrony at boot)
if [[ -f "${script_dir}/var-chrony.conf" ]]; then
    cp "${script_dir}/var-chrony.conf" "${rootfs}/usr/lib/tmpfiles.d/"
fi
```

> **Note on `etc-chrony.conf`**: The existing `etc-chrony.conf` file in this directory creates a symlink `/etc/chrony/chrony.conf -> ../../usr/share/oem-azure/chrony.conf`, which is the path used by the Flatcar ebuild. On ACL, the config lives at `/usr/lib/chrony/chrony.conf` (where `manglefs.sh` moves it and the `chronyd.service` is patched to use via `-f`). You can either skip copying `etc-chrony.conf` (the `-f` flag in `ExecStart` makes the symlink unnecessary) or update its symlink target to `../../usr/lib/chrony/chrony.conf`.

> **Note on `var-chrony.conf`**: `build_library/rpm/build_image_util.sh` (line ~459) already creates a `chrony.conf` tmpfiles entry with `d /var/lib/chrony 0755 root root -`. The oem-azure `var-chrony.conf` uses `0770 ntp ntp`. Use whichever ownership/permissions are correct for your RPM's chrony user (check with `rpm -q --scripts chrony` or `getent passwd chrony`).

### How to verify

Boot an ACL VM on Azure and check:
```bash
# Config has PTP refclock and always-step
grep -q 'refclock PHC /dev/ptp_hyperv' /usr/lib/chrony/chrony.conf && echo "OK: PTP configured"
grep -q 'makestep 1.0 -1' /usr/lib/chrony/chrony.conf && echo "OK: always-step configured"

# chrony is using PTP
chronyc sources
# Expected: line starting with #* PHC0 or similar PTP source

# Time correction test (set clock 5 years back, verify fast correction)
sudo date -s "2021-01-01"
sleep 30
date  # Should be back to current time within ~20 seconds
```

You can also run the AKS VHD content test `testChrony` in `mantle`, which validates this behavior by setting the clock 5 years in the past and checking it recovers within 100 seconds.

### What we had to do as a workaround

We added ~110 lines in AgentBaker's `vhdbuilder/scripts/linux/flatcar/tool_installs_flatcar.sh` that write the correct chrony config to `/etc/chrony/chrony.conf` (a writable path) and add a systemd drop-in to override the `ExecStart` to use our config instead of the read-only `/usr/lib/chrony/chrony.conf`.

**AgentBaker commit**: [`91fb4c3`](https://github.com/Azure/AgentBaker/commit/91fb4c3846) — _Skip ACL umask test, adjust chrony checks, update ACL issues_

---

## Task 3: Add `tmpfiles.d/logrotate.conf`

### What's wrong

`logrotate.service` fails on every boot with:
```
error creating stub state file /var/lib/logrotate/logrotate.status: No such file or directory
```

Log rotation never runs, meaning logs grow unbounded until disk fills up.

### Why it happens

ACL has an immutable rootfs with `/var` as a separate partition that is populated at boot via `systemd-tmpfiles`. The Azure Linux 3 `logrotate` RPM creates `/var/lib/logrotate/` at RPM install time (`%install` section), but does **not** ship a `tmpfiles.d` drop-in to recreate it at boot.

On a traditional mutable filesystem (like regular Azure Linux), the directory persists across reboots because `/var` is part of the rootfs. On ACL's Flatcar-style layout, `/var` must be recreated at each boot from `tmpfiles.d` declarations.

Upstream Flatcar ships `usr/lib/tmpfiles.d/logrotate.conf` for exactly this purpose. ACL does not have it because it uses the Azure Linux RPM, which doesn't include this file.

### What to change

Add a single tmpfiles.d file in `build_library/rpm/build_image_util.sh`. Note that this file already has `sudo mkdir -p "${root_fs_dir}/var/lib/logrotate"` (line ~1173) which creates the directory at build time, but that alone is insufficient — on ACL's immutable rootfs, `/var` is repopulated at boot from `tmpfiles.d` declarations, so the build-time directory doesn't persist.

Add near the existing `mkdir -p` (replace or add alongside it):
```bash
# Create tmpfiles.d entry for logrotate state directory.
# The Azure Linux 3 logrotate RPM creates /var/lib/logrotate at install time
# but doesn't ship a tmpfiles.d drop-in. On ACL's immutable rootfs, /var is
# populated at boot via systemd-tmpfiles, so the directory must be declared.
echo 'd /var/lib/logrotate 0755 root root -' | sudo tee "${root_fs_dir}/usr/lib/tmpfiles.d/logrotate.conf" > /dev/null
```

That's it — one line of content in one file. Note the use of `${root_fs_dir}` (the variable name used in `build_image_util.sh`) and `sudo tee` (the pattern used by other tmpfiles entries in the same file).

### How to verify

Boot the ACL image and check:
```bash
# Directory exists
test -d /var/lib/logrotate && echo "OK: logrotate dir exists"

# Service runs successfully
systemctl status logrotate.service
# Expected: "inactive (dead)" (oneshot, runs on timer) with exit code 0

# Force a run
sudo systemctl start logrotate.service
systemctl status logrotate.service
# Expected: "Started logrotate.service" with exit status 0

# State file created
test -f /var/lib/logrotate/logrotate.status && echo "OK: state file exists"
```

### What we had to do as a workaround

We added `mkdir -p /var/lib/logrotate` in AgentBaker's `pre-install-dependencies.sh`, which creates the directory at VHD build time. Note that `build_library/rpm/build_image_util.sh` in acl-scripts also has `mkdir -p "${root_fs_dir}/var/lib/logrotate"` (line ~1173), so the directory exists at build time — but both of these are insufficient because on ACL's immutable rootfs, `/var` is repopulated at boot from `tmpfiles.d` declarations. The proper fix is the `tmpfiles.d` entry shown above.

**AgentBaker commit**: [`5d762e8`](https://github.com/Azure/AgentBaker/commit/5d762e88b6) — _fix: create /var/lib/logrotate on ACL before enabling logrotate.timer_

---

## Task 4: Add boot-time systemd preset processing

### What's wrong

When services are configured via Ignition (Flatcar's provisioning system) with `enabled: true`, Ignition writes a **preset file** at `/etc/systemd/system-preset/20-ignition.preset`. On upstream Flatcar, `systemd-preset-all.service` processes this file at first boot and creates the enable symlinks. On ACL, this service doesn't exist, so the preset file is never processed and Ignition-enabled services never start.

This affects any ACL user who uses Ignition to enable custom services, not just AKS.

### Why it happens

Three things combine:

1. **`systemd-preset-all.service` does not exist**: This service was introduced in upstream systemd v256. Azure Linux's systemd v255 does not include it. Azure Linux handles presets at RPM install time via a `%post` scriptlet in the systemd RPM spec, which only runs during package installation — not at boot.

2. **First-boot detection fails**: Azure Linux's systemd is built with `-Dfirst-boot-full-preset=true`, which makes PID 1 internally run preset-all on first boot. But this requires `ConditionFirstBoot=yes` to be true, which requires `/etc/machine-id` to be empty or missing. Despite both AgentBaker's VHD cleanup and ACL's `build_image_util.sh` emptying machine-id, something repopulates it before PID 1 evaluates the condition.

3. **`99-default-disable.preset` catch-all**: Azure Linux ships `/usr/lib/systemd/system-preset/99-default-disable.preset` with a `disable *` catch-all (the Flatcar overlay has a similar `99-default.preset`). Even if `systemctl preset` were somehow re-run, it would disable everything not explicitly listed in higher-priority preset files.

### What to change

**Option A (recommended)**: Create a simple early-boot service that processes preset files, equivalent to upstream's `systemd-preset-all.service`:

Create a new systemd service file (e.g., via `manglefs.sh` or as a new file in the image):

```ini
# /usr/lib/systemd/system/acl-preset-all.service
[Unit]
Description=Apply Preset Settings
DefaultDependencies=no
Conflicts=shutdown.target
After=local-fs.target
Before=sysinit.target

[Service]
Type=oneshot
ExecStart=/usr/bin/systemctl preset-all
RemainAfterExit=yes

[Install]
WantedBy=sysinit.target
```

Enable it statically (don't rely on the preset mechanism to enable the preset service!):
```bash
sudo mkdir -p "${root_fs_dir}/usr/lib/systemd/system/sysinit.target.wants"
sudo ln -sf ../acl-preset-all.service \
   "${root_fs_dir}/usr/lib/systemd/system/sysinit.target.wants/acl-preset-all.service"
```

> **Important**: The `99-default-disable.preset` catch-all (`disable *`) means `systemctl preset-all` will disable any service that isn't explicitly listed in a higher-priority preset file. Ignition writes its presets to `/etc/systemd/system-preset/20-ignition.preset` (priority 20, higher than 99), so Ignition-enabled services will be correctly enabled. However, be aware that this `preset-all` will also re-apply the `disable *` catch-all to any other services without explicit preset entries. Test to confirm no critical services are unexpectedly disabled.

**Option B**: Fix the first-boot detection chain so that systemd's built-in `-Dfirst-boot-full-preset=true` logic works correctly. This requires investigating why `/etc/machine-id` is populated before PID 1 evaluates `ConditionFirstBoot=yes`. Key places to check:
- `build_image_util.sh` line ~1242 (removes machine-id from lowerdir `/usr/share/flatcar/etc`)
- Bootengine `initrd-setup-root` (may remove blank machine-id, causing systemd to regenerate it)
- `systemd-machine-id-setup` in initrd (may run before switch-root)

### How to verify

Create a test Ignition config that enables a simple custom service with `enabled: true`:
```yaml
systemd:
  units:
    - name: test-preset.service
      enabled: true
      contents: |
        [Unit]
        Description=Test Preset
        [Service]
        Type=oneshot
        ExecStart=/bin/echo "preset works"
        [Install]
        WantedBy=multi-user.target
```

Boot a VM with this Ignition config and check:
```bash
# Service should be enabled AND should have run
systemctl is-enabled test-preset.service  # Expected: enabled
systemctl status test-preset.service      # Expected: inactive (dead), exit 0

# The preset file should exist
cat /etc/systemd/system-preset/20-ignition.preset
# Expected: "enable test-preset.service"
```

### What we had to do as a workaround

In AgentBaker's Ignition config (`parts/linux/cloud-init/flatcar.yml`), we bypass the preset mechanism entirely by using Ignition's `storage.links` to directly create enable symlinks:
```yaml
storage:
  links:
    - path: /etc/systemd/system/sysinit.target.wants/ignition-file-extract.service
      target: /etc/systemd/system/ignition-file-extract.service
```

This works for our specific services, but any other Ignition user on ACL who uses `enabled: true` will hit the same issue.

**AgentBaker commits**:
- [`a108d2b`](https://github.com/Azure/AgentBaker/commit/a108d2b457) — _fix: add explicit enable symlink for ignition-bootcmds.service on ACL_
- [`16c956ad`](https://github.com/Azure/AgentBaker/commit/16c956ad7c) — _e2e: add debug diagnostics for Issue 9 (Ignition preset mechanism)_

---

## Task 5: Fix `update-ssh-keys` cross-device mv bug

### What's wrong

`update-ssh-keys-after-ignition.service` fails on boot with:
```
mv: cannot create regular file '/home/core/.ssh/authorized_keys': File exists
```

SSH still works (waagent separately creates `authorized_keys`), but the failed service pollutes logs and causes health check noise.

### Why it happens

ACL replaces Flatcar's Rust `update-ssh-keys` binary with a bash script at `build_library/rpm/additional_files/update-ssh-keys`. (The Rust package `coreos-base/update-ssh-keys` is `SKIP`ped in `package_catalog.sh` line 244.)

The bash script's `regenerate()` function creates a temp file in `/tmp` and then tries to `mv` it to `/home/core/.ssh/authorized_keys`:

```bash
temp_file=$(mktemp)                    # creates in /tmp (tmpfs)
# ... build content ...
mv "$temp_file" "$KEYS_FILE"           # target is on /home (ext4)
```

The `mv` fails because:
- `/tmp` is a tmpfs filesystem, `/home` is ext4 — **different filesystems**
- Cross-device `mv` cannot use atomic `rename()` syscall
- It falls back to copy+delete, which fails with `EEXIST` when waagent already created the target file
- The script uses `set -euo pipefail`, so the failure causes immediate exit

Flatcar's Rust binary creates temp files in the same directory as the target (same filesystem), so `rename()` atomically replaces the file even if it exists.

### What to change

In `build_library/rpm/additional_files/update-ssh-keys`, change two `mktemp` calls:

**1. In the `regenerate()` function** (line 136):
```bash
# BEFORE:
temp_file=$(mktemp)

# AFTER:
temp_file=$(mktemp "${KEYS_FILE}.XXXXXX")
```

**2. In the `add|force-add` case** (line 185):
```bash
# BEFORE:
temp_key=$(mktemp)

# AFTER:
temp_key=$(mktemp "${key_file}.XXXXXX")
```

By creating temp files adjacent to their targets (same directory, same filesystem), `mv` can use the atomic `rename()` syscall, which **overwrites** existing files — matching the Rust binary's behavior.

### How to verify

```bash
# Boot an ACL VM and check the service
systemctl status update-ssh-keys-after-ignition.service
# Expected: inactive (dead) with exit code 0 (not failed)

# Verify authorized_keys exists
test -f /home/core/.ssh/authorized_keys && echo "OK"

# Manual test: run update-ssh-keys directly
sudo /usr/bin/update-ssh-keys -a test_key < /dev/null 2>&1
echo $?  # Expected: 0
```

### What we had to do as a workaround

We added `update-ssh-keys-after-ignition.service` to our E2E test failure allowlist, suppressing the test failure since the actual SSH behavior is unaffected.

**AgentBaker commit**: [`965f7fd`](https://github.com/Azure/AgentBaker/commit/965f7fd1d6) — _e2e: allowlist update-ssh-keys-after-ignition.service for Flatcar/ACL_

---

## Task 6: Remove or fix `/etc/profile.d/umask.sh`

### What's wrong

ACL ships `/etc/profile.d/umask.sh` with conditional umask values (`umask 002` when user/group names match, `umask 022` otherwise). Neither value meets CIS hardening standards (CIS requires `umask 027` or stricter). AKS VHD content tests verify CIS compliance and fail on ACL.

### Why it happens

Upstream Flatcar (version 4593+) removed `/etc/profile.d/umask.sh` entirely. ACL inherits this file from the Azure Linux base packages (`setup` RPM).

### What to change

**Option A (recommended — match Flatcar)**: Remove the file during image build:
```bash
# In build_library/rpm/build_image_util.sh during image finalization:
sudo rm -f "${root_fs_dir}/etc/profile.d/umask.sh"
```

**Option B**: Update the file contents to meet CIS standards:
```bash
echo 'umask 027' | sudo tee "${root_fs_dir}/etc/profile.d/umask.sh" > /dev/null
```

### How to verify

```bash
# After building the image, check:
test ! -f /etc/profile.d/umask.sh && echo "OK: file removed"
# OR if Option B:
grep -q 'umask 027' /etc/profile.d/umask.sh && echo "OK: CIS compliant"
```

### What we had to do as a workaround

We skip the `umask.sh` validation test for ACL/Flatcar in AgentBaker's VHD content tests, since ACL uses an overlay filesystem on `/etc` which prevents persistent modification of the file at VHD build time.

**AgentBaker commit**: [`91fb4c3`](https://github.com/Azure/AgentBaker/commit/91fb4c3846) — _Skip ACL umask test, adjust chrony checks, update ACL issues_

---

## Task 7: Evaluate cloud-init distro detection

### What's wrong

Cloud-init logs a warning during boot:
```
Unable to load distro implementation for acl. Using default distro implementation instead.
```

### Why it happens

Cloud-init's `distro` library does not recognize `acl` as a distro ID (from `/etc/os-release`). It falls back to a generic default implementation.

### What to change

This is an investigation item. Options:
1. Contribute an ACL distro class to cloud-init upstream
2. Configure cloud-init to treat ACL as `flatcar` via `/etc/cloud/cloud.cfg`
3. Accept the warning if default behavior is sufficient

### How to verify

```bash
cloud-init status --long
# Check for warnings in /var/log/cloud-init.log
grep -i "distro" /var/log/cloud-init.log
```

---

## Appendix A: Root cause pattern — RPM `SKIP` gaps

Most of the issues above stem from a single pattern: packages that are `SKIP`ped in `build_library/rpm/package_catalog.sh` provide files that are needed on Azure VMs but have no alternative source in RPM mode.

Current `SKIP`ped packages that cause issues:

| Package | Line | What's missing | Task |
|---------|------|----------------|------|
| `sys-apps/azure-vm-utils` | 248 | udev disk rules + NVMe ID binary | Task 1 |
| `coreos-base/oem-azure` | 246 | chrony config, drop-ins, tmpfiles | Task 2 |
| `coreos-base/update-ssh-keys` | 244 | Rust binary (bash replacement has bugs) | Task 5 |

When reviewing other `SKIP`ped packages, consider whether they provide files that are needed for Azure VM operation.

## Appendix B: Root cause pattern — missing `tmpfiles.d` entries

ACL's Flatcar-style immutable rootfs uses a separate `/var` partition populated at boot via `systemd-tmpfiles`. Azure Linux RPMs assume a persistent rootfs, so directories created at RPM install time under `/var` don't persist across reboots on ACL.

Any RPM that creates directories under `/var/lib/`, `/var/log/`, or `/var/cache/` at install time without shipping a corresponding `tmpfiles.d` drop-in will break on ACL.

Known instances:
| RPM | Missing `tmpfiles.d` entry | Task | Notes |
|-----|---------------------------|------|-------|
| `logrotate` | `d /var/lib/logrotate 0755 root root -` | Task 3 | No tmpfiles.d entry exists anywhere; `build_image_util.sh` only does build-time `mkdir` |
| `chrony` | `d /var/lib/chrony 0770 ntp ntp -` | Task 2 | `build_image_util.sh` already has a tmpfiles entry (line ~459) but uses `0755 root root`; the oem-azure overlay's `var-chrony.conf` uses `0770 ntp ntp` |

Consider auditing all installed RPMs for this pattern:
```bash
# Find RPMs that create directories under /var but don't ship tmpfiles.d entries
rpm -qa --queryformat '%{NAME}\n' | while read pkg; do
    has_var=$(rpm -ql "$pkg" 2>/dev/null | grep -c '^/var/')
    has_tmpfiles=$(rpm -ql "$pkg" 2>/dev/null | grep -c 'tmpfiles.d')
    if [[ $has_var -gt 0 && $has_tmpfiles -eq 0 ]]; then
        echo "MISSING: $pkg has /var files but no tmpfiles.d entry"
    fi
done
```

## Appendix C: Changes handled in AgentBaker (no base image changes needed)

The following changes were also made in AgentBaker and no changes specifically needed in the ACL base image — they're listed here for awareness only.

| Change | What it does | Files | Commit |
|--------|-------------|-------|--------|
| OS detection plumbing | Extended `isFlatcar()` to recognize ACL, added `isACL()` helper | `cse_helpers.sh`, `vhd-scanning.sh` | [`683984748f`](https://github.com/Azure/AgentBaker/commit/683984748f) |
| Package resolution fallback | When `components.json` has no `acl` entry, falls back to `flatcar` entries | `cse_helpers.sh` | [`fcf079e68b`](https://github.com/Azure/AgentBaker/commit/fcf079e68b), [`7fbee096df`](https://github.com/Azure/AgentBaker/commit/7fbee096df) |
| Disable iptables.service | Clears Azure Linux's default iptables rules that block Cilium eBPF host routing (same as Mariner/AzureLinux) | `install-dependencies.sh` | [`1b3e82ee52`](https://github.com/Azure/AgentBaker/commit/1b3e82ee52) |
| Disable systemd-resolved cache | Repoints `resolv.conf` to upstream DNS so localdns works (same as Mariner/AzureLinux) | `install-dependencies.sh` | [`6d7d99c6a4`](https://github.com/Azure/AgentBaker/commit/6d7d99c6a4) |
| CA cert path routing | Routes CA certs to `/etc/pki/ca-trust/source/anchors` with `update-ca-trust` (ACL uses RHEL-style trust, not `update-ca-certificates`) | `cse_config.sh`, `acl/update_certs.service`, Packer JSON templates | [`1e414f4ec9`](https://github.com/Azure/AgentBaker/commit/1e414f4ec9), [`66dc0acafa`](https://github.com/Azure/AgentBaker/commit/66dc0acafa) |
| Kubelet/kubectl install path | Routes ACL through `isFlatcar()` so kubelet/kubectl are installed from URL (not package manager) | `cse_config.sh` | [`e466ce277f`](https://github.com/Azure/AgentBaker/commit/e466ce277f) |
| waagent PATH fix | Uses `waagent` via PATH instead of hardcoded `/usr/sbin/waagent` in packer templates (ACL installs it at `/usr/bin/waagent`) | Packer JSON templates | [`91d7246adb`](https://github.com/Azure/AgentBaker/commit/91d7246adb) |
