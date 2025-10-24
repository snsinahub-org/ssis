# SSIS Build & Deploy Composite Action

Minimal README (no previous version found). This action builds SSIS solutions and deploys .ispac packages using either Microsoft CDN or a GitHub release asset for the SSIS DevOps Tools installer.

## Inputs (summary)
- project_path (required): Root path containing the .sln / .dtproj files.
- project_configuration (required): Build configuration (e.g. Development).
- output_path (required): Directory to copy build artifacts.
- file_extenstion (optional, default *.sln): Solution file filter.
- ssis_action (required): One of build, build-sa, ssis-deploy-sql, ssis-deploy-win, ssis-deploy-multiple-win.
- ssis_tool_source (optional, default microsoft): microsoft (download from Microsoft CDN) or release (download asset SSISDevOpsTools-1.0.0.0.exe from tag v1.0.0 in this repo).

## Installer Source
When ssis_tool_source=release the action fetches release tag v1.0.0 and downloads asset SSISDevOpsTools-1.0.0.0.exe using the GitHub API. For public access no extra token beyond GITHUB_TOKEN in this repo is needed.

## Basic Usage Examples

Build:
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: build
    project_path: .
    project_configuration: Development
    output_path: build_out
```

Standalone SSISBuild (build-sa):
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: build-sa
    project_path: .
    project_configuration: Development
    output_path: build_out
```

Deploy (Windows auth) using Microsoft CDN:
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-win
    ssis_source: path\to\package.ispac
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

Deploy (Windows auth) using release asset:
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-win
    ssis_tool_source: release
    ssis_source: path\to\package.ispac
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

Deploy (SQL auth):
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

Deploy multiple .ispac:
```yaml
- uses: snsinahub-org/ssis@main
  with:
    ssis_action: ssis-deploy-multiple-win
    ssis_source: path\to\ispac\directory
    ssis_destination: /SSISDB/Folder
    ssis_server: your-ssis-host.example.com
```

## Troubleshooting (brief)
- SSL/TLS error on download: Check outbound HTTPS to download.microsoft.com (for microsoft) or api.github.com (for release).
- Asset not found (release mode): Ensure tag v1.0.0 and asset name SSISDevOpsTools-1.0.0.0.exe exist.
- Missing SSISDeploy: Installer download or extraction failed; inspect logs before deployment step.

## License
(Define repository license—placeholder.)

Minimal README complete.
