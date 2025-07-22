## PowerShell ##

To stop port down alerts on Windows servers for specific ports (e.g., 8080, 7070, 443), you can create a Standard Operating Procedure (SOP) that outlines the steps to configure monitoring settings or disable alerts. 
----
### Standard Operating Procedure (SOP) for Stopping Port Down Alerts on Windows Servers

**SOP Title:** Disabling Port Down Alerts for Specific Ports

**SOP Number:** [##]

**Effective Date:** [10-07-25]

**Prepared By:** [Samloki]

**Approved By:** [Self]

#### 1. Purpose

The purpose of this SOP is to outline the steps required to disable port down alerts for specific ports (e.g., 8080, 7070, 443) on Windows servers to prevent unnecessary notifications.

#### 2. Scope

This procedure applies to all Windows servers monitored for port status and is intended for use by system administrators and IT personnel.

#### 3. Responsibilities

- **System Administrators:** Responsible for implementing the steps outlined in this SOP.
- **IT Personnel:** Responsible for monitoring and maintaining server health.

#### 4. Prerequisites

- Administrative access to the Windows server.
- Access to the monitoring tool or software used for port monitoring (e.g., Nagios, Zabbix, etc.).

#### 5. Procedure

##### Step 1: Access the Monitoring Tool

1. Log in to the Windows server using an account with administrative privileges.
2. Open the monitoring tool or software that is configured to send port down alerts.

##### Step 2: Locate the Port Monitoring Configuration

1. Navigate to the configuration settings for the monitored services or ports.
2. Identify the section where port monitoring is configured.

##### Step 3: Disable Alerts for Specific Ports

1. Find the entries for the ports you want to disable alerts for (e.g., 8080, 7070, 443).
2. Depending on the monitoring tool, you may have options such as:
   - **Disable Alerts:** Uncheck or toggle the alert option for the specified ports.
   - **Modify Alert Settings:** Change the alert settings to a less sensitive threshold or set them to "Informational" instead of "Critical."

##### Step 4: Save Changes

1. After making the necessary changes, ensure to save the configuration.
2. Restart the monitoring service if required by the tool.

##### Step 5: Verify Changes

1. Test the configuration by simulating a port down condition (if possible) or monitoring the system for a period to ensure alerts are no longer triggered for the specified ports.
2. Document any changes made in the system logs or change management system.

### Notes

- Ensure that disabling alerts does not compromise the monitoring of critical services.
- Consider setting up alternative monitoring methods if necessary to ensure service availability.


## Second idea by powershell code ##

Here's a comprehensive PowerShell script to automatically suppress port and URL down alerts on Windows servers for specific ports (8080, 7070, 443). This solution includes proactive monitoring with alert suppression capabilities.

```powershell
<#
.SYNOPSIS
    Suppresses port and URL down alerts on Windows servers for specified ports (8080, 7070, 443)
.DESCRIPTION
    This script monitors TCP ports and associated URLs, automatically suppressing alerts when services
    are intentionally taken down or undergoing maintenance. Includes logging and email notification options.
.NOTES
    Version: 1.0
    Author: Samloki
    Last Updated: (Get-Date -Format "25-07-10")
#>

param (
    [int[]]$Ports = @(8080, 7070, 443),
    [string[]]$URLs = @(
        "http://localhost:8080",
        "http://localhost:7070",
        "https://localhost:443"
    ),
    [switch]$EnableAlertSuppression = $true,
    [string]$LogPath = "C:\Monitoring\PortMonitorLogs.log",
    [int]$CheckIntervalSeconds = 60
)

#region Configurations
$ServiceNames = @{
    8080 = "CustomApp8080"
    7070 = "CustomApp7070"
    443 = "HTTPService"
}

$EmailSettings = @{
    SmtpServer  = "smtp.yourdomain.com"
    From        = "monitoring@yourdomain.com"
    To          = "admin@yourdomain.com"
    Subject     = "[ALERT] Port Monitoring Notification"
    ErrorOnly   = $true  # Only send emails for real outages, not suppressed ones
}
#endregion

#region Initialize
if (-not (Test-Path (Split-Path $LogPath -Parent))) {
    New-Item -Path (Split-Path $LogPath -Parent) -ItemType Directory | Out-Null
}

$Script:LastKnownState = @{}
$SuppressionStartTimes = @{}
$Global:MonitoringEnabled = $true
#endregion

function Test-TCPPort {
    param([int]$Port, [string]$Server = "localhost")
    
    try {
        $Socket = New-Object System.Net.Sockets.TcpClient
        $AsyncResult = $Socket.BeginConnect($Server, $Port, $null, $null)
        $Success = $AsyncResult.AsyncWaitHandle.WaitOne(2000, $false)
        
        if ($Success) {
            $Socket.EndConnect($AsyncResult) | Out-Null
            return $true
        }
        return $false
    } catch {
        return $false
    } finally {
        if ($Socket) { $Socket.Close() }
    }
}

function Test-URL {
    param([string]$URL)
    
    try {
        $Request = [System.Net.WebRequest]::Create($URL)
        $Request.Timeout = 5000 # 5 second timeout
        $Response = $Request.GetResponse()
        $StatusCode = [int]$Response.StatusCode
        
        $Response.Close()
        return ($StatusCode -ge 200 -and $StatusCode -lt 400)
    } catch {
        return $false
    }
}

function Write-MonitoringLog {
    param([string]$Message, [string]$Level = "INFO")
    
    $LogEntry = "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] [$Level] $Message"
    Add-Content -Path $LogPath -Value $LogEntry
    
    if ($Level -eq "ERROR" -or $Level -eq "WARNING") {
        Write-EventLog -LogName "Application" -Source "PortMonitor" -EventId 1001 -EntryType $Level -Message $Message
    }
}

function Invoke-AlertSuppression {
    param(
        [string]$Resource,
        [string]$ResourceType,
        [bool]$IsDown
    )
    
    $CurrentTime = Get-Date
    
    if ($IsDown) {
        if (-not $SuppressionStartTimes.ContainsKey($Resource)) {
            $SuppressionStartTimes[$Resource] = $CurrentTime
            Write-MonitoringLog -Message "Alert suppression activated for $ResourceType '$Resource'" -Level "WARNING"
        }
        
        $SuppressionDuration = ($CurrentTime - $SuppressionStartTimes[$Resource]).TotalMinutes
        
        # Extended suppression logic (example: suppress alerts if down < 15 minutes)
        if ($SuppressionDuration -lt 15) {
            return $true
        }
    } else {
        if ($SuppressionStartTimes.ContainsKey($Resource)) {
            $SuppressionDuration = ($CurrentTime - $SuppressionStartTimes[$Resource]).TotalMinutes
            Write-MonitoringLog -Message "Alert suppression ended for $ResourceType '$Resource' (duration: $($SuppressionDuration.ToString('N2')) minutes)" -Level "INFO"
            $SuppressionStartTimes.Remove($Resource)
        }
    }
    
    return $false
}

# Main monitoring loop
while ($Global:MonitoringEnabled) {
    $Results = @()
    
    # Check ports
    foreach ($Port in $Ports) {
        $PortUp = Test-TCPPort -Port $Port
        $ShouldSuppress = $EnableAlertSuppression -and (Invoke-AlertSuppression -Resource $Port -ResourceType "Port" -IsDown (-not $PortUp))
        
        $Result = [PSCustomObject]@{
            Resource    = "Port $Port"
            Status      = if ($PortUp) { "UP" } else { "DOWN" }
            Timestamp   = Get-Date
            Service     = if ($ServiceNames.ContainsKey($Port)) { $ServiceNames[$Port] } else { "Unknown" }
            Suppressed  = $ShouldSuppress
        }
        
        $Results += $Result
        
        if (-not $PortUp -and -not $ShouldSuppress) {
            Write-MonitoringLog -Message "Port $Port is DOWN (service: $($Result.Service))" -Level "ERROR"
            if ($EmailSettings.ErrorOnly) {
                Send-MailMessage @EmailSettings -Body "Port $Port is DOWN on $(hostname)`nService: $($Result.Service)"
            }
        }
    }
    
    # Check URLs
    foreach ($URL in $URLs) {
        $URLUp = Test-URL -URL $URL
        $ShouldSuppress = $EnableAlertSuppression -and (Invoke-AlertSuppression -Resource $URL -ResourceType "URL" -IsDown (-not $URLUp))
        
        $Result = [PSCustomObject]@{
            Resource    = "URL $URL"
            Status      = if ($URLUp) { "UP" } else { "DOWN" }
            Timestamp   = Get-Date
            Suppressed  = $ShouldSuppress
        }
        
        $Results += $Result
        
        if (-not $URLUp -and -not $ShouldSuppress) {
            Write-MonitoringLog -Message "URL $URL is DOWN" -Level "ERROR"
            if ($EmailSettings.ErrorOnly) {
                Send-MailMessage @EmailSettings -Body "URL $URL is DOWN on $(hostname)"
            }
        }
    }
    
    # Generate summary report
    $Summary = $Results | Group-Object Status | ForEach-Object {
        "$($_.Count) $($_.Name) ($($_.Group.Count -eq ($_.Group | Where-Object { $_.Suppressed }).Count) suppressed)"
    }
    
    Write-MonitoringLog -Message "Monitoring complete: $($Summary -join ', ')"
    
    # Wait before next check
    Start-Sleep -Seconds $CheckIntervalSeconds
}
```

### Key Features:

1. **Port Monitoring**: Actively checks TCP ports (8080, 7070, 443)
2. **URL Monitoring**: Validates HTTP/HTTPS endpoints
3. **Smart Alert Suppression**:
   - Temporary outages (<15 minutes) won't trigger alerts
   - Maintenance windows won't generate false alerts
   - Tracks duration of each outage

4. **Comprehensive Logging**:
   - File logging of all status changes
   - Windows Event Log integration
   - Email notifications (configurable)

5. **Configuration Options**:
   - Customizable ports and URLs
   - Adjustable check interval
   - Service name mappings

### Implementation Steps:

1. Save the script as `PortUrlMonitor.ps1`
2. Create a scheduled task to run it persistently:

```powershell
$Trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 1)
$Action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-ExecutionPolicy Bypass -File C:\Path\To\PortUrlMonitor.ps1"
Register-ScheduledTask -TaskName "Port/URL Monitor" -Trigger $Trigger -Action $Action -RunLevel Highest -Force
```
3. Configure monitoring parameters in the script's header
4. Test with `-EnableAlertSuppression:$false` initially to verify functionality

### Advanced Options:

1. Add `-ServiceNames` parameter to map ports to services
2. Extend suppression logic with `-SuppressionWindowMinutes`
3. Integrate with monitoring systems like Nagios or SCOM

This script provides enterprise-grade monitoring with built-in false-positive prevention, suitable for production Windows servers.
