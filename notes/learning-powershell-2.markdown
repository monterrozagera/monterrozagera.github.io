---
title: Learning Powershell 2
permalink: /learning-powershell-2.html
---

# Learning PowerShell – Part 2: Advanced Concepts

## Functions and Parameters

Define reusable blocks of code using `function`.

```powershell
function Get-Greeting {
    param (
        [string]$Name = "Guest"
    )
    return "Hello, $Name!"
}

Get-Greeting -Name "Kami"
```

---

## Error Handling with Try/Catch/Finally

```powershell
try {
    Get-Content "missingfile.txt"
}
catch {
    Write-Warning "The file does not exist: $_"
}
finally {
    Write-Output "Operation finished."
}
```

You can also use `-ErrorAction Stop` to force a terminating error:

```powershell
Get-Item "C:\badpath" -ErrorAction Stop
```

---

## Creating Your Own Module

Create a reusable module by saving functions in a `.psm1` file.

```powershell
# MyTools.psm1
function Say-Hello {
    param($Name)
    "Hello from the module, $Name!"
}
```

Load it with:

```powershell
Import-Module .\MyTools.psm1
Say-Hello -Name "Kami"
```

---

## PowerShell Remoting

PowerShell can run scripts on remote systems using `WinRM` or `SSH`.

### Enable on local machine:

```powershell
Enable-PSRemoting -Force
```

### Connect to a remote machine:

```powershell
Enter-PSSession -ComputerName SERVER01 -Credential (Get-Credential)
```

### Run a command remotely:

```powershell
Invoke-Command -ComputerName SERVER01 -ScriptBlock {
    Get-Service | Where-Object {$_.Status -eq "Running"}
}
```

---

## Working with Files and JSON

### Read and convert JSON:

```powershell
$data = Get-Content "config.json" | ConvertFrom-Json
$data.ServerName
```

### Write JSON:

```powershell
$data = @{
    ServerName = "App01"
    Port = 443
}
$data | ConvertTo-Json | Set-Content "config.json"
```

---

## Securing Credentials

Avoid storing plain text passwords.

```powershell
# Save credentials securely
Get-Credential | Export-Clixml -Path "creds.xml"

# Load them securely
$cred = Import-Clixml -Path "creds.xml"
```

Use `$cred` with `Invoke-Command`, `New-PSSession`, etc.

---

## Custom Prompt (PowerShell Profile)

Customize your terminal by editing your profile script:

```powershell
notepad $PROFILE
```

Example prompt:

```powershell
function prompt {
    "[$env:USERNAME@$env:COMPUTERNAME] PS $PWD> "
}
```

---

## Unit Testing with Pester

Install the module:

```powershell
Install-Module Pester -Force
```

Create a test file:

```powershell
# MyTools.Tests.ps1
Describe "Say-Hello" {
    It "Returns correct greeting" {
        Say-Hello -Name "Kami" | Should -Be "Hello from the module, Kami!"
    }
}
```

Run tests:

```powershell
Invoke-Pester .\MyTools.Tests.ps1
```

---

## Scheduled Jobs

Unlike Task Scheduler, PowerShell can manage its own jobs:

```powershell
$trigger = New-JobTrigger -Daily -At "7am"
Register-ScheduledJob -Name "BackupJob" -ScriptBlock {
    Copy-Item "C:\Important" -Destination "D:\Backup" -Recurse
} -Trigger $trigger
```

---

## Real-World Use Case: Audit Local Admins

```powershell
Get-LocalGroupMember -Group "Administrators" | Select-Object Name, ObjectClass
```

Or for remote machines:

```powershell
Invoke-Command -ComputerName PC01 -ScriptBlock {
    Get-LocalGroupMember -Group "Administrators"
}
```

---

## Real-World Use Case: Check Disk Space Remotely

```powershell
Invoke-Command -ComputerName "Server01" -ScriptBlock {
    Get-PSDrive -PSProvider FileSystem | Where-Object {$_.Used -gt 0}
}
```

Check part 3 of my Powershell notes [here](./learning-powershell-3.markdown).