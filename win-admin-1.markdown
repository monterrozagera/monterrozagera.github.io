---
title: High CPU Usage Due to a Runaway Process on Multiple Windows Servers
permalink: /win-admin-1.html
---

# Windows Admin Scenario: High CPU Usage Due to a Runaway Process on Multiple Windows Servers

### Issue:

A fleet of Windows Server 2019 EC2 instances started experiencing **high CPU usage**, causing degraded application performance. The root cause was an unresponsive `.NET` application process consuming excessive resources. Restarting the process manually was inefficient at scale.

---

## 1. PowerShell Script for Automated Detection and Recovery

### Script Goal:

* Identify processes consuming over 80% CPU.
* Automatically restart them.
* Log the event for audit purposes.
* Run across multiple servers using PowerShell Remoting.

---

### PowerShell Script:

```powershell
# Define parameters
$ThresholdCPU = 80
$TargetProcess = "MyApp.exe"
$LogPath = "C:\Logs\ProcessMonitor.log"
$Servers = @("SERVER01", "SERVER02", "SERVER03")  # Replace with real hostnames or pull from Active Directory

# Function to check and restart the process
function Monitor-HighCPU {
    param (
        [string]$ComputerName
    )

    try {
        Invoke-Command -ComputerName $ComputerName -ScriptBlock {
            param($TargetProcess, $ThresholdCPU, $LogPath)

            $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
            $process = Get-Process -Name $TargetProcess -ErrorAction SilentlyContinue | 
                       Sort-Object CPU -Descending | 
                       Select-Object -First 1

            if ($process -and $process.CPU -gt $ThresholdCPU) {
                $logEntry = "$timestamp | $env:COMPUTERNAME | Restarting $TargetProcess (CPU: $($process.CPU)%)"
                Add-Content -Path $LogPath -Value $logEntry
                Stop-Process -Id $process.Id -Force
                Start-Process -FilePath $process.Path
            }
        } -ArgumentList $TargetProcess, $ThresholdCPU, $LogPath -ErrorAction Stop
    } catch {
        Write-Warning "Failed to connect to $ComputerName: $_"
    }
}

# Loop through servers
foreach ($HighCPU -ComputerName $server
}
```

---

### Requirements:

* PowerShell Remoting enabled (`Enable-PSRemoting`)
* Proper firewall rules and credentials
* Your account must have permission to restart processes on remote machines

---

### Example Log Output:

```
2025-07-18 10:33:05 | SERVER01 | Restarting MyApp.exe (CPU: 91.3%)
2025-07-18 10:33:12 | SERVER02 | Restarting MyApp.exe (CPU: 85.7%)
```

---

## 2. Run It via AWS Systems Manager → **Run Command**

1. Go to **AWS Console → Systems Manager → Run Command**
2. Choose: **AWS-RunPowerShellScript**
3. Select target EC2 instances
4. Paste the content of `HighCPURemediation.ps1` into the script box
5. (Optional) Set output to **CloudWatch Logs**

---

## 3. Schedule It with **State Manager** or **CloudWatch Events**

To run this **every 5 or 10 minutes**:

### Option A: **SSM State Manager**

1. Go to **Systems Manager > State Manager**
2. Create an association:

   * Document: `AWS-RunPowerShellScript`
   * Target instances: choose tag-based or specific
   * Schedule: `rate(5 minutes)`
   * Script Source: Upload or paste `HighCPURemediation.ps1`

### Option B: **CloudWatch Event Rule + SSM Automation**

1. Create a rule:

   * Event Source: Schedule → every 5 minutes
   * Target: **SSM Run Command**
   * Document: `AWS-RunPowerShellScript`
   * Input: the script from above

---

## 4. Optional: Push Logs to CloudWatch

To push logs to CloudWatch:

* Install the **CloudWatch Agent**.
* Configure it to collect `C:\SSM\CPUWatch.log`.

Example JSON snippet in CloudWatch Agent config:

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "C:\\SSM\\CPUWatch.log",
            "log_group_name": "/windows/highcpu",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```
