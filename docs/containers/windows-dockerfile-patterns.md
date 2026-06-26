# Windows Dockerfile Patterns — Complete Mastery Reference

**Domain:** Windows Container Image Engineering
**Scope:** Dockerfile syntax for Windows, base image selection, Registry manipulation, PowerShell scripting in builds, multi-stage builds, layer optimization, and real-world application patterns
**Last Updated:** 2026-06-22
**File Path:** `windows-dockerfile-patterns.md`

---

## Table of Contents

1. [How Windows Dockerfiles Differ from Linux](#1-how-windows-dockerfiles-differ-from-linux)
2. [Base Image Selection — Choosing the Right Foundation](#2-base-image-selection--choosing-the-right-foundation)
3. [Core Dockerfile Instructions — Windows Behavior](#3-core-dockerfile-instructions--windows-behavior)
4. [Shell Selection — CMD vs PowerShell](#4-shell-selection--cmd-vs-powershell)
5. [Working with the Windows Filesystem in Dockerfiles](#5-working-with-the-windows-filesystem-in-dockerfiles)
6. [Windows Registry in Dockerfiles](#6-windows-registry-in-dockerfiles)
7. [Installing Software in Windows Containers](#7-installing-software-in-windows-containers)
8. [Environment Variables and Build Arguments](#8-environment-variables-and-build-arguments)
9. [Windows Service Configuration Inside Containers](#9-windows-service-configuration-inside-containers)
10. [Multi-Stage Builds for Windows](#10-multi-stage-builds-for-windows)
11. [Networking and Port Exposure in Windows Containers](#11-networking-and-port-exposure-in-windows-containers)
12. [Volume and Data Persistence Patterns](#12-volume-and-data-persistence-patterns)
13. [User and Permission Management](#13-user-and-permission-management)
14. [Real-World Application Patterns](#14-real-world-application-patterns)
15. [Layer Optimization and Image Size](#15-layer-optimization-and-image-size)
16. [Building and Tagging Windows Images](#16-building-and-tagging-windows-images)
17. [Pushing to a Registry](#17-pushing-to-a-registry)
18. [Multi-Platform Image Manifests](#18-multi-platform-image-manifests)
19. [Troubleshooting Windows Build Failures](#19-troubleshooting-windows-build-failures)
20. [Homelab Practice Tasks](#20-homelab-practice-tasks)
- [Quick Reference](#quick-reference)

---

## 1. How Windows Dockerfiles Differ from Linux

The Dockerfile instruction set is the same across Linux and Windows — `FROM`, `RUN`, `COPY`, `ENV`, `EXPOSE`, `CMD`, `ENTRYPOINT` all exist and work the same way structurally. What changes is **everything inside those instructions** — the shell, the paths, the package management, the filesystem layout, and how the OS behaves during build.

### The Mental Model Shift

| Concept | Linux Dockerfile | Windows Dockerfile |
|---------|-----------------|-------------------|
| Default shell | `/bin/sh -c` | `cmd /S /C` |
| Package manager | `apt`, `apk`, `yum` | `winget`, `choco`, direct `.exe`/`.msi` |
| Path separator | `/` | `\` (escape as `\\` in JSON, use `/` in PowerShell) |
| Config files | `/etc/app.conf` | Registry or `C:\app\config.ini` |
| Service management | `systemd`, `supervisord` | Windows Service Manager, `sc.exe` |
| Temp directory | `/tmp` | `C:\Windows\Temp` or `%TEMP%` |
| App install location | `/usr/local/bin` | `C:\Program Files\` or `C:\app\` |
| Line endings | LF (`\n`) | CRLF (`\r\n`) — can cause script failures if mixed |
| User | `root` (default) | `ContainerAdministrator` (default) |
| Environment variable syntax | `$VAR` or `${VAR}` | `%VAR%` (cmd) or `$env:VAR` (PowerShell) |

> ⚠️ **Line ending hazard:** If you write shell scripts on Linux/WSL2 and COPY them into a Windows container, CRLF vs LF mismatches can cause silent failures. Always verify line endings when crossing platforms.

### What You Must Be In to Build Windows Images

You must be in **Windows container mode** before running any `docker build` for a Windows image. The Docker daemon determines which kernel backs the build environment.

```powershell
# Switch to Windows container mode
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon

# Verify
docker info | findstr "OSType"
# Must show: OSType: windows
```

---

## 2. Base Image Selection — Choosing the Right Foundation

Choosing the wrong base image is the single most common Windows container mistake. Each base has hard limitations on what APIs, tools, and binaries can run inside it.

### The Four Official Microsoft Base Images

#### `nanoserver` — Minimal, Fastest, Most Restrictive

```dockerfile
FROM mcr.microsoft.com/windows/nanoserver:ltsc2022
```

- **Size:** ~260–300MB compressed
- **Shell:** `cmd.exe` for basic operations — **no PowerShell by default**
- **Has:** .NET runtime (if you use the .NET nanoserver variant), basic Win32 APIs
- **Missing:** PowerShell (unless installed separately), WMI, most Win32 APIs, MSI installer support, many system DLLs
- **Use for:** .NET 6+ apps, Go binaries compiled for Windows, minimal CLI tools
- **Do NOT use for:** Anything needing PowerShell during runtime, WMI queries, MSI installs, IIS

#### `servercore` — Full Win32, PowerShell Included

```dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2022
```

- **Size:** ~3.5GB compressed
- **Shell:** `cmd.exe` and `powershell.exe`
- **Has:** Full Win32 API surface, PowerShell 5.1, .NET Framework, WMI, most system DLLs, MSI support
- **Missing:** GUI components, DirectX, some specialized APIs
- **Use for:** .NET Framework apps, IIS, anything needing PowerShell at runtime, most enterprise Windows apps
- **This is your default choice** when you are unsure

#### `server` — Nearly Complete Windows Server

```dockerfile
FROM mcr.microsoft.com/windows/server:ltsc2022
```

- **Size:** ~5GB compressed
- **Has:** Everything in servercore plus additional Windows Server features
- **Use for:** Apps that genuinely need features absent from servercore
- **Rarely needed** — servercore covers most cases

#### `windows` — General Purpose Desktop APIs

```dockerfile
FROM mcr.microsoft.com/windows:ltsc2022
```

- **Size:** ~3GB compressed
- **Has:** Broader API surface including some desktop APIs
- **Use for:** Apps that were designed for desktop Windows and need APIs not in servercore
- **Avoid if possible** — servercore is usually sufficient and better documented

### Version Tags — Pin These, Always

| Tag | Windows Version | Notes |
|-----|----------------|-------|
| `ltsc2025` | Windows Server 2025 | Newest LTS — use this if your host runs Server 2025 |
| `ltsc2022` | Windows Server 2022 / Windows 11 | Widely used current LTS — use this if your host runs Server 2022 |
| `ltsc2019` | Windows Server 2019 / Windows 10 1809 | Previous LTS |
| `ltsc2016` | Windows Server 2016 | Legacy — avoid for new builds |
| `20H2`, `21H2`, `22H2` | Semi-annual channel releases | Short support windows |

> ⚠️ **Never use `latest` for Windows base images.** Microsoft does not publish or maintain a `latest` tag for the Windows Server, Windows Server Core, or Nano Server base images at all — you must declare a specific tag every time. Beyond that, even if you pin to a moving alias, the underlying OS version can shift and break Process Isolation compatibility. Always pin to an explicit tag like `ltsc2022` or `ltsc2025`.

> 🎯 **The host and container OS versions must match (or use Hyper-V isolation).** Windows containers — unlike Linux containers — require the container's base OS build to match the host's build when using the default Process Isolation mode. If your host is Windows Server 2022, build with `ltsc2022`. If it's Server 2025, build with `ltsc2025`. Mismatches fail at `docker run` with an OS version compatibility error (see Section 19). If you need to run a container built for an older OS version on a newer host, use `docker run --isolation=hyperv` instead of rebuilding.

### .NET-Specific Base Images (Preferred for .NET Apps)

Microsoft publishes pre-built .NET images that layer on top of nanoserver or servercore. Use these instead of installing .NET yourself:

```dockerfile
# .NET 8 runtime on nanoserver (smallest for .NET apps)
FROM mcr.microsoft.com/dotnet/runtime:8.0-nanoserver-ltsc2022

# .NET 8 ASP.NET runtime (for web apps)
FROM mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022

# .NET 8 SDK (for building — use in multi-stage builds)
FROM mcr.microsoft.com/dotnet/sdk:8.0-nanoserver-ltsc2022

# .NET Framework 4.8 (legacy — requires servercore)
FROM mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022

# .NET 10 is also current — same pattern, both ltsc2022 and ltsc2025 variants exist
FROM mcr.microsoft.com/dotnet/aspnet:10.0-nanoserver-ltsc2022
FROM mcr.microsoft.com/dotnet/aspnet:10.0-nanoserver-ltsc2025
```

---

## 3. Core Dockerfile Instructions — Windows Behavior

### FROM

```dockerfile
# Standard single-stage
FROM mcr.microsoft.com/windows/servercore:ltsc2022

# Named stage for multi-stage builds
FROM mcr.microsoft.com/dotnet/sdk:8.0-nanoserver-ltsc2022 AS build-env
```

### RUN

`RUN` executes commands during the image build. On Windows, these run in either `cmd.exe` or PowerShell depending on your `SHELL` declaration (see Section 4).

```dockerfile
# cmd.exe syntax (default)
RUN mkdir C:\app

# PowerShell syntax (after SHELL declaration)
RUN New-Item -ItemType Directory -Path C:\app -Force

# Chaining commands in cmd.exe (use & or &&)
RUN mkdir C:\app & mkdir C:\app\logs & mkdir C:\app\config

# Chaining in PowerShell (use ; or separate RUN instructions)
RUN New-Item -ItemType Directory -Path C:\app, C:\app\logs, C:\app\config -Force
```

> ⚠️ **Each RUN instruction creates a new image layer.** Chain related commands together to minimize layers. But — Windows layers are large (hundreds of MB), so the layer count matters more than on Linux.

### COPY

```dockerfile
# Copy a single file
COPY myapp.exe C:\app\myapp.exe

# Copy a directory (note trailing backslash on destination)
COPY ./src C:\app\src\

# Copy with wildcards
COPY *.dll C:\app\libs\

# Copy from a build stage (multi-stage)
COPY --from=build-env C:\app\publish\ C:\app\
```

> ⚠️ **Path separator in COPY:** Docker accepts both `/` and `\` in COPY destination paths. Prefer `/` — it works on both Linux and Windows Docker and avoids escaping issues. `C:/app/` and `C:\app\` are both valid.

### ADD

```dockerfile
# ADD can fetch URLs (use sparingly — prefer COPY + RUN for clarity)
ADD https://example.com/installer.exe C:\installers\

# ADD can extract .tar.gz archives (but NOT .zip on Windows — use PowerShell Expand-Archive)
ADD archive.tar.gz C:\app\
```

> ⚠️ **ADD does not extract `.zip` files on Windows.** Use `COPY` + `RUN Expand-Archive` instead.

### WORKDIR

```dockerfile
# Sets the working directory for subsequent RUN, CMD, ENTRYPOINT, COPY, ADD
WORKDIR C:\app

# Subsequent RUN commands execute from C:\app
RUN myapp.exe --init
```

`WORKDIR` creates the directory if it does not exist. It is the Windows equivalent of `mkdir -p && cd`.

### ENV

```dockerfile
ENV APP_VERSION=1.0.0
ENV APP_HOME=C:\app
ENV LOG_LEVEL=info

# Multiple on one line
ENV APP_VERSION=1.0.0 APP_HOME=C:\app LOG_LEVEL=info
```

Access in `cmd.exe`: `%APP_VERSION%`
Access in PowerShell: `$env:APP_VERSION`

### ARG

Build-time variables — not present in the final image.

```dockerfile
ARG BUILD_NUMBER=0
ARG DOTNET_VERSION=8.0

# Use in RUN
RUN echo Build %BUILD_NUMBER%
```

```powershell
# Pass ARG at build time
docker build --build-arg BUILD_NUMBER=42 -t myapp:42 .
```

### EXPOSE

```dockerfile
# Document which ports the container listens on
EXPOSE 80
EXPOSE 443
EXPOSE 8080
```

`EXPOSE` is documentation only — it does not actually publish ports. Ports are published with `-p` in `docker run` or `ports:` in Compose.

### CMD and ENTRYPOINT

```dockerfile
# CMD — default command, can be overridden at docker run
CMD ["myapp.exe", "--config", "C:\\app\\config.ini"]

# ENTRYPOINT — fixed executable, CMD becomes its arguments
ENTRYPOINT ["myapp.exe"]
CMD ["--config", "C:\\app\\config.ini"]

# Shell form (runs via cmd /S /C — avoid for production)
CMD myapp.exe --config C:\app\config.ini
```

> ⚠️ **Always use JSON array (exec form) for CMD and ENTRYPOINT.** Shell form spawns an extra `cmd.exe` process as PID 1, which does not forward signals properly. Your container won't stop cleanly on `docker stop`.

---

## 4. Shell Selection — CMD vs PowerShell

### The Default Shell Problem

Windows Dockerfiles default to `cmd.exe`. `cmd.exe` is limited — no loops, no conditionals worth using, no object pipeline, string manipulation is painful. For any non-trivial build logic, switch to PowerShell.

### Declaring the Shell

```dockerfile
# Switch all subsequent RUN instructions to PowerShell
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]
```

The two preference variables are critical:

- `$ErrorActionPreference = 'Stop'` — makes PowerShell exit with a non-zero code on any error, which makes Docker treat the build step as failed. Without this, PowerShell errors are silent and the build continues, producing broken images.
- `$ProgressPreference = 'SilentlyContinue'` — suppresses download progress bars which spam the build log and slow down `Invoke-WebRequest`.

### Full Pattern — Switching Shell at the Right Point

```dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2022

# Use cmd.exe for early setup (faster for simple mkdir commands)
RUN mkdir C:\app

# Switch to PowerShell for complex operations
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Now all RUN instructions use PowerShell
RUN New-Item -ItemType Directory -Path C:\app\logs, C:\app\config -Force

RUN Invoke-WebRequest -Uri "https://example.com/installer.exe" -OutFile "C:\installers\installer.exe"
```

### Mixing Shells Within a Single RUN

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Call cmd.exe from within a PowerShell RUN block
RUN cmd /c "net user ContainerUser /add"

# Or use PowerShell's native approach
RUN net user ContainerUser /add
```

### PowerShell Script Files

For complex build logic, write a `.ps1` script and `COPY` + `RUN` it:

```powershell
# build-scripts/install-deps.ps1
$ErrorActionPreference = 'Stop'
$ProgressPreference = 'SilentlyContinue'

Write-Host "Installing dependencies..."

# Download a tool
Invoke-WebRequest -Uri "https://example.com/tool.zip" -OutFile "C:\Temp\tool.zip"
Expand-Archive -Path "C:\Temp\tool.zip" -DestinationPath "C:\Tools\" -Force
Remove-Item "C:\Temp\tool.zip"

Write-Host "Done."
```

```dockerfile
COPY build-scripts/ C:\build-scripts\
RUN powershell -ExecutionPolicy Bypass -File C:\build-scripts\install-deps.ps1
```

---

## 5. Working with the Windows Filesystem in Dockerfiles

### Path Rules

```dockerfile
# Both of these work in Dockerfile instructions:
COPY ./app C:/app/           # Forward slashes — recommended
COPY ./app C:\app\           # Backslashes — works but needs escaping in JSON arrays

# In JSON arrays (CMD, ENTRYPOINT, RUN exec form), backslashes must be doubled:
CMD ["C:\\app\\myapp.exe"]   # Correct
CMD ["C:\app\myapp.exe"]     # WRONG — \a, \m are escape sequences

# In PowerShell RUN blocks, forward or back both work:
RUN New-Item -Path C:\app -ItemType Directory
RUN New-Item -Path C:/app -ItemType Directory   # Also valid
```

### Common Directory Operations

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Create directory tree
RUN New-Item -ItemType Directory -Path C:\app\bin, C:\app\logs, C:\app\config, C:\app\data -Force

# Copy file and verify it exists
COPY myapp.exe C:\app\bin\
RUN if (-not (Test-Path C:\app\bin\myapp.exe)) { throw "myapp.exe not found after COPY" }

# Set file permissions (Windows ACL)
RUN icacls C:\app\data /grant "ContainerUser:(OI)(CI)F" /T

# Create a symlink
RUN New-Item -ItemType SymbolicLink -Path C:\app\current -Target C:\app\v1.0.0

# Delete files/directories
RUN Remove-Item -Path C:\Windows\Temp\* -Recurse -Force -ErrorAction SilentlyContinue
```

### Handling the Windows PATH

```dockerfile
# Add to system PATH using environment variable layering
ENV PATH="C:\app\bin;C:\tools\bin;${PATH}"

# Or via PowerShell Registry manipulation (persists in image)
RUN [Environment]::SetEnvironmentVariable('PATH', $env:PATH + ';C:\app\bin', [EnvironmentVariableTarget]::Machine)
```

> ⚠️ **ENV PATH modification in Dockerfiles is preferred** over Registry manipulation. The `ENV` instruction is processed by the Docker layer system and is reliably inherited by all subsequent instructions and by containers at runtime.

### Zip File Handling

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Download and extract a zip
RUN Invoke-WebRequest -Uri "https://example.com/app.zip" -OutFile "C:\Temp\app.zip" ; \
    Expand-Archive -Path "C:\Temp\app.zip" -DestinationPath "C:\app\" -Force ; \
    Remove-Item "C:\Temp\app.zip" -Force
```

> 🎯 **Always delete downloaded installers and zip files in the same RUN instruction** that extracted them. If you delete in a separate `RUN`, the previous layer still contains the large file and it remains in the final image size.

---

## 6. Windows Registry in Dockerfiles

The Windows Registry is a first-class configuration target in Windows Dockerfiles. Many Windows applications read their configuration exclusively from the Registry. You set Registry values during the build to pre-configure the application in the image.

### Reading and Writing Registry in RUN Instructions

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Create a Registry key
RUN New-Item -Path "HKLM:\SOFTWARE\MyApp" -Force

# Set string value (REG_SZ)
RUN Set-ItemProperty -Path "HKLM:\SOFTWARE\MyApp" -Name "InstallPath" -Value "C:\app"

# Set DWORD value (REG_DWORD)
RUN Set-ItemProperty -Path "HKLM:\SOFTWARE\MyApp" -Name "MaxConnections" -Value 100 -Type DWord

# Set multi-string value (REG_MULTI_SZ)
RUN Set-ItemProperty -Path "HKLM:\SOFTWARE\MyApp" -Name "AllowedHosts" -Value @("localhost","127.0.0.1") -Type MultiString

# Set expandable string (REG_EXPAND_SZ — expands %VARIABLES%)
RUN Set-ItemProperty -Path "HKLM:\SOFTWARE\MyApp" -Name "LogPath" -Value "%TEMP%\MyApp\logs" -Type ExpandString

# Read a value (useful for verification)
RUN (Get-ItemProperty "HKLM:\SOFTWARE\MyApp").InstallPath

# Delete a key tree
RUN Remove-Item -Path "HKLM:\SOFTWARE\MyApp" -Recurse -Force
```

### Importing a .reg File

For complex Registry configurations, write a `.reg` file and import it during build:

```reg
; app-config.reg
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\MyApp]
"InstallPath"="C:\\app"
"Version"="1.0.0"
"MaxConnections"=dword:00000064
"LogLevel"="info"

[HKEY_LOCAL_MACHINE\SOFTWARE\MyApp\Features]
"EnableCache"=dword:00000001
"EnableMetrics"=dword:00000001
```

```dockerfile
COPY app-config.reg C:\setup\app-config.reg
RUN reg import C:\setup\app-config.reg
RUN Remove-Item C:\setup\app-config.reg -Force
```

### Common Registry Locations for Application Configuration

| Registry Path | Purpose |
|--------------|---------|
| `HKLM:\SOFTWARE\<Company>\<App>` | Application settings (system-wide) |
| `HKCU:\SOFTWARE\<Company>\<App>` | Per-user settings (ContainerUser by default) |
| `HKLM:\SYSTEM\CurrentControlSet\Services\<SvcName>` | Windows service configuration |
| `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options` | Process debugging/monitoring hooks |
| `HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment` | System-wide environment variables |

### Setting Environment Variables via Registry (Persistent Across Processes)

```dockerfile
# This persists the variable in the Registry, making it available to all
# processes in the container including those started by Windows Service Manager
RUN [Environment]::SetEnvironmentVariable('MY_APP_ENV', 'production', [EnvironmentVariableTarget]::Machine)

# Verify
RUN [Environment]::GetEnvironmentVariable('MY_APP_ENV', [EnvironmentVariableTarget]::Machine)
```

---

## 7. Installing Software in Windows Containers

### Method 1: Direct Executable/MSI Install

The most reliable method for most Windows software:

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Download and install an .exe (silent install flags vary by installer)
RUN Invoke-WebRequest -Uri "https://example.com/app-installer.exe" -OutFile "C:\Temp\installer.exe" ; \
    Start-Process -FilePath "C:\Temp\installer.exe" -ArgumentList "/S", "/D=C:\app" -Wait -NoNewWindow ; \
    Remove-Item "C:\Temp\installer.exe" -Force

# Install an MSI silently
RUN Invoke-WebRequest -Uri "https://example.com/app.msi" -OutFile "C:\Temp\app.msi" ; \
    Start-Process msiexec.exe -ArgumentList "/i", "C:\Temp\app.msi", "/qn", "/norestart", "INSTALLDIR=C:\app" -Wait -NoNewWindow ; \
    Remove-Item "C:\Temp\app.msi" -Force
```

> ⚠️ **MSI installs require `servercore` or `server` base — not `nanoserver`.** Nanoserver does not have the Windows Installer service.

### Method 2: Chocolatey Package Manager

Chocolatey is the most common third-party package manager for Windows containers:

```dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2022

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Install Chocolatey
RUN Set-ExecutionPolicy Bypass -Scope Process -Force ; \
    [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072 ; \
    Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install packages
RUN choco install -y git
RUN choco install -y nodejs-lts
RUN choco install -y python3

# Or install multiple packages in one layer
RUN choco install -y git nodejs-lts python3 --no-progress
```

### Method 3: winget (Windows Package Manager)

`winget` is available in newer Windows container images but has limitations in containers (no user context, some packages require desktop components):

```dockerfile
FROM mcr.microsoft.com/windows/server:ltsc2022

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

RUN winget install --id Git.Git -e --silent --accept-package-agreements --accept-source-agreements
```

> ⚠️ **winget in containers is unreliable.** Many packages fail because they require GUI components, user sessions, or the Windows App Installer service. Prefer Chocolatey or direct downloads for containers.

### Method 4: Copying Pre-Built Binaries

The cleanest, most reproducible method — especially for Go, Rust, or self-contained .NET binaries:

```dockerfile
FROM mcr.microsoft.com/windows/nanoserver:ltsc2022

# Copy a self-contained binary directly — no installer needed
COPY --from=build-env C:\app\publish\myapp.exe C:\app\myapp.exe

WORKDIR C:\app
CMD ["myapp.exe"]
```

### Method 5: Windows Features (DISM)

For enabling built-in Windows features like IIS:

```dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2022

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Enable IIS
RUN Add-WindowsFeature Web-Server, Web-Asp-Net45, NET-Framework-Features

# Or use DISM directly
RUN dism /online /enable-feature /featurename:IIS-WebServer /all /norestart
```

### Method 6: NuGet / PowerShell Gallery

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Install NuGet provider
RUN Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force

# Install a PowerShell module
RUN Install-Module -Name SqlServer -Force -AllowClobber

# Install from PowerShell Gallery
RUN Install-Module -Name PSWindowsUpdate -Force
```

---

## 8. Environment Variables and Build Arguments

### ENV — Runtime Variables (Baked Into Image)

```dockerfile
# Single variable
ENV APP_ENV=production

# Multiple variables
ENV APP_ENV=production \
    LOG_LEVEL=warn \
    PORT=8080 \
    DB_HOST=localhost

# Variables referencing other variables
ENV APP_HOME=C:\app
ENV APP_LOG=${APP_HOME}\logs
ENV APP_CONFIG=${APP_HOME}\config
```

### ARG — Build-Time Only Variables

```dockerfile
# Declare with optional default
ARG BUILD_VERSION=dev
ARG REGISTRY=myregistry.local

# Use in RUN
RUN echo Building version %BUILD_VERSION%

# ARG can influence ENV (making build-time values available at runtime)
ARG APP_VERSION=1.0.0
ENV APP_VERSION=${APP_VERSION}
```

```powershell
# Pass ARG values at build time
docker build `
  --build-arg BUILD_VERSION=2.1.0 `
  --build-arg REGISTRY=registry.example.local `
  -t myapp:2.1.0 .
```

### Accessing Variables — Shell Syntax Reference

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

ENV MY_VAR=hello

# PowerShell access
RUN Write-Host $env:MY_VAR

# cmd.exe access (within a powershell block, use cmd /c)
RUN cmd /c "echo %MY_VAR%"

# String interpolation in PowerShell
RUN Write-Host "The value is: $env:MY_VAR"
RUN $path = "$env:APP_HOME\bin"; Write-Host $path
```

### Secrets — What NOT to Do

```dockerfile
# NEVER do this — ARG and ENV values appear in docker history
ARG DB_PASSWORD=mysecretpassword    # BAD — visible in image layers
ENV DB_PASSWORD=mysecretpassword    # BAD — baked into image permanently
```

Handle secrets at runtime via:
- `docker run -e DB_PASSWORD=...`
- Docker secrets (Swarm)
- Kubernetes Secrets mounted as env vars
- External secret managers (Vault, AWS SSM)

---

## 9. Windows Service Configuration Inside Containers

Windows Services are the Windows equivalent of Linux daemons. Many Windows applications run as services. In containers, this requires special handling because there is no Service Control Manager (SCM) running by default in minimal containers.

### Running a Windows Service in a Container

There are two patterns for keeping a container alive while a Windows Service runs inside it. Choose based on your use case.

#### Pattern A: ServiceMonitor (Production-Correct)

Microsoft ships a lightweight binary called **ServiceMonitor.exe** specifically for this problem. It is the binary that the official IIS base image uses (`C:\ServiceMonitor.exe`). It watches a named Windows Service and exits with a non-zero code the instant the service stops — which causes Docker/Kubernetes to detect the failure and restart the container.

```dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2022

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

COPY myservice/ C:\myservice\

# Register the Windows Service
RUN New-Service `
      -Name "MyService" `
      -BinaryPathName "C:\myservice\myservice.exe" `
      -DisplayName "My Application Service" `
      -StartupType Automatic `
      -Description "My containerized Windows service"

# Download ServiceMonitor, pinned to a specific release rather than
# "latest" so builds stay reproducible. Check
# https://github.com/microsoft/IIS.ServiceMonitor/releases for the
# current version before using this in a new project.
RUN Invoke-WebRequest `
      -Uri "https://github.com/microsoft/IIS.ServiceMonitor/releases/download/v2.0.1.10/ServiceMonitor.exe" `
      -OutFile "C:\ServiceMonitor.exe"

# ServiceMonitor watches "MyService" — exits immediately when the service stops
ENTRYPOINT ["C:\\ServiceMonitor.exe", "MyService"]
```

> 🎯 **Why this is better than a polling loop:** ServiceMonitor uses Windows event notification — zero CPU overhead while the service runs. It also exits the instant the service stops, giving Kubernetes a clean failure signal to trigger a Pod restart.
>
> 📌 **Pin the version, don't use `releases/latest/download/`.** A `releases/latest` URL silently resolves to whatever the newest release happens to be at build time — which means the same Dockerfile can pull a different binary today than it did last month, with no record of which version actually shipped in a given image. Pinning to an explicit tag (as above) makes builds reproducible and auditable. The note about the old `dotnetbinaries.blob.core.windows.net` download location is now historical — that location was decommissioned and current builds should only ever reference the `github.com/microsoft/IIS.ServiceMonitor/releases` location shown here.

#### Pattern B: PowerShell Event-Driven Loop (Homelab/Custom Services)

If ServiceMonitor is not an option, use `Register-ObjectEvent` for event-driven monitoring instead of polling. This drops idle CPU to zero and triggers an exit the moment the service status changes — not after waiting for the next poll interval.

```dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2022

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

COPY myservice/ C:\myservice\

RUN New-Service `
      -Name "MyService" `
      -BinaryPathName "C:\myservice\myservice.exe" `
      -StartupType Automatic

RUN sc.exe failure MyService reset= 60 actions= restart/5000/restart/10000/restart/30000

# Write the monitor script — avoids CMD array escaping issues
RUN Set-Content -Path C:\monitor.ps1 -Value @' `
Start-Service MyService `
$svc = Get-Service MyService `
Register-ObjectEvent -InputObject $svc -EventName StatusChanged -SourceIdentifier SvcMonitor -Action { `
    if ($Event.SourceEventArgs.Status -ne [System.ServiceProcess.ServiceControllerStatus]::Running) { `
        Stop-Process -Id $PID -Force `
    } `
} | Out-Null `
while ($true) { Start-Sleep 3600 } `
'@

ENTRYPOINT ["powershell", "-ExecutionPolicy", "Bypass", "-File", "C:\\monitor.ps1"]
```

> ⚠️ **Why a script file instead of inline CMD array:** A CMD JSON array like `["powershell", "-Command", "while ($true) {...}"]` gets awkward fast with nested quotes, braces, and backslash escaping across multiple array elements. Writing to a `.ps1` file in a RUN step and referencing it in ENTRYPOINT is cleaner, easier to read, and avoids all the escaping complexity.

### Using sc.exe for Service Management in Dockerfiles

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Create service via sc.exe
RUN sc.exe create MyService binPath= "C:\myservice\myservice.exe" start= auto

# Set service description
RUN sc.exe description MyService "My Application Service"

# Configure service dependencies
RUN sc.exe config MyService depend= Tcpip/Afd

# Query service status (useful for debugging build)
RUN sc.exe query MyService

# Delete a service
RUN sc.exe delete MyService
```

### IIS — The Most Common Windows Service in Containers

```dockerfile
FROM mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# The IIS base image already has IIS configured and running
# Remove default site content
RUN Remove-Item -Path C:\inetpub\wwwroot\* -Recurse -Force

# Copy your web app
COPY ./webapp C:\inetpub\wwwroot\

# Configure IIS application pool (via WebAdministration module)
RUN Import-Module WebAdministration ; \
    Set-ItemProperty "IIS:\AppPools\DefaultAppPool" -Name "managedRuntimeVersion" -Value "v4.0" ; \
    Set-ItemProperty "IIS:\AppPools\DefaultAppPool" -Name "enable32BitAppOnWin64" -Value $false

# The IIS base image CMD already starts IIS and keeps the container alive
# (via ServiceMonitor.exe watching the W3SVC service)
```

> ⚠️ **Note on browsing the IIS site from the container host:** Microsoft's own IIS image documentation notes that `http://localhost` does not reach the container from the host due to a known WinNAT behavior — this is the same underlying issue covered in Section 11's HNS port-binding gotcha. Use the container's IP address (`docker inspect`) to browse from the host, or rely on the mapped port from another machine on the network.

---

## 10. Multi-Stage Builds for Windows

Multi-stage builds let you use a large SDK image to compile your application, then copy only the compiled output into a small runtime image. This is critical for Windows — the SDK images are enormous (6GB+), but runtime images can be ~300MB.

### .NET Application Multi-Stage Build

```dockerfile
# ───────────────────────────── Stage 1: Build ─────────────────────────────
FROM mcr.microsoft.com/dotnet/sdk:8.0-windowsservercore-ltsc2022 AS build-env

WORKDIR C:\src

# Copy csproj and restore dependencies first (layer cache optimization)
COPY *.csproj .
RUN dotnet restore

# Copy everything else and build
COPY . .
RUN dotnet publish -c Release -o C:\app\publish --no-restore

# ──────────────────────────── Stage 2: Runtime ────────────────────────────
FROM mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022 AS runtime

WORKDIR C:\app

# Copy only the published output from the build stage
COPY --from=build-env C:\app\publish .

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV DOTNET_RUNNING_IN_CONTAINER=true

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Go Application Multi-Stage Build (Tiny Final Image)

Go compiles to a single self-contained binary — perfect for nanoserver:

```dockerfile
# ───────────────────────────── Stage 1: Build ─────────────────────────────
FROM golang:1.22-windowsservercore-ltsc2022 AS build-env

WORKDIR C:\src
COPY . .

# Build a Windows binary
RUN go build -ldflags="-s -w" -o C:\app\myapp.exe .

# ──────────────────────────── Stage 2: Runtime ────────────────────────────
FROM mcr.microsoft.com/windows/nanoserver:ltsc2022 AS runtime

WORKDIR C:\app
COPY --from=build-env C:\app\myapp.exe .

EXPOSE 8080
CMD ["myapp.exe"]
```

### Multi-Stage with a Setup Stage

Sometimes you need a separate stage just to download and prepare dependencies:

```dockerfile
# ──────────────────────── Stage 1: Download dependencies ──────────────────
FROM mcr.microsoft.com/windows/servercore:ltsc2022 AS downloader

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

RUN Invoke-WebRequest -Uri "https://example.com/dep1.zip" -OutFile C:\downloads\dep1.zip ; \
    Expand-Archive C:\downloads\dep1.zip -DestinationPath C:\deps\dep1\ -Force

# ───────────────────────────── Stage 2: Build ──────────────────────────────
FROM mcr.microsoft.com/dotnet/sdk:8.0-windowsservercore-ltsc2022 AS builder

COPY --from=downloader C:\deps\ C:\deps\
COPY . C:\src\
WORKDIR C:\src
RUN dotnet publish -c Release -o C:\publish

# ──────────────────────────── Stage 3: Runtime ─────────────────────────────
FROM mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022

COPY --from=builder C:\publish\ C:\app\
WORKDIR C:\app
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

---

## 11. Networking and Port Exposure in Windows Containers

### EXPOSE in Windows Dockerfiles

`EXPOSE` is purely documentation — it does not open firewall rules or publish ports. Windows containers use HNS (Host Network Service) for actual port mapping.

```dockerfile
EXPOSE 80
EXPOSE 443
EXPOSE 8080/tcp
EXPOSE 9090/udp
```

### Port Mapping at Runtime

```powershell
# Map host port 8080 to container port 80
docker run -d -p 8080:80 mywinapp:latest

# Map multiple ports
docker run -d -p 80:80 -p 443:443 mywinapp:latest

# Map to a specific host IP
docker run -d -p 192.168.1.100:80:80 mywinapp:latest

# Map all exposed ports to random host ports
docker run -d -P mywinapp:latest

# Check port mappings
docker port <container-name>
```

### Windows Container Network Modes

```powershell
# Default NAT network (most common)
docker run --network nat mywinapp:latest

# Host networking (container uses host's network stack directly)
# Note: Windows host networking is more limited than Linux
docker run --network host mywinapp:latest

# No networking
docker run --network none mywinapp:latest

# Custom named network
docker network create --driver nat mynet
docker run --network mynet mywinapp:latest
```

### The HNS Port-Binding Trap — Bind to 0.0.0.0, Not localhost

> ⚠️ **Critical runtime gotcha:** Inside a Windows container, services that bind only to `127.0.0.1` or `localhost` will **not** accept traffic coming in through the HNS-mapped host port. The port mapping appears correct (`-p 80:80`), the container runs without error, but external connections are silently refused.
>
> This is a long-standing, well-documented behavior — Microsoft's own IIS container image documentation notes the same WinNAT limitation, and it remains a live, frequently-hit issue years after Windows containers first shipped. It is not a bug specific to your setup; it is how Windows container networking currently works.

The fix is always to configure your application to bind to `0.0.0.0` (all interfaces) so HNS can route external ingress to it:

```dockerfile
# .NET / ASP.NET Core — correct binding for containers
ENV ASPNETCORE_URLS=http://+:8080      # '+' is the wildcard — binds all interfaces

# Wrong — will silently refuse external connections through port mapping
ENV ASPNETCORE_URLS=http://localhost:8080
```

For non-.NET apps, ensure the application config uses `0.0.0.0` as the listen address:

```powershell
# Verify what a running container is actually binding to
docker exec -it mycontainer powershell
netstat -an | findstr LISTENING
# Look for 0.0.0.0:<port> — that is correct
# Lines showing 127.0.0.1:<port> only are the problem
```

This is a Windows networking stack difference from Linux. On Linux, `localhost` binding inside a container can often still be reached via port mapping because of how the Linux network namespace and iptables NAT work together. On Windows, HNS uses a different path and `127.0.0.1` inside the container stays inside the container's loopback — it never surfaces to the mapped host port.

### Checking Network Config Inside a Windows Container

```powershell
# Inside a running container (via docker exec):
docker exec -it mycontainer powershell

# Inside container PowerShell:
ipconfig                                    # Container's IP address
Get-NetIPAddress                            # PowerShell equivalent
netstat -an                                 # Active connections and listening ports
Test-NetConnection -ComputerName google.com -Port 80   # Connectivity test
Resolve-DnsName google.com                  # DNS resolution test
```

---

## 12. Volume and Data Persistence Patterns

### Named Volumes

```dockerfile
# Declare a volume mount point in the image
VOLUME C:\app\data
VOLUME C:\app\logs
```

```powershell
# Docker creates and manages the volume
docker run -v appdata:C:\app\data mywinapp:latest

# Inspect volume location on host
docker volume inspect appdata
```

### Bind Mounts (Host Path → Container Path)

```powershell
# Mount a host directory into the container
docker run -v C:\HostData:C:\app\data mywinapp:latest

# Mount as read-only
docker run -v C:\HostConfig:C:\app\config:ro mywinapp:latest
```

> ⚠️ **Windows bind mount path format:** Host path must be a Windows path (`C:\HostData`). Container path must be a Windows path (`C:\app\data`). Linux-style paths do not work for Windows containers.

> ⚠️ **Bind mount ACL inheritance — `icacls` in the Dockerfile does NOT apply to bind-mounted host directories.** `icacls` permissions set during `docker build` are baked into the image layer and apply only to directories that are part of the image itself. At runtime, a bind-mounted host directory (`-v C:\HostData:C:\app\data`) brings its own Windows ACL from the host filesystem. The container image's permission settings have no effect on it.
>
> **The consequence:** If your Dockerfile ends with `USER ContainerUser` (non-admin), and the host folder `C:\HostData` was created by an Administrator without granting access to lower-privilege accounts, the container process will throw `Access Denied` when it tries to read or write the mounted directory — even though the image permissions look correct.
>
> **The fix:** On the host machine, before running the container, grant the appropriate access to the folder:
> ```powershell
> # On the HOST — grant Authenticated Users read/write on the folder to be bind-mounted
> icacls "C:\HostData" /grant "Authenticated Users:(OI)(CI)M" /T
>
> # Or grant Everyone full control (less secure, fine for homelab)
> icacls "C:\HostData" /grant "Everyone:(OI)(CI)F" /T
> ```
> This is fundamentally different from Linux, where bind-mounted directories respect UID/GID mappings that can be coordinated between host and container. Windows uses SIDs and ACLs which are evaluated at the host level independently of the container's user context.

### Persistent Data Patterns in Kubernetes

```yaml
# Windows pod with hostPath volume
spec:
  nodeSelector:
    kubernetes.io/os: windows
  containers:
    - name: mywinapp
      image: myregistry/mywinapp:ltsc2022
      volumeMounts:
        - mountPath: C:\app\data      # Windows path format — mandatory
          name: app-data
  volumes:
    - name: app-data
      hostPath:
        path: C:\k8s-storage\mywinapp
        type: DirectoryOrCreate
```

---

## 13. User and Permission Management

### Default Users in Windows Containers

| Base Image | Default User | Privilege Level |
|-----------|-------------|----------------|
| `nanoserver` | `ContainerUser` | Non-admin |
| `servercore` | `ContainerAdministrator` | Local Administrator |
| `server` | `ContainerAdministrator` | Local Administrator |

### USER Instruction

```dockerfile
# Switch to a non-admin user for runtime
USER ContainerUser

# Switch back to admin for install steps
USER ContainerAdministrator
RUN choco install -y git
USER ContainerUser
```

### Creating Custom Users

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Create a local user
RUN net user apprunner "P@ssw0rd123!" /add /y

# Add to a group
RUN net localgroup "Users" apprunner /add

# Create user with no password expiry (for service accounts)
RUN net user svcaccount "P@ssw0rd123!" /add /passwordchg:no /expires:never

# Switch to the new user
USER apprunner
```

### File System Permissions (ACLs)

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Grant full control to a user on a directory
RUN icacls C:\app\data /grant "ContainerUser:(OI)(CI)F" /T

# Grant read-only
RUN icacls C:\app\config /grant "ContainerUser:(OI)(CI)R" /T

# Remove inherited permissions and set explicit
RUN icacls C:\app\secrets /inheritance:r /grant "ContainerAdministrator:(OI)(CI)F" /T

# Common icacls permission flags:
# F  = Full control
# M  = Modify
# RX = Read & Execute
# R  = Read only
# W  = Write only
# (OI) = Object Inherit (files in directory inherit this ACE)
# (CI) = Container Inherit (subdirectories inherit this ACE)
```

---

## 14. Real-World Application Patterns

### Pattern 1: ASP.NET Core Web API (Modern, Nanoserver)

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-windowsservercore-ltsc2022 AS build
WORKDIR C:\src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o C:\publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022
WORKDIR C:\app
COPY --from=build C:\publish .
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

### Pattern 2: .NET Framework App (Legacy, Servercore)

```dockerfile
FROM mcr.microsoft.com/dotnet/framework/sdk:4.8-windowsservercore-ltsc2022 AS build
WORKDIR C:\src
COPY . .
RUN msbuild MyApp.sln /p:Configuration=Release /p:OutputPath=C:\publish

FROM mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022
WORKDIR C:\inetpub\wwwroot
COPY --from=build C:\publish .

# Configure IIS app pool for .NET 4.8
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]
RUN Import-Module WebAdministration ; \
    Set-ItemProperty "IIS:\AppPools\DefaultAppPool" managedRuntimeVersion v4.0
```

### Pattern 3: PowerShell Script Runner (Automation Container)

```dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2022

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Install required PowerShell modules
RUN Install-PackageProvider -Name NuGet -Force ; \
    Install-Module -Name SqlServer -Force ; \
    Install-Module -Name Az -Force -AllowClobber

# Copy automation scripts
COPY scripts\ C:\scripts\

# Set working directory
WORKDIR C:\scripts

# Default — run the main script
CMD ["powershell", "-ExecutionPolicy", "Bypass", "-File", "C:\\scripts\\main.ps1"]
```

### Pattern 4: Windows Console Application (Go Binary)

```dockerfile
FROM golang:1.22-windowsservercore-ltsc2022 AS build
WORKDIR C:\src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -ldflags="-s -w" -o C:\bin\app.exe .

FROM mcr.microsoft.com/windows/nanoserver:ltsc2022
COPY --from=build C:\bin\app.exe C:\app\app.exe
WORKDIR C:\app
ENV APP_ENV=production
CMD ["app.exe"]
```

### Pattern 5: SQL Server with Pre-loaded Schema

> ⚠️ **Microsoft's official SQL Server container images (`mcr.microsoft.com/mssql/server`) are Linux-based — not Windows containers.** Run them in **Linux container mode** (the default for Docker Desktop). Do not use these images in Windows container mode; they will fail to pull with an OS mismatch error. If you genuinely need SQL Server inside a Windows container, you would install SQL Server Express silently into a `servercore` base image, which results in a very large image and significant complexity — for most use cases the Linux image is the right choice.

```dockerfile
# Linux container — ensure Docker is in Linux mode (docker info | findstr "OSType" → linux)
FROM mcr.microsoft.com/mssql/server:2022-latest

ENV ACCEPT_EULA=Y
ENV SA_PASSWORD=YourStrongPassword123!
ENV MSSQL_PID=Express

COPY schema.sql /setup/schema.sql
COPY init.sh /setup/init.sh

RUN chmod +x /setup/init.sh
CMD ["/setup/init.sh"]
```

```bash
#!/bin/bash
# init.sh — starts SQL Server, waits for readiness, applies schema
set -e

/opt/mssql/bin/sqlservr &

echo "Waiting for SQL Server to start..."
for i in {1..30}; do
    /opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P "$SA_PASSWORD" \
        -C -Q "SELECT 1" > /dev/null 2>&1 && break
    sleep 2
done

/opt/mssql-tools18/bin/sqlcmd -S localhost -U SA -P "$SA_PASSWORD" \
    -C -i /setup/schema.sql
echo "Schema applied."

wait
```

---

## 15. Layer Optimization and Image Size

### Why Windows Layers Are Enormous

Windows container layers are much larger than Linux layers because:
- The Windows base images contain hundreds of DLLs that cannot be stripped
- The Windows Registry snapshot is included in each layer
- Windows filesystem metadata (ACLs, alternate data streams) adds overhead
- The windowsfilter storage driver is less efficient than OverlayFS

### Optimization Rules

**Rule 1: Combine all related RUN instructions into one**

```dockerfile
# BAD — creates 4 separate layers, each capturing full filesystem delta
RUN mkdir C:\app
RUN mkdir C:\app\logs
RUN mkdir C:\app\config
RUN mkdir C:\app\data

# GOOD — one layer
RUN New-Item -ItemType Directory -Path C:\app\logs, C:\app\config, C:\app\data -Force
```

**Rule 2: Delete downloads in the same RUN that created them**

```dockerfile
# BAD — the zip file exists in layer 1 forever even after layer 2 deletes it
RUN Invoke-WebRequest -Uri "https://example.com/app.zip" -OutFile C:\Temp\app.zip
RUN Expand-Archive C:\Temp\app.zip -DestinationPath C:\app\ -Force
RUN Remove-Item C:\Temp\app.zip -Force

# GOOD — zip never persists in any layer
RUN Invoke-WebRequest -Uri "https://example.com/app.zip" -OutFile C:\Temp\app.zip ; \
    Expand-Archive C:\Temp\app.zip -DestinationPath C:\app\ -Force ; \
    Remove-Item C:\Temp\app.zip -Force
```

**Rule 3: Use multi-stage builds**

Never ship the SDK or build tools in your final runtime image (see Section 10).

**Rule 4: Leverage layer caching — COPY dependencies before source**

```dockerfile
# Dependencies change rarely — cache them in an early layer
COPY *.csproj .
RUN dotnet restore

# Source changes frequently — copy last
COPY . .
RUN dotnet build
```

**Rule 5: Clean up Windows temp files and package caches**

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

RUN choco install -y git --no-progress ; \
    Remove-Item "C:\ProgramData\chocolatey\logs\*" -Force -ErrorAction SilentlyContinue ; \
    Remove-Item "C:\Windows\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue ; \
    Remove-Item "C:\Users\ContainerAdministrator\AppData\Local\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue
```

### Checking Image Size and Layer Breakdown

```powershell
# Check image size
docker images mywinapp

# Inspect layer history and sizes
docker history mywinapp:latest

# Detailed image inspection
docker inspect mywinapp:latest
```

---

## 16. Building and Tagging Windows Images

### Basic Build

```powershell
# Build from current directory Dockerfile
docker build -t mywinapp:1.0 .

# Build from specific Dockerfile
docker build -t mywinapp:1.0 -f Dockerfile.windows .

# Build with build args
docker build `
  --build-arg APP_VERSION=1.0.0 `
  --build-arg BUILD_ENV=production `
  -t mywinapp:1.0 .

# Build with no cache (forces full rebuild)
docker build --no-cache -t mywinapp:1.0 .

# Build and show verbose output
docker build --progress=plain -t mywinapp:1.0 .
```

### Tagging Strategy

```powershell
# Tag with OS version for clarity in hybrid environments
docker tag mywinapp:1.0 mywinapp:1.0-ltsc2022
docker tag mywinapp:1.0 mywinapp:latest-windows

# Tag for a private registry
docker tag mywinapp:1.0 myregistry.example.com:5000/myorg/mywinapp:1.0-ltsc2022

# List all tags
docker images mywinapp
```

---

## 17. Pushing to a Registry

### Pushing to a Self-Hosted Registry (e.g. Gitea, Harbor, generic OCI registry)

```powershell
# Log in to your registry
docker login myregistry.example.com:5000 -u myuser -p <token>

# Tag the image for your registry
docker tag mywinapp:1.0-ltsc2022 myregistry.example.com:5000/myuser/mywinapp:1.0-ltsc2022

# Push
docker push myregistry.example.com:5000/myuser/mywinapp:1.0-ltsc2022

# Pull (on another machine)
docker pull myregistry.example.com:5000/myuser/mywinapp:1.0-ltsc2022
```

### Pushing to Docker Hub

```powershell
docker login -u <dockerhub-username>
docker tag mywinapp:1.0-ltsc2022 <dockerhub-username>/mywinapp:1.0-ltsc2022
docker push <dockerhub-username>/mywinapp:1.0-ltsc2022
```

---

## 18. Multi-Platform Image Manifests

A manifest list (also called a multi-platform image) is a single image tag that points to different actual images depending on the OS and architecture of the machine pulling it. This is how `nginx:latest` works — Linux machines get the Linux image, Windows machines (in theory) get the Windows image.

### Building Multi-Platform Manifests

```powershell
# Build the Linux image (on a Linux machine or WSL2 in Linux mode)
docker build -t myregistry/myapp:1.0-linux-amd64 -f Dockerfile.linux .
docker push myregistry/myapp:1.0-linux-amd64

# Build the Windows image (on Windows in Windows container mode)
docker build -t myregistry/myapp:1.0-windows-ltsc2022 -f Dockerfile.windows .
docker push myregistry/myapp:1.0-windows-ltsc2022

# Create and push the manifest list (combine both under one tag)
docker manifest create myregistry/myapp:1.0 `
  myregistry/myapp:1.0-linux-amd64 `
  myregistry/myapp:1.0-windows-ltsc2022

docker manifest push myregistry/myapp:1.0
```

> ⚠️ **Most open-source software does not publish Windows container variants.** If you need a Windows version of a Linux-only image, you must build it yourself from a Windows base image. There is no automatic cross-compilation — the application must be built for Windows.

---

## 19. Troubleshooting Windows Build Failures

### Build Fails Immediately — Wrong Docker Mode

```
Error: no matching manifest for windows/amd64 in the manifest list entries
```

**Cause:** You are in Linux container mode trying to pull a Windows-only image, or vice versa.

```powershell
docker info | findstr "OSType"
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon
```

### RUN Command Fails Silently

```
Step 4/10 : RUN some-command
 ---> Running in abc123def456
 ---> 8f2a1b3c4d5e
```

No error shown but the result is wrong. **Cause:** PowerShell errors not set to stop.

```dockerfile
# Fix — always include ErrorActionPreference
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]
```

### Layer Cache Causing Stale Builds

```powershell
# Force full rebuild, ignoring cache
docker build --no-cache -t mywinapp:latest .
```

### Access Denied During File Operations

```
Access to the path 'C:\Program Files\...' is denied.
```

**Cause:** Running as `ContainerUser` (non-admin) and trying to write to a protected path.

```dockerfile
# Ensure admin user for installation steps
USER ContainerAdministrator
RUN # install operations here
USER ContainerUser
```

### Invoke-WebRequest Fails or Hangs

```powershell
# Fix 1: Use correct TLS settings
RUN [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12 ; \
    Invoke-WebRequest -Uri "https://example.com/file.zip" -OutFile "C:\Temp\file.zip" -UseBasicParsing

# Fix 2: Suppress progress (prevents hanging in non-interactive contexts)
RUN $ProgressPreference = 'SilentlyContinue' ; \
    Invoke-WebRequest -Uri "https://example.com/file.zip" -OutFile "C:\Temp\file.zip"

# Fix 3: Use curl.exe (available in servercore ltsc2022+)
RUN curl.exe -L -o C:\Temp\file.zip https://example.com/file.zip
```

### Container Exits Immediately

**Cause:** The main process (CMD/ENTRYPOINT) exits. Windows containers stop when PID 1 exits.

```dockerfile
# Bad — cmd.exe exits immediately after running the command
CMD ["cmd", "/c", "myservice.exe"]

# Good — exec form, myservice.exe becomes PID 1 directly
CMD ["myservice.exe"]

# If you need a keep-alive loop:
CMD ["powershell", "-Command", "Start-Service MyService; while ($true) { Start-Sleep 30 }"]
```

### Version Mismatch Error at Runtime

```
Error: container operating system does not match host operating system
```

**Cause:** Process isolation with mismatched OS versions.

```powershell
# Fix 1: Use Hyper-V isolation
docker run --isolation=hyperv mywinapp:ltsc2019

# Fix 2: Rebuild image with matching base image version
# Host is Windows Server 2022 → use ltsc2022 base image
# Host is Windows Server 2025 → use ltsc2025 base image
```

### MSI Install Hangs During Build

```dockerfile
# Bad — msiexec may open a GUI dialog that hangs the build
RUN msiexec /i app.msi

# Good — silent install with no restart
RUN Start-Process msiexec.exe -ArgumentList "/i", "C:\Temp\app.msi", "/qn", "/norestart" -Wait -NoNewWindow
```

---

## 20. Homelab Practice Tasks

### Task 1: Build Your First Windows Container Image

```dockerfile
# Save as Dockerfile in an empty directory
FROM mcr.microsoft.com/windows/nanoserver:ltsc2022

# Set a label
LABEL maintainer="you@example.com"
LABEL version="1.0"

# Create a directory and write a file
RUN mkdir C:\app
COPY hello.txt C:\app\hello.txt

# Default command — print the file content
CMD ["cmd", "/c", "type C:\\app\\hello.txt"]
```

```powershell
# hello.txt content: "Hello from a Windows Container!"

# Switch to Windows mode
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon

# Build
docker build -t hello-windows:1.0 .

# Run
docker run --rm hello-windows:1.0

# Inspect layers
docker history hello-windows:1.0
```

### Task 2: Registry Manipulation in a Build

```dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2022

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# Write app config to Registry
RUN New-Item -Path "HKLM:\SOFTWARE\MyHomelab" -Force ; \
    Set-ItemProperty -Path "HKLM:\SOFTWARE\MyHomelab" -Name "Environment" -Value "homelab" ; \
    Set-ItemProperty -Path "HKLM:\SOFTWARE\MyHomelab" -Name "Version" -Value "1.0.0" ; \
    Set-ItemProperty -Path "HKLM:\SOFTWARE\MyHomelab" -Name "NodeCount" -Value 6 -Type DWord

# Verify the values were written
RUN $props = Get-ItemProperty "HKLM:\SOFTWARE\MyHomelab" ; \
    Write-Host "Environment: $($props.Environment)" ; \
    Write-Host "Version: $($props.Version)" ; \
    Write-Host "NodeCount: $($props.NodeCount)"

CMD ["powershell", "-Command", "Get-ItemProperty HKLM:\\SOFTWARE\\MyHomelab"]
```

```powershell
docker build -t registry-demo:1.0 .
docker run --rm registry-demo:1.0
```

### Task 3: Multi-Stage Build — Size Comparison

```dockerfile
# Save as Dockerfile.singlestage
FROM mcr.microsoft.com/dotnet/sdk:8.0-windowsservercore-ltsc2022
WORKDIR C:\app
COPY . .
RUN dotnet publish -c Release -o C:\publish
CMD ["dotnet", "C:\\publish\\MyApp.dll"]
```

```dockerfile
# Save as Dockerfile.multistage
FROM mcr.microsoft.com/dotnet/sdk:8.0-windowsservercore-ltsc2022 AS build
WORKDIR C:\src
COPY . .
RUN dotnet publish -c Release -o C:\publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022
COPY --from=build C:\publish C:\app
CMD ["dotnet", "C:\\app\\MyApp.dll"]
```

```powershell
# Build both
docker build -f Dockerfile.singlestage -t myapp:singlestage .
docker build -f Dockerfile.multistage -t myapp:multistage .

# Compare sizes
docker images myapp
# The difference will be several GB
```

### Task 4: Push a Windows Image to a Self-Hosted Registry

```powershell
# Tag for your registry
docker tag hello-windows:1.0 myregistry.example.com:5000/myuser/hello-windows:1.0-ltsc2022

# Log in
docker login myregistry.example.com:5000

# Push
docker push myregistry.example.com:5000/myuser/hello-windows:1.0-ltsc2022

# Pull on another machine to verify
docker pull myregistry.example.com:5000/myuser/hello-windows:1.0-ltsc2022
docker run --rm myregistry.example.com:5000/myuser/hello-windows:1.0-ltsc2022
```

### Task 5: Deploy a Windows Container Image to Kubernetes

```yaml
# Save as windows-test-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-windows
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-windows
  template:
    metadata:
      labels:
        app: hello-windows
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
        - key: "os"
          operator: "Equal"
          value: "windows"
          effect: "NoSchedule"
      containers:
        - name: hello-windows
          image: myregistry.example.com:5000/myuser/hello-windows:1.0-ltsc2022
          resources:
            limits:
              memory: "256Mi"
              cpu: "500m"
```

```bash
# Apply from Linux master
kubectl apply -f windows-test-deployment.yaml
kubectl get pods -o wide
kubectl describe pod <pod-name>
```

---

## Quick Reference

### Base Image Decision Tree

```
Does your app need .NET?
  ├── Yes, .NET 6+  → mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022
  ├── Yes, .NET Framework  → mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022
  └── No .NET
        ├── Needs PowerShell at runtime?
        │     └── Yes  → mcr.microsoft.com/windows/servercore:ltsc2022
        └── No PowerShell needed, self-contained binary?
              └── Yes  → mcr.microsoft.com/windows/nanoserver:ltsc2022
```

> Substitute `ltsc2025` throughout if your host is running Windows Server 2025 — see Section 2's note on matching host and container OS versions.

### SHELL Declaration (Always Include This)

```dockerfile
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]
```

### Common PowerShell Dockerfile Operations

| Task | PowerShell Command |
|------|--------------------|
| Create directory tree | `New-Item -ItemType Directory -Path C:\app\logs, C:\app\config -Force` |
| Download file | `Invoke-WebRequest -Uri $url -OutFile C:\Temp\file.zip -UseBasicParsing` |
| Extract zip | `Expand-Archive -Path C:\Temp\file.zip -DestinationPath C:\app\ -Force` |
| Delete file/dir | `Remove-Item -Path C:\Temp\* -Recurse -Force -ErrorAction SilentlyContinue` |
| Write Registry string | `Set-ItemProperty -Path "HKLM:\SOFTWARE\App" -Name "Key" -Value "Value"` |
| Write Registry DWORD | `Set-ItemProperty -Path "HKLM:\SOFTWARE\App" -Name "Key" -Value 1 -Type DWord` |
| Create Registry key | `New-Item -Path "HKLM:\SOFTWARE\App" -Force` |
| Import .reg file | `reg import C:\setup\config.reg` |
| Add to PATH | `[Environment]::SetEnvironmentVariable('PATH', $env:PATH + ';C:\app\bin', 'Machine')` |
| Register Windows service | `New-Service -Name "MySvc" -BinaryPathName "C:\app\svc.exe" -StartupType Automatic` |
| Install Chocolatey package | `choco install -y git --no-progress` |
| Test file exists | `if (-not (Test-Path C:\app\myapp.exe)) { throw "not found" }` |
| Set file permissions | `icacls C:\app\data /grant "ContainerUser:(OI)(CI)F" /T` |

### Build Commands

| Goal | Command |
|------|---------|
| Build image | `docker build -t myapp:1.0 .` |
| Build no cache | `docker build --no-cache -t myapp:1.0 .` |
| Build with ARG | `docker build --build-arg VER=1.0 -t myapp:1.0 .` |
| Check image size | `docker images myapp` |
| Inspect layers | `docker history myapp:1.0` |
| Check daemon mode | `docker info \| findstr "OSType"` |
| Switch daemon mode | `& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon` |
| Push to registry | `docker push registry/myapp:1.0` |
| Tag image | `docker tag myapp:1.0 registry/myapp:1.0-ltsc2022` |

### Key Differences Summary — Linux vs Windows Dockerfile

| Element | Linux | Windows |
|---------|-------|---------|
| Default shell | `/bin/sh` | `cmd.exe` |
| Recommended shell | `bash` | `powershell` |
| Path separator | `/` | `\` (use `\\` in JSON) |
| Package manager | `apt-get`, `apk` | Chocolatey, direct download |
| Temp dir | `/tmp` | `C:\Windows\Temp` |
| Config location | `/etc/` | Registry or `C:\app\` |
| Service manager | `systemd` / `supervisord` | `sc.exe` / `New-Service` |
| Default user | `root` | `ContainerAdministrator` |
| Base image size | 5–50MB (alpine) | 260MB–5GB |
| Layer storage | OverlayFS | windowsfilter |

---

*File: `windows-dockerfile-patterns.md`*
*See also: `windows-linux-containers-vms-complete.md` | `kubernetes-hybrid-architecture.md`*
