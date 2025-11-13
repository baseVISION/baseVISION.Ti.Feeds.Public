# baseVISION TI Feeds - Microsoft Defender XDR Integration [Beta]

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FbaseVISION%2FbaseVISION.Ti.Feeds.Public%2Frefs%2Fheads%2Fmain%2Fbeta%2FDefenderIntegration%2Fla-basevisionti-defenderxdr.json)
![Beta](https://img.shields.io/badge/Status-Beta-orange)
![Support](https://img.shields.io/badge/Support-Community-blue)

> 🛡️ **Enhance your Microsoft Defender XDR with high-quality threat intelligence**

This integration automatically synchronizes baseVISION threat intelligence indicators with Microsoft Defender XDR, providing your security team with actionable threat intelligence directly within your endpoint detection and response platform.

## 🌟 Overview

The Microsoft Defender XDR integration is a Logic App-based solution that:

- **Automatically syncs** high-confidence IOCs from baseVISION feeds to Microsoft Defender XDR
- **Filters intelligently** to avoid Microsoft's 15,000 IOC limit by excluding Microsoft-native indicators
- **Runs continuously** with 30-minute intervals for near real-time protection
- **Respects confidence levels** by only importing IOCs with 100% confidence scores
- **Supports multiple IOC types** including IP addresses, URLs, file hashes, and domains

## 🎯 Key Features

### 🔄 **Automated Synchronization**
- **Frequency**: Every 30 minutes
- **Filtering**: Only high-confidence (100%) indicators
- **Deduplication**: Excludes IOCs already present in Microsoft feeds
- **Batch Processing**: Optimized for Microsoft Defender XDR API limits

### 📊 **Intelligent IOC Management**
- **IOC Types Supported**:
  - 🌐 IP Addresses (Source, Destination, Generic)
  - 🔗 URLs and domains
  - 📄 File hashes (SHA-256, excludes MD5/SHA1)
  - 📧 Email addresses
- **Action Configuration**: Configurable response (Warn, Block, Audit, BlockAndRemediate)
- **Expiration Handling**: Automatic cleanup of expired indicators

### 🛡️ **Security & Compliance**
- **Managed Identity**: Secure authentication without stored credentials
- **Least Privilege**: Minimal required permissions

## 🏗️ Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│  Microsoft      │    │   Logic App      │    │   Microsoft         │
│  Sentinel       │────│   (15 min        │────│   Defender XDR      │
│  (TAXII Feed)   │    │   intervals)     │    │   (IOC Import)      │
└─────────────────┘    └──────────────────┘    └─────────────────────┘
         │                       │                        │
         │              ┌────────▼────────┐              │
         │              │  Kusto Query    │              │
         └──────────────│  (Filtering &   │──────────────┘
                        │  Deduplication) │
                        └─────────────────┘
```

## 🚀 Quick Start

### Prerequisites

Before deploying this integration, ensure you have:

- ✅ **baseVISION TI Feeds** configured in Microsoft Sentinel via TAXII connector
- ✅ **Microsoft Sentinel workspace** with Log Analytics
- ✅ **Microsoft Defender XDR** subscription and access
- ✅ **Azure permissions** to deploy Logic Apps and assign roles
- ✅ **Azure PowerShell** or Cloud Shell access for post-deployment configuration

### Step 1: Deploy the Logic App

Click the deploy button to start the automated deployment:

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FbaseVISION%2FbaseVISION.Ti.Feeds.Public%2Frefs%2Fheads%2Fmain%2Fbeta%2FDefenderIntegration%2Fla-basevisionti-defenderxdr.json)

#### Deployment Parameters

| Parameter | Description | Default Value | Required |
|-----------|-------------|---------------|----------|
| **PlaybookName** | Name for the Logic App resource | `la-basevisionti-defenderxdr` | Yes |
| **Action** | Defender XDR action for IOCs | `Audit` | Yes |
| **TaxiiFeedName** | Name of your TAXII feed in Sentinel | `baseVISION-SOC-TI-Feed` | Yes |
| **workspace** | Log Analytics workspace name | `<Enter Log Analytics Workspace>` | Yes |

> ⚠️ **Important**: Use the exact TAXII feed name as configured in your Microsoft Sentinel TAXII connector.

### Step 2: Configure API Permissions

After deployment, you must grant the managed identity access to the Microsoft Defender XDR API.

#### Option A: Azure Cloud Shell (Recommended)

1. Open [Azure Cloud Shell](https://shell.azure.com)
2. Run the following PowerShell script:

```powershell
# Configuration - Update these values
$miName = "la-basevisionti-defenderxdr-uami"  # Managed Identity name
$appId = "fc780465-2017-40d4-a0c5-307022471b92"  # Microsoft Defender XDR API App ID
$permissionsToAdd = @("Ti.ReadWrite.All")  # Required API permission

# Connect and configure
Connect-AzureAD

# Get the managed identity
$miObjectID = (Get-AzADServicePrincipal -DisplayName $miName).Id
if (-not $miObjectID) {
    Write-Error "Managed Identity '$miName' not found. Please check the deployment."
    exit 1
}

# Get the Defender API service principal
$app = Get-AzureADServicePrincipal -Filter "AppId eq '$appId'"
if (-not $app) {
    Write-Error "Microsoft Defender XDR API not found."
    exit 1
}

# Assign permissions
foreach ($permission in $permissionsToAdd) {
    $role = $app.AppRoles | Where-Object Value -Like $permission | Select-Object -First 1
    
    if ($role) {
        try {
            New-AzureADServiceAppRoleAssignment -Id $role.Id -ObjectId $miObjectID -PrincipalId $miObjectID -ResourceId $app.ObjectId
            Write-Host "✅ Successfully assigned '$permission' permission" -ForegroundColor Green
        }
        catch {
            Write-Warning "Failed to assign '$permission': $($_.Exception.Message)"
        }
    }
    else {
        Write-Warning "Permission '$permission' not found in app roles"
    }
}

Write-Host "🎉 Permission configuration completed!" -ForegroundColor Green
```

#### Option B: Local PowerShell

If you prefer to run the script locally:

1. Install Azure PowerShell modules:
   ```powershell
   Install-Module -Name Az -AllowClobber -Scope CurrentUser
   Install-Module -Name AzureAD -AllowClobber -Scope CurrentUser
   ```

2. Run the same script as above after connecting to your Azure tenant

### Step 3: Verify Deployment

1. **Check Logic App Status**:
   - Navigate to your Logic App in the Azure portal
   - Verify it's enabled and running
   - Check the run history for successful executions

2. **Verify Microsoft Defender XDR Integration**:
   - Open Microsoft Defender XDR portal
   - Navigate to **Settings** > **Indicators**
   - Look for indicators tagged with your TAXII feed name

3. **Monitor Performance**:
   - Review Logic App execution logs
   - Check for any authentication or API errors
   - Validate IOC import counts and timing

## 📋 Configuration Options

### Action Types

Configure how Microsoft Defender XDR responds to imported IOCs:

| Action | Description | Use Case |
|--------|-------------|----------|
| **Audit** | Log detections without blocking | Testing and monitoring |
| **Warn** | Alert users about potential threats | User awareness |
| **Block** | Prevent access to malicious resources | Active protection |
| **BlockAndRemediate** | Block and clean up threats | Maximum protection |

### IOC Filtering Logic

The integration uses sophisticated Kusto queries to:

- ✅ Include only high-confidence indicators (>99%)
- ✅ Exclude IOCs already present in Microsoft feeds
- ✅ Filter out weak hash types (MD5, SHA1)
- ✅ Respect Microsoft Defender XDR's 15,000 IOC limit
- ✅ Handle IOC expiration automatically

## 🔧 Troubleshooting

### Common Issues

#### ❌ **Logic App Authentication Errors**
**Symptoms**: HTTP 401/403 errors in Logic App runs
**Solution**: 
1. Verify the managed identity has the correct permissions
2. Re-run the PowerShell permission script
3. Check that the API App ID is correct

#### ❌ **No IOCs Being Imported**
**Symptoms**: Logic App runs successfully but no IOCs appear in Defender XDR
**Solution**:
1. Verify TAXII feed name matches exactly
2. Check that baseVISION indicators are present in Sentinel
3. Confirm indicators meet confidence threshold (>99%)

#### ❌ **API Rate Limiting**
**Symptoms**: HTTP 429 errors or throttling messages
**Solution**:
1. The Logic App includes retry logic and batching
2. Consider reducing execution frequency if needed
3. Monitor Microsoft Defender XDR API quotas

### Debugging Steps

1. **Check Logic App Execution History**:
   ```
   Azure Portal > Logic Apps > [Your Logic App] > Overview > Runs history
   ```

2. **Review Kusto Query Results**:
   ```kusto
   // Run this query in your Sentinel workspace to see available IOCs
   ThreatIntelIndicators
   | where ConfidenceScore > 99 
     and Active == true 
     and SourceSystem == "baseVISION-SOC-TI-Feed"
   | summarize count() by ObservableKey
   ```

## 📊 Monitoring & Analytics

### Key Metrics to Track

- **IOC Import Rate**: Number of indicators imported per execution
- **Execution Success Rate**: Percentage of successful Logic App runs
- **API Response Times**: Performance of Microsoft Defender XDR API calls
- **Error Rates**: Failed authentications or API errors

### Recommended Alerts

Set up Azure Monitor alerts for:
- Logic App execution failures
- Authentication errors
- API quota exhaustion
- Unexpected drops in IOC import volume

## 🔐 Security Considerations

### Data Protection
- All API communications use HTTPS/TLS encryption
- Managed identity eliminates credential storage
- Audit logs track all IOC imports and modifications

### Access Control
- Principle of least privilege for managed identity
- Role-based access control for Logic App management
- Regular review of API permissions

### Compliance
- Supports enterprise compliance requirements
- Full audit trail of all IOC operations
- Integration with Azure Security Center

## 🆕 Beta Status & Feedback

This integration is currently in **beta status**. We encourage feedback and contributions:

- 🐛 **Report Issues**: [GitHub Issues](https://github.com/baseVISION/baseVISION.Ti.Feeds.Public/issues)
- 💡 **Feature Requests**: Share your enhancement ideas
- 📖 **Documentation**: Help improve these guides
- 🤝 **Community**: Join discussions with other users

### Known Limitations

- Currently supports IOCs with 100% confidence only
- Limited to specific IOC types (IP, URL, Hash, Domain)
- Requires manual permission configuration post-deployment

## 📚 Additional Resources

### Documentation
- [Microsoft Defender XDR API Documentation](https://docs.microsoft.com/en-us/microsoft-365/security/defender/api-overview)
- [Azure Logic Apps Documentation](https://docs.microsoft.com/en-us/azure/logic-apps/)
- [Microsoft Sentinel TAXII Connector](https://docs.microsoft.com/en-us/azure/sentinel/connect-threat-intelligence-taxii)

### Support
- **Community Support**: GitHub Issues and Discussions
- **Documentation**: Comprehensive guides and troubleshooting

---

**Ready to enhance your threat detection?** Deploy the integration now and start leveraging high-quality threat intelligence in Microsoft Defender XDR!

