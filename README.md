# SSIS Build & Deploy Composite GitHub Action

This action builds SQL Server Integration Services (SSIS) solutions and deploys `.ispac` packages to an SSIS Catalog using either:
- The Microsoft CDN installer for SSIS DevOps Tools.
- A release asset in this same repository (tag `v1.0.0`, asset `SSISDevOpsTools-1.0.0.0.exe`).

It supports building solution (`*.sln`) + project (`*.dtproj`) files, standalone SSISBuild packaging, single / multiple package deployment with either Windows or SQL authentication.

---

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `project_path` | yes (for build actions) | — | Root path containing solutions / projects. |
| `project_configuration` | yes (for build actions) | — | Build configuration (e.g. `Development`, `Release`). |
| `output_path` | yes (for build actions) | — | Directory to copy built artifacts / `.ispac` files. |
| `file_extenstion` | no | `*.sln` | Solution file glob filter. |
| `ssis_action` | yes | — | One of: `build`, `build-sa`, `ssis-deploy-sql`, `ssis-deploy-win`, `ssis-deploy-multiple-win`. |
| `ssis_tool_source` | no | `microsoft` | `microsoft` (download from Microsoft CDN) or `release` (download from repo release asset). |
| `insecure_skip_cert_validation` | no | `false` | LAST RESORT: `true` bypasses TLS certificate validation only for installer download (unsafe; use only temporarily). |

(Additional deployment-related inputs such as `ssis_source`, `ssis_destination`, `ssis_server`, `ssis_username`, `ssis_password` are implied by deploy actions; keep them consistent with your workflow—if not yet defined in `action.yml`, add them before use.)

---

## Actions / Modes

| Action | Purpose |
|--------|---------|
| `build` | Use Visual Studio (devenv) to build solutions and copy compiled SSIS artifacts. |
| `build-sa` | Use SSISBuild CLI (from DevOps Tools) to produce `.ispac` output directly. |
| `ssis-deploy-win` | Deploy a single `.ispac` using Windows authentication. |
| `ssis-deploy-multiple-win` | Deploy all `.ispac` files found under a directory (Windows auth). |
| `ssis-deploy-sql` | Deploy a single `.ispac` using SQL authentication (`-u` / `-p`). |

---

## Installer Source Selection

Set `ssis_tool_source`:

- `microsoft`: Downloads from  
  `https://download.microsoft.com/download/5/1/4/5144b772-d3b0-4e1c-a05b-5376f2ea0fc1/SSISDevOpsTools-1.0.0.0.exe`

- `release`: Queries GitHub release tag `v1.0.0` in this repository and downloads asset `SSISDevOpsTools-1.0.0.0.exe` (via `browser_download_url`).

If corporate firewall blocks the Microsoft CDN, use `release`.

---

## Certificate / TLS Handling

The action:
1. Ensures TLS 1.2 is enabled (adds it if missing).
2. Optionally (if `insecure_skip_cert_validation=true`) overrides certificate validation ONLY during installer download, then restores the original callback. This is dangerous—prefer installing proper root/intermediate certificates and keeping `false`.

Do NOT leave the insecure bypass enabled permanently.

---

## Usage Examples

### 1. Build Solutions (devenv)
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: build
    project_path: .
    project_configuration: Development
    output_path: build_out
```

### 2. Standalone SSISBuild Packaging
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: build-sa
    project_path: .
    project_configuration: Development
    output_path: build_out
```

### 3. Deploy (Windows Auth) via Microsoft CDN
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-win
    ssis_tool_source: microsoft
    ssis_source: path\to\package.ispac
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

### 4. Deploy (Windows Auth) via Release Asset
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-win
    ssis_tool_source: release
    ssis_source: path\to\package.ispac
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

### 5. Deploy (SQL Auth)
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-sql
    ssis_source: path\to\package.ispac
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
    ssis_username: sqluser
    ssis_password: ${{ secrets.SQL_PASSWORD }}
```

### 6. Deploy Multiple Packages (Windows Auth)
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-multiple-win
    ssis_source: path\to\ispac\directory
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

### 7. Temporary Insecure Certificate Bypass (NOT Recommended)
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
Use only if you are diagnosing certificate chain issues; remove afterward.

---

## Proxy / Firewall Guidance

Ensure outbound HTTPS access to (depending on source):
- Microsoft CDN: `download.microsoft.com`
- GitHub APIs: `api.github.com`
- GitHub release hosts: `github-releases.githubusercontent.com`, `objects.githubusercontent.com`

If using a corporate proxy:
- Configure system or environment (`HTTP_PROXY`, `HTTPS_PROXY`).
- Install proxy root CA into Trusted Root store (if SSL inspection enabled).

---

## Troubleshooting

| Symptom | Possible Cause | Recommended Action |
|---------|----------------|-------------------|
| `Could not create SSL/TLS secure channel` | Blocked egress / SSL interception without trust | Verify proxy rules, install root CA, check `Test-NetConnection download.microsoft.com -Port 443`. |
| Asset not found in release mode | Wrong tag or asset missing | Confirm release tag `v1.0.0` and asset name `SSISDevOpsTools-1.0.0.0.exe`. |
| `SSISDeploy` not recognized | Installer download/extraction failed | Inspect earlier logs; add diagnostics (DNS, port tests). |
| Build fails finding `devenv` | Visual Studio missing or vswhere path not detected | Install VS Build Tools or full VS with SSIS extensions; verify vswhere output. |
| `.ispac` not deployed | Incorrect `ssis_source` or permissions | Confirm path exists; verify package format; ensure runner has network connectivity to SSIS server. |

Diagnostic one-liners you can temporarily add:
```powershell
Resolve-DnsName download.microsoft.com
Test-NetConnection api.github.com -Port 443
[Net.ServicePointManager]::SecurityProtocol
```

---

## Security Considerations

- Avoid permanent use of `insecure_skip_cert_validation`.
- Consider adding future checksum verification for the installer (SHA256).
- If repository is ever made private, cross-repo use of release assets may fail; plan a public distribution approach if needed.

---

## Roadmap (Optional Enhancements)

- Add checksum input for installer integrity.
- Caching extracted tools between runs.
- Central diagnostic flag for network/TLS tests.
- Skip-download input for pre-provisioned runners.
- Support browser download fallback for Microsoft if CDN blocked.

---

## License

(Provide license text or reference here—currently placeholder.)

---

## Disclaimer

This action performs network downloads of tooling. Always verify compliance with organizational policies before enabling certificate validation bypass or using external endpoints.

---
