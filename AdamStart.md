# Azure Guardrails Solution Accelerator - Documentation

## Executive Summary

The **Azure Guardrails Solution Accelerator** is an enterprise-grade automated compliance monitoring and auditing solution for Microsoft Azure environments. It provides continuous, automated assessment of cloud security controls against 13 standardized security guardrails that align with industry standards (ITSG-33, PBMM) and government compliance frameworks.

### What It Does

This solution automates the evaluation of Azure cloud environments to ensure adherence to security best practices and regulatory requirements. It operates as a "compliance-as-code" platform that:

1. **Continuously Monitors** Azure subscriptions and resources for compliance with 13 security guardrails
2. **Automatically Assesses** over 45 individual compliance checks covering identity, access, data protection, network security, and operational controls
3. **Logs Results** to Azure Log Analytics for analysis, reporting, and audit trails
4. **Provides Multi-Tenant Support** via Azure Lighthouse for managed service providers
5. **Generates Reports** through Azure Workbooks and centralized dashboards

### Key Capabilities

- **Automated Compliance Checks**: Executes scheduled compliance audits without manual intervention
- **13 Security Guardrails**: Comprehensive coverage of identity, access, encryption, network, and operational controls
- **Flexible Deployment**: Single-tenant or multi-tenant (MSP) deployment models
- **Multi-Cloud Profiles**: Selective enable/disable of checks based on cloud usage patterns (6 profiles)
- **Localization Support**: English and French-Canadian language support
- **Break-Glass Account Monitoring**: Special handling for emergency access accounts
- **Performance Telemetry**: Execution time, memory usage, and compliance statistics tracking

### Target Users

- **Government Agencies**: Requiring ITSG-33 and PBMM compliance
- **Enterprise IT Security Teams**: Managing large Azure deployments
- **Managed Service Providers**: Monitoring compliance across multiple customer tenants
- **Cloud Architects**: Enforcing security baselines and governance policies

### Value Proposition

Traditional compliance assessments are manual, error-prone, and time-consuming. This solution provides:

- **Continuous Compliance**: Real-time monitoring vs. periodic manual audits
- **Reduced Risk**: Early detection of non-compliant configurations
- **Audit Readiness**: Automated evidence collection and reporting
- **Cost Efficiency**: Reduces manual compliance effort by 80%+
- **Scalability**: Handles single subscriptions to large multi-tenant environments

---

## Technical Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Azure Tenant                             │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │          Azure Automation Account (MSI)                  │  │
│  │                                                          │  │
│  │  ┌─────────────┐     ┌─────────────┐                   │  │
│  │  │  main.ps1   │────>│ backend.ps1 │                   │  │
│  │  │ Orchestrator│     │  Updates    │                   │  │
│  │  └──────┬──────┘     └─────────────┘                   │  │
│  │         │                                               │  │
│  │         │ Loads modules.json                           │  │
│  │         │ Executes 13 Guardrail Modules                │  │
│  │         v                                               │  │
│  │  ┌─────────────────────────────────────────┐           │  │
│  │  │  Guardrail Check Modules (45+ checks)   │           │  │
│  │  │  - GR1: Protect Identities (6 checks)   │           │  │
│  │  │  - GR2: Manage Access (9 checks)        │           │  │
│  │  │  - GR3: Secure Endpoints (2 checks)     │           │  │
│  │  │  - GR4-13: Additional controls          │           │  │
│  │  └─────────────────────────────────────────┘           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐  │
│  │ Key Vault    │  │ Storage      │  │ Log Analytics       │  │
│  │ - Secrets    │  │ - Modules    │  │ - Compliance Logs   │  │
│  │ - BG Accts   │  │ - Config     │  │ - Exception Logs    │  │
│  │ - Workspace  │  │ - Workbooks  │  │ - Telemetry Data    │  │
│  └──────────────┘  └──────────────┘  └─────────────────────┘  │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Azure Resources Being Monitored                       │    │
│  │  - Users, Groups, Service Principals                   │    │
│  │  - Azure Policies, Role Assignments                    │    │
│  │  - Network (VNets, NSGs, Subnets)                      │    │
│  │  - Storage Accounts, Key Vaults                        │    │
│  │  - App Services, Azure Functions                       │    │
│  │  - Log Analytics Workspaces                            │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Component Architecture

#### 1. **Execution Engine**

**Azure Automation Account**
- Runs PowerShell runbooks on schedule (typically daily)
- Uses Managed Service Identity (MSI) for authentication
- Executes `main.ps1` (primary orchestrator) and `backend.ps1` (maintenance tasks)

**Main Orchestrator (`main.ps1`)**
- **Initialization**: Connects to Azure, loads configuration, retrieves secrets from Key Vault
- **Pre-Caching**: Fetches all user raw data once (performance optimization)
- **Module Loading**: Reads `modules.json` to determine which checks to execute
- **Dynamic Execution**: Each module loaded, configured, and invoked with specific parameters
- **Result Aggregation**: Collects compliance status, errors, and additional data
- **Logging**: Pushes results to Log Analytics workspace
- **Telemetry**: Tracks execution time, memory usage, compliance statistics

**Backend Processor (`backend.ps1`)**
- Updates ITSG control mapping reference data from GitHub
- Checks for solution updates
- Registers Lighthouse resource providers (multi-tenant scenarios)
- Updates tenant information

#### 2. **Compliance Check Modules**

Each of the 13 guardrails contains multiple PowerShell modules (`.psm1` + `.psd1` manifest):

**Module Structure**:
```
/src/GUARDRAIL X <Name>/
    ├── Check-<Function>.psm1        # Check implementation
    ├── Check-<Function>.psd1        # Module manifest
    └── GR-ComplianceChecks-Msgs.psd1 # Localized messages
```

**Module Contract**:
- **Input**: Configuration parameters (via `$vars` object from main.ps1)
- **Processing**: Queries Azure/Graph APIs, evaluates compliance rules
- **Output**: Returns PSCustomObject with:
  - `ComplianceResults`: Array of compliance check results
  - `Errors`: Array of error objects (if any)
  - `AdditionalResults`: Optional extra data (custom logs)

**Compliance Result Schema**:
```powershell
@{
    ControlName = "GUARDRAIL X"
    ItemName = "<Resource/User/Policy Name>"
    ComplianceStatus = $true/$false
    Comments = "<Details or remediation guidance>"
    itsgcode = "<ITSG-33 control ID>"
    ReportTime = "<ISO-8601 timestamp>"
    Required = $true/$false  # Mandatory vs. recommended
}
```

#### 3. **Data Storage Layer**

**Azure Key Vault**
- **Secrets**:
  - `BGA1`, `BGA2`: Break-glass account UPNs
  - `WorkSpaceKey`: Log Analytics shared key for data ingestion
  - `gsaConfigExportLatest`: Exported configuration JSON
- **Access**: Automation Account MSI has `Get` and `List` permissions

**Azure Storage Account**
- **Container: `guardrailsstorage`**
  - Module ZIP files (deployed code)
  - `modules.json` configuration
- **Container: `configuration`**
  - Exported configuration backups
- **Container: `workbooks`** (optional)
  - Azure Workbook templates for reporting

**Azure Log Analytics Workspace**
- **Table: `GuardrailsCompliance_CL`**
  - Primary compliance check results
  - Columns: ControlName, ItemName, ComplianceStatus, Comments, itsgcode, ReportTime, Required
- **Table: `GuardrailsComplianceException_CL`**
  - Error logs from failed modules or checks
- **Table: `CaCDebugMetrics_CL`** (optional, if debug metrics enabled)
  - Performance telemetry: execution time, memory usage, item counts
  - Permission snapshot data

#### 4. **Configuration Management**

**`modules.json`** - Master configuration file defining all checks:
```json
{
  "ModuleName": "Check-CloudAccountsMFA",
  "Status": "Enabled",
  "Control": "GUARDRAIL 1: PROTECT USER ACCOUNTS AND IDENTITIES",
  "Script": "Check-CloudAccountsMFA -WorkSpaceID $WorkSpaceID -workspaceKey $WorkspaceKey ...",
  "Required": true,
  "Profiles": [1, 2, 3, 4, 5, 6],
  "variables": [...],
  "secrets": [...]
}
```

**Multi-Cloud Profiles** (1-6):
- **Profile 1**: Basic cloud usage
- **Profile 2**: Standard enterprise
- **Profile 3**: Government/regulated industries (default)
- **Profile 4-6**: Custom profiles for specific use cases

**`config.json`** - Deployment parameters:
- Resource names (automation account, storage, Key Vault)
- Policy IDs (PBMM, allowed locations)
- Break-glass account UPNs
- Retention days, department info, locale

#### 5. **API Integrations**

**Microsoft Graph API**
- User accounts (all users, admin accounts)
- Group memberships
- Conditional access policies
- MFA settings and authentication methods
- Application service principals
- Directory role assignments
- Cross-tenant access settings

**Azure Resource Manager API**
- Subscriptions and management groups
- Azure Policy definitions and assignments
- Resource groups and tags
- Network resources (VNets, NSGs, subnets)
- Storage accounts configuration
- App Service configuration
- Key Vault settings
- Log Analytics workspace configuration

**Azure Log Analytics Data Ingestion API**
- Custom log ingestion via HTTP POST
- Table schema auto-created on first write
- Retention managed by Log Analytics workspace

#### 6. **Centralized Reporting (Multi-Tenant)**

**For Managed Service Providers**:

**Customer Components**:
- Deployed in each customer tenant
- Exports compliance data to central Log Analytics workspace

**Provider Components**:
- Central Azure Function App with:
  - **HTTP Trigger** (`grfunchttp`): Receives compliance data from customers
  - **Timer Trigger** (`grtimerfunction`): Scheduled data aggregation
- Centralized Log Analytics workspace
- Azure Workbooks for multi-tenant compliance dashboards

**Azure Lighthouse Integration**:
- Provider manages customer subscriptions via delegated access
- Resource provider registration automation
- Unified reporting across all managed tenants

---

### Data Flow

```
1. SCHEDULED TRIGGER
   └─> Azure Automation Account runbook starts

2. AUTHENTICATION & INITIALIZATION
   ├─> Connect-AzAccount (MSI)
   ├─> Retrieve configuration from Key Vault
   ├─> Load modules.json from Storage Blob
   └─> Pre-fetch all user data (Graph API)

3. MODULE EXECUTION LOOP (for each enabled module)
   ├─> Load module-specific variables/secrets
   ├─> Invoke Check-* function
   ├─> Query Azure/Graph APIs
   ├─> Evaluate compliance rules
   └─> Return ComplianceResults object

4. RESULT PROCESSING
   ├─> Add ReportTime, Required fields
   ├─> Calculate compliance statistics
   └─> Track performance metrics (time, memory)

5. DATA INGESTION
   ├─> POST to Log Analytics Data Collector API
   ├─> Write to GuardrailsCompliance_CL table
   └─> Write errors to GuardrailsComplianceException_CL

6. REPORTING & ANALYSIS
   ├─> Azure Workbooks query Log Analytics
   ├─> KQL queries aggregate compliance status
   └─> Dashboards show trends and violations
```

---

### Deployment Architecture

#### Option 1: Single-Tenant Deployment

```powershell
# Deploy to a single Azure tenant
Deploy-GuardrailsSolutionAccelerator `
    -ConfigFilePath ".\config.json" `
    -KeyVaultName "kv-guardrails" `
    -ResourceGroup "rg-guardrails" `
    -AutomationAccountName "aa-guardrails"
```

**Infrastructure Created**:
- 1 Resource Group
- 1 Azure Automation Account
- 1 Azure Key Vault
- 1 Azure Storage Account
- 1 Log Analytics Workspace

#### Option 2: Multi-Tenant (MSP) Deployment

**Per Customer Tenant**:
- Full guardrails deployment (automation, storage, Key Vault)
- Export compliance data to provider's central Log Analytics

**Provider Tenant**:
- Central Log Analytics workspace
- Azure Function App for data aggregation
- Centralized reporting workbooks

**Azure Lighthouse**:
- Provider has delegated access to customer subscriptions
- Can manage guardrails deployments centrally

---

### Security Architecture

**Authentication**:
- Managed Service Identity (MSI) for Azure resource access
- No stored credentials in code or configuration files

**Authorization**:
- Automation Account MSI assigned:
  - `Reader` on subscriptions/management groups
  - `Directory Readers` in Azure AD (for user/group queries)
  - Key Vault `Get` and `List` permissions

**Secret Management**:
- All secrets stored in Azure Key Vault
- Workspace key for Log Analytics ingestion
- Break-glass account UPNs
- Service principal credentials (if needed)

**Data Protection**:
- All data encrypted at rest (Azure Storage/Key Vault encryption)
- Data in transit protected via HTTPS/TLS 1.2+
- Logs stored in Log Analytics with configurable retention (default: 730 days)

**Least Privilege**:
- Automation Account has read-only access to resources
- No ability to modify production resources
- Limited to compliance data collection and logging

---

## PowerShell Script Analysis

### PowerShell Scripts Inventory

The repository contains **60+ PowerShell scripts** across multiple categories:

#### Core Runbooks (2)
1. **`setup/main.ps1`** (629 lines)
   - Primary orchestrator for compliance checks
   - Module loading, execution, and result aggregation
   
2. **`setup/backend.ps1`** (241 lines)
   - ITSG data updates
   - Tenant information maintenance
   - Lighthouse provider registration

#### Guardrail Check Modules (45+)

Located in `/src/GUARDRAIL X <Name>/`:

**GUARDRAIL 1: Protect User Accounts and Identities**
- `Check-CloudAccountsMFA.psm1` - MFA policy validation
- `Check-DedicatedAdminAccounts.psm1` - Admin account separation
- `Check-BreakGlassAccounts.psm1` - Emergency account validation
- `Check-MFAEnforcement.psm1` - MFA configuration checks
- `Check-AdminAccountEvents.psm1` - Admin activity logging
- `Check-MFAExclusion.psm1` - MFA exclusion list validation

**GUARDRAIL 2: Manage Access**
- `Check-ExternalUsers.psm1` - Guest account monitoring
- `Check-DeprecatedUsers.psm1` - Inactive account detection
- `Check-GuestAccountReview.psm1` - Guest access review
- `Check-ConditionalAccessPolicy.psm1` - Conditional access validation
- `Check-RoleAssignmentReview.psm1` - Permission review
- `Check-AdminRoleAssignments.psm1` - Privileged role monitoring

**GUARDRAIL 3: Secure Endpoints**
- `Check-AdminAccess.psm1` - Admin access policy
- `Check-ConsoleAccess.psm1` - Console access policy

**GUARDRAIL 4: Enterprise Monitoring Accounts**
- `Check-ServicePrincipal.psm1` - Service principal validation
- `Check-ServicePrincipalCredentials.psm1` - Credential expiration
- `Check-FinOpsTools.psm1` - FinOps tooling compliance

**GUARDRAIL 5: Data Location**
- `Check-AllowedLocations.psm1` - Geographic restriction policy

**GUARDRAIL 6: Protection of Data-at-Rest**
- `Check-ProtectionDataAtRest.psm1` - Encryption at rest policy

**GUARDRAIL 7: Protection of Data-in-Transit**
- `Check-StorageAccountTLSversion.psm1` - TLS version enforcement
- `Check-AppServiceHTTPSConfiguration.psm1` - HTTPS-only validation
- `Check-KeyVaultCertificateExpiry.psm1` - Certificate expiration
- `Check-FrontDoorHTTPSConfiguration.psm1` - Front Door HTTPS
- `Check-APIManagementHTTPSConfiguration.psm1` - API Management HTTPS
- (5 more TLS/HTTPS checks for various Azure services)

**GUARDRAIL 8: Network Segmentation and Separation**
- `Check-SubnetComplianceStatus.psm1` - Subnet design validation

**GUARDRAIL 9: Network Security Services**
- `Check-VNetComplianceStatus.psm1` - Virtual network validation
- `Check-NetworkWatcher.psm1` - Network watcher configuration
- `Check-NSGComplianceStatus.psm1` - NSG configuration
- `Check-NetworkInterfaces.psm1` - NIC configuration

**GUARDRAIL 10: Cyber Defense Services**
- `Check-CyberDefenseServices.psm1` - Security services validation

**GUARDRAIL 11: Logging and Monitoring**
- `Check-ServiceHealthAlerts.psm1` - Service health alerting
- `Check-DefenderForCloudAlerts.psm1` - Defender alert configuration

**GUARDRAIL 12: Configuration of Cloud Marketplaces**
- `Check-PrivateMarketplace.psm1` - Private marketplace configuration

**GUARDRAIL 13: Plan for Continuity**
- `Check-BreakGlassAccount.psm1` - Break-glass account validation
- `Check-BreakGlassAuthenticationMethods.psm1` - Auth method validation
- `Check-BreakGlassOwnership.psm1` - Account ownership
- `Check-BreakGlassCredentials.psm1` - Credential management
- `Check-BreakGlassMFAExclusion.psm1` - MFA exclusion validation

#### Common & Utility Modules

**`src/Guardrails-Common/`**:
- `GR-Common.psm1` - Shared utility functions (tag parsing, blob operations, MFA exclusions)
- `blob-functions.psm1` - Azure Storage operations

**`src/GuardrailsSolutionAcceleratorSetup/`** (15+ modules):
- `Deploy-GuardrailsSolutionAccelerator.psm1` - Main deployment orchestrator
- `Confirm-GSAPrerequisites.psm1` - Prerequisite validation
- `Deploy-GSACoreResources.psm1` - Infrastructure deployment
- `Get-GSAExportedConfig.psm1` - Configuration export
- `Deploy-GSACentralizedReportingCustomerComponents.psm1` - Customer reporting
- `Deploy-GSACentralizedReportingProviderComponents.psm1` - Provider reporting
- (9 more deployment/configuration modules)

#### Tools & Utilities

**`tools/`**:
- `zipemall.ps1` - Package modules into ZIP archives
- `Update-ModuleVersions.ps1` - Update module version numbers
- `Check-ZippedModuleVersions.ps1` - Validate ZIP file versions
- `Create-manifestpsd1.psm1` - Generate module manifests
- `Reformat-Sarif.ps1` - SARIF security report formatting

**`tools/CentralView/`** (Azure Functions):
- `grfunchttp/run.ps1` - HTTP endpoint for data ingestion
- `grtimerfunction/run.ps1` - Scheduled aggregation
- `setup/setup.ps1` - Function app deployment
- `profile.ps1` - Function app configuration

---

## Issues Section: PowerShell Script Analysis

### Critical Issues

#### 1. **Brittle Tag Parsing (High Risk)**

**Location**: `src/Guardrails-Common/GR-Common.psm1`, lines 1-84

**Problem**: Multiple functions use naive string splitting on "=" and ";" delimiters to parse Azure resource tags. This approach fails when tag values contain these characters.

**Affected Functions**:
- `Get-Tags` (line 1-21)
- `Get-PBMMTags` (line 23-48)
- `Get-AllowedLocationPolicy` (line 50-66)
- `Get-PBMMExclusionPolicy` (line 68-84)

**Example Vulnerable Code**:
```powershell
$tags = $resource.Tags
$tagValue = ($tags -split ';' | Where-Object { $_ -match "^$tagName=" }) -replace "^$tagName=", ""
```

**Risk**: 
- If a tag value contains `=` or `;`, parsing will fail or produce incorrect results
- Example: `Department=R&D;Development` would split incorrectly
- Could cause compliance checks to miss resources or report false negatives

**Remediation Needed**:
```powershell
# Use Azure's native tag property instead of string parsing
if ($resource.Tags.ContainsKey($tagName)) {
    $tagValue = $resource.Tags[$tagName]
}
```

**Impact**: Medium (tags are typically controlled, but still fragile)

---

#### 2. **Missing JSON Structure Validation (High Risk)**

**Location**: `setup/main.ps1`, lines 148-149

**Problem**: Configuration retrieval from Key Vault uses chained JSON parsing without null/structure validation.

**Vulnerable Code**:
```powershell
$RuntimeConfig = Get-AzKeyVaultSecret -VaultName $KeyVaultName -Name 'gsaConfigExportLatest' `
    -AsPlainText -ErrorAction Stop | ConvertFrom-Json | Select-Object -Expand runtime
```

**Risk**:
- If JSON structure changes, `Select-Object -Expand runtime` fails silently
- If `runtime` property missing, script crashes without helpful error message
- Could cause complete runbook failure

**Similar Issues**:
- Line 77: `$config.psobject.properties | ForEach-Object { ... }`
- Line 82: `Select-Object -Expand runtime` (repeated pattern)

**Remediation Needed**:
```powershell
$configSecret = Get-AzKeyVaultSecret -VaultName $KeyVaultName -Name 'gsaConfigExportLatest' -AsPlainText -ErrorAction Stop
$config = $configSecret | ConvertFrom-Json

if (-not $config.PSObject.Properties['runtime']) {
    throw "Configuration JSON missing required 'runtime' property"
}

$RuntimeConfig = $config.runtime
```

**Impact**: High (causes runbook failure, no data collected)

---

#### 3. **Hardcoded External Dependency (Medium Risk)**

**Location**: `setup/backend.ps1`, line 70

**Problem**: Hardcoded GitHub URL for ITSG control data with no fallback mechanism.

**Code**:
```powershell
$itsgURL="https://raw.githubusercontent.com/ssc-spc-ccoe-cei/workshop-test-azure-cac/main/setup/itsg33-ann4a-eng.csv"
```

**Risk**:
- If GitHub is unavailable, ITSG data updates fail
- If repository is renamed/deleted, all deployments break
- No cached fallback or retry logic

**Remediation Needed**:
- Store ITSG data in Azure Storage as primary source
- Use GitHub as secondary/update mechanism
- Add retry logic with exponential backoff

**Impact**: Medium (degrades functionality but doesn't stop main checks)

---

#### 4. **Incomplete Error Context (Medium Risk)**

**Location**: `setup/main.ps1`, lines 535-547

**Problem**: Exception handling loses original error count and context.

**Code**:
```powershell
catch {
    if ($moduleErrors -lt 1) {
        $moduleErrors = 1  # Forces error count to 1, losing actual count
    }
    # Original error details logged but error count lost
    Add-LogEntry 'Error' "Failed to invoke the module execution script for module '$($module.moduleName)', script '$sanitizedScriptblock' with error: $_"
}
```

**Risk**:
- If module throws 5 errors, only 1 is reported in telemetry
- Makes debugging and root cause analysis difficult
- Misleading statistics in compliance reports

**Remediation Needed**:
```powershell
catch {
    $moduleErrors++  # Increment instead of overriding
    $exceptionMessage = $_.Exception.Message
    $scriptContext = $_.InvocationInfo.PositionMessage
    Add-LogEntry 'Error' "Module execution failed: $exceptionMessage`n$scriptContext"
}
```

**Impact**: Medium (affects observability, not compliance accuracy)

---

### Design & Quality Issues

#### 5. **Memory Management Strategy (Low-Medium Risk)**

**Location**: `setup/main.ps1`, lines 558-564

**Problem**: Garbage collection triggered every 3 modules (arbitrary interval).

**Code**:
```powershell
if ($moduleCount % 3 -eq 0) {
    Write-Output "Clearing memory after $moduleCount modules..."
    [System.GC]::Collect()
    [System.GC]::WaitForPendingFinalizers()
    [System.GC]::Collect()
}
```

**Issues**:
- No adaptive threshold based on actual memory usage
- Could cause unnecessary GC pauses if modules are lightweight
- Could be insufficient if modules leak memory

**Recommendation**:
```powershell
$currentMemoryMB = [System.GC]::GetTotalMemory($false) / 1MB
if ($currentMemoryMB -gt $memoryThresholdMB) {
    Write-Output "Memory usage ($currentMemoryMB MB) exceeded threshold ($memoryThresholdMB MB), collecting..."
    [System.GC]::Collect()
}
```

**Impact**: Low (performance optimization, not functional issue)

---

#### 6. **Incomplete Pagination Handling (Medium-High Risk)**

**Location**: Multiple modules querying Graph API

**Problem**: Comment in code indicates pagination awareness, but implementation may be incomplete.

**Example**: `Check-CloudAccountsMFA.psm1` comment:
```powershell
# (using paginated query to handle >100 policies)
```

**Risk**:
- Graph API returns max 100 items per page by default
- If pagination not fully implemented, items 101+ are missed
- Critical for tenants with >100 users, policies, or resources

**Common Pattern to Verify**:
```powershell
$result = Invoke-MgGraphRequest -Uri $uri
$allItems = $result.value

# Check if pagination is handled
while ($result.'@odata.nextLink') {
    $result = Invoke-MgGraphRequest -Uri $result.'@odata.nextLink'
    $allItems += $result.value
}
```

**Modules to Audit**:
- All user enumeration functions
- Policy listing functions
- Resource queries (VNets, storage accounts, etc.)

**Impact**: High (causes false negatives, missed violations)

---

#### 7. **Magic Strings and Hardcoded Values (Low Risk)**

**Location**: Multiple locations

**Examples**:
1. **Reserved subnet list** (`main.ps1`, line 95):
   ```powershell
   [System.Environment]::SetEnvironmentVariable('ReservedSubnetList', 
       "GatewaySubnet,AzureFirewallSubnet,AzureBastionSubnet,AzureFirewallManagementSubnet,RouteServerSubnet", 
       [System.EnvironmentVariableTarget]::Process)
   ```

2. **Table names** (`main.ps1`, line 92):
   ```powershell
   [System.Environment]::SetEnvironmentVariable('LogType', 'GuardrailsCompliance', ...)
   ```

3. **Default locale** (`main.ps1`, line 143-145):
   ```powershell
   If ($Locale -eq $null) {
       $Locale = "en-CA"
   }
   ```

**Issues**:
- Cannot be customized without code changes
- Should be in configuration file
- Makes testing difficult

**Recommendation**: Move to `config.json` or `modules.json`

**Impact**: Low (maintenance burden, not functional)

---

#### 8. **Inconsistent Error Handling Patterns (Medium Risk)**

**Problem**: Different modules use inconsistent error handling approaches.

**Pattern 1**: ErrorAction Stop with try-catch (good)
```powershell
try {
    $result = Get-AzResource -ErrorAction Stop
}
catch {
    Write-Error "Failed to get resources: $_"
}
```

**Pattern 2**: No ErrorAction specified (risky)
```powershell
$result = Get-AzResource  # May silently fail
```

**Pattern 3**: Bare try-catch (loses context)
```powershell
try {
    Do-Something
}
catch {
    Write-Error $_  # No context about what failed
}
```

**Impact**: Medium (inconsistent reliability, debugging difficulty)

---

#### 9. **Performance Concerns with User Data Fetching (Medium-High Risk)**

**Location**: `setup/main.ps1`, line 340

**Problem**: `FetchAllUserRawData` loads **all users** in tenant regardless of subscription scope.

**Risk**:
- Large tenants (100K+ users) may timeout Azure Automation job (3-hour limit)
- Excessive API calls to Microsoft Graph
- High memory consumption
- Unnecessary data fetching if only evaluating specific subscriptions

**Recommendation**:
- Add subscription-scoped filtering
- Implement incremental loading with batching
- Cache results across multiple runbook executions (use Storage Blob)

**Impact**: High for large tenants (job timeout, incomplete checks)

---

#### 10. **Module Return Contract Not Enforced (Medium Risk)**

**Location**: `setup/main.ps1`, line 449

**Problem**: Modules expected to return specific structure, but no validation enforces it.

**Expected Contract**:
```powershell
@{
    ComplianceResults = @(...)  # Array of results
    Errors = @(...)            # Array of errors
    AdditionalResults = @{     # Optional extra data
        logType = "CustomLog"
        records = @(...)
    }
}
```

**Risk**:
- If module returns different structure, main.ps1 may crash
- No type checking on return values
- Silent failures if properties missing

**Example Vulnerable Code**:
```powershell
$results = $NewScriptBlock.Invoke()
# No validation that $results has required properties
New-LogAnalyticsData -Data $results.ComplianceResults  # Could be null
```

**Recommendation**:
```powershell
$results = $NewScriptBlock.Invoke()

# Validate return structure
if (-not ($results.PSObject.Properties['ComplianceResults'])) {
    Write-Error "Module did not return ComplianceResults property"
    continue
}

if ($results.ComplianceResults -isnot [array]) {
    Write-Error "ComplianceResults must be an array"
    continue
}
```

**Impact**: Medium (causes pipeline failures)

---

#### 11. **Lack of Retry Logic for Transient Failures (Medium Risk)**

**Problem**: No retry mechanism for Azure/Graph API calls that may fail transiently.

**Common Transient Errors**:
- HTTP 429 (Too Many Requests)
- HTTP 503 (Service Unavailable)
- Network timeouts
- Authentication token expiration

**Current Behavior**: Single failure causes module to report error and skip checks

**Recommendation**: Implement retry with exponential backoff
```powershell
function Invoke-WithRetry {
    param($ScriptBlock, $MaxRetries = 3, $InitialDelaySeconds = 2)
    
    $attempt = 0
    while ($attempt -lt $MaxRetries) {
        try {
            return & $ScriptBlock
        }
        catch {
            $attempt++
            if ($attempt -ge $MaxRetries) { throw }
            
            $delay = $InitialDelaySeconds * [Math]::Pow(2, $attempt - 1)
            Write-Warning "Attempt $attempt failed, retrying in $delay seconds..."
            Start-Sleep -Seconds $delay
        }
    }
}
```

**Impact**: Medium (reduces reliability in production)

---

#### 12. **Credential Handling in Logs (Low-Medium Risk)**

**Location**: Multiple locations

**Problem**: Break-glass UPNs and other secrets passed as plain text parameters.

**Example**:
```powershell
Check-BreakGlassAccount -FirstBreakGlassUPN $FirstBreakGlassUPN -SecondBreakGlassUPN $SecondBreakGlassUPN
```

**Risk**:
- If verbose logging enabled, UPNs visible in logs
- Multi-user automation account could expose secrets
- Job history shows parameter values

**Mitigation Present**:
- `main.ps1` sanitizes workspace key in error logs (line 544)
- `$workspaceKey` replaced with `***` in log entries

**Recommendation**: 
- Extend sanitization to all sensitive parameters
- Use SecureString for secret parameters
- Audit all Write-Output/Write-Verbose for secret leakage

**Impact**: Low-Medium (information disclosure risk)

---

#### 13. **No Unit Test Coverage (Medium Risk)**

**Problem**: No visible unit tests in repository.

**Missing**:
- Module function tests
- Mocking of Azure/Graph API calls
- Compliance logic validation
- Configuration parsing tests

**Impact**:
- Refactoring is risky (no regression detection)
- Bug fixes may introduce new bugs
- Cannot validate behavior in isolation

**Recommendation**:
- Add Pester test framework
- Create tests for each module function
- Mock Azure cmdlets for isolated testing

**Impact**: Medium (maintenance and quality risk)

---

#### 14. **String Interpolation in Dynamic Script Blocks (Low Risk)**

**Location**: `setup/main.ps1`, line 544

**Problem**: Complex string expansion on script blocks could be vulnerable to injection.

**Code**:
```powershell
$sanitizedScriptblock = $($ExecutionContext.InvokeCommand.ExpandString(($moduleScript -ireplace '\$workspaceKey', '***')))
```

**Risk**:
- If `$moduleScript` contains user-controlled input, could execute arbitrary code
- Current usage appears safe (script comes from `modules.json`)
- Pattern is concerning for future modifications

**Recommendation**:
- Avoid `ExpandString` on dynamic content
- Use parameterized approach instead of string replacement
- Add input validation on module script sources

**Impact**: Low (currently safe, but dangerous pattern)

---

#### 15. **Circular Dependencies in Setup Modules (Low Risk)**

**Location**: `src/GuardrailsSolutionAcceleratorSetup/GuardrailsSolutionAcceleratorSetup.psd1`

**Problem**: Module manifest imports 5 sub-modules, but dependency order not explicit.

**Code**:
```powershell
NestedModules = @(
    'Deploy-GuardrailsSolutionAccelerator.psm1',
    'Confirm-GSAPrerequisites.psm1',
    'Deploy-GSACoreResources.psm1',
    'Get-GSAExportedConfig.psm1',
    'Show-GSADeploymentSummary.psm1'
)
```

**Risk**:
- If modules have interdependencies, import order matters
- Not obvious which prerequisites are needed
- Could cause issues if one module fails to import

**Recommendation**:
- Document module dependencies
- Add `RequiredModules` to manifests
- Implement dependency resolution logic

**Impact**: Low (appears to work, but fragile)

---

### Summary of Issues by Severity

| Severity | Count | Examples |
|----------|-------|----------|
| **Critical** | 0 | None identified |
| **High** | 4 | Brittle tag parsing, JSON validation, pagination, user data fetching |
| **Medium** | 8 | Error context, retry logic, contract enforcement, credential handling |
| **Low** | 3 | Magic strings, GC strategy, circular dependencies |

### Recommendations Priority

1. **Immediate** (High/Critical):
   - Fix JSON structure validation (prevents runbook failures)
   - Audit and fix pagination in all Graph API calls (prevents false negatives)
   - Add retry logic for transient failures

2. **Short-Term** (Medium):
   - Refactor tag parsing to use native Azure tag properties
   - Enforce module return contract with validation
   - Improve error handling consistency

3. **Long-Term** (Low):
   - Add comprehensive unit test coverage
   - Move hardcoded values to configuration
   - Optimize memory management strategy

---

## Appendix: Module Inventory

### Total PowerShell Code Statistics
- **Total Scripts**: 60+
- **Total Lines of Code**: ~12,000+ (estimated)
- **Languages**: PowerShell 5.1+, PowerShell 7.0+ (some tools)
- **Dependencies**: Az.* modules, Microsoft.Graph.* modules

### Infrastructure as Code
- **Bicep Templates**: 20+ files
- **ARM Templates**: Legacy templates being migrated to Bicep
- **GitHub Actions Workflows**: 9 CI/CD pipelines

### Key External Dependencies
- **Azure Modules**: Az.Accounts, Az.Resources, Az.Storage, Az.KeyVault, Az.Automation, Az.OperationalInsights, Az.Monitor, Az.Network
- **Microsoft Graph**: Microsoft.Graph.Authentication, Microsoft.Graph.Users, Microsoft.Graph.Groups, Microsoft.Graph.Identity.*
- **Other**: PSScriptAnalyzer (linting), Pester (testing framework - not currently used)

---

*Document Generated: 2026-01-16*
*Last Updated: 2026-01-16*
*Version: 1.0*
