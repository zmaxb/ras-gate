[English](README.md) | [Русский](README.ru.md)

# RasGate

RasGate puts a small HTTP API in front of the 1C:Enterprise `rac` utility. A
client sends the same argument list it would pass to `rac`; RasGate runs the
configured executable and returns its exit code, output, and execution time.

```text
HTTP client -> RasGate -> RAC -> RAS -> 1C cluster
```

The service is deliberately thin. It does not run a shell, parse RAC output, or
try to model clusters, infobases, and sessions as its own domain API.

There is no web interface at `/`. A browser request to the root path therefore
gets a normal JSON `404` response. Use `/rasgate/status` to check the service.

## Requirements

To run RasGate you need Windows or Linux, a compatible `rac` executable, and
network access to RAS. Release archives are self-contained and include the .NET
runtime.

Building from source requires .NET SDK 10. GNU Make and Bash are needed for the
Makefile and release scripts. Docker deployment requires Docker Compose.

## API key

`POST /rac/execute` is protected by an API key. RasGate refuses to start when
`RasGate:ApiKey` is missing or invalid. The key must be 32 to 512 characters
long, with no leading or trailing whitespace.

RasGate uses the standard .NET configuration system, so the key can be stored
in `appsettings.json` in both development and production. When running from
source, edit `src/RasGate.Web/appsettings.json`. For a published application,
edit the `appsettings.json` next to the RasGate executable. Add `ApiKey` to the
existing `RasGate` section:

```json
{
  "RasGate": {
    "InstanceName": "RasGate Application",
    "ApiKey": "<paste-a-random-key-of-at-least-32-characters-here>"
  }
}
```

Replace the value between angle brackets. The repository copy of this file is
tracked by Git and deliberately ships without a key. Never commit a real key.
If production keeps the key in this file, restrict access to the application
directory, deployment artifacts, and backups that contain it.

The safer option for local development is .NET User Secrets. It keeps the key
outside the working tree:

```bash
api_key="$(openssl rand -hex 32)"
dotnet user-secrets set \
  "RasGate:ApiKey" "$api_key" \
  --project src/RasGate.Web/RasGate.Web.csproj
```

PowerShell:

```powershell
$apiKey = [guid]::NewGuid().ToString("N") + [guid]::NewGuid().ToString("N")
dotnet user-secrets set `
  "RasGate:ApiKey" $apiKey `
  --project src/RasGate.Web/RasGate.Web.csproj
```

An environment variable is an alternative for any deployment and overrides
the value from `appsettings.json`. The double underscore is the .NET
replacement for `:` in a configuration key:

```bash
export RasGate__ApiKey="$(openssl rand -hex 32)"
./RasGate.Web
```

Docker Compose reads the key from `RASGATE_API_KEY` in your local `.env` file.
Whichever method you choose, clients must send that same value in the
`X-Api-Key` header.

## Run locally

Set the path to RAC in `src/RasGate.Web/appsettings.json` or through the
`Rac__ExecutablePath` environment variable:

```json
{
  "RasGate": {
    "InstanceName": "RasGate Application"
  },
  "Rac": {
    "ExecutablePath": "/opt/1cv8/x86_64/rac"
  }
}
```

On Windows, escape backslashes in JSON:

```json
"ExecutablePath": "C:\\Program Files\\1cv8\\8.3.27.2214\\bin\\rac.exe"
```

Start the project and check both status endpoints:

```bash
dotnet run --project src/RasGate.Web/RasGate.Web.csproj

curl http://127.0.0.1:5050/rasgate/status
curl http://127.0.0.1:5050/rac/status
```

Run a harmless command through the API:

```bash
api_key='<the same key configured in RasGate>'
curl \
  --request POST \
  --header 'Content-Type: application/json' \
  --header "X-Api-Key: ${api_key}" \
  --data '{"arguments":["--version"]}' \
  http://127.0.0.1:5050/rac/execute
```

The default bind address is localhost. If the service must be reachable over a
network, change `Urls`, restrict the port with a firewall, and terminate TLS in
RasGate or a trusted reverse proxy.

### Validate configuration

Check the configuration without opening an HTTP port or running RAC:

```bash
./RasGate.Web --validate-config
```

The command returns `0` when the configuration is valid and a nonzero exit code
otherwise. The API key is never printed.

## Run as a service

The Windows and Linux archives include service installation scripts. You can
still run the same executable from a console when needed.

### Windows service

1. Extract the Windows archive to its permanent directory, for example
   `C:\Program Files\RasGate`.
2. Configure `RasGate:ApiKey` and `Rac:ExecutablePath` in
   `appsettings.json`.
3. Open Windows PowerShell as Administrator in that directory.
4. Run `.\install-service.ps1`.

Check or restart the service:

```powershell
Get-Service -Name RasGate
Restart-Service -Name RasGate
Invoke-RestMethod http://127.0.0.1:5050/rasgate/status
```

Remove the service without deleting the configuration, logs, or application
files:

```powershell
.\uninstall-service.ps1
```

### systemd service

1. Extract the Linux archive and configure `appsettings.json`.
2. Run `sudo ./install-service.sh`.

The installer copies RasGate to `/opt/rasgate`, creates an unprivileged
`rasgate` user, installs `rasgate.service`, and starts it.

```bash
systemctl status rasgate.service
sudo systemctl restart rasgate.service
journalctl -u rasgate.service -f
curl http://127.0.0.1:5050/rasgate/status
```

Remove the service without deleting `/opt/rasgate`, its configuration, or logs:

```bash
sudo /opt/rasgate/uninstall-service.sh
```

The scripts do not install RAC or change firewall and TLS settings. RasGate
keeps the address configured in `Urls`.

## Configuration

| Setting | Default and allowed values |
|---|---|
| `Urls` | `http://127.0.0.1:5050` |
| `RasGate:InstanceName` | Name returned by `/rasgate/status` |
| `RasGate:ApiKey` | Required secret, 32-512 characters |
| `Rac:ExecutablePath` | `rac`, an absolute path, or a command available through `PATH` |
| `Rac:TimeoutSeconds` | `30`, range 1-3600 |
| `Rac:StatusCacheSeconds` | `30`, range 1-300 |
| `Rac:MaxConcurrentProcesses` | `4`, range 1-32 per RasGate instance |
| `Rac:MaxOutputBytes` | `4194304`, maximum `16777216` per output stream |
| `Rac:MaxArgumentCount` | `128`, range 1-128 |
| `Rac:MaxArgumentBytes` | `8192`, UTF-8 bytes per argument |
| `Rac:MaxTotalArgumentBytes` | `24576`, total UTF-8 bytes; not less than `MaxArgumentBytes` |

Any setting can be overridden with an environment variable by replacing `:`
with `__`, for example `Rac__TimeoutSeconds=60`. Configuration is read at
startup; restart RasGate after changing it.

## Docker Compose

Copy the example environment file and fill in the key and RAC directory:

```bash
cp .env.example .env
```

`RAC_HOST_PATH` must point to a directory containing the Linux `rac` binary and
the libraries it needs. The directory is mounted read-only at `/opt/1c/rac`.

```bash
docker compose up --build --detach
docker compose down
```

The container runs as a non-root user with a read-only root filesystem. Logs go
to the `logs` volume and `/tmp` is backed by tmpfs. By default, port 5050 is
published only on `127.0.0.1`.

## HTTP API

| Method | Path | Authentication | Purpose |
|---|---|---|---|
| `GET` | `/rasgate/status` | none | RasGate name and version |
| `GET` | `/rac/status` | none | cached RAC availability and version |
| `POST` | `/rac/execute` | `X-Api-Key` | run a RAC command |

`/rac/status` always returns HTTP `200`; check `data.available` to learn whether
RAC can be started. The probe has its own cache and does not consume a command
execution slot.

An execution request contains an array of arguments:

```json
{
  "arguments": ["cluster", "list", "localhost:1545"]
}
```

A completed call looks like this:

```json
{
  "success": true,
  "data": {
    "outcome": "succeeded",
    "exitCode": 0,
    "standardOutput": "...",
    "standardError": "",
    "durationMilliseconds": 42,
    "timedOut": false
  }
}
```

`success` describes the HTTP operation, not the result of RAC. Read `outcome`,
`exitCode`, and `timedOut` together:

- `succeeded`: RAC exited with code 0;
- `failed`: RAC returned a non-zero exit code;
- `unknown`: RasGate cannot prove the external result.

Do not automatically retry an `unknown` result, a disconnected request after
process start, an output-limit failure, or a cleanup failure. The command may
already have changed cluster state. RasGate does not retry commands and cannot
provide exactly-once execution for arbitrary RAC operations.

Typical API errors are `400 bad_request`, `401 unauthorized`,
`429 rac_capacity_exceeded`, `502 rac_output_limit_exceeded`,
`502 rac_execution_outcome_unknown`, and `503 rac_unavailable`.

Responses include `X-Trace-Id`, which can be used to find the matching server
log entry. OpenAPI JSON is available at `/openapi/v1.json` in `Development`.

## Related projects

RasGate is part of the [Ras Ecosystem](https://github.com/RasEcosystem):

- [RasHub](https://github.com/RasEcosystem/ras-hub) — the central
  management service that connects RasStudio to RasGate and provides a unified
  API;
- [RasStudio Mono](https://github.com/RasEcosystem/ras-studio-mono) — an
  experimental monolithic web client for administering 1C:Enterprise clusters
  through RasHub and RasGate.

## License

See [LICENSE](LICENSE).
