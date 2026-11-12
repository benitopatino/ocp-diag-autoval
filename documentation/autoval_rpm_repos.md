# AutoVal RPM Repos Explained (CentOS Stream 10)

Written Nov 11, 2026. See also [`autoval_setup_guide_centos10.md`](autoval_setup_guide_centos10.md), Parts 1, 3 and DUT step 5.

An AutoVal lab uses two kinds of RPM repo:

1. **Distribution repos** (BaseOS, AppStream, CRB, EPEL 10). You enable them; nothing is downloaded by hand.
2. **`autoval-tools`**, a local repo on the controller that **you download and build yourself**. DUTs install test tools from it during every test run.

## 1. Distribution repos

These are the normal CentOS Stream 10 repos that every machine uses for its own packages.

| Repo | What it is | How to enable |
|---|---|---|
| BaseOS, AppStream | Core CentOS packages: Python, fio, nvme-cli, smartmontools, nfs-utils, and so on | On by default |
| CRB | Extra build and development packages; EPEL depends on it | `dnf config-manager --set-enabled crb` |
| EPEL 10 | Community packages not in CentOS. For AutoVal that's mainly `sshpass` | `dnf install epel-release` (after enabling CRB) |

- **Controller:** needs all four (it installs `sshpass` from EPEL).
- **DUTs:** need BaseOS and AppStream for their base install. The kickstart takes care of that.

## 2. The `autoval-tools` repo

### Why it exists

SSD tests install their own tools on the DUT at the start of every run (`SystemUtils.install_rpms` in ocp-diag-autoval-ssd). When the `yum_repo` site setting is set, the command it runs **shuts out every other repo**:

```text
sudo dnf -y --allowerasing --disablerepo=* --enablerepo=autoval-tools install <pkg>
```

(`-b` is added when the test force-installs.) That has two consequences:

- Every package a test installs, **and everything those packages depend on**, must be in `autoval-tools`. dnf can't fill gaps from BaseOS or EPEL during a test.
- The repo id in the DUT's `.repo` file must match `yum_repo` in `site_settings.json` **exactly**. The older docs use both `autoval-tools` and `autoval_tools`. Pick one and use it everywhere.

### What goes in it

| Packages | Where you get them | Needed by |
|---|---|---|
| smartmontools, hdparm, sdparm, sg3_utils, fio, fio-engine-libaio | CentOS Stream 10 repos | Every storage test (`StorageTestBase`, `TOOLS` list) |
| boost-program-options | CentOS Stream 10 repos | fio runs (`fio_runner.py`) |
| gdisk | CentOS Stream 10 repos | Boot-drive partition handling (`boot_drive_partition_utils.py`) |
| nvme-cli, libnvme | CentOS Stream 10 repos | nvme tests |
| golang | CentOS Stream 10 repos | iogo tests (large dependency tree) |
| sshpass | EPEL 10 | `StorageTestBase`, `fio_runner.py` |
| fiosynth | Built yourself from [FioSynth](https://github.com/facebookincubator/FioSynth) | `fio_synth_flash` and the synthflash workloads |
| Pinned versions: `nvme-cli-2.8-1.hs.el9` (`nvme_cli_with_nvme_version.json`), `nvme-cli-2.10.2` + `libnvme-1.10` (FDP setup in `fdp_utils.py`) | Rebuilt by your team for el10, or change the pins to versions you host | nvme_cli version tests, FDP tests |
| switchtec, sedutil, lsiutil / scrtnycli, agfhc | Vendor RPMs or your own builds | Switch, SED and HBA tests only |
| **All dependencies of the above** | Pulled in automatically by the download below | Required because of `--disablerepo=*` |

### How to build it

The controller serves the repo from the NFS share at `/shared/autoval/pkgs/el10`.

**Download the packages with their full dependency closure.** Doing it inside a clean CentOS 10 container means nothing already installed on the controller is skipped:

```bash
dnf install -y podman
mkdir -p /shared/autoval/pkgs/el10
podman run --rm -v /shared/autoval/pkgs/el10:/out:Z quay.io/centos/centos:stream10 bash -c '
  dnf install -y dnf-plugins-core epel-release &&
  dnf config-manager --set-enabled crb &&
  dnf download --resolve --alldeps --destdir /out \
    smartmontools hdparm sdparm sg3_utils fio fio-engine-libaio \
    sshpass boost-program-options gdisk nvme-cli libnvme golang'
```

- `--resolve` adds the dependencies.
- `--alldeps` also includes dependencies a minimal system already has. That makes the repo bigger (golang especially), but a DUT never fails a test because a library is missing.

**Build fiosynth** (no distribution packages it):

```bash
git clone https://github.com/facebookincubator/FioSynth
cd FioSynth
python3 -m pip install --user setuptools   # bdist_rpm needs setuptools on Python 3.12
python3 setup.py bdist_rpm                 # needs rpm-build installed
cp dist/*.noarch.rpm /shared/autoval/pkgs/el10/
```

**Add pinned-version and vendor RPMs** by copying them into the same folder.

**Generate the repo metadata:**

```bash
createrepo_c /shared/autoval/pkgs/el10
# after adding or removing RPMs later:
createrepo_c --update /shared/autoval/pkgs/el10
```

The metadata must be regenerated every time the folder's contents change.

### Configure the controller and DUTs

**Controller**, in `/etc/autoval/site_settings.json`:

```json
"yum_repo": "autoval-tools"
```

**Each DUT**, in `/etc/yum.repos.d/autoval.repo` (the DUT reaches the folder through its NFS mount of `/shared/autoval`):

```ini
[autoval-tools]
name=AutoVal test tools (el10)
baseurl=file:///shared/autoval/pkgs/el10
enabled=1
gpgcheck=0
```

**Check from a DUT:**

```bash
dnf --disablerepo='*' --enablerepo=autoval-tools makecache
dnf --disablerepo='*' --enablerepo=autoval-tools list available | head
dnf --disablerepo='*' --enablerepo=autoval-tools install -y smartmontools
```

The last command runs exactly the way a test would. If it fails with `nothing provides ...`, a dependency is missing from the repo.

## 3. The alternative: no local repo

Leave `yum_repo` out of `site_settings.json`. Tests then run a plain `sudo dnf -y --allowerasing install <pkg>`, so each DUT uses its normal CentOS and EPEL repos.

| | Local `autoval-tools` repo | Distribution repos only |
|---|---|---|
| Same tool versions on every DUT | Yes | No: whatever the mirrors have that day |
| Works on an isolated lab network | Yes | No: DUTs need internet or a mirror |
| In-house builds (fiosynth, pinned nvme-cli, vendor tools) | Yes | No |
| Effort | Build and refresh the repo | None |

For a lab, use the local repo.

## 4. Mixed CentOS 9 and 10 DUTs

While DUTs are upgraded pool by pool, keep the two releases apart:

- `/shared/autoval/pkgs/el9/` for CentOS Stream 9 DUTs
- `/shared/autoval/pkgs/el10/` for CentOS Stream 10 DUTs

Point each DUT's `autoval.repo` `baseurl` at its own release's folder. Never mix el9 and el10 RPMs in one repo.

## 5. Common problems

| Symptom on the DUT | Cause | Fix |
|---|---|---|
| `No match for argument: <pkg>` | Package isn't in the repo, or metadata is stale | Add the RPM, then `createrepo_c --update` |
| `nothing provides <library> needed by <pkg>` | A dependency is missing (because of `--disablerepo=*`) | Re-download with `dnf download --resolve --alldeps` |
| `Unknown repo: 'autoval-tools'` | Repo id doesn't match `yum_repo` | Make them identical (`autoval-tools` vs `autoval_tools`) |
| `Cannot download repomd.xml` / empty repo | Share not mounted on the DUT | `mount /shared/autoval`; check the fstab entry and the controller's NFS export |
| fio version check fails | fio older than 3.32 (default `fio_ver`) | Put a newer fio in the repo; match the controller's fio for `remote_fio` |
| `mkfs.btrfs: command not found` | btrfs isn't available on CentOS Stream 10 | Use `xfs` or `ext4` in the control file |
