# GitHub Actions VHDX Caching
This repository demonstrates a technique for speeding up GitHub Actions workflows on Windows runners by caching tool installations in a VHDX (Virtual Hard Disk) file.

## Overview
GitHub Actions workflows often require installing the same tools repeatedly across different runs. This process can be time-consuming, especially for large dependencies like development environments, SDKs, and build tools.

This project showcases how to:
1. Create a VHDX file once
2. Install tools inside the VHDX
3. Cache the VHDX file between workflow runs
4. Mount the VHDX in subsequent workflows

By using this approach, you can significantly reduce the setup time for your workflows.

## Workflows

### `cache-vhdx.yml`

This workflow creates and populates a VHDX file with common development tools. This workflows populates the following tools, but the concept is applicable to other development tools.

- PowerShell 7
- Node.js
- Git for Windows
- Visual Studio Build Tools
- .NET SDK

The workflow:
1. Creates a new VHDX file
2. Mounts it as the `X:` drive
3. Installs tools to this drive
4. Dismounts the VHDX
5. Caches the VHDX file for future workflow runs

### `test-cache-vhdx.yml`

This workflow demonstrates how to use the cached VHDX:

1. Restores the VHDX from the cache
2. Mounts it as the `X:` drive
3. Updates the system PATH to include the tools
4. Verifies the tool installations

## How It Works

### Step 1: Create and populate the VHDX

The first workflow (`cache-vhdx.yml`) runs either on manual workflow dispatch or when changes are made to the workflow file itself. It:

```yaml
- name: Create and Mount VHDX
  run: |
    $Drive = New-VHD -Path D:\dev_drive.vhdx -SizeBytes 50GB -Dynamic |
        Mount-VHD -PassThru | 
        Initialize-Disk -PassThru |
        New-Partition -UseMaximumSize |
        Format-Volume -FileSystem NTFS -Confirm:$false -Force
    $Drive | Get-Partition | Set-Partition -NewDriveLetter 'X'
```

After installing tools, it saves the VDHX to the GitHub Actions cache:

```yaml
- name: Save VHDX to Cache
  uses: actions/cache/save@v4
  with:
    path: D:\dev_drive.vhdx
    key: vhdx-cache-${{ github.run_id }}
```

### Step 2: Use the cached VHDX

Subsequent workflows can restore and mount the VHDX:

```yaml
- name: Restore VHDX from Cache
  uses: actions/cache/restore@v4
  with:
    path: D:\dev_drive.vhdx
    key: vhdx-cache
    fail-on-cache-miss: true

- name: Mount VHDX
  run: |
    $Drive = Mount-VHD -Path D:\dev_drive.vhdx -PassThru
    $Drive | Get-Disk | Get-Partition | Where-Object { $_.Type -eq 'Basic' } | Set-Partition -NewDriveLetter 'X'
```

## Benefits
- **Faster Workflows**: Dramtically reduces the time needed to install tools
- **Consistency**: Ensures the same tool versions across workflow runs
- **Reduced Network Usage**: Downloads dependencies only once
- **Simpler Workflows**: Reduces the need for complex setup steps

## Limitations
- **Cache Size**: GitHub Actions has cache size limits (currently 10GB per repository)
- **Windows Only**: This technique is specific to Windows runners
- **Cache Expiration**: GitHub Actions caches may expire after a period of inactivity

## Setup Instructions
1. Create a `.github/workflows` directory in your repository
2. Add the two workflow files (`cache-vhdx.yml` and `test-cache-vhdx.yml`)
3. Run the `cache-vhdx` workflow manually to create and cache the VHDX
4. Now any workflow that needs these tools can use the cached VHDX

## Customization
You can modify the `cache-vhdx.yml` workflow to install different tools or adjust the VHDX size based on your needs.

## Requirements
- GitHub Actions with Windows runners (`windows-2025` used in this example)
- Actions that need Windows-specific development tools
