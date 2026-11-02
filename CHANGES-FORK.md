# Changes in this fork

This is a fork of [sidoh/esp8266_milight_hub](https://github.com/sidoh/esp8266_milight_hub),
based on upstream `master` at commit `23102c7` (release 1.13.0 + 5 commits, tagged
`pre-group-fix` here). Upstream has not merged changes since late 2024, so these fixes
are maintained here.

The fork focuses on one problem: **MQTT state of individual groups and of group 0 ("all")
getting out of sync**, e.g. in Home Assistant (upstream issue
[#703](https://github.com/sidoh/esp8266_milight_hub/issues/703)).

## Credits

This fork builds on upstream pull request
[#779 "MQTT fixes"](https://github.com/sidoh/esp8266_milight_hub/pull/779) by
**Cody Cutrer ([@ccutrer](https://github.com/ccutrer))**. Thanks also to Chris Mullins
([@sidoh](https://github.com/sidoh)) for the original project.

## This is more than PR #779

PR #779 is the starting point, but this fork is **not** just PR #779 applied as-is.
It contains a fix to one part of the PR and an additional feature that is not part of
any upstream pull request:

| # | Change | Origin |
|---|---|---|
| 1 | Publish child group states for commands to group 0 | **PR #779**, taken over essentially unchanged |
| 2 | Re-publish all aliased states on MQTT (re)connect | Idea from **PR #779**, **fixed in this fork**: as written in the PR, unchanged states were skipped, so nothing was re-published |
| — | NULL checks for unknown remote types / missing states | **New in this fork** |
| 3 | Group 0 on/off state follows the individual groups | **New in this fork**, not part of any upstream PR |
| 4 | Web UI loads its assets from the device instead of a dead CDN path | **PR #923** (fix commit only, without the rest of that PR) |

## Changes

### 1. Publish child group states for commands to group 0 (from PR #779)

Commits: `d453355`

A command to group 0 already updated the hub's internal state of every individual group
(`GroupStateStore::set`), but only group 0's state was published via MQTT. Entities for
groups 1..n kept showing stale values. `BulbStateUpdater::enqueueUpdate` now also enqueues
all child groups when group 0 changes. This part is taken from PR #779 essentially unchanged.

### 2. Re-publish all aliased states on MQTT (re)connect (idea from PR #779, fixed)

Commits: `d453355`

PR #779 enqueues all aliased groups on MQTT connect. However, the updater only publishes
states whose MQTT dirty flag is set, so unchanged states were silently skipped and nothing
was re-published. This fork adds `BulbStateUpdater::enqueueRefresh()`, which sets the dirty
flag before enqueuing (and makes `GroupState::setMqttDirty()` an actual, non-inline
function). Result: retained MQTT state is current again after a restart of the hub or
the broker.

Also adds NULL checks for unknown remote types and missing states.

### 3. Derive group 0 on/off state from the individual groups (new in this fork)

Commits: `2a328e6`

After a command to an individual group, group 0 is set to **ON** if any group is on
(or in night mode) and to **OFF** if all are off, and then published. Previously group 0
kept its last state, e.g. "all" still showed "on" after every lamp had been switched off
one by one.

Only groups that have an alias are considered, because a command to group 0 also switches
groups without any lamp attached (e.g. group 4 of a 4-zone remote). If no group of the
device has an alias, all groups are considered. Brightness and other fields of group 0
are not recomputed.

### 4. Load web UI assets from the device (from PR #923)

Commits: `d2c55c8`

The page loader (`web2/src/pageload.js`) first tried to fetch the CSS/JS bundles from
`cdn.jsdelivr.net/gh/sidoh/esp8266_milight_hub@web2/...`. That branch no longer exists, so
every page load made failing requests to an external server before falling back to the
copy on the device. The loader now uses the on-device bundles directly. The device serves
them gzipped with long-lived cache headers and hashed file names, so after the first load
they come from the browser cache.

Taken from upstream pull request
[#923](https://github.com/sidoh/esp8266_milight_hub/pull/923) by
[@TehRobot-Assistant](https://github.com/TehRobot-Assistant) (fixes upstream issue #864).
Only the actual fix commit was taken over. The PR is based on the unreleased upstream
`1.13.1` branch, and its other changes are not part of this fork. `dist/index.html.gz.h`
was regenerated with `npm run build` in `web2/`. The CSS/JS bundles are unchanged
(identical content).

### Documentation: README state topic example (from PR #863)

Commits: `fd926b3`

The README used `milight/states/...` as the example state topic, while the default
since 1.13 is `milight/state/...` (`lib/Settings/Settings.h`). Taken over from upstream
pull request [#863](https://github.com/sidoh/esp8266_milight_hub/pull/863) by
[@soif](https://github.com/soif) (see also issue #860). Documentation only, no firmware change.

## Versions

| Version | Contents |
|---|---|
| `1.13.0-kkcb.1` | Changes 1–3 above |
| `1.13.0-kkcb.2` | Change 4 (web UI without CDN) |

## Building

Same as upstream (`pio run -e nodemcuv2`). On Apple Silicon without Rosetta, the
toolchain of `espressif8266@~4` does not run natively; building inside a Linux container
works:

```sh
docker run --rm -v "$PWD":/work -w /work python:3.12-slim bash -c \
  'apt-get update -qq && apt-get install -y -qq git >/dev/null &&
   git config --global --add safe.directory /work &&
   pip install -q platformio && pio run -e nodemcuv2'
```
