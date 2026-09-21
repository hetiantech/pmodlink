# pmodlink

**A Lua-scripted industrial IoT gateway.** Connects Modbus, OPC DA/UA, MQTT and HTTP —
custom protocols and custom logic at the edge.

pmodlink (ModLink Gateway) is a Windows application for connecting shop-floor equipment
to upper-level systems. It acquires data from Modbus TCP/RTU devices and OPC DA servers,
republishes it over OPC DA and OPC UA, forwards it upstream over MQTT, and adapts to
everything else through Lua scripts that run inside the gateway.

> **Status: pre-release.** Features and license terms may change before the first
> production release.

---

## Contents

- [Editions](#editions)
- [Capabilities](#capabilities)
- [Requirements](#requirements)
- [Installation](#installation)
- [Activation](#activation)
- [Documentation](#documentation)
- [Support](#support)
- [License](#license)
- [Safety notice](#safety-notice)

---

## Editions

The table below summarises the licensed scope of each edition. The
[commercial license agreement](LICENSE.txt) is authoritative.

|                        | Core                                  | Professional                                                                 |
| ---------------------- | ------------------------------------- | ---------------------------------------------------------------------------- |
| Tags                   | up to 5,000                           | up to 50,000                                                                 |
| Acquisition            | single acquisition host               | up to 10 parallel acquisition instances                                      |
| OPC UA server          | 1 instance                            | 1 instance, Basic256Sha256 and application instance certificate management    |
| Lua scripting          | yes                                   | yes                                                                          |
| History archiving      | —                                     | retention policies and backfill                                              |
| Windows service mode   | —                                     | yes                                                                          |
| Version updates        | perpetual use of the licensed version | 1 year of upgrades from the issue date                                       |

Without a license key the application still runs, in an unlicensed mode with reduced
capacity. The limits currently in force are shown on the **License** page.

## Capabilities

### Data acquisition

- **Modbus TCP** and **Modbus RTU** (serial), and **OPC DA** as a data source.
- Per-device polling interval, response timeout, retry count and retry delay.
- Per-tag configuration: register type (coil, discrete input, holding register, input
  register), data type, byte order, scaling factor, engineering unit, and whether the tag
  may be written to.
- Per-tag quality. When communication with a device fails, its tags are marked with a
  communication-failure quality and keep their last known value, so an operator sees
  stale data rather than a value that appears to be refreshing. Quality returns to good
  automatically once the device responds again.

### Serving and forwarding

- **OPC DA server** — publish collected tags to OPC DA clients, with configurable server
  identity, update rate, group limit, asynchronous mode and DCOM support.
- **OPC UA server** (default port 4840) — tag quality is reported to clients as OPC UA
  status codes, and the source timestamp is preserved from the moment of acquisition so
  that a client can distinguish a live value from a stale one. While a device is
  unavailable, the last usable value is served together with a
  no-communication/last-usable-value status.
- **MQTT uplink** — publish tag values to a broker, with automatic reconnect.

### History

- Periodic sampling of tag values into a local database, with a configurable sampling
  interval and retention period.
- Views for browsing and querying archived values.

### Scripting

pmodlink is extended with **Lua 5.4** scripts that run inside the gateway. Scripts can
read and write tags, react to tag-change events, run timers, raise alarms, query history,
keep state between runs, publish over MQTT, and open serial or TCP sessions to implement
custom protocols or arbitrary logic at the edge.

Scripting runs in a sandbox with configurable execution-time and memory limits and an
allow-list of modules. Script access to serial ports and network targets is disabled by
default and is enabled explicitly, with port and target allow-lists and timeout caps.

### Operations

- Runs as a desktop application or as a Windows service, with optional run at startup.
- Logging with a master on/off switch, selectable severity levels, optional log file
  output, an in-application log viewer and log export.

### User interface

Dashboard, Devices, Tags, Monitor, History, Log, Lua, OPC DA, OPC UA, Browse, License and
Settings. The interface is available in English and Chinese.

## Requirements

- Windows 7 SP1 or later. pmodlink is a 32-bit application.
- Administrator rights to install.
- For OPC DA connectivity, the **OPC Core Components Redistributable (x86)** from the
  OPC Foundation must be installed on the machine. The OPC Core Components *SDK* is
  **not** sufficient — it does not contain the Automation Wrapper. This third-party
  runtime is not included with pmodlink.
- A license key for licensed operation.

## Installation

1. If OPC DA connectivity is required, install the OPC Core Components Redistributable
   (x86) first.
2. Run the pmodlink installer. Administrator rights are required; the default
   installation directory is under `Program Files`.
3. Start pmodlink, configure your devices and tags, then activate it on the **License**
   page.

The installation and deployment guide ([INSTALL.md](INSTALL.md)) covers the OPC Core
Components procedure in detail, together with Windows service installation, DCOM setup
for remote OPC DA access, and troubleshooting.

## Activation

Activation is offline and bound to the machine. The **License** page displays the machine
code for the computer; supply that code when ordering to receive a license key, and enter
the key on the same page. No internet connection is required, and no operational or
personal data is transmitted as part of activation.

If a hardware change alters the machine code — a motherboard or network interface
replacement, for example — the application will ask to be re-activated. A replacement key
for the same licensed machine is issued at no charge.

## Documentation

- **[INSTALL.md](INSTALL.md)** — installation, OPC Core Components, Windows service,
  DCOM and deployment.
- **Lua scripting reference** — built into the application. Open it with the **Help**
  button on the Lua page; it is also shipped as an offline manual.

Both are currently written in Chinese.

## Support

- Licensing, sales and upgrades: **support@pmodlink.com**
- Product website: **https://www.pmodlink.com/**

## License

**pmodlink is commercial, proprietary software. It is not open source.**

Use of the software is governed by the [ModLink Gateway Commercial License
Agreement](LICENSE.txt). In summary — and the agreement prevails — the license is
non-exclusive, non-transferable and non-sublicensable, it permits installation on the
number of computers stated in the license particulars, and each license key is bound to
one machine through its machine code. Reverse engineering, redistribution, and use of the
software to provide a hosted or managed service to third parties are not permitted.

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
