# AutoVal Setup Guide: Controller and DUT on CentOS Stream 10

Snapshot exported Nov 11, 2026 from the working doc.

## How it fits together

```text
+-------------------------------------+                    +-------------------------------------+
| Controller (test server)            |  SSH :22 as root   | DUT (one per hosts file)            |
|  autoval_test_runner in venv        | -----------------> |  root login with controller key     |
|  /etc/autoval: settings, hosts,     |                    |  mounts /shared/autoval (NFSv4)     |
|    controls, suites                 |  NFSv4 :2049       |  autoval.repo -> autoval-tools      |
|  /autoval/logs, /tmp/autoval        | <----------------- |  boot drive + drives under test     |
|  exports /shared/autoval:           |                    +-------------------------------------+
|    results/ and pkgs/el10 RPM repo  |
+-------------------------------------+                    +-------------------------------------+
                  |          SSH :22 via sshpass           | BMC (optional)                      |
                  +--------------------------------------> |  OpenBMC: checks and power cycles   |
                                                           |  set as oob_addr in the hosts file  |
                                                           +-------------------------------------+
```

All test traffic starts on the controller. The only connection a DUT opens is its NFS mount of `/shared/autoval`, which carries both the results and the RPM repo tests install from. Set the parts up in the order of this guide: controller first, then each DUT, then verify.

## Before you start

This guide rebuilds a working AutoVal environment from bare metal: one **controller** (test server) that runs AutoVal, and any number of **DUTs** that AutoVal tests over SSH. Everything targets CentOS Stream 10. Commands marked *controller* or *DUT* run as root unless noted. Every requirement below was checked against the ocp-diag-autoval and ocp-diag-autoval-ssd code.

### Hardware

| Item | Requirement | Why |
| --- | --- | --- |
| Controller CPU | x86-64-v3 (roughly Intel Haswell / AMD Zen 1 or newer) | Stream 10 won't install on older CPUs |
| Controller disk | Enough for `/shared/autoval` (results + RPM repo) and `/autoval` (logs) | Long fio runs and suite logs add up; size for your retention period |
| DUT CPU | x86-64-v3 | Same Stream 10 baseline |
| DUT boot drive | A dedicated boot drive, separate from the drives under test. Note its PCI address (BDF, e.g. `0000:01:00.0`) | Control files use `boot_drive_physical_location` to keep tests off it. **SSD tests are destructive** |
| DUT BMC (optional) | OpenBMC reachable from the controller | Needed for BMC checks and AC/DC/warm power-cycle tests (`drive_data_integrity`) |

### Network

| From | To | Port / protocol | Used for |
| --- | --- | --- | --- |
| Controller | DUT | TCP 22 (SSH), ICMP | All test commands, file copies, `ping`/`ping6` reachability check |
| DUT | Controller | TCP 2049 (NFSv4) | Mounting `/shared/autoval` (results and the RPM repo) |
| Controller | BMC | TCP 22, ICMP (UDP 623 if you use IPMI-over-LAN) | BMC commands, power cycles |

- Hostnames in `hosts.json` must resolve from the controller (DNS or `/etc/hosts`), and the controller's name must resolve from every DUT.
- If you use IPv6, configure it on both sides. Parts of the code try `scp -6` and `ping6` first.
- Keep clocks in sync (chrony) so log timestamps line up across controller and DUTs.

### Accounts and access

| Where | Account | Requirement |
| --- | --- | --- |
| Controller | A run user (this guide uses `autoval`) | Owns the AutoVal venv and log dirs. Needs **passwordless sudo**: the BMC helper runs `sudo sshpass ...` |
| DUT | `root` | **AutoVal connects as root by default**, and file transfers (`put_file`, `scp`, `rsync`) always use root. Root must accept the controller's SSH key |
| BMC | BMC login | Set `oob_username` / `oob_password` per host in `hosts.json`. Without them the code falls back to OpenBMC defaults (`root` / `0penBmc`) |

### Decisions to make first

- **Shared storage:** this guide uses NFS for `/shared/autoval`, since GlusterFS isn't available on Stream 10. Adjust Part 2 if you choose differently.
- **Where DUTs get RPMs:** a local `autoval-tools` repo on the share (recommended, and required for in-house builds like fiosynth), or the normal CentOS + EPEL repos. Part 3 explains the trade-off.
- **Paths:** this guide keeps the defaults from `site_settings.json` (`/autoval/logs`, `/autoval/results`, `/tmp/autoval`). If you change them, change them in Part 6 as well.

## Controller part 1: OS, repos and system packages

1. **Install CentOS Stream 10** (Minimal Install is enough). Before installing, confirm the CPU level from any Linux live environment:

   ```bash
   /lib64/ld-linux-x86-64.so.2 --help | grep x86-64-v
   # needs: x86-64-v3 (supported, searched)
   ```
2. **Set hostname, network and time:**

   ```bash
   hostnamectl set-hostname autoval-ctrl.example.com
   nmcli con show                       # find the connection name
   nmcli con mod <conn> ipv4.method manual ipv4.addresses 10.0.0.10/24 ipv4.gateway 10.0.0.1 ipv4.dns 10.0.0.2
   nmcli con mod <conn> ipv6.method auto  # or manual, if your lab uses static IPv6
   nmcli con up <conn>
   dnf install -y chrony && systemctl enable --now chronyd
   ```

   Stream 10 uses NetworkManager keyfiles only. Old `ifcfg-*` files are not read.
3. **Enable CRB and EPEL 10** (`sshpass` comes from EPEL, and EPEL needs CRB):

   ```bash
   dnf install -y dnf-plugins-core
   dnf config-manager --set-enabled crb
   dnf install -y epel-release
   dnf -y update && reboot
   ```
4. **Install the system packages AutoVal uses on the controller:**

   ```bash
   dnf install -y \
     python3 python3-pip python3-devel gcc git \
     openssh-clients sshpass rsync tar iputils iproute \
     ipmitool lshw jq \
     nfs-utils createrepo_c dnf-utils rpm-build \
     fio fio-engine-libaio \
     policycoreutils-python-utils firewalld
   ```

| Package | Why AutoVal needs it on the controller |
| --- | --- |
| `python3`, `python3-pip`, `python3-devel`, `gcc` | Python 3.12 runtime and building wheels (pandas and friends, if no wheel exists) |
| `git` | Cloning the two AutoVal repos |
| `openssh-clients`, `ssh-agent`/`ssh-add` | SSH to DUTs, agent forwarding (`transport/ssh.py`) |
| `sshpass` (EPEL) | `scp`/`rsync`/`ssh` to DUTs and BMCs with a password (`transport/ssh.py`, `host/bmc.py`) |
| `rsync`, `tar` | Copying tools to DUTs, archiving logs into `resultsdir` (`site_utils.py`) |
| `iputils` | `ping`/`ping6` reachability check before every test (`host/host.py`) |
| `ipmitool`, `lshw` | BMC helper commands (`host/bmc.py`) |
| `jq` | Used by some storage\_hw\_eng tests and handy for checking JSON configs |
| `nfs-utils` | Serving `/shared/autoval` (Part 2) |
| `createrepo_c`, `dnf-utils`, `rpm-build` | Building the `autoval-tools` repo and the fiosynth RPM (Part 3) |
| `fio`, `fio-engine-libaio` | Only for `remote_fio` runs, where the controller drives fio. The controller's fio must be 3.32 or newer (the default `fio_ver` check) and should match the DUTs |
| `policycoreutils-python-utils`, `firewalld` | `semanage` for SELinux labels, and opening ports |

5. **Create the run user** with passwordless sudo:

   ```bash
   useradd -m autoval
   echo 'autoval ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/autoval
   chmod 440 /etc/sudoers.d/autoval
   ```

   You can run AutoVal as root instead. A named user keeps `suite_logs/<user>/...` folders meaningful, because the suite runner names them after the user who runs it.

## Controller part 2: directories, shared storage, SELinux and firewall

AutoVal needs local log and temp directories on the controller, plus one shared directory that every DUT mounts. The shared directory holds results and the `autoval-tools` RPM repo.

| Path | Site setting | Purpose | Owner |
| --- | --- | --- | --- |
| `/autoval/logs` | `control_server_logdir` | Per-test logs on the controller, archived into `resultsdir` at the end | `autoval` |
| `/tmp/autoval` | `control_server_tmpdir` | Scratch space, deleted after each test | `autoval` |
| `/shared/autoval/results` | `resultsdir` | Final archived results and `suite_logs/<user>/...` | `autoval` |
| `/autoval/repository` | `repository_dir` | Holds `tools/` (files tests copy to DUTs, e.g. latency-monitor settings), `golden_config/`, `bin/` (firmware) | `autoval` |
| `/shared/autoval/pkgs/el10` | — | The `autoval-tools` RPM repo (Part 3) | root |

1. **Give `/shared` its own filesystem** (recommended), for example XFS on a spare disk or LVM volume:

   ```bash
   mkfs.xfs /dev/<spare-disk>
   mkdir -p /shared
   echo '/dev/<spare-disk> /shared xfs defaults 0 0' >> /etc/fstab
   systemctl daemon-reload && mount /shared
   ```
2. **Create the directories:**

   ```bash
   mkdir -p /autoval/logs /autoval/repository/{tools,golden_config,bin} /tmp/autoval
   mkdir -p /shared/autoval/results /shared/autoval/pkgs/el10
   chown -R autoval:autoval /autoval /tmp/autoval /shared/autoval/results
   ```

   `/tmp/autoval` disappears on reboot, and AutoVal recreates subfolders as it needs them. To keep the top folder across reboots, add `d /tmp/autoval 0755 autoval autoval -` to `/etc/tmpfiles.d/autoval.conf`.
3. **Export `/shared/autoval` over NFS.** DUTs mount it as root, so it needs `no_root_squash`. Limit it to your lab subnet:

   ```bash
   echo '/shared/autoval 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)' > /etc/exports.d/autoval.exports
   systemctl enable --now nfs-server
   exportfs -rav
   ```
4. **Open the firewall:**

   ```bash
   firewall-cmd --permanent --add-service=ssh --add-service=nfs
   firewall-cmd --reload
   ```

   NFSv4 only needs TCP 2049. Add `mountd` and `rpc-bind` only if a client needs NFSv3.
5. **Keep SELinux enforcing.** Exports work with the default `nfs_export_all_rw` boolean. Check it, and relabel if you moved files in from elsewhere:

   ```bash
   getsebool nfs_export_all_rw        # expect: on
   restorecon -Rv /shared /autoval
   ausearch -m avc -ts recent         # any denials show up here
   ```

## Controller part 3: the autoval-tools RPM repo

SSD tests install their own tools on the DUT at the start of each run (`SystemUtils.install_rpms` in ocp-diag-autoval-ssd). What it runs depends on the `yum_repo` site setting:

| `yum_repo` | Command run on the DUT | What the DUT needs |
| --- | --- | --- |
| Set, e.g. `"autoval-tools"` | `sudo dnf -y --allowerasing --disablerepo=* --enablerepo=autoval-tools install <pkg>` (plus `-b` when force-installing) | **Every package and all of its dependencies** in that one repo. A repo whose id matches `yum_repo` exactly |
| Not set | `sudo dnf -y --allowerasing install <pkg>` | Working CentOS + EPEL repos (internet or a mirror). Can't supply in-house builds like fiosynth |

Use the local repo. It gives every DUT the same tool versions and works on isolated lab networks.

> The repo id and `yum_repo` must match character for character. The repo docs use `autoval-tools` but the site-settings example uses `autoval_tools`. Pick one (this guide uses `autoval-tools`).

### Packages to put in the repo

| Package | Installed by | Where to get it |
| --- | --- | --- |
| smartmontools, hdparm, sdparm, sg3\_utils, fio, fio-engine-libaio | `StorageTestBase` (`TOOLS` list), every storage test | Stream 10 repos |
| sshpass | `StorageTestBase`, `fio_runner.py` | EPEL 10 |
| boost-program-options | `fio_runner.py` | Stream 10 repos |
| gdisk | `boot_drive_partition_utils.py` | Stream 10 repos |
| nvme-cli, libnvme | nvme tests; FDP setup pins `nvme-cli-2.10.2` + `libnvme-1.10`; `nvme_cli_with_nvme_version.json` pins `nvme-cli-2.8-1.hs.el9` | Stream 10 repos for the current version. Pinned versions need your own el10 rebuilds, or change the pins |
| golang | `iogo` tests | Stream 10 repos (large dependency tree) |
| fiosynth | `fio_synth_flash`, synthflash workloads | Build from source (below) |
| switchtec, sedutil, lsiutil / scrtnycli, agfhc | switch, SED and HBA tests | Vendor RPMs or your own builds. Only needed if you run those tests |

### Build the repo

The safest way to capture the full dependency closure is to download inside a clean Stream 10 container. That way nothing already installed on the controller is skipped.

```bash
dnf install -y podman
podman run --rm -v /shared/autoval/pkgs/el10:/out:Z quay.io/centos/centos:stream10 bash -c '
  dnf install -y dnf-plugins-core epel-release &&
  dnf config-manager --set-enabled crb &&
  dnf download --resolve --alldeps --destdir /out \
    smartmontools hdparm sdparm sg3_utils fio fio-engine-libaio \
    sshpass boost-program-options gdisk nvme-cli libnvme golang'
```

**Build fiosynth** (it isn't packaged by any distribution):

```bash
git clone https://github.com/facebookincubator/FioSynth
cd FioSynth
python3 -m pip install --user setuptools   # bdist_rpm needs setuptools on Python 3.12
python3 setup.py bdist_rpm
cp dist/*.noarch.rpm /shared/autoval/pkgs/el10/
```

Copy any vendor or pinned-version RPMs into the same folder, then generate the metadata:

```bash
createrepo_c /shared/autoval/pkgs/el10
# after adding RPMs later:
createrepo_c --update /shared/autoval/pkgs/el10
```

The metadata must be regenerated every time RPMs are added or removed.

### Notes

- If any DUTs stay on CentOS Stream 9 for a while, keep their RPMs in `/shared/autoval/pkgs/el9` and point those DUTs at that folder. Never mix el9 and el10 RPMs in one repo.
- `btrfs-progs` isn't available on CentOS Stream 10 (no btrfs support in RHEL), so test options that format btrfs won't work. Use `xfs` or `ext4`.
- `dnf download --resolve --alldeps` also pulls packages a minimal install already has. That makes the repo bigger, but it means a DUT never stalls on a missing library.

## Controller part 4: install AutoVal in a Python venv

Run everything in this part **as the `autoval` user** (`sudo -iu autoval`). The controller needs to reach PyPI, or an internal mirror, for the Python packages.

1. **Clone both repos side by side.** The SSD repo expects core at `../ocp-diag-autoval`:

   ```bash
   mkdir -p ~/repos && cd ~/repos
   git clone https://github.com/opencomputeproject/ocp-diag-autoval.git
   git clone https://github.com/opencomputeproject/ocp-diag-autoval-ssd.git
   # or your team's forks/branches
   ```
2. **Create the venv and add the packages the code imports but doesn't declare yet:**

   ```bash
   python3 -m venv ~/autoval-env
   source ~/autoval-env/bin/activate
   pip install --upgrade pip build
   pip install PyYAML attrs pandas packaging
   ```

   | Package | Needed by | Symptom if missing |
   | --- | --- | --- |
   | PyYAML | core `suite_runner.py` | `--suite` fails with `No module named 'yaml'` |
   | attrs, pandas, packaging | SSD `tests/storage_hw_eng/` | Those tests fail to import |
3. **Install core, then SSD.** Pick one style:

   *Fixed install*, for a stable lab controller:

   ```bash
   cd ~/repos/ocp-diag-autoval && python -m build && pip install dist/ocptv_autoval-0.0.1.tar.gz
   cd ~/repos/ocp-diag-autoval-ssd && python -m build && pip install --no-deps dist/ocptv_autoval_ssd-0.0.1.tar.gz
   ```

   *Editable install*, if you change the code. Edits take effect without rebuilding:

   ```bash
   pip install -e ~/repos/ocp-diag-autoval
   pip install --no-deps -e ~/repos/ocp-diag-autoval-ssd
   ```

   Core's dependencies (`setuptools==70.0.0`, `ocptv==0.1.6`, `six`, `iopath`, `paramiko`) install automatically. `setuptools` matters: the code uses `pkg_resources`, and Python 3.12 venvs don't include it.
4. **Load the environment at login.** Part 6 creates the settings file this points to:

   ```bash
   cat >> ~/.bashrc <<'EOF'
   source ~/autoval-env/bin/activate
   export SITE_SETTINGS=/etc/autoval/site_settings.json
   EOF
   ```
5. **Check the install:**

   ```bash
   python -c "import autoval, autoval_ssd, yaml, pandas, attr, packaging; print('imports ok')"
   autoval_test_runner --help
   ```

> **Run AutoVal through the `autoval_test_runner` command**, not `python -m autoval.autoval_test_runner`. The suite runner starts each test by executing `sys.argv[0]` directly. With the console script that is the venv's launcher. With `python -m` it is the module file itself, which then has to be executable and finds `python3` through `PATH`.

## Controller part 5: SSH keys and BMC access

AutoVal connects to DUTs with paramiko as **root**, using the key files listed in `ssh_key_path` (Part 6). It also shells out to `ssh`, `scp` and `rsync` (via `sshpass`), always as `root@<host>`.

1. **Create a passphrase-less key** as the `autoval` user. Agent forwarding runs `ssh-add` non-interactively, so a passphrase would break it:

   ```bash
   ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
   ```
2. **Install the public key for root on every DUT.** Stream 10 defaults to `PermitRootLogin prohibit-password`: root may log in with a key but not a password. That means `ssh-copy-id root@dut` with a password won't work. Use one of these:
   - Put the key in the DUT kickstart (`%post`, shown in the DUT section). Best for a fleet.
   - Log in as another user with sudo and append it:

     ```bash
     cat ~/.ssh/id_ed25519.pub | ssh admin@dut01 'sudo install -d -m 700 /root/.ssh && sudo tee -a /root/.ssh/authorized_keys >/dev/null && sudo chmod 600 /root/.ssh/authorized_keys && sudo restorecon -R /root/.ssh'
     ```
3. **Accept each DUT's host key once** and confirm key login works:

   ```bash
   for h in dut01 dut02; do ssh -o StrictHostKeyChecking=accept-new root@$h 'hostname; id -u'; done
   ```

   Paramiko auto-adds unknown host keys, so this step is mostly a check. When a DUT is reinstalled and its host key changes, AutoVal runs `ssh-keygen -R <host>` and retries. You can also clear it yourself with `ssh-keygen -R dut01`.
4. **BMC access**. Per host, `hosts.json` takes `oob_addr`, `oob_username` and `oob_password` (or `oob1_addr`, `oob1_username`, ... for extra BMCs). Without credentials the code uses OpenBMC defaults (`root` / `0penBmc`). Check from the controller:

   ```bash
   sshpass -p '<bmc-password>' ssh -o StrictHostKeyChecking=no root@dut01-bmc 'hostname'
   ping -c 3 dut01-bmc      # or ping6
   ```

   **Every host entry currently needs an `oob_addr`.** In the current code, a host without one crashes before the test starts, with `AttributeError: 'NoneType' object has no attribute 'hostname'` from `host/bmc.py` (verified Nov 2026). `BMC.__init__` reads the connection's hostname before checking whether a BMC connection exists. Until that's fixed, a DUT with no BMC needs a placeholder `oob_addr`, such as the DUT's own hostname, and must not run BMC or power-cycle tests. The proper fix is a small guard in `BMC.__init__`.
5. **Older BMCs and crypto policy.** Stream 10's default crypto policy rejects SHA-1 signatures and DSA keys. If an old BMC fails with `no matching host key type` or a signature error, add a per-host exception in `~/.ssh/config` rather than weakening the system-wide policy:

   ```text
   Host dut01-bmc
       HostKeyAlgorithms +ssh-rsa
       PubkeyAcceptedAlgorithms +ssh-rsa
   ```

   This fixes the `ssh`/`sshpass` paths only. Paramiko doesn't read `~/.ssh/config`, so a BMC that paramiko can't negotiate with needs a firmware update.

## Controller part 6: settings, hosts, control files and suites

Keep your site's config outside the repos so a reinstall or `git pull` never overwrites it. This guide uses `/etc/autoval/`:

```bash
sudo install -d -o autoval -g autoval -m 750 /etc/autoval /etc/autoval/hosts /etc/autoval/controls /etc/autoval/suites
```

### site\_settings.json

AutoVal reads the file named by the `SITE_SETTINGS` environment variable. Use an absolute path. A bare file name is looked up inside the installed package's `cfg/site_settings/` folder instead.

```json
{
  "control_server_logdir": "/autoval/logs/",
  "control_server_tmpdir": "/tmp/autoval/",
  "dut_logdir": "/autoval/logs/",
  "dut_tmpdir": "/tmp/autoval/",
  "resultsdir": "/shared/autoval/results/",
  "ssh_key_path": ["/home/USERNAME/.ssh/id_ed25519"],
  "repository_dir": "/autoval/repository/",
  "plugin_config_path": "plugins/plugin_config.json",
  "test_utils_plugin_config_path": "plugins/test_utils_plugin_config.json",
  "cleanup_dut_logdirs": false,
  "yum_repo": "autoval-tools"
}
```

| Key | Required | What it does |
| --- | --- | --- |
| `control_server_logdir` | Yes | Controller log dir. Each test gets a subfolder, archived into `resultsdir` at the end |
| `control_server_tmpdir` | Yes | Controller scratch dir, deleted after each test |
| `dut_logdir`, `dut_tmpdir` | Yes | Created on the DUT for each test |
| `resultsdir` | Yes | Where archived results and `suite_logs/<user>/<date>/` go |
| `ssh_key_path` | Yes | List of private keys. `USERNAME` is replaced with the local user running AutoVal. Missing files are silently skipped. **The shipped default `/USERNAME/.ssh/id_rsa` points to a path that doesn't exist on a normal system: fix it** |
| `repository_dir` | No (default `autoval/`) | Base for `tools/`, `golden_config/` and `bin/` on the controller |
| `plugin_config_path`, `test_utils_plugin_config_path` | No | Plugin wiring, relative to the package. Leave the defaults |
| `cleanup_dut_logdirs` | No (default **true** when absent) | Delete DUT log dirs after each test. `false` keeps them for debugging |
| `yum_repo` | No | Repo id DUTs install test tools from (Part 3). Must match the `[id]` in the DUT's `.repo` file |
| `dut_critical_services` | No | List of services checked on the DUT |
| `enable_ocp_diag_output` | No (default false) | Turn on OCP Diag JSONL output for every test (a control file can also set it) |

### hosts.json (the `--config` file)

One file per DUT matches how the team works, since each host runs on its own. `/etc/autoval/hosts/dut01.json`:

```json
{
  "hosts": [
    {
      "hostname": "dut01.example.com",
      "oob_addr": "dut01-bmc.example.com",
      "oob_username": "root",
      "oob_password": "<bmc-password>"
    }
  ]
}
```

| Field | Default | Notes |
| --- | --- | --- |
| `hostname` | — (required) | Must resolve from the controller |
| `username` | `root` | Leave unset. File transfers use root regardless |
| `password` | none | DUT password for the `sshpass` copy paths. Key login makes it optional |
| `sudo` | false | Prefix commands with `sudo`. Only needed for a non-root `username` |
| `oob_addr`, `oob_username`, `oob_password` | none / `root` / `0penBmc` | BMC. `oob1_addr` and so on for more BMCs |
| `local_mode` | false | Run commands on the controller instead of over SSH (for development) |

The file can hold passwords, so `chmod 600` it.

### Control files (`--test_control`)

A control file is a flat JSON object of test parameters. Tests read it with `self.test_control.get(...)`. `--args '{...}'` on the command line overrides individual keys. Most storage tests use these keys from `StorageTestBase`:

| Key | Default | Meaning |
| --- | --- | --- |
| `drive_type` | none | `ssd`, `hdd`, ... |
| `drive_interface` | none | `nvme`, `sata`, `sas` |
| `drives` | `[]` (auto-discover) | Explicit list, e.g. `["nvme1n1", "nvme2n1"]` |
| `boot_drive_physical_location` | `""` | **Set it.** BDF of the boot drive, e.g. `0000:01:00.0`, so discovery can exclude it |
| `include_boot_drive` / `only_boot_drive` | false / false | Test the boot drive too / only the boot drive |
| `drive_model` | none | Only test drives of this model |
| `ignore_smart`, `collect_drive_data` | false / true | SMART checks and drive log collection around the test |
| `remove_partition`, `format_partition` | true / false | Partition handling on test drives |
| `enable_ocp_diag_output` | site setting | OCP Diag JSONL output for this test |

Find the boot drive's BDF on the DUT with:

```bash
findmnt -no SOURCE /       # e.g. /dev/nvme0n1p3
readlink -f /sys/block/nvme0n1/device/device   # ends in the BDF, e.g. .../0000:01:00.0
```

Start from the control JSONs shipped next to each test (`src/autoval_ssd/tests/<test>/control*.json`) and copy them into `/etc/autoval/controls/` with your BDF filled in.

### Suite YAML (`--suite`)

A suite is a YAML list. The open-source runner uses only three keys per entry:

```yaml
- cmd: autoval_ssd.tests.nvme_cli.nvme_cli
  args:
    test_params:                 # written to that test's test_control.json
      drive_type: ssd
      drive_interface: nvme
      boot_drive_physical_location: "0000:01:00.0"
  continue_on_fail: true
```

- **`continue_on_fail`** decides whether the suite keeps going after a failure. The bundled suites in `cfg/test_suites/` say `continue_on_error`, which the runner ignores, so they **stop at the first failed test**. Add `continue_on_fail: true` in your copies.
- `name`, `timeout`, `min_hosts`, `max_hosts`, `execution_type`, `hostage`, `skip_health_check` and `runner` are carried over from the internal runner. The open-source runner ignores them, so **`timeout` is not enforced**.
- Each test runs as a separate `autoval_test_runner` process with a copy of the `--config` file (`remote_cfg.json`). Its logs land under `resultsdir/suite_logs/<user>/<date>/<test>_<date>/`.

## DUT setup

A DUT needs much less than the controller: root SSH access from the controller, the share mounted, the repo configured, and a handful of base tools the tests call but don't install themselves. AutoVal creates its own DUT directories (`dut_logdir`, `dut_tmpdir`) at the start of each test.

### 1. Install the OS

- Install CentOS Stream 10 (Minimal) on the **boot drive only**. Leave the drives under test unpartitioned. Storage tests reformat, sanitize and overwrite them.
- Set hostname, network (IPv4 and IPv6 if used) and chrony as on the controller (Part 1, step 2). The DUT's name must match the `hostname` in its hosts file.
- Keep firewalld and SELinux on. The default firewalld zone already allows SSH and ICMP. NFS is client-only on the DUT, so no ports need opening.
- Make sure the DUT boots unattended: power-cycle tests (warm, DC, AC) reboot it and then wait for SSH to come back.

### 2. Base packages

Tests install the storage tools from the repo themselves (Part 3). These are called but **not** installed by the tests, so put them in the image:

```bash
dnf install -y \
  openssh-server rsync tar sudo which file python3 gcc \
  nfs-utils pciutils lsscsi dmidecode lshw ipmitool \
  parted gdisk mdadm xfsprogs e2fsprogs \
  psmisc procps-ng jq nvme-cli
```

| Package | Used for |
| --- | --- |
| `rsync`, `tar` | Controller copies files with `rsync` to `root@dut` and archives DUT logs |
| `python3` | `ssd_fileappend` runs a Python script on the DUT |
| `gcc` | `fsync` test compiles `fsync.c` on the DUT |
| `pciutils`, `lsscsi`, `dmidecode`, `lshw`, `ipmitool` | Drive and PCI discovery (`lspci`, `setpci`, `lsscsi`), system info, in-band BMC queries |
| `parted`, `gdisk`, `mdadm`, `xfsprogs`, `e2fsprogs` | `partprobe`, `sgdisk`, software RAID tests, `mkfs.xfs` / `mkfs.ext4` |
| `psmisc`, `procps-ng` | `killall`, `pkill`, `ps` used to stop fio and other workloads |
| `jq` | storage\_hw\_eng tests edit JSON files on the DUT with `jq` |
| `nvme-cli` | Lets you check drives before the first run. Tests may replace the version from the repo |

The test-installed tools (smartmontools, hdparm, sdparm, sg3\_utils, fio, sshpass, boost-program-options, gdisk, golang, fiosynth, and so on) can be preinstalled too. The tests still force-reinstall them from `autoval-tools`.

### 3. Root SSH access

Add the controller's public key to `/root/.ssh/authorized_keys` (Part 5, step 2). Keep the Stream 10 default `PermitRootLogin prohibit-password`. If someone changed it, restore it:

```bash
grep -ri permitrootlogin /etc/ssh/sshd_config /etc/ssh/sshd_config.d/   # want prohibit-password (or yes)
systemctl restart sshd
```

### 4. Mount the share

```bash
mkdir -p /shared/autoval
echo 'autoval-ctrl.example.com:/shared/autoval /shared/autoval nfs4 defaults,_netdev,x-systemd.automount 0 0' >> /etc/fstab
systemctl daemon-reload
mount /shared/autoval && df -h /shared/autoval
```

`_netdev` and `x-systemd.automount` stop the boot from hanging if the controller is briefly unreachable after a power cycle. The share mounts on first access.

### 5. Point dnf at autoval-tools

The section name must equal `yum_repo` in `site_settings.json`:

```bash
cat > /etc/yum.repos.d/autoval.repo <<'EOF'
[autoval-tools]
name=AutoVal test tools (el10)
baseurl=file:///shared/autoval/pkgs/el10
enabled=1
gpgcheck=0
EOF
dnf --disablerepo='*' --enablerepo=autoval-tools makecache
dnf --disablerepo='*' --enablerepo=autoval-tools list available | head
```

### 6. Prepare the drives under test

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,MOUNTPOINTS
nvme list
findmnt -no SOURCE /                             # identify the boot drive
readlink -f /sys/block/nvme0n1/device/device     # its BDF for boot_drive_physical_location
```

- No drive under test may be mounted, in `/etc/fstab`, part of LVM, or part of an active md RAID. Stop leftovers with `mdadm --stop /dev/md<N>` and clear old signatures with `wipefs -a /dev/<drive>` (**destroys data**, only on test drives).
- Tests mount scratch filesystems under `/mnt/havoc*` (for example `/mnt/havoc_nvme1n1`). Don't use those paths for anything else.
- Firmware-update tests read images from `repository_dir/bin` on the controller (`/autoval/repository/bin`).

### 7. BMC (if present)

Give the BMC an address the controller can reach, record its login in the hosts file (`oob_addr`, `oob_username`, `oob_password`), and test it from the controller (Part 5, step 4).

### Kickstart snippet for a DUT fleet

Add to your DUT kickstart so every reinstall comes back test-ready:

```bash
%packages
@^minimal-environment
openssh-server
rsync
tar
sudo
which
file
python3
gcc
nfs-utils
pciutils
lsscsi
dmidecode
lshw
ipmitool
parted
gdisk
mdadm
xfsprogs
e2fsprogs
psmisc
jq
nvme-cli
chrony
%end

%post
install -d -m 700 /root/.ssh
echo 'ssh-ed25519 AAAA...your-controller-key... autoval@autoval-ctrl' >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
restorecon -R /root/.ssh
mkdir -p /shared/autoval
echo 'autoval-ctrl.example.com:/shared/autoval /shared/autoval nfs4 defaults,_netdev,x-systemd.automount 0 0' >> /etc/fstab
cat > /etc/yum.repos.d/autoval.repo <<'EOF'
[autoval-tools]
name=AutoVal test tools (el10)
baseurl=file:///shared/autoval/pkgs/el10
enabled=1
gpgcheck=0
EOF
systemctl enable chronyd
%end
```

## Verify the setup

Work from the cheapest check to the most invasive. Run as `autoval` on the controller unless noted. Stop at the first failure and use the troubleshooting table below.

- [ ] **Environment:** `echo $SITE_SETTINGS` prints `/etc/autoval/site_settings.json`, and `python -m json.tool $SITE_SETTINGS` parses it
- [ ] **Python:** `python -c "import autoval, autoval_ssd, yaml, pandas"` and `autoval_test_runner --help` both work
- [ ] **Writable dirs:** `touch /autoval/logs/.t /tmp/autoval/.t /shared/autoval/results/.t && rm /autoval/logs/.t /tmp/autoval/.t /shared/autoval/results/.t`
- [ ] **SSH as root with the configured key:** `ssh -i ~/.ssh/id_ed25519 root@dut01.example.com 'hostname; id -u'` prints the DUT name and `0`
- [ ] **DUT share and repo** (on the DUT): `df -h /shared/autoval`, then `dnf --disablerepo='*' --enablerepo=autoval-tools install -y smartmontools`
- [ ] **BMC** (if any): `sshpass -p '<pw>' ssh root@dut01-bmc hostname`
- [ ] **Dry run:** `autoval_test_runner autoval.tests.connect_test --config /etc/autoval/hosts/dut01.json --dry_run`
- [ ] **Connect test** (non-destructive: SSH to the DUT, ping the BMC):

  ```bash
  autoval_test_runner autoval.tests.connect_test \
    --config /etc/autoval/hosts/dut01.json \
    --args '{"enable_ocp_diag_output": true}'
  ```

  It should end with a passing summary. Results appear under `/shared/autoval/results/`, with `test_results.json`, `test_steps.json` and the OCP Diag `.jsonl` file.
- [ ] **First storage test, on a dedicated DUT only.** SSD tests format and overwrite drives. Check the control file's `boot_drive_physical_location` first:

  ```bash
  autoval_test_runner autoval_ssd.tests.nvme_cli.nvme_cli \
    --config /etc/autoval/hosts/dut01.json \
    --test_control /etc/autoval/controls/nvme_cli.json
  ```

  In the log, confirm that the RPM installs come from `autoval-tools` and the boot drive is excluded from the test drives.
- [ ] **Short suite:** a two-entry YAML with `continue_on_fail: true`:

  ```bash
  autoval_test_runner --suite /etc/autoval/suites/smoke.yaml --config /etc/autoval/hosts/dut01.json
  ls /shared/autoval/results/suite_logs/$USER/
  ```

Where to look when something fails: the console output first, then the test's folder under `control_server_logdir` (while it runs) or `resultsdir` (after). For suites, each test's debug log is copied into its `suite_logs/<user>/<date>/<test>_<date>/` folder.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Site setting <name> not found` | `SITE_SETTINGS` unset, misspelled, or a relative path | `export SITE_SETTINGS=/etc/autoval/site_settings.json` (absolute). Check all required keys exist |
| `Authentication Failure : Failed to establish ssh connection` | Key path in `ssh_key_path` doesn't exist (skipped silently), key not in DUT root's `authorized_keys`, or the key has a passphrase | Fix the path (`/home/USERNAME/...`). Install the key for root. Use a passphrase-less key |
| `No module named 'yaml'` (or `attr`, `pandas`) | Undeclared Python dependencies | `pip install PyYAML attrs pandas packaging` in the venv |
| `No module named 'pkg_resources'` | setuptools missing (Python 3.12 venv, `PYTHONPATH` dev setup) | `pip install setuptools==70.0.0` |
| `sshpass: command not found` | EPEL not enabled on the controller (or DUT) | Enable CRB + EPEL 10, then `dnf install sshpass` |
| DUT: `No match for argument` / `nothing provides ...` during test setup | Package or a dependency missing from `autoval-tools`, or metadata not regenerated | Add the RPMs (`dnf download --resolve --alldeps`) and rerun `createrepo_c --update` |
| DUT: `Unknown repo: 'autoval-tools'` | Repo id in `autoval.repo` doesn't match `yum_repo` (`autoval-tools` vs `autoval_tools`) | Make them identical |
| Suite stops after the first failed test | Bundled YAMLs use `continue_on_error`, which the runner ignores | Add `continue_on_fail: true` to each entry |
| Suite tests fail to start (`Permission denied`, wrong Python) | Suite launched with `python -m autoval.autoval_test_runner` | Launch with the `autoval_test_runner` command from the venv |
| Test picks the boot drive, or refuses to run | Missing or wrong `boot_drive_physical_location` | Set the BDF from `readlink -f /sys/block/<dev>/device/device` |
| DUT hangs at boot or the share is empty after a power cycle | NFS mount at boot without network ordering | Use `_netdev,x-systemd.automount` in `/etc/fstab`. Check the controller firewall allows `nfs` |
| NFS mount fails: `access denied` | Export doesn't cover the DUT's subnet, or exports not reloaded | Fix `/etc/exports.d/autoval.exports`, then `exportfs -rav` |
| `Permission denied` writing logs on the controller | Directory owned by root, or an SELinux label | `chown -R autoval:autoval /autoval /shared/autoval/results`, then `restorecon -Rv`. Check `ausearch -m avc` |
| BMC: `no matching host key type` / signature errors | Old BMC firmware vs Stream 10 crypto policy | Per-host `HostKeyAlgorithms +ssh-rsa` in `~/.ssh/config`, or update the BMC firmware |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` after a reinstall | Stale host key | `ssh-keygen -R dut01.example.com` |
| fio version check fails | DUT or controller fio older than 3.32 (default `fio_ver`) | Put a newer fio in the repo. Install the same version on the controller for `remote_fio` |
| `mkfs.btrfs: command not found` | btrfs isn't available on Stream 10 | Use `xfs` or `ext4` in the control file |
| `AttributeError: 'NoneType' object has no attribute 'hostname'` (host/bmc.py) before the test starts | Host entry has no `oob_addr` | Add `oob_addr` (the BMC, or the DUT's own hostname as a placeholder if it has no BMC), or apply the `BMC.__init__` fix |

## Reference

### Files and locations

| What | Where | Machine |
| --- | --- | --- |
| AutoVal code | `~autoval/repos/ocp-diag-autoval`, `~autoval/repos/ocp-diag-autoval-ssd` | Controller |
| Python venv | `~autoval/autoval-env` | Controller |
| Site settings | `/etc/autoval/site_settings.json` (`$SITE_SETTINGS`) | Controller |
| Hosts, controls, suites | `/etc/autoval/hosts/`, `/etc/autoval/controls/`, `/etc/autoval/suites/` | Controller |
| SSH key | `~autoval/.ssh/id_ed25519` | Controller |
| Logs / scratch | `/autoval/logs`, `/tmp/autoval` | Controller and DUT |
| Results | `/shared/autoval/results` | Controller (NFS export) |
| RPM repo | `/shared/autoval/pkgs/el10` | Controller, mounted on DUTs |
| Tools, golden configs, firmware | `/autoval/repository/{tools,golden_config,bin}` | Controller |
| DUT repo file | `/etc/yum.repos.d/autoval.repo` | DUT |
| DUT share mount | `/shared/autoval` (fstab, NFSv4) | DUT |
| DUT authorized key | `/root/.ssh/authorized_keys` | DUT |

### Older docs this guide supersedes for Stream 10

- `ocp-diag-autoval-ssd/src/autoval_ssd/docs/gluster_configuration.md`: GlusterFS isn't available on Stream 10. Use NFS (Part 2).
- `ocp-diag-autoval-ssd/src/autoval_ssd/docs/rpm_repo_hosting.md`: still correct in outline. This guide adds the el10 folder, full-dependency downloads and the repo-id match.
- `ocp-diag-autoval/documentation/centos10_migration_plan.md`: the migration checklist this setup feeds into.

### Sources

- [ocp-diag-autoval](https://github.com/opencomputeproject/ocp-diag-autoval) and [ocp-diag-autoval-ssd](https://github.com/opencomputeproject/ocp-diag-autoval-ssd): code and docs every requirement here was checked against (`site_utils.py`, `transport/ssh.py`, `connection_dispatcher.py`, `credentials.py`, `suite_runner.py`, `system_utils.py`, `storage_test_base.py`)
- [FioSynth](https://github.com/facebookincubator/FioSynth): source for the fiosynth RPM
- [CentOS Stream 10 release notes](https://centos.org/centos10/)
- [Red Hat Bugzilla 2307291: sshpass in EPEL 10](https://bugzilla.redhat.com/show_bug.cgi?id=2307291)
