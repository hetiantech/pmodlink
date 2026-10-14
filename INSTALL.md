# pmodlink — Installation Guide

Version: pmodlink 1.4.1 (32-bit / Nuitka standalone build)
Systems: Windows 7 SP1 / 8.1 / 10 / 11 (a 32-bit application, runs on 64-bit Windows)

> **The short version**: the target machine must have the **OPC Core Components
> Redistributable** installed (administrator rights required). The SDK alone is **not
> enough** — it does not contain `opcdaauto.dll`, so OPC DA still will not work. Confirm
> with the two checks in §2.3.
>
> **Windows 7 SP1 target machines additionally require** **KB2999226** (Universal C
> Runtime) first, otherwise the application **exits immediately on start-up** (see §1).

---

## 1. System requirements

| Item | Requirement | Notes |
|---|---|---|
| Operating system | Windows 7 **SP1** / 8.1 / 10 / 11 | The application is **32-bit** and runs directly on 64-bit Windows. On Windows 7 it **must be SP1** (RTM is not supported) |
| **Windows 7 SP1 prerequisite** | **KB2999226** (Universal C Runtime) | **A required prerequisite, not an optional optimisation**: the build statically imports 15 `api-ms-win-crt-*` modules plus `ucrtbase.dll`. Without them the application **exits immediately on start-up** reporting a missing `api-ms-win-crt-*.dll` / `ucrtbase.dll`. Windows 8.1 and later include the runtime. Download the **x86** package: [Windows6.1-KB2999226-x86.msu](https://github.com/hetiantech/pmodlink/releases/download/KB2999226/Windows6.1-KB2999226-x86.msu) — or search `KB2999226` in Windows Update / the Microsoft Update Catalog. On **64-bit Windows 7** install the x64 package of the same update instead, then check that `%SystemRoot%\SysWOW64\ucrtbase.dll` exists: that is the 32-bit runtime this application loads |
| Runtimes | **No** separate VC++ Redistributable | `msvcp140.dll` / `vcruntime140.dll` / `mfc140u.dll` ship inside the application directory |
| Privileges | Unpacking and running needs **no** administrator rights; registering the OPC components **does** | see §2 |
| Disk | ≥ 300 MB (including history data) | |
| Other | No .NET Framework needed | The OPC DA client uses COM directly, not the .NET wrappers |

---

## 2. Required dependency: OPC Core Components Redistributable (★)

### 2.1 Why it is required

pmodlink's OPC DA client reaches servers through **`OPC.Automation.1`** (the OPC DA
Automation Wrapper), which is provided by **`opcdaauto.dll`** and depends on a set of
runtime components. Those exist **only in the Redistributable package**:

| Component | Purpose |
|---|---|
| `opcdaauto.dll` | The Automation Wrapper — `OPC.Automation.1` itself |
| `opcproxy.dll`, `opccomn_ps.dll`, `opc_aeps.dll`, `opchda_ps.dll`, `opcbc_ps.dll`, `opcSec_PS.dll` | COM proxies/stubs (required for out-of-process marshalling) |
| `OpcEnum.exe` | Enumerates registered OPC DA servers, local and remote |

**Why "I already installed OPC Core Components" may still not be enough**: the OPC
Foundation ships two different packages.

- **SDK** — aimed at developers; contains headers, IDL and the `.NET` wrappers
  (`OpcRcw.*.dll`) only. It does **not** contain `opcdaauto.dll` or the runtime
  components listed above.
  Observed on a machine with the SDK 2.20 installed:

  ```
  C:\Program Files (x86)\OPC Foundation\Core Components 2.00\
  └── Bin\  OpcRcw.Ae.dll  OpcRcw.Batch.dll  OpcRcw.Comn.dll
            OpcRcw.Da.dll  OpcRcw.Dx.dll     OpcRcw.Hda.dll  OpcRcw.Sec.dll

  Recursive search for opcdaauto.dll under the OPC Foundation folder → no results
  ```

  After installing the SDK, creating `OPC.Automation.1` reports
  **`CO_E_CLASSSTRING` / "Invalid class string"**.
- **Redistributable** (the runtime / redistributable package) — this is the one to install
  on client machines; it contains every component in the table above.

### 2.2 Installation steps (administrator rights required)

1. Make sure pmodlink is closed.
2. Install the **OPC Core Components Redistributable**:
   - the **x86** build is mandatory (our application is 32-bit; the 64-bit
     `opcdaauto.dll` is of no use to it);
   - on 64-bit Windows, installing both x86 and x64 is recommended if other 64-bit OPC
     applications are present.

   **Download**

   - pmodlink mirror:
     <https://pmodlink.com/download/opc-redistributables-3.00.108.zip>
   - OPC Foundation official page (a free Registered User account is required):
     <https://opcfoundation.org/developer-tools/samples-and-tools-classic/core-components>

   > The 3.00.108 build on the mirror contains the **x64** installer only, while pmodlink
   > needs the **x86** one. Use the official page above to obtain the x86 package; this
   > section will be updated once the mirror carries it.

3. If the package did not register the enumerator automatically, register it once:

   ```
   C:\Windows\SysWOW64\OpcEnum.exe /RegServer
   ```

4. If the package did not register the Wrapper (some trimmed-down builds do not), register
   the 32-bit DLL:

   ```
   C:\Windows\SysWOW64\regsvr32.exe /s opcdaauto.dll
   ```

   > Note: the **32-bit `regsvr32` under `SysWOW64` is mandatory**. Registering a 32-bit
   > DLL with the 64-bit one reports success, but 32-bit processes still cannot use it.

### 2.3 Verification (both checks must pass)

**Check 1: the file the registry points at really exists**

```
reg query "HKLM\SOFTWARE\Classes\CLSID\{28E68F9A-8D75-11D1-8DC3-3C302A000000}\InprocServer32" /ve /reg:32
```

Expected output (observed):

```
HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{28E68F9A-8D75-11D1-8DC3-3C302A000000}\InprocServer32
    (Default)    REG_SZ    C:\Windows\SysWow64\opcdaauto.dll
```

Then confirm that the file at that path really exists — a leftover registry entry with the
DLL deleted is a common state.

> **`/reg:32` is not optional**: on 64-bit Windows, a 64-bit PowerShell reading
> `HKLM:\SOFTWARE\Classes\CLSID\{…}` directly **cannot see** the 32-bit entries and will
> wrongly conclude "not installed". `/reg:32` reads the 32-bit registry view explicitly.

**Check 2: create an instance from 32-bit PowerShell and enumerate servers**

```
C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe -Command "(New-Object -ComObject 'OPC.Automation.1').GetOPCServers()"
```

Expected output (lists the registered OPC DA servers; an empty result also passes if no
OPC DA server is installed):

```
Matrikon.OPC.Simulation.1
```

> **32-bit PowerShell under `SysWOW64` is mandatory**: only a 32-bit process sees the
> 32-bit COM registrations, which is exactly the view pmodlink (32-bit) has at runtime.

Both checks passing means the machine can run OPC DA. Check 1 only means the registry
entry is stale and the DLL is gone; check 2 reporting `Invalid class string`
(`CO_E_CLASSSTRING`) means nothing is registered at all.

### 2.4 Common pitfalls

| Symptom | Cause |
|---|---|
| "OPC Core Components" installed, still `Invalid class string` | The **SDK** was installed, not the Redistributable |
| Registered with 64-bit `regsvr32`, still unusable | 32-bit processes only see 32-bit registrations; use `SysWOW64\regsvr32.exe` |
| Only the x64 components were installed | The **x86** build is mandatory |
| `opcdaauto.dll` was merely copied into a system directory | Unregistered is the same as absent — it must be registered (or registered in-process by the application, see §7) |
| "The registry has no entry" during verification | A 64-bit PowerShell was used. 32-bit registrations appear in the 32-bit view only: add `/reg:32` when querying, and verify with the 32-bit PowerShell under `SysWOW64` |

---

## 3. Installing pmodlink

1. Unpack the **entire release directory** to the target location (for example
   `D:\pmodlink`). **Do not copy `pmodlink.exe` on its own** — the application depends on
   the Python/wx runtime and the subdirectories next to it.
2. Directory contents:

   | Path | Purpose |
   |---|---|
   | `pmodlink.exe` | The application |
   | `modlink_da.dll` | OPC DA server module (Rust, loaded by the executable) |
   | `config\default.toml` | **Factory configuration template** (do not edit it here — settings changed in the UI are stored in the database and take precedence) |
   | `data\` | Runtime data. `data\modlink.db` is the **only file that needs backing up** (configuration + history + license); `data\pki\` holds the OPC UA certificates (`own\` server certificate **including its private key**, `trusted\certs\` trusted client certificates, `rejected\` rejected certificates) — back that up too and treat it as key material |
   | `scripts\` | Lua script directory, entry point `init.lua` by default |
   | `help\` | Built-in help pages (Lua API reference) |
   | `LICENSE.txt` | License agreement |
   | `wx\`, `lupa\` | Python extension modules (user interface / Lua engine) — do not delete |
   | `asyncua\`, `cryptography\`, `cffi\`, `pycparser\`, `sortedcontainers\`, `dateutil\`, `pytz\`, `aiofiles\`, `aiosqlite\` | OPC UA server and crypto backend. These are shipped as **source** (compiling them with 32-bit Nuitka runs out of memory during analysis) — delete them and the OPC UA page reports that the crypto backend / server is unavailable |
   | `python3.dll`, `python38.dll`, `msvcp140.dll`, `vcruntime140.dll`, `libcrypto-1_1.dll`, `libssl-1_1.dll`, … | Python / VC / OpenSSL runtimes. **`python3.dll` in particular must not be deleted**: the crypto extension modules import it by name, and without it you get `DLL load failed while importing _rust` |
3. Start `pmodlink.exe`. The first start creates `data\modlink.db` automatically.
4. If you tick Autostart under **Settings → General**, an entry is written to
   `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`.

---

## 4. First-time configuration

1. **License** — on the **License** page, copy the machine code and send it to your
   supplier to obtain a license key, then paste the key there.
2. **Devices** — **Devices** page → Add.
   - Modbus TCP: put `IP:port` in `Address` (port 502 by default);
   - Modbus RTU: put `COM3` or similar in `Serial Port` and set baud rate, data bits and
     parity;
   - **OPC DA**: set `Protocol` to `opcda` and use **Browse...** next to `Prog ID` to pick
     from the registered servers (or type it in). Remote servers also need `Remote Host`
     and a DCOM configuration (see §5).
3. **Tags** — **Tags** page → Add (manually) or **Browse OPC DA** (tick entries in the
   address space to generate them in bulk).
4. **Live check** — watch values and quality on the **Monitor** page; select a device on
   the **Devices** page and use **Register Test** / **OPC DA Probe** for single-point
   read/write tests.
5. **Scripts** — create and edit scripts on the **Lua** page and press **Run** to see the
   result; the script directory is `scripts\` next to the executable.
6. **Logs** — the **Log** page filters and exports by level; look here first when
   troubleshooting.
7. **OPC UA** — three endpoints are enabled by default (None + Sign + SignAndEncrypt). The
   server certificate is self-signed into `data\pki\own\` on first start; hand
   `modlink-server.der` to any client that must connect to an encrypted endpoint. To
   control who may write tags, add username/password accounts on the **OPC UA** page
   (`admin` may write, `user` is read-only).

---

## 5. Remote OPC DA (DCOM) and firewall

Both the **client** (this machine) and the **server** (the machine hosting the OPC server)
must be configured, and **the account and password must be identical on both**. Key points:

- allow DCOM on both machines: `135/TCP` plus the RPC dynamic port range;
- on the server: `dcomcnfg` → Component Services → Computers → My Computer → Properties →
  COM Security, and grant the account running the OPC server "Local/Remote Access and
  Launch" permissions;
- the target machine must be enumerable: remotely this requires `OpcEnum.exe` to be
  registered and reachable over DCOM;
- this application offers **Auto Configure DCOM** on the **OPC DA** page as a one-click
  setup (administrator rights required).

If enumeration comes back empty, remember "cannot enumerate ≠ cannot connect": just type
the Prog ID in by hand and try to connect.

---

## 6. Upgrading and backup

- **Upgrade**: replace the whole application directory; **do not delete `data\`**.
- **Backup**: `data\modlink.db` (configuration, history, license) plus `data\pki\` (OPC UA
  certificates and private key — essential if you use encryption; lose them and every
  client has to import the new certificate).
- **Rollback**: keep the previous application directory and start from there (the database
  is backward compatible).

---

## 7. Known limitations (this version)

| Item | Notes |
|---|---|
| **OPC UA serves plaintext and encrypted endpoints at the same time** | By default three endpoints are offered — `None` (plaintext), `Basic256Sha256_Sign` and `Basic256Sha256_SignAndEncrypt` — with anonymous, username/password and client-certificate identities. The server certificate is self-signed into `data\pki\own\` when an encrypted endpoint first starts (**it contains the private key — back it up and protect it**); a client needs `modlink-server.der` to connect to an encrypted endpoint, and encrypted endpoints additionally require the client's own certificate under `data\pki\trusted\certs\`. Rejected certificates are filed under `data\pki\rejected\`. **The plaintext endpoint stays available**: on a trusted network only, untick the `None` policy under **Settings → OPC UA** (keeping at least one encrypted endpoint is recommended — with the plaintext policy removed and no certificate on the machine, the server falls back to plaintext only). It listens on `0.0.0.0:4840` by default; a specific host name can be configured. Security policy and identity changes **require a restart**, and the trusted directory is scanned once at start-up |
| OPC UA users and roles | The **OPC UA** page adds and removes username/password accounts: `admin` may write tags, `user` is read-only, anonymous clients are read-only. Passwords are stored in the database as salted PBKDF2-SHA256; the UI only shows whether an account is enabled |
| OPC DA depends on external components | The Redistributable must be installed per §2; this application **does not ship** `opcdaauto.dll` (OPC Foundation components are subject to their own redistribution terms) |
| Windows 7 | Must be **SP1** with KB2999226 installed. In addition the shipped **`cryptography` is pinned to 42.0.8**: from 43.0.3 on, its native extension statically depends on `bcryptprimitives.dll!ProcessPrng`, an export that only Windows 8 introduced, which on Windows 7 shows up as "double-click does nothing / the window flashes and disappears". Do not upgrade that package (see the notes in `requirements.txt`) |
| Modbus RTU | Requires a working serial-port driver (for USB adapters install the vendor driver first and confirm the COM port is visible in Device Manager) |

---

## 8. Uninstalling

1. Untick Autostart under **Settings → General** (removes the start-up entry);
2. exit the application and delete the application directory;
3. to remove all data, delete the `data\` directory;
4. the OPC runtime components (`opcdaauto.dll`, `OpcEnum`) are normally shared by several
   applications — uninstalling or unregistering them is **not recommended**.

---

## 9. Troubleshooting

| Symptom | Most likely cause | Action |
|---|---|---|
| Exits on double-click / reports missing `api-ms-win-crt-*` | Windows 7 without the UCRT | install KB2999226 (§1) |
| `Invalid class string` / `CO_E_CLASSSTRING` | `opcdaauto.dll` not registered | §2.2 + §2.3 |
| OPC DA device will not connect, log reports class not registered | As above, or the SDK was installed | §2 |
| "Browse OPC DA" list is empty | `OpcEnum` not registered / DCOM permissions / firewall | try local enumeration first, then check the remote configuration (§5) |
| OPC UA client is refused on `4840` | The server is not running: not enabled in Settings, changed without a restart, or the port is taken | look at the `Running:` line on the **OPC UA** page — `Disabled` / `FAILED - ...` states the reason; setting changes require a restart |
| Cannot connect to an OPC UA encrypted endpoint (client reports an untrusted certificate / `BadSecurityChecksFailed`) | Trust is **mutual** — both directions are required | ① import the server's `data\pki\own\modlink-server.der` into the client; ② put the client's own certificate (DER) into the server's `data\pki\trusted\certs\`; ③ **restart pmodlink** (the trusted directory is scanned once at start-up). Rejected certificates stay in `data\pki\rejected\` — the file name tells you which one |
| OPC UA username/password is rejected | Account missing, disabled, or wrong password | on the **OPC UA** page confirm the account is listed and enabled; use Change Password to reset it |
| OPC UA connects but writing a value fails on permissions | Role restrictions | only `admin` may write tags; `user` and anonymous clients are read-only |
| Windows 7 SP1: double-click does nothing / window flashes and disappears | The shipped `cryptography` is too new: from 43 on, its native extension depends on `bcryptprimitives!ProcessPrng`, which only Windows 8 has | rebuild with the pinned version (`requirements.txt` pins `cryptography==42.0.8`); the current package is fixed, so make sure you are running the latest build |
| The OPC UA page reports "the crypto backend cannot be loaded on this machine" | Same as the previous line: the crypto backend failed to load. The application still starts, but certificates and encrypted endpoints are unavailable | same as the previous line |
| Remote server connection times out | DCOM permissions, mismatched accounts, firewall | §5 |
| Writing a tag reports `0xC0040006` | That tag is read-only (OPC_E_BADRIGHTS) | expected behaviour; use a writable tag or do not write |
| All Modbus values are Bad | Network/serial link down, wrong slave address or function code | isolate it with a single-point read using **Register Test** |
| UI opens but the OPC DA page's buttons are greyed out | pywin32 missing (unusual environment) | cannot happen with a normal release package — please report it |

---

## 10. Pre-delivery checklist

- [ ] Operating system supported (Windows 7 requires SP1 + KB2999226)
- [ ] **OPC Core Components Redistributable** (x86) installed
- [ ] §2.3 check 1 passes (the `opcdaauto.dll` the registry points at exists)
- [ ] §2.3 check 2 passes (32-bit PowerShell can create `OPC.Automation.1`)
- [ ] Application directory unpacked **completely** (not just the exe)
- [ ] First start succeeds and `data\modlink.db` is created
- [ ] License activated (the License page shows the licensed state)
- [ ] At least one device and one tag configured, live values on the Monitor page
- [ ] The **OPC UA** page shows `Running: yes` with 3 endpoints (None + Sign + SignAndEncrypt)
- [ ] (When encrypted clients are used) the client has imported `data\pki\own\modlink-server.der` and its own certificate is in `data\pki\trusted\certs\`
- [ ] (When username/password is used) accounts are created with the intended roles (writing needs `admin`)
- [ ] (Remote OPC DA) DCOM and firewall configured per §5
