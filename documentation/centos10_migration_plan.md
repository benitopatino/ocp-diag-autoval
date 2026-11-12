# AutoVal Controller Migration Plan: CentOS Stream 9 → 10

Snapshot exported Nov 11, 2026 from the working doc.

## Summary

**Scope:** the AutoVal controller (test server) and all DUTs. Both move from CentOS Stream 9 to Stream 10.

**What the controller does today** (from the ocp-diag-autoval and ocp-diag-autoval-ssd repos):

- Runs `autoval_test_runner` (single tests and `--suite` YAML runs) in Python.
- Reaches DUTs and BMCs over SSH (paramiko) plus `sshpass` + `scp`/`rsync`/`ssh` shell-outs, `ssh-agent` forwarding, and `ping`/`ping6` reachability checks.
- Serves the shared `/shared/autoval` filesystem (GlusterFS per `gluster_configuration.md`) that holds `resultsdir` and is mounted on every DUT.
- Hosts the `autoval-tools` DNF repo (`createrepo_c` over `/shared/autoval/pkgs/`) that DUTs install fio, nvme-cli, smartmontools and similar packages from.

**Why now:** CentOS Stream 9 reaches end of life on **May 31, 2027**. Stream 10 is supported until **May 31, 2030**.

**Bottom line:** the Python code is ready for Stream 10's Python 3.12 (tested, see below). The real work is infrastructure:

1. **GlusterFS is not built for Stream 10.** This blocks a like-for-like rebuild and needs a decision first.
2. **CPU must support x86-64-v3** (roughly Haswell / Zen 1 or newer) or Stream 10 will not install.
3. **Four Python dependencies are imported but not declared** in `pyproject.toml`. They probably came from system RPMs on the old box, and a clean 3.12 venv will not have them.
4. **Stricter SSH crypto defaults** may break `sshpass`/paramiko connections to older BMCs and DUTs.

There is no supported in-place upgrade from Stream 9 to Stream 10. Plan a **fresh build alongside the current controller**, then cut over.

**DUTs:** every DUT is reinstalled with Stream 10 too. After storage, the biggest jobs are rebuilding the `autoval-tools` repo for el10 (every RPM plus its dependencies) and re-validating the command-output parsers against newer tools and kernel 6.12. See "DUT upgrade: additional work" below.

## What changes from Stream 9 to Stream 10

| Area | Stream 9 | Stream 10 | Impact on AutoVal |
| --- | --- | --- | --- |
| Python | 3.9 | 3.12 | Code compiles and tests pass on 3.12. venvs no longer pre-install setuptools, which `pkg_resources` needs |
| x86 baseline | x86-64-v2 | x86-64-v3 | Older controller hardware will not boot the installer |
| Kernel / GCC | 5.14 / 11 | 6.12 / 14 | Low impact on the controller |
| GlusterFS | Storage SIG packages | Not built for c10s | **Blocker** for `/shared/autoval` as documented |
| sshpass | EPEL 9 | EPEL 10 (1.09) | Enable EPEL 10 + CRB |
| Networking | ifcfg + keyfiles | NetworkManager keyfiles | Recreate any `ifcfg-*` config with `nmcli` |
| Modularity | AppStream modules | Removed (plain RPMs) | Any `dnf module` steps in runbooks need rewriting |
| EOL | 2027-05-31 | 2030-05-31 | — |

## Findings from the repos

| # | Finding | Where | Severity | Action |
| --- | --- | --- | --- | --- |
| 1 | Shared results storage relies on GlusterFS server + client | `autoval_ssd/docs/gluster_configuration.md`; `resultsdir` lives on it | **Blocker** | Choose a replacement (see Decision section) |
| 2 | `yaml` imported, PyYAML not declared. `--suite` runs fail with `ModuleNotFoundError` | core `lib/suite_runner.py` | High | Add `PyYAML` to core `dependencies` |
| 3 | `attrs`, `pandas`, `packaging`, `yaml` imported, not declared | ssd `tests/storage_hw_eng/...` (`data_record.py`, `ssd_test_base.py`, `component_test_base.py`, `ssd_synthflash_workload.py`) | High for storage_hw_eng tests | Add to ssd `dependencies`, or an optional extra |
| 4 | `sshpass` used for scp, rsync, ssh to DUTs and BMCs | core `transport/ssh.py`, `host/bmc.py` | High | Install from EPEL 10 on the controller |
| 5 | Default SSH key is `id_rsa`, `StrictHostKeyChecking=no`, paramiko `AutoAddPolicy` | `site_settings.json`, `transport/ssh.py` | Medium | Test against the oldest BMC/DUT. Consider ed25519 keys |
| 6 | `pkg_resources` + pinned `setuptools==70.0.0` | core `plugin_manager.py`, `generic_utils.py`, `file_actions.py` | Medium | Works on 3.12 with a DeprecationWarning. Make sure setuptools is installed in the venv. Longer term, move to `importlib.resources` |
| 7 | Controller-local commands: `ping`/`ping6`, `ssh-agent`/`ssh-add`, `tar`, and BMC commands (incl. `ipmitool`) run locally when a host has no `oob_addr` | core `host/host.py`, `host/bmc.py`, `site_utils.py` | Medium | Include in the package list (below) |
| 8 | IPv6 assumed in places (`scp -6`, `ping6`, Gluster `transport.address-family=inet6`) | core `transport/ssh.py`, Gluster doc | Medium | Configure and verify IPv6 on the new controller |
| 9 | Docs reference `scripts/install_glusterfs_automounter.sh`, which is not in the repo | `gluster_configuration.md` | Low | Restore it, or drop it if Gluster is replaced |
| 10 | `get_centos_release()` parses `/etc/centos-release` | ssd `component_test_base.py` | Low | Regex parses "CentOS Stream release 10" correctly. Confirm on a real el10 DUT |
| 11 | Deprecated APIs: `Thread.setName()`, `datetime.utcfromtimestamp()` | core `autoval_thread.py:87`, ssd `lmparser.py` | Low (warnings only) | Clean up when convenient |
| 12 | Package classifiers list only Python 3.8 | both `pyproject.toml` | Cosmetic | Add 3.12 |

## DUT upgrade: additional work

Moving the DUTs to Stream 10 adds three large jobs: rebuild `autoval-tools` for el10, re-validate every output parser against el10 tool versions, and reinstall each DUT without losing its AutoVal setup.

| # | Finding | Where | Severity | Action |
| --- | --- | --- | --- | --- |
| D1 | RPM installs run `dnf --disablerepo=* --enablerepo=autoval-tools`, so every dependency has to come from that repo alone | ssd `lib/utils/system_utils.py` (`install_rpms`) | High | Build the el10 repo with the full dependency closure (`dnf download --resolve --alldeps`) on a clean DUT image |
| D2 | Packages the tests install: smartmontools, hdparm, sdparm, fio, fio-engine-libaio, sg3_utils, sshpass, boost-program-options, gdisk, golang, nvme-cli/libnvme, switchtec, plus site tools (fio-synth, sedutil, agfhc) | `storage_test_base.py` (`TOOLS`), `fio_runner.py`, `boot_drive_partition_utils.py`, `iogo.py`, `fdp_utils.py` | High | Get an el10 build of each. Rebuild in-house and vendor RPMs (fio-synth, `*.hs.el9` packages) for el10 |
| D3 | Pinned el9 versions: `nvme-cli-2.8-1.hs.el9`, and `nvme-cli-2.10.2` + `libnvme-1.10` for FDP | `nvme_cli_with_nvme_version.json`, `fdp_utils.py` | High | Rebuild those versions for el10, or move the pins to the versions you will host |
| D4 | Parsers read nvme-cli, smartctl, lspci, dmidecode, hdparm and sdparm output plus about 39 `/sys` and `/proc` paths. Kernel 6.12 and newer tools can change that output | ssd `lib/utils/...`. Mocks in `unittest/mock/util_outputs/` are el9 captures | High | Capture outputs from an el10 DUT, diff them against the mocks, and add el10 mock cases where they differ |
| D5 | Code built or run on the DUT: `gcc` (fsync), `golang` (iogo, `ioT6.go`), `python3` (fileappend) | `tests/fsync/fsync.py`, `tests/iogo/iogo.py`, `ssd_fileappend.py` | Medium | Put `gcc` in the DUT image (tests don't install it). Test `ioT6.go` with el10's Go |
| D6 | With `remote_fio`, the fio version is checked on the controller as well as the DUT | `fio_runner.py` | Medium | Install the same fio build on the controller as on the DUTs |
| D7 | The Gluster client isn't available on el10 DUTs either | DUT `/etc/fstab` | Blocker (same as #1) | Rules out keeping Gluster on a separate server. Use NFS |
| D8 | Reinstalling a DUT wipes its local setup: `authorized_keys`, the share mount in `/etc/fstab`, `autoval.repo`, sudoers | each DUT | Medium | Script the DUT post-install (kickstart `%post` or Ansible) and prove it on one DUT first |

## Python 3.12 compatibility test (already run)

Run on CPython 3.12.16 in a clean venv with the declared core dependencies (`setuptools==70.0.0`, `ocptv==0.1.6`, `six`, `iopath`, `paramiko`):

- **SSD unit tests:** 240 run in about 19s. The only failures are the **3 known `test_system_utils` failures** (same as on 3.10). `test_nvme_drive` was skipped for time and should be run on the new box.
- **Compile check** of both packages with `SyntaxWarning` treated as errors: clean. No invalid escape sequences.
- **Import check:** every core and ssd module imported except `autoval.lib.suite_runner` (missing PyYAML, finding #2).
- **No removed stdlib modules in use** (`imp`, `distutils`, `asyncore`, `crypt`, `telnetlib` and similar were checked).
- `autoval_test_runner --help` works.
- Only warnings: `pkg_resources is deprecated` and `setName() is deprecated`.
- Core has no runnable unit tests, so its coverage comes from the hardware validation in Phase 4.

## Decision needed: replacing GlusterFS

The CentOS Storage SIG Gluster maintainer has said they are not planning to build for c10s.

| Option | Pros | Cons | Effort |
| --- | --- | --- | --- |
| **A. NFS v4.2 server on the controller** (recommended) | `nfs-utils` is in BaseOS and supported for the life of Stream 10. Same `/shared/autoval` path, so `site_settings.json` doesn't change. Simpler than Gluster for a single-server volume | Need to change `/etc/fstab` on every DUT. Single server (same as the current one-brick Gluster setup) | Low |
| B. Keep Gluster on a separate Stream 9 box | Not viable: Stream 10 DUTs can't install the Gluster client either | Stream 9 EOL is May 2027, so this only delays the problem | Low now, repeat later |
| C. Build Gluster from source or COPR for el10 | Keeps the current design | Unsupported and unmaintained. You own every patch | High, ongoing |
| D. CephFS | Scales, maintained | Heavy for a single test-server volume | High |

**Recommendation:** Option A. The current Gluster setup is a single brick on the controller, so it is already a single-server share in practice.

## Migration checklist

### Phase 0: Discovery (current controller)

- [ ] Confirm the CPU supports x86-64-v3: `/lib64/ld-linux-x86-64.so.2 --help | grep v3` should show `x86-64-v3 (supported, searched)`
- [ ] Record how AutoVal is installed (venv vs system Python, built sdists vs `PYTHONPATH`) and capture `pip freeze`
- [ ] Export the package list: `dnf repoquery --userinstalled > pkgs-userinstalled.txt`
- [ ] Back up configs: `site_settings.json` (all variants), `hosts.json` files, custom control JSONs and suite YAMLs, `~/.ssh` (keys, `known_hosts`, config), `/etc/ssh/ssh_config.d/`, `/etc/fstab`, `/etc/yum.repos.d/`, sudoers, crontabs, custom systemd units, firewall rules
- [ ] Capture Gluster state: `gluster volume info autoval`, `gluster volume status`
- [ ] Measure `/shared/autoval` and `/autoval/results`, and decide what to migrate vs archive
- [ ] Inventory DUTs and BMCs: OS version, SSH server version (`ssh -v`), whether `oob_addr` is used
- [ ] Check whether any DUT still runs CentOS 7 (yum, older repo metadata support)

### Phase 1: Decisions

- [ ] Pick the shared storage replacement (see above)
- [ ] Decide the DUT rollout: all at once or pool by pool (waves mean the repo serves el9 and el10 side by side for a while)
- [ ] Choose new controller hardware or VM if the current CPU is not x86-64-v3
- [ ] Set the cutover window and the rollback owner

### Phase 2: Code changes (both repos)

- [ ] Core `pyproject.toml`: add `PyYAML` to `dependencies`, and add the Python 3.12 classifier
- [ ] SSD `pyproject.toml`: add `attrs`, `pandas`, `packaging`, `PyYAML` (or an extra such as `storage_hw_eng`)
- [ ] Keep `setuptools` installed in the venv for `pkg_resources`. Optionally replace it with `importlib.resources`
- [ ] Optional: `t.name = threadname` instead of `setName`, and timezone-aware `datetime.fromtimestamp(..., tz=timezone.utc)`
- [ ] Update `gluster_configuration.md` and `rpm_repo_hosting.md` (or add an NFS guide) for Stream 10
- [ ] Run the SSD unit tests on 3.12, including `test_nvme_drive`

### Phase 3: Build the new controller

- [ ] Install CentOS Stream 10 (minimal) and set the hostname, IPv4 and IPv6 with `nmcli`
- [ ] Enable CRB + EPEL 10: `dnf config-manager --set-enabled crb && dnf install -y epel-release`
- [ ] Install packages: `dnf install -y python3 python3-pip git sshpass rsync openssh-clients iputils tar ipmitool lshw createrepo_c nfs-utils`
- [ ] Create the venv and install AutoVal (core first, then ssd), as in the repo's CLAUDE.md setup steps
- [ ] Restore SSH keys (`chmod 600`), `site_settings.json` and hosts files
- [ ] Create `/autoval/logs`, `/autoval/results`, `/tmp/autoval`, `/autoval/repository` (or whatever site settings point to)
- [ ] Set up shared storage: export `/shared/autoval`, open the firewall (`firewall-cmd --add-service=nfs --permanent`), fix SELinux contexts
- [ ] Rebuild the `autoval-tools` repo: copy RPMs to `/shared/autoval/pkgs/`, run `createrepo_c`
- [ ] Test-mount the share on one DUT and run `dnf --disablerepo='*' --enablerepo=autoval-tools makecache`

### Phase 3b: DUT image and el10 repo

- [ ] Check x86-64-v3 support on every DUT platform (same `ld.so --help` check as Phase 0)
- [ ] Build a Stream 10 DUT image (kickstart) with `gcc`, `python3`, `nfs-utils`, `openssh-server`, and the AutoVal post-install: keys, share mount, `autoval.repo`, sudoers
- [ ] Split the repo by release: `/shared/autoval/pkgs/el9/` and `/shared/autoval/pkgs/el10/`, with each DUT's `autoval.repo` pointing at its own directory while both fleets run
- [ ] On a clean DUT image, download the RPMs with their full dependency closure: `dnf download --resolve --alldeps --destdir=/shared/autoval/pkgs/el10 smartmontools hdparm sdparm fio fio-engine-libaio sg3_utils sshpass boost-program-options gdisk golang nvme-cli libnvme switchtec`. Add the rebuilt in-house RPMs, then run `createrepo_c`
- [ ] Rebuild the pinned nvme-cli/libnvme versions for el10, or update `nvme_cli_with_nvme_version.json` and `fdp_utils.py`
- [ ] On one upgraded DUT, run every `install_rpms` path with only `autoval-tools` enabled
- [ ] Capture el10 outputs (`nvme list`, `nvme id-ctrl`, `nvme smart-log`, `smartctl -a`, `lspci -vvv`, `dmidecode`, `hdparm -I`, `sdparm`), diff them against `unittest/mock/util_outputs/`, then fix parsers and add el10 mocks
- [ ] Run each test once on a dedicated, wipeable el10 DUT (the tests are destructive) and compare with an el9 DUT of the same platform
- [ ] Upgrade the rest of the fleet one pool at a time

### Phase 4: Validation (one non-production DUT)

- [ ] `autoval_test_runner --dry_run` with a real `hosts.json`
- [ ] SSH, `sshpass` scp/rsync and `ssh-agent` forwarding to the DUT
- [ ] BMC reachability: `sshpass` to the BMC, plus `ipmitool` from the controller where `oob_addr` isn't set
- [ ] Run a single non-destructive test end to end, and check that logs and results land in `resultsdir` (including the planned `suite_logs/<user>/<DUT host>/...` layout)
- [ ] Run a short `--suite` YAML (exercises PyYAML)
- [ ] Check RPM auto-install on the DUT from `autoval-tools`
- [ ] Compare against the old controller: same test, same DUT, same pass/fail and artifacts
- [ ] Validate with the oldest BMC firmware and oldest DUT OS in the fleet

### Phase 5: Cutover

- [ ] Announce a test freeze and let in-flight runs finish
- [ ] Final sync of results: `rsync -aHAX` from old `/shared/autoval` to new
- [ ] Move the controller identity (DNS name or IP), or update DUT `/etc/fstab`, `autoval.repo` `baseurl` and `hosts.json`
- [ ] Remount the share on all DUTs and run `df /shared/autoval` on each
- [ ] Run the smoke suite on every DUT pool
- [ ] Keep the old controller intact (powered off or isolated) for at least 2 weeks

### Phase 6: Rollback

- [ ] Trigger: any blocker in Phase 5 that can't be fixed within the window
- [ ] Steps: point DNS/IP and DUT mounts back at the old controller, restart `glusterd`, copy back any results written to the new share
- [ ] Results written during the failed window: rsync new → old before switching back

## Potential problems and mitigations

| Problem | Symptom | Mitigation |
| --- | --- | --- |
| Gluster packages unavailable | `dnf install glusterfs-server` finds nothing | Replace with NFS (Option A) before the build |
| CPU lacks x86-64-v3 | Installer refuses to run or the kernel panics at boot | Check in Phase 0. Use newer hardware or set the VM CPU model to `host` or a v3 model |
| Undeclared Python deps | `ModuleNotFoundError: No module named 'yaml'` (or `attr`, `pandas`) | Phase 2 dependency fixes. Interim: `pip install PyYAML attrs pandas packaging` |
| setuptools missing in a 3.12 venv | `ModuleNotFoundError: pkg_resources` in `PYTHONPATH` dev mode | `pip install setuptools==70.0.0` in the venv |
| SSH algorithm mismatch with old BMCs or DUTs | `no matching host key type` or signature errors, paramiko `SSHException` | Use RSA keys with rsa-sha2-256/512 or ed25519. As a temporary per-host exception, add `HostKeyAlgorithms`/`PubkeyAcceptedAlgorithms` in `~/.ssh/config`. Avoid lowering the system-wide crypto policy |
| `sshpass` not installed | `sshpass: command not found` during scp, rsync or BMC steps | Enable EPEL 10 + CRB, then install `sshpass` |
| SELinux denials on new paths or exports | Permission denied despite correct Unix perms; AVCs in `ausearch -m avc` | `semanage fcontext` + `restorecon` on `/shared/autoval`, `/autoval`. Don't disable SELinux |
| Firewall blocks NFS or the repo | DUT mount hangs, `dnf makecache` times out | Open `nfs` (and `mountd`/`rpc-bind` if using NFSv3) |
| Old network config ignored | Interface comes up without IPv6 or with the wrong address | Recreate with `nmcli`. Verify `ping6` to DUTs and BMCs |
| Repo metadata unreadable by very old DUTs | CentOS 7 yum errors on repodata | Test from the oldest DUT. If needed, generate metadata with gzip compression (`createrepo_c --general-compress-type=gz`) |
| Stream rolling updates change behaviour mid-campaign | A tool version changes between runs | Schedule updates between campaigns. Use `dnf versionlock` for critical packages |
| `ocptv==0.1.6` on 3.12 | Only import-tested so far | Covered by Phase 4 end-to-end runs |
| Results history lost or split | Old results missing after cutover | Final rsync, and keep the old controller read-only for 2 weeks |
| Dependency missing from the el10 repo | dnf on the DUT reports nothing provides \<library\>, and the test setup fails | Build the repo with dnf download --resolve --alldeps on a clean image. Test install\_rpms on one DUT with only autoval-tools enabled |
| Tool output format changed on el10 | Parser exceptions, or wrong SMART, firmware or drive-inventory values that pass silently | Diff el10 captures against the mocks before running real tests. Compare el9 and el10 results on the same platform |

## Open questions for the team

- Is the current controller's CPU x86-64-v3 capable?
- In what order do the DUT pools upgrade, and how long will el9 and el10 DUTs run side by side?
- Are any DUTs still on CentOS 7?
- NFS, or another option, for `/shared/autoval`?
- How much results history needs to move?
- Who owns rollback during the cutover window?

## Handy commands

```bash
# CPU level
/lib64/ld-linux-x86-64.so.2 --help | grep x86-64-v

# Package inventory on the old box
dnf repoquery --userinstalled | sort > pkgs-userinstalled.txt

# New box: repos + base tools
dnf config-manager --set-enabled crb
dnf install -y epel-release
dnf install -y python3 python3-pip git sshpass rsync openssh-clients iputils tar ipmitool lshw createrepo_c nfs-utils

# AutoVal venv
python3 -m venv ~/autoval-env && source ~/autoval-env/bin/activate
pip install build setuptools==70.0.0 PyYAML attrs pandas packaging
(cd ocp-diag-autoval && python -m build && pip install dist/ocptv_autoval-0.0.1.tar.gz)
(cd ocp-diag-autoval-ssd && python -m build && pip install --no-deps dist/ocptv_autoval_ssd-0.0.1.tar.gz)

# NFS export (Option A)
echo '/shared/autoval *(rw,sync,no_root_squash)' >> /etc/exports
systemctl enable --now nfs-server
firewall-cmd --add-service=nfs --permanent && firewall-cmd --reload
# DUT fstab: controller:/shared/autoval /shared/autoval nfs4 defaults,_netdev 0 0

# RPM repo
createrepo_c /shared/autoval/pkgs/

# SELinux check
ausearch -m avc -ts recent
```

## Sources

- [CentOS Stream 10 release notes](https://centos.org/centos10/)
- [CentOS download page with EOL dates](https://www.centos.org/download)
- [CentOS devel list: glusterfs not built for c10s](https://lists.centos.org/hyperkitty/list/devel@lists.centos.org/message/ZFQT34CXRAGYXBHTURQJZ4ZZAY6PA6PD)
- [Red Hat Bugzilla 2307291: sshpass in EPEL 10](https://bugzilla.redhat.com/show_bug.cgi?id=2307291)
- [OpenLogic: CentOS Stream 10 changes](https://www.openlogic.com/blog/centos-stream-10)
