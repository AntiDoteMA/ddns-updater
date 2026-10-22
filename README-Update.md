# Changes to the upstream repository

This file documents every change made to [qdm12/ddns-updater](https://github.com/qdm12/ddns-updater)
in this fork, so it is clear what is local and what is upstream.

Everything is additive except for the two changed defaults listed below. The
upstream behaviour can be restored by setting the corresponding environment
variables.

---

## 1. Summary

| | Upstream | This fork |
| --- | --- | --- |
| Update period (default) | `5m` | **`1m`** |
| Web UI listening port (default) | `:8000` | **`:12345`** |
| Reload the records configuration | not possible (restart required) | **`GET /reload`** |
| Change the period at runtime | not possible (restart required) | **`GET /reload?period=30s`** |
| DNS resolution | system resolver only | **DNS over HTTPS** (`RESOLVER_DOH_URL`) |
| Web UI buttons | none (the `/update` endpoint existed but had no button) | **Update now**, **Reload config**, period input, **Live refresh toggle** |
| Windows installation | Docker image or manual | **Inno Setup installer** (`windows/`) |

### Why the web UI port changed to 12345

Port 8000 is one of the most commonly used ports on the internet: it is the
default for a large number of web frameworks, development servers and container
images (Jupyter, many Node and Python applications, several official Docker
images). Running ddns-updater next to any of them means the web UI fails to
bind with `address already in use`, and the user has to discover the
`LISTENING_ADDRESS` variable to move it out of the way.

`:12345` is a registered port that is not used by any well known service, and
it is easy to remember and to type. Using it as the default means the web UI
works out of the box in the vast majority of setups.

This only changes the default. The port can still be set to anything with
`LISTENING_ADDRESS` or the equivalent flag, and `8000` remains perfectly
valid. The rationale is also documented in the upstream `README.md`, in the
"Why is the web UI listening on port 12345?" section.

---

## 2. New features

### 2.1 Reload the configuration from the web UI

`GET /reload` re-reads the records configuration file, rebuilds the records with
their history restored from the persistent database, and replaces them in the
in-memory database. The web UI shows the new records immediately, and the
periodic updater starts using them on its next cycle.

- If the configuration file cannot be read or parsed, the endpoint returns
  `500` with the error message and **the previous records are kept**.
- Record history is keyed by domain, owner and IP version, so it survives a
  reload and survives entries being reordered in the configuration file.
- The web UI gained a **Reload config** button.

### 2.2 Change the period at runtime

Environment variables are fixed for the lifetime of a process, so a new
settings file was introduced: `data/settings.json` by default
(`SETTINGS_FILEPATH` to override).

```json
{
  "PERIOD": "1m",
  "UPDATE_COOLDOWN_PERIOD": "1m"
}
```

- The file is read at startup and re-read on every reload.
- `GET /reload?period=30s` sets the period, **persists it in the settings file**
  and applies it to the running updater. The new interval takes effect on the
  next tick.
- The web UI gained a period input and a **Set period** button.
- The settings file takes precedence over the `PERIOD` and
  `UPDATE_COOLDOWN_PERIOD` environment variables.

### 2.3 DNS over HTTPS

New setting `RESOLVER_DOH_URL`, for example
`https://cloudflare-dns.com/dns-query`. When set, it is used to resolve the
provider host names and the record host names, and it takes precedence over
`RESOLVER_ADDRESS` and the system resolver.

This was needed because some networks (routers and the ISP resolvers behind
them) filter dynamic DNS domains and return `NXDOMAIN` for them, while plain
DNS on port 53 is blocked on the LAN. DNS over HTTPS on port 443 is not.

Go's resolver on Windows only speaks plain DNS to the servers listed in the
network adapter configuration (`net/dnsconfig_windows.go` in the Go standard
library), and it has no DNS over HTTPS support, so this cannot be solved with
the operating system settings alone.

The HTTP client is also routed through the DNS over HTTPS resolver, since the
provider API host name needs to be resolved too.

### 2.4 Windows installation

`windows/ddns-updater.iss` is an [Inno Setup](https://jrsoftware.org/isinfo.php)
script producing a single setup program which installs the application to
`C:\Tools\ddns-updater`, registers a scheduled task and starts it.

The task runs as the **SYSTEM** account with an **at startup** trigger, so the
application runs at boot without needing anyone to log in. The web UI stays
bound to the loopback interface and remains reachable from the desktop session.

### 2.5 Live refresh in the web UI

The web UI gained a **Live: on / Live: off** toggle button. When enabled, the
records table refreshes every 15 seconds by fetching the page and replacing
only the table body, so the toolbar and the period input keep their state and
the page does not flicker. The table is only replaced when its content
actually changed.

The choice is remembered in the browser's local storage, so it survives a page
reload. It is **off by default**.

The **Update now** button also refreshes the table once it completes, so the
new status is visible without waiting for the next tick.

---

## 3. New environment variables

| Variable | Default | Description |
| --- | --- | --- |
| `RESOLVER_DOH_URL` | *(empty)* | DNS over HTTPS server URL, for example `https://cloudflare-dns.com/dns-query` |
| `SETTINGS_FILEPATH` | `<datadir>/settings.json` | JSON settings file, re-read at runtime |

---

## 4. New HTTP endpoints

| Endpoint | Description |
| --- | --- |
| `GET /reload` | Re-reads the records configuration and the settings file |
| `GET /reload?period=30s` | Sets the period, persists it and reloads |

Both are unauthenticated, like the rest of the web UI. Keep the listening
address on the loopback interface unless the network is trusted.

---

## 5. New files

| File | Lines | Description |
| --- | --- | --- |
| `internal/doh/doh.go` | 233 | DNS over HTTPS client (RFC 8484), built on `github.com/miekg/dns` |
| `internal/doh/doh_test.go` | 110 | Offline tests against a fake DNS over HTTPS server |
| `internal/settings/settings.go` | 121 | JSON settings file, read and written at runtime |
| `internal/settings/settings_test.go` | 153 | Tests for the settings file |
| `internal/reload/service.go` | 187 | Reload service: rebuilds the records and applies the settings |
| `internal/server/reload.go` | 51 | `/reload` handler |
| `internal/server/reload_test.go` | 102 | Tests for the handler |
| `internal/update/service_test.go` | 49 | Tests for `SetPeriod` and `SetCooldown` |
| `windows/ddns-updater.iss` | | Inno Setup script |
| `windows/install-scheduled-task.ps1` | | Registers and starts the scheduled task |
| `windows/uninstall-scheduled-task.ps1` | | Stops and removes the scheduled task |
| `windows/run-ddns-updater.cmd` | | Launcher holding the environment variables |
| `windows/run-ddns-updater.vbs` | | Hidden launcher used by the scheduled task |

---

## 6. Modified files

| File | Change |
| --- | --- |
| `cmd/ddns-updater/main.go` | Wiring: the HTTP client resolves through DNS over HTTPS, the reload service is created and passed to the server, the settings file is applied at startup, `readRecords` moved to `internal/reload` |
| `internal/config/update.go` | Default period `5m` → `1m` |
| `internal/config/server.go` | Default listening address `:8000` → `:12345` |
| `internal/config/resolver.go` | New `RESOLVER_DOH_URL` setting |
| `internal/config/paths.go` | New `SettingsFile` path setting |
| `internal/config/settings_test.go` | Golden settings summary updated |
| `internal/data/memory.go` | New `SetAll()` to replace the records atomically |
| `internal/resolver/resolver.go` | Returns a `LookupIPer` interface, uses DNS over HTTPS when configured, adds an `IPResolver` adapter for the health package |
| `internal/resolver/settings.go` | New `DOHURL` setting |
| `internal/server/handler.go` | `reloader` field and `/reload` route |
| `internal/server/interfaces.go` | New `ConfigReloader` interface |
| `internal/server/server.go` | `New()` takes the reloader |
| `internal/server/ui/index.html` | Toolbar with the new buttons and the period input |
| `internal/server/ui/static/styles.css` | Styling for the toolbar and the status message |
| `internal/update/service.go` | `SetPeriod` and `SetCooldown`, ticker reset, concurrent-safe access |
| `internal/backup/service.go` | The backup now archives the **configured** configuration file instead of hardcoding `<datadir>/config.json` |
| `Dockerfile`, `docker-compose.yml`, `k8s/base/*` | Port updated to 12345 |
| `README.md` | Documented `RESOLVER_DOH_URL` |
| `.gitignore` | Excludes the Windows build artefacts and the local configuration |

---

## 7. Behaviour notes and gotchas

- **The configuration file must not have a UTF-8 BOM.** PowerShell's
  `Set-Content -Encoding UTF8` adds one and the application then fails with
  `cannot unmarshal common settings: invalid character 'ï»¿'`. Write it with:

  ```powershell
  [System.IO.File]::WriteAllText($path, $json, (New-Object System.Text.UTF8Encoding($false)))
  ```

- **Relative paths resolve against the working directory**, not against the
  executable location. The launcher sets the working directory explicitly.
- **The web UI has no authentication.** Keep it on `127.0.0.1`.
- A name which has no address of the requested type returns the error
  `no such host`, matching the Go resolver, so that the updater treats a
  missing AAAA record as "no IPv6 address" rather than retrying.

### File layout

The records configuration lives in the data directory, as it does upstream:

```text
C:\Tools\ddns-updater\
├── ddns-updater.exe
└── data\
    ├── config.json         the records to watch and the credentials
    ├── settings.json       the runtime settings, re-read at runtime
    ├── updates.json        the record history
    └── ddns-updater.log    the application log
```

`CONFIG_FILEPATH` and `SETTINGS_FILEPATH` point at those files explicitly, so
the layout does not depend on the working directory.

Upstream's backup service archived `<datadir>/config.json` unconditionally,
which ignores the `CONFIG_FILEPATH` setting. It now archives the configured
file instead, which also fixes it for anyone setting `CONFIG_FILEPATH` to
another location.

### The `windows/` directory mirrors the installed layout

`windows/` holds everything needed to run the application on Windows, and
mirrors the installed layout exactly:

```text
windows\
├── ddns-updater.exe        the built executable
├── data\
│   └── config.json         the records to watch and the credentials
├── run-ddns-updater.cmd    the launcher
├── run-ddns-updater.vbs    the hidden launcher
├── install-scheduled-task.ps1
├── uninstall-scheduled-task.ps1
├── ddns-updater.iss        the Inno Setup script
```

Copying the whole directory to another location works as is: the scripts find
their own path (`%~dp0`, `$PSScriptRoot`), the launcher creates the `data`
directory if it is missing, and it falls back to `config.json` in the root of
the installation directory if there is none in `data\`. Only
`install-scheduled-task.ps1` has to be re-run afterwards, since the scheduled
task stores the absolute path of the launcher at registration time.

---

## 8. Testing

```powershell
go build ./...
go test ./internal/doh/... ./internal/settings/... ./internal/server/... ./internal/update/... ./internal/config/...
```

> `go test ./...` also runs the provider tests, which call the real provider
> APIs and can hang without network access to them.

---

## 9. Building and installing on Windows

`windows/output/ddns-updater-setup.exe` is the **Inno Setup repackage
of `windows/ddns-updater.iss`**. It is a self-contained setup program: anybody
can run it to install the application, and anybody can regenerate it themselves
from the `.iss` script, which is plain text and needs no secrets.

To use the ready-made installer:

```powershell
# Install (elevated, it registers a scheduled task)
windows\output\ddns-updater-setup.exe
```

To repackage it from source:

```powershell
# Build the executable
$env:CGO_ENABLED = "0"
go build -trimpath -ldflags "-s -w -X 'main.version=local'" `
    -o windows\ddns-updater.exe ./cmd/ddns-updater

# Repackage it with Inno Setup
"C:\Program Files (x86)\Inno Setup 6\ISCC.exe" windows\ddns-updater.iss
# -> windows\output\ddns-updater-setup.exe
```

The `.iss` script only needs [Inno Setup 6](https://jrsoftware.org/isinfo.php)
and the Go toolchain. It ships the executable, the launcher scripts and an
**example configuration**, then registers and starts the scheduled task.

### The shipped example configuration

The package ships `windows/data/config.json` with a single placeholder entry:

```json
{
  "_comment": "Replace the values below with your own provider credentials, then press the 'Reload config' button in the web UI.",
  "settings": [
    {
      "provider": "duckdns",
      "domain": "example.duckdns.org",
      "token": "00000000-0000-0000-0000-000000000000",
      "ip_version": "ipv4",
      "ipv6_suffix": ""
    }
  ]
}
```

Each user replaces the values with their own credentials and presses
**Reload config** in the web UI, so no restart is needed.

> The upstream example configuration is **not** usable as is: its godaddy
> placeholder secret does not match the provider's validation regex, and the
> application exits at startup on invalid settings. The example shipped here
> passes validation, so the application starts and simply reports
> `bad authentication` until the token is replaced.

The file is installed with the `onlyifdoesntexist` flag, so **reinstalling
never overwrites an existing configuration**.

The installer stops the running application before replacing the executable,
registers the scheduled task and starts it. Uninstalling removes the task and
the application but **keeps `data\config.json`**, so the credentials survive a
reinstall. The rest of the `data` directory is removed on uninstall.

> The `windows/output/` directory is git-ignored, so the setup program is not
> part of the repository. Everyone builds their own from the `.iss` script.

---

## 10. Compatibility with upstream

- No upstream behaviour is removed. The two changed defaults are restored by
  setting `PERIOD=5m` and `LISTENING_ADDRESS=:8000`.
- The new environment variables are optional and default to a no-op.
- The new HTTP endpoints are additive.
- The provider implementations are untouched.
