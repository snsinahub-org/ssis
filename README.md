# SSIS Build & Deploy Composite GitHub Action

This action builds SQL Server Integration Services (SSIS) solutions and deploys `.ispac` packages. It can obtain the SSIS DevOps Tools installer from one of four sources:

- `microsoft`: Download from Microsoft CDN.
- `release`: Download a published release asset (`v1.0.0`, asset `SSISDevOpsTools-1.0.0.0.exe`) in this repository.
- `repo`: Use a bundled copy committed in this repository (default path `SSISDevOpsTools-1.0.0.0.exe` at repo root).
- `local`: Use a pre-existing installer already present on the self-hosted runner (no network required). Provide path via `ssis_local_installer_path`.

## Inputs

- `project_path` (required): Root path containing solutions/projects.
- `project_configuration` (required): Build configuration (e.g. Development).
- `output_path` (required): Destination for built artifacts.
- `file_extenstion` (optional, default `*.sln`): Solution file filter.
- `ssis_action` (required): One of `build`, `build-sa`, `ssis-deploy-sql`, `ssis-deploy-win`, `ssis-deploy-multiple-win`.
- `ssis_tool_source` (optional, default `microsoft`): `microsoft | release | repo | local`.
- `ssis_repo_installer_path` (optional, default `SSISDevOpsTools-1.0.0.0.exe`): Path inside repository used when `ssis_tool_source=repo`.
- `ssis_local_installer_path` (optional): Absolute/accessible path on runner when `ssis_tool_source=local`.
- `insecure_skip_cert_validation` (optional, default `false`): LAST RESORT—temporarily bypass certificate validation only for installer download (not recommended long-term).
- `ssis_source` / `ssis_destination` / `ssis_server` / `ssis_username` / `ssis_password`: Deployment-related values depending on deploy action.

## Source Modes

| Mode      | Use Case                                               | Network Need                               |
|-----------|--------------------------------------------------------|---------------------------------------------|
| microsoft | Standard scenario, public CDN available                | `download.microsoft.com`                    |
| release   | CDN blocked, GitHub allowed                            | `api.github.com`, `github-releases.githubusercontent.com` |
| repo      | Ship a known version inside the repo                   | Only repo clone                             |
| local     | Fully isolated runner or controlled internal copy      | None for installer (path must exist)        |

## Security / TLS

TLS 1.2 is added if missing. Setting `insecure_skip_cert_validation: true` disables certificate validation solely during installer acquisition and then restores it. Prefer fixing trust (install enterprise root CA) over using insecure bypass.

## Examples

Build:
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: build
    project_path: .
    project_configuration: Development
    output_path: build_out
```

Standalone SSISBuild:
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: build-sa
    project_path: .
    project_configuration: Development
    output_path: build_out
```

Deploy (Win, Microsoft source):
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-win
    ssis_tool_source: microsoft
    ssis_source: path\to\package.ispac
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

Deploy (Win, Release asset):
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-win
    ssis_tool_source: release
    ssis_source: path\to\package.ispac
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

Deploy (Win, Bundled repo copy):
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-win
    ssis_tool_source: repo
    ssis_source: path\to\package.ispac
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

Deploy (Win, Local installer on runner):
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-win
    ssis_tool_source: local
    ssis_local_installer_path: D:\tools\SSISDevOpsTools-1.0.0.0.exe
    ssis_source: path\to\package.ispac
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

Deploy (SQL auth):
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-sql
    ssis_tool_source: microsoft
    ssis_source: path\to\package.ispac
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
    ssis_username: sqluser
    ssis_password: ${{ secrets.SQL_PASSWORD }}
```

Deploy multiple packages (Win):
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-multiple-win
    ssis_tool_source: release
    ssis_source: path\to\ispac\directory
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

Temporary insecure download (NOT recommended):
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-win
    ssis_tool_source: microsoft
    insecure_skip_cert_validation: true
    ssis_source: path\to\package.ispac
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

## Local Mode Notes

- Ensure the path given in `ssis_local_installer_path` is readable by the runner user.
- Can reside on an attached volume, network share (with permissions), or pre-provisioned tools directory.
- Action copies the file locally as `./SSISDevOpsTools.exe` for consistent extraction.

## Troubleshooting

| Symptom | Cause | Action |
|---------|-------|--------|
| SSL/TLS error (microsoft/release) | Blocked egress / untrusted proxy cert | Verify network, install root CA, avoid insecure bypass. |
| Asset not found (release) | Wrong tag or missing asset | Confirm tag `v1.0.0` and asset name. |
| Installer not found (repo/local) | Path incorrect | Check `ssis_repo_installer_path` or `ssis_local_installer_path` value. |
| `SSISDeploy` not recognized | Installer failed or path not appended | Review install logs; ensure extraction succeeded. |
| `devenv` not found | VS Build Tools / SSIS extensions missing | Install Visual Studio components or adjust PATH. |

Quick diagnostics (PowerShell):
```powershell
Resolve-DnsName download.microsoft.com
Test-NetConnection api.github.com -Port 443
[Net.ServicePointManager]::SecurityProtocol
```

## Proxy

Allow endpoints depending on source:
- Microsoft: `download.microsoft.com`
- Release: `api.github.com`, `github-releases.githubusercontent.com`
- Repo/local: none (for installer), only general GitHub clone access if repo mode.

Set env vars if needed:
```powershell
$env:HTTPS_PROXY="http://proxy.example:8080"
$env:HTTP_PROXY="http://proxy.example:8080"
```

## Security

- Prefer verifying installer integrity (future: add checksum input).
- Avoid long-term `insecure_skip_cert_validation`.
- Local / repo modes shift trust to your internal distribution—ensure binary provenance.

## License

(Add license details here.)

Minimal README end.
