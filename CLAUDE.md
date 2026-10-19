# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
bundle install
bin/kitchen list                                # default-<platform>, 3 instances
bin/kitchen converge default-debian-13-3008-2
bin/kitchen verify  default-debian-13-3008-2
bin/kitchen test    default-debian-13-3008-2
pre-commit run --all-files
```

The single suite `default` applies `cron._mapdata` then `cron`, with the pillar
`test/salt/pillar/cron.sls`.

## Architecture

Standard template layout under `cron/`: `init.sls` includes `package`,
`config`, `service`, each importing `cron` from `map.jinja` (the five-level
`os*map.yaml` merge). Only three keys come from the maps: `pkg`, `service`,
`enabled` (`defaults.yaml` says `cronie`/`cron`; `osfamilymap.yaml` sets Debian
to `cron`, RedHat to `crond`, Arch and Gentoo to `cronie`). `osarchmap.yaml`
sets `arch`, which no state reads; `osmap.yaml` and `osfingermap.yaml` are
empty.

- `package/install.sls` — `cron.install`: `pkg.installed` on `cron.pkg`.
- `service/running.sls` — `cron.service`: `service.running` + enabled, or
  `service.dead` + disabled when `cron:enabled` is false.
- `config/file.sls` — no file is managed (no `files/`, no TOFS). It loops over
  pillar `cron:tasks` → `cron.<task>` with `cron.present`/`cron.absent`
  (`type`, default `present`; `identifier` is the task key; `user` defaults to
  `root`; optional `minute`, `hour`, `daymonth`, `month`, `dayweek`, `comment`,
  `special`, `commented`), and over `cron:env` → `cron.env_present`/`env_absent`.
  `pillar.example` documents the keys. Schedule fields and `comment` are
  rendered as `'{{ value }}'`, so a single quote in a value breaks the YAML;
  `name` and the env `value` are not quoted at all.

`docs/README.rst` describing the config state as managing `cron.conf` from a
template in `cron/files` is upstream text that does not match the code.

## Tests

- InSpec (`test/integration/default/controls/`) only checks the package and the
  service. Nothing verifies the crontab entries, and there are no `_mapdata`
  reference dumps under `test/integration/default/files/`: the `_mapdata` state
  writes `/tmp/salt_mapdata_dump.yaml` but no control compares it.
- `cron/saltcheck-tests/*.tst` do check the tasks and env entries (via
  `cron.get_entry` / `cron.raw_cron`, mapping `special` to the module's `spec`),
  but neither Kitchen nor the CI runs saltcheck.
