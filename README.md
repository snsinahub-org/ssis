# SSIS Build & Deploy Composite GitHub Action

This action builds SQL Server Integration Services (SSIS) solutions and deploys `.ispac` packages. It can obtain the SSIS DevOps Tools installer from:
- Microsoft CDN (`ssis_tool_source: microsoft`)
- This repository’s release tag `v1.0.0` (`ssis_tool_source: release`)
- A bundled copy committed at the repository root (`ssis_tool_source: repo`, default path `SSISDevOpsTools-1.0.0.0.exe`)

## Inputs

- project_path (required): Root path containing solutions/projects.
- project_configuration (required): Build configuration (e.g. Development).
- output_path (required): Destination for built artifacts.
- file_extenstion (optional, default *.sln): Solution filter.
- ssis_action (required): build | build-sa | ssis-deploy-sql | ssis-deploy-win | ssis-deploy-multiple-win.
- ssis_tool_source (optional, default microsoft): microsoft | release | repo.
- ssis_repo_installer_path (optional, default SSISDevOpsTools-1.0.0.0.exe): Relative path to bundled installer when using repo source.
- insecure_skip_cert_validation (optional, default false): LAST RESORT—if true, temporarily bypasses certificate validation only for the installer download.
- ssis_source / ssis_destination / ssis_server / ssis_username / ssis_password: Used by deploy actions (single or multiple, Windows or SQL auth).

## Source Modes

| Mode | Use Case | Network Need |
|------|----------|--------------|
| microsoft | Standard; fetch from Microsoft CDN | Access to download.microsoft.com |
| release | Firewall blocks CDN; prefer GitHub release asset | Access to api.github.com & github-releases.githubusercontent.com |
| repo | Fully offline or tightly controlled; artifact pre-committed | No external network for installer (only repo clone) |

## Security / TLS

TLS 1.2 is added if missing. Setting `insecure_skip_cert_validation: true` disables certificate validation only during download and then restores it—avoid permanent use (risk of MITM).

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

Deploy multiple packages:
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-multiple-win
    ssis_tool_source: release
    ssis_source: path\to\ispac\directory
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

Temporary (NOT recommended) insecure download:
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

## Troubleshooting

- SSL/TLS error: Verify egress (download.microsoft.com or api.github.com). Prefer fixing certificate trust over using insecure bypass.
- Asset not found (release): Confirm tag v1.0.0 and asset name SSISDevOpsTools-1.0.0.0.exe.
- Missing SSISDeploy: Installer download/extraction failed—check earlier logs.
- devenv not found: Ensure Visual Studio Build Tools / SSIS components installed.

## Proxy

Configure `HTTP_PROXY` / `HTTPS_PROXY` or system proxy; allow:
- download.microsoft.com (microsoft mode)
- api.github.com + github-releases.githubusercontent.com (release mode)

Repo mode minimizes external calls for installer.

## License

(Add license details here.)

Minimal README end.
