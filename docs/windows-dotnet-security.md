# Windows and .NET Pipeline Security Guide

The Techstream DevSecOps Framework primarily uses examples from Linux, container, and JVM/Node.js ecosystems. This guide fills the gap for organizations building .NET applications on Windows-based CI/CD infrastructure. The core DevSecOps principles are identical, but the specific tooling, configuration, and pipeline patterns differ.

This guide covers security scanning for .NET applications, Windows-based build agent hardening, NuGet dependency security, and Windows-specific secrets management.

---

## .NET Application Security Scanning

### SAST — Static Application Security Testing for .NET

**Primary tools:**

| Tool | Type | Strengths | Integration |
|------|------|-----------|-------------|
| **Semgrep** | SAST | Supports C# rules; cross-platform; open source | GitHub Actions, GitLab CI, Azure Pipelines |
| **Security Code Scan** | SAST | .NET-specific; detects injection, XSS, CSRF, SSRF | MSBuild analyzer; Roslyn-based |
| **Roslyn Security Guard** | SAST (Roslyn analyzer) | Runs during build; developer IDE feedback | Added as NuGet package to .csproj |
| **SonarQube / SonarCloud** | SAST + quality | Deep C# analysis; tracks trends | MSBuild sonar scanner |
| **Veracode / Checkmarx** | Commercial SAST | Comprehensive .NET coverage | API-based; integrates with Azure DevOps |

**Security Code Scan as a Roslyn analyzer (runs at build time):**

```xml
<!-- Add to your .csproj — security issues become build warnings or errors -->
<ItemGroup>
  <PackageReference Include="SecurityCodeScan.VS2019" Version="5.6.7">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

```xml
<!-- Directory.Build.props — escalate security warnings to errors for CI enforcement -->
<PropertyGroup Condition="'$(CI)' == 'true'">
  <TreatWarningsAsErrors>false</TreatWarningsAsErrors>
  <!-- Only escalate security analyzer codes, not all warnings -->
  <WarningsAsErrors>SCS0001;SCS0005;SCS0007;SCS0018;SCS0026;SCS0027</WarningsAsErrors>
</PropertyGroup>
```

**Semgrep for C# (Azure Pipelines):**

```yaml
- task: CmdLine@2
  displayName: 'SAST — Semgrep C# scan'
  inputs:
    script: |
      pip install semgrep
      semgrep \
        --config "p/csharp" \
        --config "p/owasp-top-ten" \
        --sarif \
        --output semgrep.sarif \
        --error \
        .
  continueOnError: false

- task: PublishBuildArtifacts@1
  displayName: 'Archive SAST report'
  condition: always()
  inputs:
    pathToPublish: 'semgrep.sarif'
    artifactName: 'sast-results'
```

---

### SCA — NuGet Dependency Security Scanning

**NuGet ecosystem-specific considerations:**

- NuGet packages are published to nuget.org; there is no equivalent to npm's `npm audit` built-in
- .NET 8+ includes `dotnet list package --vulnerable` for basic CVE checking
- Trivy supports NuGet scanning via filesystem scan mode

**Built-in .NET vulnerability check:**

```bash
# List packages with known vulnerabilities (requires .NET 8+)
dotnet list package --vulnerable --include-transitive

# Output example:
# The following sources were used:
#    https://api.nuget.org/v3/index.json
#
# Project `MyApp` has the following vulnerable packages
#   [net8.0]:
#   Top-level Package      Requested   Resolved   Severity   Advisory URL
#   > System.Net.Http      4.3.0       4.3.0      High       https://github.com/...
```

**Trivy NuGet scanning in Azure Pipelines:**

```yaml
- script: |
    # Install Trivy
    wget -qO- https://github.com/aquasecurity/trivy/releases/download/v0.58.1/trivy_0.58.1_Linux-64bit.tar.gz \
      | tar xz -C /usr/local/bin trivy

    # Scan NuGet packages (filesystem mode detects packages.lock.json and .csproj)
    trivy fs \
      --format json \
      --output trivy-nuget-results.json \
      --severity CRITICAL,HIGH \
      --ignore-unfixed \
      --exit-code 0 \
      .

    # Gate run
    trivy fs \
      --format table \
      --severity CRITICAL,HIGH \
      --ignore-unfixed \
      --exit-code 1 \
      .
  displayName: 'SCA — Trivy NuGet scan'
  continueOnError: false
  condition: succeeded()
```

**NuGet lockfile enforcement:**

Lockfiles are the .NET equivalent of `package-lock.json` or `Gemfile.lock`. Enable them to ensure reproducible, auditable builds:

```xml
<!-- Enable NuGet lock file in your .csproj or Directory.Build.props -->
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
  <!-- In CI, do not allow updating the lockfile — fail if it's out of sync -->
  <RestoreLockedMode Condition="'$(CI)' == 'true'">true</RestoreLockedMode>
</PropertyGroup>
```

Commit the `packages.lock.json` file to your repository. The `RestoreLockedMode` flag causes `dotnet restore` to fail if the lockfile does not match the resolved packages — protecting against dependency substitution attacks.

---

### SBOM Generation for .NET

```bash
# Generate CycloneDX SBOM using the CycloneDX .NET tool
dotnet tool install --global CycloneDX

# Generate SBOM from solution or project file
dotnet CycloneDX ./MyApp.sln \
  --output ./sbom \
  --filename sbom-myapp.json \
  --json

# Alternatively, use Trivy for SBOM generation (consistent with other ecosystems)
trivy fs \
  --format cyclonedx \
  --output sbom-filesystem.json \
  .
```

---

## Windows Build Agent Security Hardening

Windows-based build agents introduce different attack surfaces than Linux containers. The primary concerns are persistent state, Windows-specific privilege escalation paths, and Active Directory integration.

### Agent Hardening Checklist

**Operating system baseline:**

- [ ] Run agents on Windows Server Core (not Desktop Experience); minimize installed components
- [ ] Apply CIS Benchmark for Windows Server 2022 at Level 1; enforce via Group Policy or DSC
- [ ] Enable Windows Defender with real-time protection; exclude only specific build output directories (not the entire workspace)
- [ ] Enable Windows Event Log forwarding to your SIEM (Security, System, and Application channels minimum)
- [ ] Enable PowerShell Script Block Logging (HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging)
- [ ] Disable SMBv1 (a common lateral movement vector): `Set-SmbServerConfiguration -EnableSMB1Protocol $false`
- [ ] Disable Windows Remote Management (WinRM) if not required; if required, restrict to specific management IP ranges

**Agent account:**

```powershell
# Create a dedicated, limited-privilege service account for the build agent
# Do not run the agent as SYSTEM, LocalService, or an administrator account
New-LocalUser -Name "build-agent" -Description "CI/CD Build Agent - no interactive login"
  -NoPassword -AccountNeverExpires -PasswordNeverExpires

# Assign only required permissions
# The agent needs access to:
# - Its working directory
# - The temp directory
# - Any build toolchain directories
$agentWorkDir = "C:\agent\_work"
$acl = Get-Acl $agentWorkDir
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
  "build-agent", "Modify", "ContainerInherit,ObjectInherit", "None", "Allow"
)
$acl.SetAccessRule($rule)
Set-Acl $agentWorkDir $acl
```

**Ephemeral agents (strongly recommended):**

The most effective agent hardening is to make agents ephemeral. Pre-provision an agent VM image and re-provision after each job. This eliminates persistent state that attackers can exploit between jobs.

```powershell
# Azure DevOps scale sets — ephemeral agents from a managed VM image
# Defined in Azure DevOps: Project Settings → Agent pools → Add pool → Azure virtual machine scale set

# The VM image should be built with Packer:
# 1. Start from Windows Server Core base
# 2. Install only required toolchain (dotnet SDK, Git, NuGet, Docker if needed)
# 3. Apply CIS hardening
# 4. Install Azure Pipelines agent in deprovisioned state
# 5. Capture as a Managed Image
# 6. Azure DevOps provisions agents from this image on demand and destroys after job

# PowerShell DSC to enforce agent VM configuration baseline
Configuration BuildAgentBaseline {
  Node "build-agent-vm" {
    WindowsFeature RemoveInternetExplorer {
      Name   = "Internet-Explorer-Optional-amd64"
      Ensure = "Absent"
    }

    Registry DisableSMBv1 {
      Key       = "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters"
      ValueName = "SMB1"
      ValueData = "0"
      ValueType = "DWord"
      Ensure    = "Present"
    }

    Registry EnablePSScriptBlockLogging {
      Key       = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
      ValueName = "EnableScriptBlockLogging"
      ValueData = "1"
      ValueType = "DWord"
      Ensure    = "Present"
    }
  }
}
```

---

## .NET Secrets Management

### NuGet Feed Authentication

Private NuGet feeds (Azure Artifacts, GitHub Packages, Nexus) require authenticated access. Do not store credentials in `NuGet.Config` files committed to the repository.

**Azure Artifacts with managed identity (Azure Pipelines):**

```yaml
# No credentials required — Azure Pipelines uses the pipeline's managed identity
- task: NuGetAuthenticate@1
  displayName: 'Authenticate to Azure Artifacts'
  # Automatically uses the pipeline's OIDC identity — no stored credentials
```

**GitHub Packages with OIDC:**

```yaml
- name: Authenticate to GitHub Packages
  run: |
    dotnet nuget add source \
      --username ${{ github.actor }} \
      --password ${{ secrets.GITHUB_TOKEN }} \
      --store-password-in-clear-text \
      --name github \
      "https://nuget.pkg.github.com/${{ github.repository_owner }}/index.json"
```

### Application Secrets in .NET

**Development secrets (never commit to source control):**

```bash
# .NET Secret Manager — stores secrets outside the project directory
# Development only; not suitable for staging/production

dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Database" "Server=localhost;Database=app;..."
dotnet user-secrets set "ApiKeys:ThirdParty" "sk-..."

# Secrets are stored in:
# Windows: %APPDATA%\Microsoft\UserSecrets\<UserSecretsId>\secrets.json
# macOS/Linux: ~/.microsoft/usersecrets/<UserSecretsId>/secrets.json
# Never committed to source control
```

**Production secrets — Azure Key Vault integration:**

```csharp
// Program.cs — load secrets from Azure Key Vault at startup
// Uses DefaultAzureCredential: works with managed identity in Azure, developer login locally
using Azure.Identity;
using Microsoft.Extensions.Configuration;

var builder = WebApplication.CreateBuilder(args);

// Only add Key Vault in non-development environments
if (!builder.Environment.IsDevelopment())
{
    var keyVaultName = builder.Configuration["KeyVaultName"];
    var keyVaultUri = new Uri($"https://{keyVaultName}.vault.azure.net/");

    builder.Configuration.AddAzureKeyVault(
        keyVaultUri,
        new DefaultAzureCredential()
    );
}
```

```csharp
// In your code — secrets referenced by name, never hardcoded
public class DatabaseService
{
    private readonly string _connectionString;

    public DatabaseService(IConfiguration configuration)
    {
        // Key Vault secret named "ConnectionStrings--Database" (-- is the hierarchy separator)
        _connectionString = configuration["ConnectionStrings:Database"]
            ?? throw new InvalidOperationException("Database connection string is not configured");
    }
}
```

---

## Windows Containers in .NET Pipelines

When .NET applications must run in Windows containers (required for older .NET Framework applications), apply these additional controls:

### Windows Container Base Image Selection

| Scenario | Recommended Base Image | Avoid |
|----------|----------------------|-------|
| .NET 8+ (modern) | `mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022` | Full Windows Server base |
| .NET Framework 4.8 | `mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022` | `windowsservercore:latest` (unpatched) |
| Console app / service | `mcr.microsoft.com/dotnet/runtime:8.0-nanoserver-ltsc2022` | Full Windows Server |

**Multi-stage build for .NET Windows containers:**

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0-nanoserver-ltsc2022 AS build
WORKDIR /src

# Copy solution and restore (separate from copy to leverage layer caching)
COPY ["MyApp/*.csproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.csproj" --locked-mode

# Copy source and build
COPY . .
RUN dotnet build "MyApp/MyApp.csproj" -c Release --no-restore

# Stage 2: Publish
FROM build AS publish
RUN dotnet publish "MyApp/MyApp.csproj" -c Release -o /app/publish \
    --no-restore \
    --no-build \
    /p:UseAppHost=false

# Stage 3: Runtime — minimal Nano Server image
FROM mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022 AS runtime
WORKDIR /app

# Run as non-privileged ContainerUser (built into Nano Server)
USER ContainerUser

COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Windows Container Vulnerability Scanning

```yaml
# Trivy scans Windows container images — requires Docker daemon with Windows container support
# or Trivy's filesystem mode against the extracted layers

- name: Scan Windows container image (Trivy)
  run: |
    trivy image \
      --format json \
      --output trivy-windows-image.json \
      --severity CRITICAL,HIGH \
      --ignore-unfixed \
      --exit-code 0 \
      ${{ env.IMAGE_NAME }}:${{ github.run_id }}

    # Gate
    trivy image \
      --severity CRITICAL,HIGH \
      --ignore-unfixed \
      --exit-code 1 \
      ${{ env.IMAGE_NAME }}:${{ github.run_id }}
```

---

## Azure DevOps-Specific Security Controls

Organizations using Azure DevOps for .NET pipelines should configure these platform-level controls:

### Pipeline Permissions

```yaml
# azure-pipelines.yml — explicit permission scoping
# Principle of least privilege for pipeline token
resources:
  repositories:
    - repository: self
      type: git

trigger:
  branches:
    include:
      - main

variables:
  - group: app-production-vars  # Only the variables group this pipeline needs

# Explicitly scope what the pipeline's access token can do
# This cannot be set in YAML — configure in Project Settings → Agent pools → Security
# and Project Settings → Pipelines → Settings → Limit job authorization scope
```

**Project-level settings to enforce (Project Settings → Settings):**

- [ ] "Limit job authorization scope to current project for non-release pipelines" — **On**
- [ ] "Limit job authorization scope to referenced Azure DevOps repositories" — **On**
- [ ] "Protect access to repositories in YAML pipelines" — **On**
- [ ] "Disable creation of classic pipelines" — **On** (force YAML-as-code only)

### Secure Files and Variable Groups

```yaml
# Download a secure file (certificate, signing key) from Azure Pipelines Library
- task: DownloadSecureFile@1
  name: signingCert
  inputs:
    secureFile: 'code-signing-certificate.pfx'

- script: |
    # Use the file path from the task output
    # The file is automatically deleted after the job completes
    signtool sign /f "$(signingCert.secureFilePath)" \
      /p "$(CERT_PASSWORD)" \
      /tr "http://timestamp.digicert.com" \
      MyApp.exe
  displayName: 'Sign application binary'
  env:
    CERT_PASSWORD: $(CERT_PASSWORD)  # From variable group; marked as secret
```

---

## Cross-References

| Topic | Document |
|-------|---------|
| Core DevSecOps principles | [Framework](framework.md) |
| CI/CD pipeline security architecture | [Secure CI/CD Reference Architecture](../../secure-ci-cd-reference-architecture/docs/framework.md) |
| Azure Pipelines secure pipeline template | [Azure Pipelines Template](../../secure-pipeline-templates/templates/azure-pipelines-secure-pipeline.yml) |
| Secrets lifecycle management | [Secret Lifecycle Management](secret-lifecycle-management.md) |
| Container security (general) | [Cloud Security DevSecOps](../../cloud-security-devsecops/docs/framework.md) |
| Supply chain security for NuGet | [Software Supply Chain Security Framework](../../software-supply-chain-security-framework/docs/framework.md) |
