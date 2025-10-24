name: 'action yml'
description: 'dotnet restore / build & deploy SSIS solutions'
inputs:
  project_path:  
    description: 'path'
    required: true
  project_configuration:  
    description: 'build configuration (e.g. Development)'
    required: true
  output_path:  
    description: 'output_path'
    required: true  
  file_extenstion:  
    description: 'solution file extension filter'
    required: false
    default: '*.sln'
  ssis_action:  
    description: 'build | build-sa | ssis-deploy-sql | ssis-deploy-win | ssis-deploy-multiple-win'
    required: true

  ssis_tool_source:
    description: 'microsoft | release | repo | local (repo: bundled file; local: use runner path you supply via ssis_local_installer_path)'
    required: false
    default: 'microsoft'

  insecure_skip_cert_validation:
    description: 'LAST RESORT: true bypasses TLS certificate validation ONLY for installer download (do not use permanently)'
    required: false
    default: 'false'

  ssis_repo_installer_path:
    description: 'Relative path (inside repo) to bundled installer when ssis_tool_source=repo'
    required: false
    default: 'SSISDevOpsTools-1.0.0.0.exe'

  ssis_local_installer_path:
    description: 'Absolute or runner-accessible path to SSISDevOpsTools*.exe when ssis_tool_source=local'
    required: false
    default: ''

  ssis_source:
    description: '.ispac file path OR directory (for deploy actions)'
    required: false
  ssis_destination:
    description: 'SSIS Catalog path (e.g. /SSISDB/Folder)'
    required: false
  ssis_server:
    description: 'Target SSIS / SQL Server host'
    required: false
  ssis_username:
    description: 'SQL auth username (for ssis-deploy-sql)'
    required: false
  ssis_password:
    description: 'SQL auth password (for ssis-deploy-sql)'
    required: false

outputs:
  random-number:
    description: "Random number"
    value: ${{ steps.random-number-generator.outputs.random-number }}

runs:
  using: "composite"
  steps:
    - name: random number generator
      id: random-number-generator
      shell: bash
      run: echo "random-number=$RANDOM" >> "$GITHUB_OUTPUT"

    - name: build (devenv)
      id: ssis-build
      shell: powershell
      if: ${{ inputs.ssis_action == 'build' }}
      run: |
        $ErrorActionPreference = "Stop"

        $Env:PATH += ";C:\Program Files (x86)\Microsoft Visual Studio\Installer"
        $VS_FILE = vswhere -products * -format json | jq '.[0]' | jq .productPath
        $TMP = Split-Path -Path $VS_FILE.replace('\\', '\')
        $VS_PATH = $TMP.replace('"', '')
        $Env:PATH += ";$VS_PATH"
        
        cd "${{ inputs.project_path }}"
        if (-Not (Test-Path -Path "${{ inputs.output_path }}")) {
          mkdir "${{ inputs.output_path }}" | Out-Null
        }

        Get-ChildItem -Path . -Filter "${{ inputs.file_extenstion }}" -File -Name | ForEach-Object {
          $proj = "${{ inputs.project_path }}/" + $_
          echo "Processing solution: $proj"
          $directory = Split-Path -Path "$proj"
          Push-Location $directory

          Get-ChildItem -Path "$directory" -Recurse -Depth 1 -Filter "*.dtproj" -File -Name | ForEach-Object {
            echo "Building project $_"
            $bin_path = Split-Path -Path "$directory\$_"
            devenv "$_" /build "${{ inputs.project_configuration }}" /Project "$_"
            $bin_path = $bin_path.TrimEnd('\')
            echo "::set-output name=gh_bin_path::$(echo $bin_path)"
            Get-ChildItem -Path "$bin_path\bin\${{ inputs.project_configuration }}" -Recurse -File -Name | ForEach-Object {
              Copy-Item -Path "$bin_path\bin\${{ inputs.project_configuration }}\$_" -Destination "${{ inputs.output_path }}"
            }
          }

          Pop-Location
        }

    - name: build standalone (SSISBuild)
      id: ssis-build-sa
      shell: powershell
      if: ${{ inputs.ssis_action == 'build-sa' }}
      run: |
        $ErrorActionPreference = "Stop"
        echo "Standalone SSISBuild mode"

        if (-Not (Test-Path -Path "${{ inputs.output_path }}")) {
          mkdir -f "${{ inputs.output_path }}" | Out-Null
        } else {
          Remove-Item -Path "${{ inputs.output_path }}" -Recurse -Force
          mkdir -f "${{ inputs.output_path }}" | Out-Null
        }

        $TMP_DIR = "C:\ssis_temp\1"
        if (Test-Path $TMP_DIR) { Remove-Item -Path $TMP_DIR -Recurse -Force }
        mkdir -f $TMP_DIR | Out-Null

        if (-Not (Get-Command SSISBuild -ErrorAction SilentlyContinue)) {
          echo "Installing SSISBuild tooling"
          $current = [Net.ServicePointManager]::SecurityProtocol
          if (($current -band [Net.SecurityProtocolType]::Tls12) -eq 0){
            [Net.ServicePointManager]::SecurityProtocol = $current -bor [Net.SecurityProtocolType]::Tls12
          }
          $insecure = ("${{ inputs.insecure_skip_cert_validation }}").ToLower() -eq "true"
          if ($insecure) {
            Write-Host "WARNING: Insecure cert validation bypass ENABLED for installer download."
            $prevCallback = [System.Net.ServicePointManager]::ServerCertificateValidationCallback
            [System.Net.ServicePointManager]::ServerCertificateValidationCallback = { param($sender,$cert,$chain,$errors) $true }
          }

          try {
            $installerOut = "./SSISDevOpsTools.exe"
            switch ("${{ inputs.ssis_tool_source }}".ToLower()) {
              'microsoft' {
                Invoke-RestMethod -Uri "https://download.microsoft.com/download/5/1/4/5144b772-d3b0-4e1c-a05b-5376f2ea0fc1/SSISDevOpsTools-1.0.0.0.exe" -OutFile $installerOut
              }
              'release' {
                $tag="v1.0.0"
                $asset="SSISDevOpsTools-1.0.0.0.exe"
                $repo="snsinahub-org/ssis"
                $api="https://api.github.com/repos/$repo/releases/tags/$tag"
                $headers=@{ Authorization="Bearer $env:GITHUB_TOKEN"; Accept="application/vnd.github+json"; "X-GitHub-Api-Version"="2022-11-28"; "User-Agent"="ssis-action" }
                $rel=Invoke-RestMethod -Headers $headers -Uri $api -Method GET
                $a=$rel.assets | Where-Object { $_.name -eq $asset }
                if(-not $a){ throw "Asset $asset not found in release $tag." }
                $downloadUrl = $a.browser_download_url
                Invoke-WebRequest -Uri $downloadUrl -OutFile $installerOut
              }
              'repo' {
                $repoPath = Join-Path $env:GITHUB_ACTION_PATH "${{ inputs.ssis_repo_installer_path }}"
                if (-Not (Test-Path $repoPath)) { throw "Bundled installer not found at $repoPath" }
                Copy-Item $repoPath $installerOut -Force
              }
              'local' {
                $localPath = "${{ inputs.ssis_local_installer_path }}"
                if ([string]::IsNullOrWhiteSpace($localPath)) { throw "ssist_tool_source=local requires ssis_local_installer_path to be set." }
                if (-Not (Test-Path $localPath)) { throw "Local installer not found at $localPath" }
                Copy-Item $localPath $installerOut -Force
              }
              default {
                throw "Unknown ssis_tool_source value '${{ inputs.ssis_tool_source }}'. Use microsoft | release | repo | local."
              }
            }
          }
          finally {
            if ($insecure) {
              [System.Net.ServicePointManager]::ServerCertificateValidationCallback = $prevCallback
              Write-Host "Certificate validation callback restored."
            }
          }

          ./SSISDevOpsTools.exe /Q /C /T:$TMP_DIR | Wait-Job
          $Env:Path += ";$TMP_DIR"
        }

        $directory="${{ inputs.project_path }}"
        $build="$TMP_DIR\a"
        if (Test-Path $build) { Remove-Item -Path $build -Recurse -Force }
        mkdir -f "$build" | Out-Null

        cd "${{ inputs.project_path }}"
        Get-ChildItem -Path . -Recurse -Depth 1 -Filter "*.dtproj" -File -Name | ForEach-Object {
          echo "Packaging project $_"
          SSISBuild -project:"${{ inputs.project_path }}\$_" -configuration:"${{ inputs.project_configuration }}" -output:"$build" -log:DIAG
        }

        Get-ChildItem -Path "$build" -Recurse -File -Name | ForEach-Object {
          Copy-Item -Path "$build\$_" -Destination "${{ inputs.output_path }}"
        }

    - name: deploy (sql auth)
      id: ssis-deploy-sql
      shell: powershell
      if: ${{ inputs.ssis_action == 'ssis-deploy-sql' }}
      run: |
        $ErrorActionPreference = "Stop"
        if (-Not (Get-Command SSISDeploy -ErrorAction SilentlyContinue)) {
          echo "installing SSISDeploy"
          $current=[Net.ServicePointManager]::SecurityProtocol
          if (($current -band [Net.SecurityProtocolType]::Tls12) -eq 0){
            [Net.ServicePointManager]::SecurityProtocol = $current -bor [Net.SecurityProtocolType]::Tls12
          }
          $insecure = ("${{ inputs.insecure_skip_cert_validation }}").ToLower() -eq "true"
          if ($insecure) {
            Write-Host "WARNING: Insecure cert validation bypass ENABLED for installer download."
            $prevCallback=[System.Net.ServicePointManager]::ServerCertificateValidationCallback
            [System.Net.ServicePointManager]::ServerCertificateValidationCallback = { param($sender,$cert,$chain,$errors) $true }
          }

          try {
            $installerOut="./SSISDevOpsTools.exe"
            switch ("${{ inputs.ssis_tool_source }}".ToLower()) {
              'microsoft' {
                Invoke-RestMethod -Uri "https://download.microsoft.com/download/5/1/4/5144b772-d3b0-4e1c-a05b-5376f2ea0fc1/SSISDevOpsTools-1.0.0.0.exe" -OutFile $installerOut
              }
              'release' {
                $tag="v1.0.0"
                $asset="SSISDevOpsTools-1.0.0.0.exe"
                $repo="snsinahub-org/ssis"
                $api="https://api.github.com/repos/$repo/releases/tags/$tag"
                $headers=@{ Authorization="Bearer $env:GITHUB_TOKEN"; Accept="application/vnd.github+json"; "X-GitHub-Api-Version"="2022-11-28"; "User-Agent"="ssis-action" }
                $rel=Invoke-RestMethod -Headers $headers -Uri $api -Method GET
                $a=$rel.assets | Where-Object { $_.name -eq $asset }
                if(-not $a){ throw "Asset $asset not found in release $tag." }
                $downloadUrl = $a.browser_download_url
                Invoke-WebRequest -Uri $downloadUrl -OutFile $installerOut
              }
              'repo' {
                $repoPath = Join-Path $env:GITHUB_ACTION_PATH "${{ inputs.ssis_repo_installer_path }}"
                if (-Not (Test-Path $repoPath)) { throw "Bundled installer not found at $repoPath" }
                Copy-Item $repoPath $installerOut -Force
              }
              'local' {
                $localPath = "${{ inputs.ssis_local_installer_path }}"
                if ([string]::IsNullOrWhiteSpace($localPath)) { throw "ssist_tool_source=local requires ssis_local_installer_path to be set." }
                if (-Not (Test-Path $localPath)) { throw "Local installer not found at $localPath" }
                Copy-Item $localPath $installerOut -Force
              }
              default { throw "Unknown ssis_tool_source '${{ inputs.ssis_tool_source }}'." }
            }

            if (Test-Path -Path $TMP_DIR) { Remove-Item -Path $TMP_DIR -Recurse -Force }
            mkdir $TMP_DIR | Out-Null
            ./SSISDevOpsTools.exe /Q /C /T:$TMP_DIR | Wait-Job
            $Env:Path += ";$TMP_DIR"
          }
          finally {
            if ($insecure){
              [System.Net.ServicePointManager]::ServerCertificateValidationCallback = $prevCallback
              Write-Host "Certificate validation callback restored."
            }
          }
        }

        echo "SQL Deploy started"
        SSISDeploy -s:"${{ inputs.ssis_source }}" -d:catalog`;${{ inputs.ssis_destination }}`;${{ inputs.ssis_server }} -at:sql -u:${{ inputs.ssis_username }} -p:${{ inputs.ssis_password }}

    - name: deploy (win auth single)
      id: ssis-deploy-win
      shell: powershell
      if: ${{ inputs.ssis_action == 'ssis-deploy-win' }}
      run: |
        $ErrorActionPreference = "Stop"
        $TMP_DIR="C:\ssis_temp\1"
        if (Test-Path $TMP_DIR) { Remove-Item -Path $TMP_DIR -Recurse -Force }
        mkdir $TMP_DIR | Out-Null

        if (-Not (Get-Command SSISDeploy -ErrorAction SilentlyContinue)) {
          echo "Installing SSISDeploy"
          $current=[Net.ServicePointManager]::SecurityProtocol
          if (($current -band [Net.SecurityProtocolType]::Tls12) -eq 0){
            [Net.ServicePointManager]::SecurityProtocol = $current -bor [Net.SecurityProtocolType]::Tls12
          }
          $insecure = ("${{ inputs.insecure_skip_cert_validation }}").ToLower() -eq "true"
          if ($insecure){
            Write-Host "WARNING: Insecure cert validation bypass ENABLED for installer download."
            $prevCallback=[System.Net.ServicePointManager]::ServerCertificateValidationCallback
            [System.Net.ServicePointManager]::ServerCertificateValidationCallback = { param($sender,$cert,$chain,$errors) $true }
          }

          try {
            $installerOut="./SSISDevOpsTools.exe"
            switch ("${{ inputs.ssis_tool_source }}".ToLower()) {
              'microsoft' {
                Invoke-RestMethod -Uri "https://download.microsoft.com/download/5/1/4/5144b772-d3b0-4e1c-a05b-5376f2ea0fc1/SSISDevOpsTools-1.0.0.0.exe" -OutFile $installerOut
              }
              'release' {
                $tag="v1.0.0"
                $asset="SSISDevOpsTools-1.0.0.0.exe"
                $repo="snsinahub-org/ssis"
                $api="https://api.github.com/repos/$repo/releases/tags/$tag"
                $headers=@{ Authorization="Bearer $env:GITHUB_TOKEN"; Accept="application/vnd.github+json"; "X-GitHub-Api-Version"="2022-11-28"; "User-Agent"="ssis-action" }
                $rel=Invoke-RestMethod -Headers $headers -Uri $api -Method GET
                $a=$rel.assets | Where-Object { $_.name -eq $asset }
                if(-not $a){ throw "Asset $asset not found in release $tag." }
                $downloadUrl = $a.browser_download_url
                Invoke-WebRequest -Uri $downloadUrl -OutFile $installerOut
              }
              'repo' {
                $repoPath = Join-Path $env:GITHUB_ACTION_PATH "${{ inputs.ssis_repo_installer_path }}"
                if (-Not (Test-Path $repoPath)) { throw "Bundled installer not found at $repoPath" }
                Copy-Item $repoPath $installerOut -Force
              }
              'local' {
                $localPath = "${{ inputs.ssis_local_installer_path }}"
                if ([string]::IsNullOrWhiteSpace($localPath)) { throw "ssist_tool_source=local requires ssis_local_installer_path to be set." }
                if (-Not (Test-Path $localPath)) { throw "Local installer not found at $localPath" }
                Copy-Item $localPath $installerOut -Force
              }
              default { throw "Unknown ssis_tool_source '${{ inputs.ssis_tool_source }}'." }
            }
            ./SSISDevOpsTools.exe /Q /C /T:$TMP_DIR | Wait-Job
            $Env:Path += ";$TMP_DIR"
          }
          finally {
            if ($insecure){
              [System.Net.ServicePointManager]::ServerCertificateValidationCallback = $prevCallback
              Write-Host "Certificate validation callback restored."
            }
          }
        }

        echo "Win Deploy started"
        SSISDeploy -s:"${{ inputs.ssis_source }}" -d:catalog`;${{ inputs.ssis_destination }}`;${{ inputs.ssis_server }} -at:win 

    - name: deploy (win auth multiple)
      id: ssis-deploy-win-multiple
      shell: powershell
      if: ${{ inputs.ssis_action == 'ssis-deploy-multiple-win' }}
      run: |
        $ErrorActionPreference = "Stop"
        $TMP_DIR="C:\ssis_temp\1"
        if (Test-Path $TMP_DIR) { Remove-Item -Path $TMP_DIR -Recurse -Force }
        mkdir $TMP_DIR | Out-Null

        if (-Not (Get-Command SSISDeploy -ErrorAction SilentlyContinue)) {
          echo "Installing SSISDeploy"
          $current=[Net.ServicePointManager]::SecurityProtocol
          if (($current -band [Net.SecurityProtocolType]::Tls12) -eq 0){
            [Net.ServicePointManager]::SecurityProtocol = $current -bor [Net.SecurityProtocolType]::Tls12
          }
          $insecure = ("${{ inputs.insecure_skip_cert_validation }}").ToLower() -eq "true"
          if ($insecure){
            Write-Host "WARNING: Insecure cert validation bypass ENABLED for installer download."
            $prevCallback=[System.Net.ServicePointManager]::ServerCertificateValidationCallback
            [System.Net.ServicePointManager]::ServerCertificateValidationCallback = { param($sender,$cert,$chain,$errors) $true }
          }

          try {
            $installerOut="./SSISDevOpsTools.exe"
            switch ("${{ inputs.ssis_tool_source }}".ToLower()) {
              'microsoft' {
                Invoke-RestMethod -Uri "https://download.microsoft.com/download/5/1/4/5144b772-d3b0-4e1c-a05b-5376f2ea0fc1/SSISDevOpsTools-1.0.0.0.exe" -OutFile $installerOut
              }
              'release' {
                $tag="v1.0.0"
                $asset="SSISDevOpsTools-1.0.0.0.exe"
                $repo="snsinahub-org/ssis"
                $api="https://api.github.com/repos/$repo/releases/tags/$tag"
                $headers=@{ Authorization="Bearer $env:GITHUB_TOKEN"; Accept="application/vnd.github+json"; "X-GitHub-Api-Version"="2022-11-28"; "User-Agent"="ssis-action" }
                $rel=Invoke-RestMethod -Headers $headers -Uri $api -Method GET
                $a=$rel.assets | Where-Object { $_.name -eq $asset }
                if(-not $a){ throw "Asset $asset not found in release $tag." }
                $downloadUrl = $a.browser_download_url
                Invoke-WebRequest -Uri $downloadUrl -OutFile $installerOut
              }
              'repo' {
                $repoPath = Join-Path $env:GITHUB_ACTION_PATH "${{ inputs.ssis_repo_installer_path }}"
                if (-Not (Test-Path $repoPath)) { throw "Bundled installer not found at $repoPath" }
                Copy-Item $repoPath $installerOut -Force
              }
              'local' {
                $localPath = "${{ inputs.ssis_local_installer_path }}"
                if ([string]::IsNullOrWhiteSpace($localPath)) { throw "ssist_tool_source=local requires ssis_local_installer_path to be set." }
                if (-Not (Test-Path $localPath)) { throw "Local installer not found at $localPath" }
                Copy-Item $localPath $installerOut -Force
              }
              default { throw "Unknown ssis_tool_source '${{ inputs.ssis_tool_source }}'." }
            }
            ./SSISDevOpsTools.exe /Q /C /T:$TMP_DIR | Wait-Job
            $Env:Path += ";$TMP_DIR"
          }
          finally {
            if ($insecure){
              [System.Net.ServicePointManager]::ServerCertificateValidationCallback = $prevCallback
              Write-Host "Certificate validation callback restored."
            }
          }
        }

        echo "Win Deploy (multiple) started"
        cd "${{ inputs.ssis_source }}"
        Get-ChildItem -Path "${{ inputs.ssis_source }}" -Recurse -Depth 1 -Filter "*.ispac" -File -Name | ForEach-Object {
          SSISDeploy -s:"${{ inputs.ssis_source }}\$_" -d:catalog`;${{ inputs.ssis_destination }}`;${{ inputs.ssis_server }} -at:win 
        }
