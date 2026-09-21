# pmodlink

**A Lua-scripted industrial IoT gateway.** Connects Modbus, OPC DA/UA, MQTT and HTTP —
custom protocols and custom logic at the edge.

pmodlink (ModLink Gateway) is a Windows desktop application that acquires data from
Modbus TCP/RTU devices and OPC DA servers, republishes it over OPC DA and OPC UA,
publishes upstream over MQTT, and lets you implement custom protocols and logic with
sandboxed Lua 5.4 scripts.

> **Status: 0.1.0, pre-release.** Interfaces, configuration keys and license terms may
> change before the first production release.

---

## Contents

- [Editions](#editions)
- [Features](#features)
- [Repository layout](#repository-layout)
- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Lua scripting](#lua-scripting)
- [Building a standalone executable](#building-a-standalone-executable)
- [Testing](#testing)
- [Documentation](#documentation)
- [License](#license)
- [Safety notice](#safety-notice)

---

## Editions

|                                            | Core                                               | Professional                                                          |
| ------------------------------------------ | -------------------------------------------------- | --------------------------------------------------------------------- |
| Tags                                       | up to 5,000                                        | up to 50,000                                                          |
| Acquisition                                | single acquisition host                            | up to 10 parallel acquisition instances                               |
| OPC UA server                              | 1 instance                                         | 1 instance, Basic256Sha256 and application instance certificate management |
| Lua scripting                              | yes                                                | yes                                                                   |
| Local history archiving                    | —                                                  | retention policies and backfill                                       |
| Windows service mode                       | —                                                  | yes                                                                   |
| Version updates                            | perpetual use of the licensed version              | 1 year of upgrades from the issue date                                |

The application runs without a license key in an unlicensed mode with reduced capacity;
the currently enforced limits are displayed on the **License** page. See
[LICENSE.txt](LICENSE.txt) for the authoritative terms.

## Features

### Data acquisition

- **Modbus TCP** and **Modbus RTU** (serial), plus **OPC DA** as a data source.
- Per-device polling interval, response timeout, retry count and retry delay.
- Per-tag configuration: function code (`1` Coil, `2` Discrete Input, `3` Holding
  Register, `4` Input Register), data type, byte order, scaling factor, unit and
  write-enable flag.
- Per-tag quality tracking. On a communication failure the affected tags switch to
  `BadCommFailure` while keeping their last-known value, so stale data is visible
  rather than silently refreshed.

### Serving

- **OPC DA server** (COM, implemented in the Rust `modlink_da.dll`), with configurable
  ProgID/CLSID, update rate, group limit, async mode and DCOM support.
- **OPC UA server** (default port `4840`). Tag quality is propagated to clients as
  OPC UA status codes with the source timestamp preserved from the last successful
  acquisition, so clients can distinguish a live value from a stale one.

### History

- Periodic sampling into SQLite with a configurable interval and retention period.
- History queries are also reachable from Lua through `modlink.query_history`.

### Scripting

- Sandboxed **Lua 5.4** runtime (lupa) with module allow-listing and execution-time
  and memory limits. See [`config/default.toml`](config/default.toml) `[lua]`.
- Optional serial and TCP sessions for custom protocols, **disabled by default** and
  guarded by port/target allow-lists and timeout caps (`[lua_comm]`).
- Optional MQTT uplink (`[mqtt]`).

### Operations

- Windows service mode and run-at-startup registration.
- Logging with a master on/off switch, a configurable set of captured levels,
  optional log file output, in-app log viewer and log export.

### User interface

Twelve pages: Dashboard, Devices, Tags, Monitor, History, Log, Lua, OPC DA, OPC UA,
Browse, License, Settings. The desktop UI is available in English and Chinese.

## Repository layout

```
modlink/                Python desktop application
  core/                 configuration, tag database, event bus, log buffer
  modbus/               Modbus TCP/RTU clients and the polling scheduler
  opcda/                OPC DA client and server browsing (via OPC.Automation.1)
  opcua/                OPC UA server
  lua/                  Lua engine, script API, serial/TCP session layer
  dll/                  ctypes wrappers for modlink_da.dll and its license API
  storage/              SQLite persistence
  service/              Windows service, DCOM setup, autostart
  ui/                   wxPython UI (pages/, widgets/, i18n)
rust/                   Rust components (see "Rust components" below)
  modlink-da/           OPC DA server DLL; license validation lives here. Depends on
                        crates from the surrounding Rust workspace
  modlink-keygen/       License issuing CLI (vendor-side tool; not for distribution)
website/                Product site: FastAPI backend + Vue 3 frontend
config/default.toml     Shipped configuration template
help/index.html         Offline Lua API manual
scripts/                Sample Lua scripts
tests/                  pytest suite
```

## Requirements

**Runtime**

- Windows. The x86 build is 32-bit and targets Windows 7 SP1 and later.
- For OPC DA functionality, the **OPC Core Components Redistributable (x86)** must be
  installed on the target machine. The OPC Core Components *SDK* is **not** sufficient —
  it does not contain `opcdaauto.dll`. See [INSTALL.md](INSTALL.md) (Chinese) for the
  full procedure and for the verification commands.

**Development and build**

- Python **3.8.x, 32-bit**. The 32-bit interpreter is required: it must match the
  32-bit OPC DA automation wrapper.
- Runtime dependencies are pinned in [requirements.txt](requirements.txt):
  wxPython, pymodbus, asyncua, lupa, pywin32, toml.
- Rust toolchain (stable) for the components under `rust/`. Rebuilding `modlink_da.dll`
  additionally requires the surrounding Rust workspace, for the reasons described under
  [Rust components](#rust-components).
- Nuitka and LLVM-MinGW (32-bit) for building the standalone executable.

## Quick start

```bat
py -3.8-32 -m pip install -r requirements.txt
python -m modlink
```

Command-line options:

```bat
python -m modlink                 :: GUI (default)
python -m modlink --service       :: run as a Windows service
python -m modlink --setup-dcom    :: configure DCOM permissions
```

## Configuration

[`config/default.toml`](config/default.toml) is the **shipped template** only. It
documents the available keys and their defaults:

| Section    | Purpose                                                                |
| ---------- | ---------------------------------------------------------------------- |
| `general`  | instance name, language, logging switch / levels / log file            |
| `opcda`    | OPC DA server identity, update rate, group limit, DCOM                 |
| `opcua`    | OPC UA server switch, endpoint URI/hostname, port, publish interval     |
| `lua`      | script directory, entry script, execution-time and memory limits, allowed modules |
| `lua_comm` | serial / TCP sessions for scripts: switches, session cap, timeouts, allow-lists |
| `mqtt`     | MQTT uplink defaults used by `modlink.mqtt.*`                          |
| `history`  | sampling enable, interval, retention                                   |

Devices and tags, and everything edited through the **Settings** page, are persisted in
the SQLite database rather than the TOML file, so replacing the template or recompiling
does not discard user configuration.

## Lua scripting

Scripts live in `scripts/` (configurable). `init.lua` is the entry point.
See [help/index.html](help/index.html) for the full API reference.

```lua
-- Read a tag by ID
local temp = modlink.read_tag("temperature")
modlink.log("Current temperature: " .. tostring(temp))

-- Run something every 5 seconds
modlink.timer.every(5000, function()
    local val = modlink.read_tag("temperature")
    if val > 80 then
        modlink.raise_alarm("temperature", "High",
            "Temperature exceeds 80: " .. tostring(val))
    end
end)

-- React to tag changes (high frequency: log at debug level)
modlink.event.on("TagChanged", function(data)
    modlink.log(string.format("%s: %s -> %s [%s]",
        tostring(data.tag_id), tostring(data.old_value),
        tostring(data.new_value), tostring(data.quality)), "debug")
end)
```

API groups: `modlink.log`, `read_tag` / `read_tags`, `write_tag` / `write_tags`,
`raise_alarm`, `query_history`, `get_tags` / `get_tag_info` / `get_device_status`,
`now_ms`, `timer.*`, `event.*` (`TagChanged`, …), `storage.get` / `storage.set`,
`mqtt.*`, `serial.open`, `net.tcp`, `comm.*`.

Additional sample scripts: [`scripts/mqtt_uplink.lua`](scripts/mqtt_uplink.lua) and
[`scripts/custom_protocol.lua`](scripts/custom_protocol.lua).

## Building a standalone executable

[`build.bat`](build.bat) produces a self-contained 32-bit build with Nuitka:

```
dist\__main__.dist\pmodlink.exe
```

The script pins the paths of the 32-bit Python interpreter and the LLVM-MinGW i686
compiler at the top of the file; adjust them to your machine before running. The build
excludes a number of unused transitive dependencies (numpy, pywin32 sample/test trees,
unused serial sub-packages, unused Lua runtimes) — see the comment block in `build.bat`
before removing any `--nofollow-import-to` flag.

The Rust components are built separately. Note that `modlink-da` is **not buildable
from this directory alone** (see below).

### Rust components

`rust/modlink-da` produces `modlink_da.dll`, the OPC DA server. It takes path
dependencies on `modlink-core` and `modlink-opcda` from the surrounding Rust workspace,
and the 32-bit linker plus the Windows 7 compatibility flags for the
`i686-pc-windows-gnu` target are configured in that workspace's `.cargo/config.toml`.
Build it from the workspace root:

```bat
cargo build --workspace --release --target i686-pc-windows-gnu
```

Output: `target\i686-pc-windows-gnu\release\modlink_da.dll`.

`rust/modlink-keygen`, the vendor-side license issuing CLI, is self-contained and can be
built on its own:

```bat
cd rust
cargo build --release -p modlink-keygen
```

## Testing

```bat
python -m pytest tests
```

Some tests need the 32-bit Python 3.8 environment that has the runtime dependencies
installed:

- `tests/test_ui_app.py` drives the real wxPython window and is run directly rather than
  through pytest:
  ```bat
  C:\Python38-32\python.exe tests\test_ui_app.py
  ```
- `tests/test_lua.py` requires `lupa`, and the integration tests that load
  `modlink_da.dll` require an interpreter whose architecture matches the DLL (32-bit).

## Documentation

All documents below are currently written in Chinese.

| Document                                                 | Contents                                      |
| -------------------------------------------------------- | --------------------------------------------- |
| [INSTALL.md](INSTALL.md)                                 | Installation, OPC Core Components, deployment |
| [help/index.html](help/index.html)                       | Lua API reference and examples                |
| [opcda-client-design.md](opcda-client-design.md)         | OPC DA client design notes                    |
| [wxwidgets-migration-plan.md](wxwidgets-migration-plan.md) | UI framework migration notes                |
| [pre-release-test-plan.md](pre-release-test-plan.md)     | Pre-release test plan and environment notes   |
| [website/README.md](website/README.md)                   | Product site (development)                    |
| [website/deploy/README.md](website/deploy/README.md)     | Product site deployment (nginx + systemd)     |

## License

**pmodlink is commercial, proprietary software. It is not open source.**

Use of the software is governed by the [ModLink Gateway Commercial License
Agreement](LICENSE.txt). In summary — and the agreement prevails — the license is
non-exclusive, non-transferable and non-sublicensable, it permits installation on the
number of computers stated in the license particulars, and each license key is bound to
one machine through its machine code. Reverse engineering, redistribution and use to
provide a hosted service to third parties are not permitted.

Activation is verified locally by the application; no connection to the licensor is
required, and no operational or personal data is transmitted as part of activation.

Licensing, sales and activation enquiries: **support@pmodlink.com**

Third-party open-source components (lupa, wxPython/wxWidgets, protocol and serial
libraries) are governed by their own licenses.

## Safety notice

This software is intended for industrial data acquisition, monitoring and integration.
It is **not** designed, certified or intended for use in the control of machinery where
a malfunction could cause injury or death, in safety-instrumented or emergency shutdown
systems, in medical devices, or in any application requiring functional-safety
certification.

Read and write paths, Lua scripts, alarms and interlocks must be designed, tested and
validated by the integrator, with independent protection measures kept in place outside
this software. Any reading, write-back or control action performed through pmodlink is a
convenience function, not a safety function.
