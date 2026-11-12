# CentOS Stream 10 Migration: Before We Start

Written Nov 11, 2026. This checklist covers what to settle before rebuilding the AutoVal controller and DUTs. It goes with:

- [`centos10_migration_plan.md`](centos10_migration_plan.md): the migration plan, findings and phased checklist
- [`autoval_setup_guide_centos10.md`](autoval_setup_guide_centos10.md): step-by-step controller and DUT setup on Stream 10

The plan and guide cover **how** to rebuild. This list covers **what to decide first**. Items are in priority order.

## 1. Decide which tests and suites we actually run

This sets the size of everything else: which RPMs go in the el10 repo, what needs rebuilding, and what to validate.

Some tests depend on internal tools that aren't in the open-source code, so they won't work on a fresh install whatever the OS:

- `hwc power_reset` (QLC power-cycle test, `qlc_drive_data_integrity.py`)
- `enclosure-util`
- `upick_path`, `data_plugin` and `deploy_tool` (site settings and helpers carried over from upstream)

- [ ] List the suites and tests we run in production
- [ ] Mark any that rely on the internal-only tools above
- [ ] Use the list to scope the el10 RPM repo and the validation runs

## 2. Find the source for our in-house RPMs

These need el10 builds. If nobody knows where they're built, this is the biggest risk to the schedule.

- [ ] fio-synth / fiosynth
- [ ] `*.hs.el9` builds (e.g. `nvme-cli-2.8-1.hs.el9`)
- [ ] Pinned versions for FDP: `nvme-cli-2.10.2`, `libnvme-1.10`
- [ ] Vendor tools: sedutil, switchtec, lsiutil / scrtnycli, agfhc
- [ ] For each: owner, source or spec location, and how to rebuild for el10

## 3. Capture baseline results on CentOS 9 now

Kernel 6.12 and newer tools can change output formats and performance numbers. Without a baseline we can't tell a real regression from an OS difference.

- [ ] Run the key tests on each DUT platform while still on CentOS 9
- [ ] Save the results (and raw tool outputs: `nvme`, `smartctl`, `lspci`, `dmidecode`, `hdparm`, `sdparm`) somewhere outside `/shared/autoval`
- [ ] After migrating, compare el10 against these, including the fio and latency-monitor thresholds in `cfg/`

## 4. Make the small code fixes before the rebuild

So the new controller starts clean. Branch only. Review before committing.

- [ ] Add `PyYAML` to core `pyproject.toml` (`suite_runner.py` imports it)
- [ ] Add `attrs`, `pandas`, `packaging`, `PyYAML` to the SSD `pyproject.toml` (storage_hw_eng tests)
- [ ] Fix the default `ssh_key_path` in `site_settings.json` (`/USERNAME/.ssh/id_rsa` isn't a real path)
- [ ] Choose one repo id, `autoval-tools` or `autoval_tools`, and use it in settings, docs and DUT `.repo` files
- [ ] Add `continue_on_fail: true` to the suite YAMLs we use (the bundled ones say `continue_on_error`, which the runner ignores)
- [ ] Add Python 3.12 to the package classifiers
- [ ] Fix `BMC.__init__` in `lib/host/bmc.py`: a host entry without `oob_addr` currently crashes before the test starts (`'NoneType' object has no attribute 'hostname'`)

## 5. Pilot the setup guide on spare hardware

The setup guide was written from the code but hasn't been run end to end.

- [ ] One spare controller (a VM is fine if its CPU type is x86-64-v3 or "host") and one spare DUT
- [ ] Follow the guide exactly and record every fix
- [ ] Update the guide before touching the production lab

## 6. Keep site config in version control

Makes the next rebuild a re-run instead of a rewrite.

- [ ] Private repo for `/etc/autoval` (site settings, hosts files, control files, suites)
- [ ] Controller and DUT kickstart files in the same repo
- [ ] Keep BMC passwords out of git (a separate secrets file, or a vault)

## 7. Check with upstream first

The code is synced from an internal upstream.

- [ ] Does upstream already have el10 / Python 3.12 changes?
- [ ] Does upstream have a replacement for Gluster storage?
- [ ] Agree how our fixes flow back, to avoid a fork

## 8. Sequence the suite-log enhancement

Don't change logging and the OS at the same time, or failures become hard to pin down.

- [ ] Decide: finish the suite-log enhancement before the migration, or after the new controller passes validation

## 9. Answer the open questions and set a date

- [ ] Is the current controller's CPU x86-64-v3?
- [ ] NFS, or something else, for `/shared/autoval`?
- [ ] In what order do the DUT pools upgrade, and how long will el9 and el10 run side by side?
- [ ] Are any DUTs still on CentOS 7?
- [ ] How much results history moves to the new controller?
- [ ] Who owns rollback during cutover?
- [ ] Set the target date, working back from CentOS Stream 9 end of life: **May 31, 2027**
