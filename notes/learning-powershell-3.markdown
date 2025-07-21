---
title: Learning Powershell 3
permalink: /learning-powershell-3.html
---

# Learning PowerShell – Part 3: Real-World Scenarios

These are some of the use cases for Powershell.

## Scenario 1: Audit Installed Software

Useful for inventory, compliance, or incident response.

```powershell
Get-ItemProperty HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* |
Select-Object DisplayName, DisplayVersion, Publisher, InstallDate |
Sort-Object DisplayName
```

For remote audit:

```powershell
Invoke-Command -ComputerName "Workstation01" -ScriptBlock {
    Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
    Select-Object DisplayName, DisplayVersion
}
```

---

## Scenario 2: Find Large Files on a Drive

```powershell
Get-ChildItem -Path C:\ -Recurse -File -ErrorAction SilentlyContinue |
Where-Object { $_.Length -gt 500MB } |
Sort-Object Length -Descending |
Select-Object FullName, @{Name="SizeMB"; Expression={"{0:N2}" -f ($_.Length / 1MB)}}
```

This is great for cleaning up disk space.

---

## Scenario 3: Restart a Service on Multiple Machines

```powershell
$computers = @("Server01", "Server02", "Server03")

Invoke-Command -ComputerName $computers -ScriptBlock {
    Restart-Service -Name "Spooler" -Force
}
```

Useful for applying patches or resetting services.

---

## Scenario 4: Get Last Logon for Domain Users (AD)

Requires `RSAT` and `ActiveDirectory` module.

```powershell
Get-ADUser -Filter * -Property LastLogonDate |
Select-Object Name, SamAccountName, LastLogonDate |
Sort-Object LastLogonDate -Descending
```

---

## Scenario 5: Export Local Admin Members to CSV

```powershell
$computers = @("PC01", "PC02", "PC03")

foreach ($computer in $computers) {
    Invoke-Command -ComputerName $computer -ScriptBlock {
        Get-LocalGroupMember -Group "Administrators"
    } | Select-Object @{Name="ComputerName"; Expression={$computer}}, Name |
    Export-Csv -Path "admins_$computer.csv" -Append -NoTypeInformation
}
```

This helps during audits or privilege reviews.

---

## Scenario 6: Verify Antivirus Status

For Defender:

```powershell
Get-MpComputerStatus | Select-Object AMServiceEnabled, AntivirusEnabled, RealTimeProtectionEnabled
```

Use this in compliance checks or incident triage.

---

## Scenario 7: Send Alert Email on Disk Threshold

```powershell
$threshold = 15
$drives = Get-PSDrive -PSProvider FileSystem

foreach ($drive in $drives) {
    $freePercent = ($drive.Free / $drive.Used) * 100
    if ($freePercent -lt $threshold) {
        Send-MailMessage -To "admin@domain.com" -From "monitor@domain.com" `
            -Subject "Low Disk Space Alert on $($drive.Name): $([math]::Round($freePercent))% free" `
            -SmtpServer "smtp.domain.com"
    }
}
```

---

## Scenario 8: Backup Configuration Files

```powershell
$sourcePaths = @(
    "C:\ProgramData\App\Config.xml",
    "C:\inetpub\wwwroot\web.config"
)

foreach ($path in $sourcePaths) {
    Copy-Item -Path $path -Destination "D:\Backups\" -Force
}
```

Combine with `Task Scheduler` or `Register-ScheduledJob` for automation.

---

## Scenario 9: Monitor a Website’s Health

```powershell
$response = Invoke-WebRequest -Uri "https://yourwebsite.com" -UseBasicParsing

if ($response.StatusCode -ne 200) {
    Write-Warning "Website is down! Status code: $($response.StatusCode)"
}
```

Enhance with logging and email for production use.

---

## Scenario 10: Reset Password for Multiple Users

```powershell
Import-Module ActiveDirectory

$users = Get-Content .\userlist.txt
foreach ($user in $users) {
    Set-ADAccountPassword -Identity $user -Reset -NewPassword (ConvertTo-SecureString "N3wP@ss!" -AsPlainText -Force)
    Enable-ADAccount -Identity $user
}
```

You can combine this with CSV input and reporting.

---

## Bonus: Create a Self-Updating Log Script

```powershell
$date = Get-Date -Format "yyyy-MM-dd_HH-mm"
$logPath = "C:\Logs\HealthCheck_$date.log"

Get-Service | Where-Object {$_.Status -ne "Running"} | Out-File $logPath
```

Run on a schedule and email/report log files automatically.
