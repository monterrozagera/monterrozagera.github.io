---
title: Learning Powershell 1
permalink: /learning-powershell-1.html
---

# Learning PowerShell – Part 1:

PowerShell is a task-based command-line shell and scripting language built on .NET. It is widely used for automation of system administration tasks across Windows, Linux, and macOS.

This article walks you through PowerShell basics, common commands, and some advanced scripting examples.

---

## What is PowerShell?

PowerShell combines the power of the command line with the flexibility of scripting. It uses **cmdlets** (command-lets), which are specialized .NET classes implementing a specific operation.

Example of a cmdlet:

```powershell
Get-Process
```

This lists all currently running processes.

---

## Basic Commands

### List Files

```powershell
Get-ChildItem
```

Shorthand:

```powershell
ls
```

Output:

```
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----       2025-07-20     10:14                Documents
-a----       2025-07-18     14:00           2048 notes.txt
```

---

### Change Directory

```powershell
Set-Location "C:\Users\Kami\Documents"
```

Shorthand:

```powershell
cd "C:\Users\Kami\Documents"
```

---

### Create a File

```powershell
New-Item -Path . -Name "example.txt" -ItemType "File"
```

---

### Read File Contents

```powershell
Get-Content example.txt
```

---

### Write to a File

```powershell
"Hello World from PowerShell!" | Out-File example.txt
```

---

### Append to a File

```powershell
"Adding more text..." | Add-Content example.txt
```

---

## Variables and Objects

```powershell
$name = "Kami"
Write-Output "Hello, $name!"
```

Everything in PowerShell is an object. You can access properties like this:

```powershell
$date = Get-Date
$date.DayOfWeek
```

---

## Loops and Conditionals

### If-Else

```powershell
$number = 10
if ($number -gt 5) {
    "Greater than 5"
} else {
    "5 or less"
}
```

### ForEach Loop

```powershell
$items = 1..5
foreach ($i in $items) {
    Write-Output "Number: $i"
}
```

---

## Installing Modules

You can install modules from the PowerShell Gallery:

```powershell
Install-Module -Name Az -Scope CurrentUser
```

---

## Getting Help

PowerShell has built-in help:

```powershell
Get-Help Get-Process
```

To update help files:

```powershell
Update-Help
```

---

## Automating Tasks

### Scheduled Task Example

Create a task that runs a script every day at 9 AM:

```powershell
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-File C:\Scripts\backup.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At 9am
Register-ScheduledTask -Action $action -Trigger $trigger -TaskName "DailyBackup" -Description "Runs backup script"
```

---

## Example Script

```powershell
# backup.ps1
$source = "C:\Users\Kami\Documents"
$destination = "D:\Backups\Documents"
Copy-Item -Path $source -Destination $destination -Recurse
```

---

## Test Your Script

Use `-WhatIf` to preview commands without making changes:

```powershell
Remove-Item C:\Temp\* -WhatIf
```

Check part 2 of my Powershell notes [here](./learning-powershell-2.markdown).