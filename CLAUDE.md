# CLAUDE.md — ocp-diag-autoval

Core AutoVal hardware-validation test framework (PyPI name `ocptv-autoval`, import name `autoval`). Provides the test runner, `TestBase`, validators, host/BMC connections and logging/results handling. Test suites live in sibling repos — mainly `../ocp-diag-autoval-ssd` (import name `autoval_ssd`), which pins `ocptv-autoval==0.0.1`. Keep the two repos compatible.

## Layout (`src/autoval/`)
- `autoval_test_runner.py` — entry point (`python -m autoval.autoval_test_runner` or the `autoval_test_runner` console script). `cli/cli_parser.py` defines the CLI.
- `lib/test_base.py` — `TestBase`, the abstract parent of every test. Lifecycle: `__init__` → `setup()` → `execute()` (must be implemented) → `cleanup()`; `on_fail()` runs when a test fails. `self.test_control` holds the control-file params (plus `--args` overrides); `self.host` / `self.localhost` are the DUT and controller.
- `lib/suite_runner.py` — runs `--suite` YAML files: a list of entries with `cmd` (dotted test module), optional `args`, `continue_on_fail`. Each test gets its own log dir under `suite_logs`. Imports `yaml` (PyYAML), which is **not** declared in `pyproject.toml` yet — install it yourself.
- `lib/test_args.py` — test-control / CLI argument plumbing.
- `lib/connection/` — `connection_dispatcher.py` (host + BMC connections, `localhost`), `connection_factory.py`, `connection_utils.py`.
- `lib/transport/` — `ssh.py` (paramiko client, `sshpass` scp/rsync/ssh shell-outs, `SSHAgent` forwarding) and `local.py` (commands on the controller).
- `lib/host/` — `host.py` (ping/ping6 reachability, `run`), `bmc.py`, `system.py`, `credentials.py`, `component/` (`COMPONENT` enum and per-component classes: bmc, bios, cpu, nic, asic, dut).
- `lib/utils/` — `autoval_utils.py` (validators, used as `self.validate_*`), `autoval_errors.py` (`ErrorType`), `autoval_exceptions.py`, `autoval_log.py` (`AutovalLog`), `site_utils.py` (site settings, log/results/tmp dirs, `yum_repo`), `result_handler.py` (`test_results.json`), `autoval_output.py` (OCP Diag JSONL output), `file_actions.py`, `generic_utils.py`, `autoval_thread.py`, `bg_runner.py`, `manifest.py`, config-check helpers.
- `lib/test_utils/` — background test-utility runners (`test_utils_base.py`, `test_utils_runner.py`, `bg_runner.py`).
- `plugins/` — `plugin_manager.py` + `plugin_config.json` (swappable implementations, e.g. `config_datasource`).
- `cfg/site_settings/` — `site_settings.json` (default) and `site_settings_devserver.json`; `cfg/thresholds/` (e.g. `latency_monitor.json`).
- `tests/connect_test.py` — the only test in core: SSH to the DUT and ping the BMC. Use it as a smoke test and a minimal example.
- `documentation/` — `autoval_core.md` (architecture, site settings, OCP Diag output), `contribution_guide.md` (how to write a test), `centos10_migration_plan.md` (CentOS Stream 9 → 10 plan for controller and DUTs), `autoval_setup_guide_centos10.md` (step-by-step controller + DUT setup on Stream 10).

## Setup
```bash
python -m venv env && source env/bin/activate
pip install build PyYAML            # PyYAML is needed for --suite but undeclared
python -m build && pip install ./dist/ocptv_autoval-0.0.1.tar.gz
```
For development without rebuilding: `PYTHONPATH=src` plus the runtime deps `setuptools==70.0.0 ocptv==0.1.6 six iopath paramiko PyYAML`. `setuptools` must be installed explicitly in Python 3.12+ venvs because the code uses `pkg_resources` (`plugin_manager.py`, `generic_utils.py`, `file_actions.py`).

## Running
```bash
export SITE_SETTINGS=src/autoval/cfg/site_settings/site_settings.json
python -m autoval.autoval_test_runner autoval.tests.connect_test --config ./hosts.json
python -m autoval.autoval_test_runner <dotted.test.module> --config ./hosts.json \
    --test_control path/to/control.json [--args '{"key": value}'] [--debug] [--dry_run]
python -m autoval.autoval_test_runner --suite path/to/suite.yaml --config ./hosts.json
```
- `SITE_SETTINGS` must be exported first. Keys: `control_server_logdir`, `control_server_tmpdir`, `dut_logdir`, `dut_tmpdir`, `resultsdir`, `ssh_key_path` (list), `plugin_config_path`, `repository_dir`, `test_utils_plugin_config_path`, `cleanup_dut_logdirs`, optional `yum_repo`.
- `hosts.json` has a `hosts` list; each entry uses `hostname`, `oob_addr` (BMC; currently required, see below), `username`, `password`, `sudo`, `local_mode`, `force_ssh`.
- Known bug (Nov 2026): a host entry without `oob_addr` crashes in `lib/host/bmc.py` (`BMC.__init__` reads `connection.hostname` while the connection is `None`). Workaround: a placeholder `oob_addr`. Fix: guard `BMC.__init__` for a `None` connection.
- Add `"enable_ocp_diag_output": true` to a control file to get OCP Diag JSONL output.
- Our team runs each host on its own; multi-host runs aren't used.
- Never run tests against real hosts unless the user explicitly asks and names the host. SSD tests built on this framework are destructive.

## Controller dependencies outside Python
The framework shells out on the controller to `sshpass`, `ssh`/`scp`/`rsync`, `ssh-agent`/`ssh-add`, `ping`/`ping6`, `tar`, and `ipmitool`. IPv6 is assumed in places (`scp -6`, `ping6`). On CentOS Stream 10, `sshpass` comes from EPEL 10 (needs CRB enabled).

## Tests
Core has no unit-test suite; the `test_*.py` files under `lib/` are framework modules, not tests. Verify changes by:
- importing every module under Python 3.12 (`python -W error::SyntaxWarning -m compileall -q src/autoval`, then import-walk the package);
- `autoval_test_runner --help` and a `--dry_run`;
- running the `../ocp-diag-autoval-ssd` unit tests, which exercise core through mocks (see that repo's CLAUDE.md).

Known state (Nov 2026): compiles and imports cleanly on Python 3.10 and 3.12, except `lib/suite_runner.py` when PyYAML is missing. Warnings only: `pkg_resources` deprecation, `Thread.setName()` in `autoval_thread.py`.

## Lint / format
`make format` (ufmt), `make lint_all` (ufmt check), `make lint` (pre-commit, ufmt v2.0.0). Format before committing.

## Conventions
- New test: subclass `TestBase`, read params with `self.test_control.get("key", default)` in `__init__`, implement `execute()`, validate with `self.validate_*` using `component=COMPONENT.<X>` and `error_type=ErrorType.<Y>`, and log with `self.log_info` / `AutovalLog`. Call `super().setup()` / `super().cleanup()` when overriding them. See `documentation/contribution_guide.md`.
- Match the header and typing style of neighboring files (many carry `pyre-fixme` comments; only a few have `# pyre-unsafe`).
- Code is synced from an internal upstream ("havoc" names, e.g. `[HAVOC SYSTEM INFO ERROR]` prefixes). Keep changes minimal and don't rename those identifiers unless asked. The ssd unit tests expect some of these strings.
- Keep `remote_cfg.json` and the suite DEBUG log handling as upstream has them. Put suite-log enhancements (planned layout: `suite_logs/<user>/<DUT host>/<suite>_<timestamp>_<Zeus test id>/<test>_<timestamp>/`, Zeus test id from a CLI arg) in separate files rather than inside `suite_runner.py`.
- New non-Python assets must match the package-data glob in `pyproject.toml` (`*.json`) or they won't ship.
- Changes that affect `autoval_ssd` must keep `../ocp-diag-autoval-ssd` working; run its unit tests.
