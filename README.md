# SSIS Build & Deploy Composite Action

A comprehensive GitHub composite action for building and deploying SQL Server Integration Services (SSIS) projects. This action supports both Visual Studio-based builds and standalone builds using Microsoft's SSIS DevOps Tools, with flexible deployment options for SQL Server using Windows or SQL authentication.

The action provides two methods for downloading SSIS DevOps Tools:
- **Microsoft CDN** (default): Downloads from Microsoft's official download servers
- **Release Asset**: Downloads from this repository's release v1.0.0 (asset: `SSISDevOpsTools-1.0.0.0.exe`), useful when corporate firewalls block Microsoft CDN access

The download method is controlled via the `ssis_tool_source` input parameter.

## Features

- **Build SSIS Solutions**: Build .dtproj projects using Visual Studio 2017
- **Standalone Build (SSISBuild)**: Build SSIS projects without Visual Studio installation using SSIS DevOps Tools
- **Deploy with Windows Authentication**: Deploy .ispac packages to SQL Server using integrated Windows authentication
- **Deploy with SQL Authentication**: Deploy .ispac packages using SQL Server authentication credentials
- **Multi-File Deployment**: Deploy multiple .ispac files from a directory in a single action
- **Flexible Tool Source**: Download SSIS DevOps Tools from Microsoft CDN or from this repository's release assets
- **Automatic Tool Installation**: Automatically installs and configures SSIS DevOps Tools when needed

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `project_path` | Yes | - | Path to the SSIS project directory containing .sln or .dtproj files |
| `project_configuration` | Yes | - | Build configuration (e.g., "Development", "Production") |
| `output_path` | Yes | - | Output directory path for built .ispac files |
| `file_extenstion` | No | `*.sln` | File extension filter for project files |
| `ssis_action` | Yes | - | Action to perform: `build`, `build-sa`, `ssis-deploy-win`, `ssis-deploy-sql`, or `ssis-deploy-multiple-win` |
| `ssis_tool_source` | No | `microsoft` | Source for SSIS DevOps Tools download: `microsoft` (Microsoft CDN) or `release` (repository release asset) |
| `ssis_source` | No* | - | Source .ispac file or directory for deployment (*required for deploy actions) |
| `ssis_destination` | No* | - | Destination path in SSISDB (e.g., `/SSISDB/MyFolder`) (*required for deploy actions) |
| `ssis_server` | No* | - | SQL Server instance name (*required for deploy actions) |
| `ssis_username` | No* | - | SQL Server username (*required for `ssis-deploy-sql` action) |
| `ssis_password` | No* | - | SQL Server password (*required for `ssis-deploy-sql` action) |

## ssis_tool_source Explained

The `ssis_tool_source` input determines where SSIS DevOps Tools are downloaded from:

### `microsoft` (Default)
Downloads from Microsoft's official CDN:
- URL: `https://download.microsoft.com/download/5/1/4/5144b772-d3b0-4e1c-a05b-5376f2ea0fc1/SSISDevOpsTools-1.0.0.0.exe`
- No authentication required
- Requires access to `download.microsoft.com`

### `release`
Downloads from this repository's GitHub release:
- Release tag: `v1.0.0`
- Asset name: `SSISDevOpsTools-1.0.0.0.exe`
- Uses GitHub API (`api.github.com`) and GitHub releases CDN (`github-releases.githubusercontent.com`)
- For public repositories, no Personal Access Token (PAT) is required
- Uses the `GITHUB_TOKEN` environment variable automatically provided by GitHub Actions
- Downloads via GitHub API's asset endpoint with `Accept: application/octet-stream` header

**When to use `release`:**
- Corporate firewall blocks `download.microsoft.com`
- Restricted network environments with GitHub access only
- Consistent tooling version control via repository releases
- Air-gapped or offline deployment scenarios (with release artifact cached)

**Note:** Future versions may adopt `browser_download_url` for simpler direct download from GitHub's CDN.

## Usage Examples

### Build with Visual Studio

Build SSIS projects using Visual Studio 2017 installed on the runner:

```yaml
- name: Build SSIS with Visual Studio
  uses: snsinahub-org/ssis@main
  with:
    ssis_action: 'build'
    project_path: 'C:\Projects\MySSISProject'
    output_path: 'C:\Output'
    project_configuration: 'Development'
```

### Standalone Build (build-sa)

Build SSIS projects without Visual Studio using SSIS DevOps Tools:

```yaml
- name: Standalone SSIS Build
  uses: snsinahub-org/ssis@main
  with:
    ssis_action: 'build-sa'
    project_path: 'C:\Projects\MySSISProject'
    output_path: 'C:\Output'
    project_configuration: 'Development'
    ssis_tool_source: 'microsoft'  # Optional, this is the default
```

### Deploy with Windows Authentication (Microsoft CDN)

Deploy a single .ispac file using Windows authentication:

```yaml
- name: Deploy SSIS Package (Windows Auth)
  uses: snsinahub-org/ssis@main
  with:
    ssis_action: 'ssis-deploy-win'
    ssis_source: 'C:\Output\MyProject.ispac'
    ssis_destination: '/SSISDB/Production/MyProject'
    ssis_server: 'SQL-SERVER-01'
    ssis_tool_source: 'microsoft'
```

### Deploy with Windows Authentication (Release Asset)

Deploy using the release asset source (useful when Microsoft CDN is blocked):

```yaml
- name: Deploy SSIS Package (Release Source)
  uses: snsinahub-org/ssis@main
  with:
    ssis_action: 'ssis-deploy-win'
    ssis_source: 'C:\Output\MyProject.ispac'
    ssis_destination: '/SSISDB/Production/MyProject'
    ssis_server: 'SQL-SERVER-01'
    ssis_tool_source: 'release'
```

### Deploy with SQL Authentication

Deploy using SQL Server authentication:

```yaml
- name: Deploy SSIS Package (SQL Auth)
  uses: snsinahub-org/ssis@main
  with:
    ssis_action: 'ssis-deploy-sql'
    ssis_source: 'C:\Output\MyProject.ispac'
    ssis_destination: '/SSISDB/Production/MyProject'
    ssis_server: 'SQL-SERVER-01'
    ssis_username: ${{ secrets.SQL_USERNAME }}
    ssis_password: ${{ secrets.SQL_PASSWORD }}
    ssis_tool_source: 'microsoft'
```

### Deploy Multiple .ispac Files

Deploy all .ispac files from a directory:

```yaml
- name: Deploy Multiple SSIS Packages
  uses: snsinahub-org/ssis@main
  with:
    ssis_action: 'ssis-deploy-multiple-win'
    ssis_source: 'C:\Output'
    ssis_destination: '/SSISDB/Production'
    ssis_server: 'SQL-SERVER-01'
    ssis_tool_source: 'microsoft'
```

## Proxy / Firewall Guidance

This action requires network access to download SSIS DevOps Tools and communicate with SQL Server. Ensure the following endpoints are accessible from your GitHub Actions runner:

### Microsoft CDN Source (`ssis_tool_source: 'microsoft'`)
- **Required endpoints:**
  - `download.microsoft.com` (HTTPS/443) - for downloading SSISDevOpsTools.exe

### Release Asset Source (`ssis_tool_source: 'release'`)
- **Required endpoints:**
  - `api.github.com` (HTTPS/443) - for GitHub API calls to retrieve release metadata
  - `github-releases.githubusercontent.com` (HTTPS/443) - for downloading release assets

### Corporate Firewall Scenarios

**Scenario 1:** Microsoft CDN is blocked
- Use `ssis_tool_source: 'release'` to download from GitHub releases
- Ensure `api.github.com` and `github-releases.githubusercontent.com` are whitelisted

**Scenario 2:** Both Microsoft CDN and GitHub are accessible
- Use default `ssis_tool_source: 'microsoft'` for faster, direct download

**Scenario 3:** Highly restricted network
- Pre-install SSIS DevOps Tools on self-hosted runners
- Tools will not be re-downloaded if already available in PATH

### SQL Server Connectivity
- Ensure network connectivity from the runner to the target SQL Server instance
- For Windows authentication, the runner must be domain-joined or have appropriate credentials configured
- For SQL authentication, ensure SQL Server is configured for mixed-mode authentication

## Troubleshooting

### SSL/TLS Connection Errors

If you encounter SSL/TLS handshake failures when downloading tools:

```powershell
# Temporarily force TLS 1.2 (add before the action step)
- run: |
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
  shell: powershell
```

### Asset Not Found Error

**Error:** `Asset SSISDevOpsTools-1.0.0.0.exe not found in release v1.0.0`

**Solutions:**
1. Verify the release `v1.0.0` exists in the repository with the asset `SSISDevOpsTools-1.0.0.0.exe`
2. Check if the repository is public (private repos may require additional token permissions)
3. Switch to `ssis_tool_source: 'microsoft'` if the release asset is unavailable

### Missing SSISDeploy or SSISBuild Command

**Error:** `SSISDeploy/SSISBuild: The term 'SSISDeploy' is not recognized...`

**Solutions:**
1. Ensure SSIS DevOps Tools downloaded successfully (check action logs)
2. Verify the tool extraction directory (`C:\ssis_temp\1`) was created
3. Manually verify tool availability:
   ```powershell
   - run: |
       Get-Command SSISDeploy -ErrorAction SilentlyContinue
       Get-Command SSISBuild -ErrorAction SilentlyContinue
     shell: powershell
   ```

### Visual Studio Path Issues (for `build` action)

**Error:** `devenv: The term 'devenv' is not recognized...`

**Solutions:**
1. Ensure Visual Studio 2017 is installed on the runner
2. Verify `vswhere.exe` is available in the expected path
3. Check Visual Studio installation:
   ```powershell
   - run: |
       & "C:\Program Files (x86)\Microsoft Visual Studio\Installer\vswhere.exe" -products * -format json
     shell: powershell
   ```

### Network Diagnostics

Add diagnostic commands to troubleshoot connectivity issues:

```yaml
- name: Network Diagnostics
  run: |
    # Test DNS resolution
    Resolve-DnsName download.microsoft.com
    Resolve-DnsName api.github.com
    
    # Test connectivity
    Test-NetConnection -ComputerName download.microsoft.com -Port 443
    Test-NetConnection -ComputerName api.github.com -Port 443
  shell: powershell
```

### Permission Errors

Ensure the GitHub Actions runner has:
- Write permissions to output directories (`output_path`)
- Execute permissions for downloaded tools
- Appropriate SQL Server permissions for deployment actions

## Security Considerations

### Downloaded Executable Integrity

Currently, this action downloads `SSISDevOpsTools.exe` without checksum verification. While the Microsoft CDN source is trusted, consider these security best practices:

1. **Use Microsoft CDN for production workflows** when possible (official Microsoft source)
2. **Verify release assets** if using `ssis_tool_source: 'release'` - ensure the release is from trusted maintainers
3. **Future enhancement:** Checksum verification will be added in future versions (see Roadmap)

### Recommendations

- Store SQL credentials (`ssis_username`, `ssis_password`) in GitHub Secrets, never hardcode
- Use service accounts with minimal required permissions for SQL Server deployments
- Audit and review release assets before using `ssis_tool_source: 'release'`
- Consider using self-hosted runners with pre-installed tools for air-gapped environments

### Repository Visibility

If you make this repository **private** and use `ssis_tool_source: 'release'`:
- The action may fail to download release assets without proper token permissions
- The automatic `GITHUB_TOKEN` should work for most cases, but verify access
- Consider using `ssis_tool_source: 'microsoft'` or pre-installing tools

## Roadmap

Potential improvements and features under consideration:

- **Checksum Verification**: Add SHA256 checksum validation for downloaded executables (add `ssis_tool_checksum` input)
- **Browser Download URL**: Use GitHub's `browser_download_url` for simpler release asset downloads
- **Tool Caching**: Cache SSIS DevOps Tools between workflow runs to reduce download time
- **Diagnostics Mode**: Add `enable_diagnostics` input to enable verbose logging and connectivity tests
- **Skip Download Option**: Add `skip_tool_download` input for runners with pre-installed tools
- **Retry Logic**: Implement retry mechanism for network failures during downloads
- **Custom Tool Version**: Support downloading different versions of SSIS DevOps Tools
- **Multi-platform Support**: Investigate Linux/macOS support for build scenarios using containers

Contributions and feature requests are welcome! Please open an issue to discuss proposed changes.

## Prerequisites

### For Build Actions
- **`build` action**: Requires Visual Studio 2017 installed on the runner
- **`build-sa` action**: No Visual Studio required; SSIS DevOps Tools are automatically installed

### For Deploy Actions
- SQL Server with SSISDB catalog configured
- Appropriate permissions:
  - Windows authentication: Domain-joined runner or appropriate credentials
  - SQL authentication: Valid SQL Server login with SSIS deployment permissions

### For All Actions
- Windows-based GitHub Actions runner (this is a Windows-only action)
- JQ for Windows (for JSON parsing): Install via `chocolatey install jq`

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please feel free to submit issues, feature requests, or pull requests.

## Support

For issues, questions, or contributions:
- Open an issue in this repository
- Review existing issues and discussions
- Check the Troubleshooting section above

---

**Note:** This action is designed for Windows runners only and requires appropriate network access for tool downloads and SQL Server connectivity.
