# Lab 2.6 — Active Directory Organisational Units

## Purpose

This lab demonstrates practical Active Directory administration using PowerShell, focusing on:

* Organisational Units (OUs)
* Object placement
* Accidental-deletion protection
* User and computer administration
* Validation
* Basic automation
* Configuration troubleshooting

The methodology throughout the lab was:

> **Do something → immediately verify it → understand the result → continue.**

The lab environment was:

| Component  | Value                              |
| ---------- | ---------------------------------- |
| Domain     | `corp.lab`                         |
| Domain DN  | `DC=corp,DC=lab`                   |
| PowerShell | Windows PowerShell 5.1             |
| AD Module  | Active Directory PowerShell module |
| Server     | Windows Server 2025 VM             |

---

## 1. Establish the Domain DN

We first obtained the domain Distinguished Name so that subsequent commands could construct Active Directory paths reliably.

```powershell
$DomainDN = (Get-ADDomain).DistinguishedName
```

**Result:**

```text
DC=corp,DC=lab
```

---

## 2. Create the Main `Lab-IT` OU

The `Lab-IT` OU provides a dedicated administrative area for the objects used by this lab.

```powershell
New-ADOrganizationalUnit `
    -Name "Lab-IT" `
    -Path $DomainDN `
    -ProtectedFromAccidentalDeletion $true `
    -ErrorAction Stop
```

The command completed without error.

**Resulting DN:**

```text
OU=Lab-IT,DC=corp,DC=lab
```

Accidental-deletion protection was enabled because this is an administrative safeguard against unintended OU deletion.

---

## 3. Create `Lab-Users`

A dedicated OU was created for user objects.

```powershell
New-ADOrganizationalUnit `
    -Name "Lab-Users" `
    -Path "OU=Lab-IT,$DomainDN" `
    -ProtectedFromAccidentalDeletion $true `
    -ErrorAction Stop
```

**Resulting DN:**

```text
OU=Lab-Users,OU=Lab-IT,DC=corp,DC=lab
```

An OU is an administrative/container structure and is distinct from an Active Directory group.

A user can:

* reside inside an OU for organisational and administrative purposes;
* belong to one or more groups for access and membership purposes.

---

## 4. Create `Lab-Computers`

A separate OU was created for computer objects.

```powershell
New-ADOrganizationalUnit `
    -Name "Lab-Computers" `
    -Path "OU=Lab-IT,$DomainDN" `
    -ProtectedFromAccidentalDeletion $true `
    -ErrorAction Stop
```

**Resulting DN:**

```text
OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
```

The intended structure was:

```text
corp.lab
└── Lab-IT
    ├── Lab-Users
    └── Lab-Computers
```

---

## 5. Verify the OU Structure

Rather than assuming the OUs were created correctly, Active Directory was queried directly.

```powershell
Get-ADOrganizationalUnit -Filter "Name -like 'Lab-*'" |
Select-Object Name, DistinguishedName, ProtectedFromAccidentalDeletion |
Sort-Object DistinguishedName
```

**Output:**

```text
Name          DistinguishedName                         ProtectedFromAccidentalDeletion
----          -----------------                         -------------------------------
Lab-Computers OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
Lab-IT        OU=Lab-IT,DC=corp,DC=lab
Lab-Users     OU=Lab-Users,OU=Lab-IT,DC=corp,DC=lab
```

The output confirmed that the intended OU hierarchy existed.

---

## 6. Verify Accidental-Deletion Protection

The protection state of each permanent OU was independently verified.

### `Lab-IT`

```powershell
Get-ADOrganizationalUnit `
    -Identity "OU=Lab-IT,$DomainDN" `
    -Properties ProtectedFromAccidentalDeletion |
Select-Object Name, ProtectedFromAccidentalDeletion
```

**Output:**

```text
Name   ProtectedFromAccidentalDeletion
----   -------------------------------
Lab-IT                            True
```

### `Lab-Users`

```powershell
Get-ADOrganizationalUnit `
    -Identity "OU=Lab-Users,OU=Lab-IT,$DomainDN" `
    -Properties ProtectedFromAccidentalDeletion |
Select-Object Name, ProtectedFromAccidentalDeletion
```

**Output:**

```text
Name      ProtectedFromAccidentalDeletion
----      -------------------------------
Lab-Users                            True
```

### `Lab-Computers`

```powershell
Get-ADOrganizationalUnit `
    -Identity "OU=Lab-Computers,OU=Lab-IT,$DomainDN" `
    -Properties ProtectedFromAccidentalDeletion |
Select-Object Name, ProtectedFromAccidentalDeletion
```

**Output:**

```text
Name          ProtectedFromAccidentalDeletion
----          -------------------------------
Lab-Computers                            True
```

All three permanent OUs were protected.

---

## 7. Locate the Existing Automation User

The existing automation account was:

```text
Name: Lab Automation User01
SamAccountName: labauto01
```

Its initial location was checked.

```powershell
Get-ADUser -Identity "labauto01" |
Select-Object Name, SamAccountName, DistinguishedName
```

**Initial result:**

```text
Name                  SamAccountName DistinguishedName
----                  -------------- -----------------
Lab Automation User01 labauto01      CN=Lab Automation User01,CN=Users,DC=corp,DC=lab
```

The account was initially located in the default `Users` container rather than the new `Lab-Users` OU.

---

## 8. Obtain the `Lab-Users` OU

The OU object was retrieved so that its Distinguished Name could be reused.

```powershell
$LabUsersOU = Get-ADOrganizationalUnit `
    -Identity "OU=Lab-Users,OU=Lab-IT,$DomainDN"
```

The Distinguished Name was then checked:

```powershell
$LabUsersOU.DistinguishedName
```

**Output:**

```text
OU=Lab-Users,OU=Lab-IT,DC=corp,DC=lab
```

---

## 9. Move the User into `Lab-Users`

The user's current Distinguished Name was obtained.

```powershell
$UserDN = (Get-ADUser -Identity "labauto01").DistinguishedName
```

The object was then moved into the target OU.

```powershell
Move-ADObject `
    -Identity $UserDN `
    -TargetPath $LabUsersOU.DistinguishedName `
    -ErrorAction Stop
```

The command completed without error.

---

## 10. Verify the User's New Location

```powershell
Get-ADUser -Identity "labauto01" |
Select-Object Name, SamAccountName, Enabled, DistinguishedName
```

**Output:**

```text
Name                  SamAccountName Enabled DistinguishedName
----                  -------------- ------- -----------------
Lab Automation User01 labauto01         True CN=Lab Automation User01,OU=Lab-Users,OU=Lab-IT,DC=corp,DC=lab
```

This confirmed that:

* the user still existed;
* the account remained enabled;
* the user was now located in `Lab-Users`.

---

## 11. Create a Computer Object

An AD computer object was created to represent a test device.

This was an **Active Directory computer object**, not a physical or virtual computer being joined to the domain.

The object was deliberately created disabled because this exercise concerned AD object administration and placement rather than domain joining.

```powershell
New-ADComputer `
    -Name "LAB-TEST-PC01" `
    -Path "OU=Lab-Computers,OU=Lab-IT,$DomainDN" `
    -Enabled $false `
    -ErrorAction Stop
```

The command completed without error.

---

## 12. Verify the Computer Object

```powershell
Get-ADComputer -Identity "LAB-TEST-PC01" |
Select-Object Name, Enabled, DistinguishedName
```

**Output:**

```text
Name          Enabled DistinguishedName
----          ------- -----------------
LAB-TEST-PC01   False CN=LAB-TEST-PC01,OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
```

This confirmed that:

* the computer object existed;
* it was disabled;
* it was located in `Lab-Computers`.

---

## 13. Create a Disposable `Lab-Temporary` OU

A temporary OU was created specifically to demonstrate accidental-deletion protection safely.

```powershell
New-ADOrganizationalUnit `
    -Name "Lab-Temporary" `
    -Path "OU=Lab-IT,$DomainDN" `
    -ProtectedFromAccidentalDeletion $true `
    -ErrorAction Stop
```

The command completed without error.

---

## 14. Verify `Lab-Temporary` Protection

```powershell
Get-ADOrganizationalUnit `
    -Identity "OU=Lab-Temporary,OU=Lab-IT,$DomainDN" `
    -Properties ProtectedFromAccidentalDeletion |
Select-Object Name, DistinguishedName, ProtectedFromAccidentalDeletion
```

**Output:**

```text
Name          DistinguishedName                         ProtectedFromAccidentalDeletion
----          -----------------                         -------------------------------
Lab-Temporary OU=Lab-Temporary,OU=Lab-IT,DC=corp,DC=lab                            True
```

The temporary OU was confirmed to be protected.

---

## 15. Remove Protection Deliberately

Because `Lab-Temporary` existed specifically for the deletion exercise, its protection was deliberately removed.

```powershell
Set-ADOrganizationalUnit `
    -Identity "OU=Lab-Temporary,OU=Lab-IT,$DomainDN" `
    -ProtectedFromAccidentalDeletion $false `
    -ErrorAction Stop
```

The state was then verified:

```powershell
Get-ADOrganizationalUnit `
    -Identity "OU=Lab-Temporary,OU=Lab-IT,$DomainDN" `
    -Properties ProtectedFromAccidentalDeletion |
Select-Object Name, ProtectedFromAccidentalDeletion
```

**Output:**

```text
Name          ProtectedFromAccidentalDeletion
----          -------------------------------
Lab-Temporary                           False
```

Only after explicitly confirming that protection was disabled was the OU deleted.

---

## 16. Delete the Disposable OU

```powershell
Remove-ADOrganizationalUnit `
    -Identity "OU=Lab-Temporary,OU=Lab-IT,$DomainDN" `
    -Confirm:$false `
    -ErrorAction Stop
```

The command completed without error.

---

## 17. Verify Deletion

The OU was queried directly:

```powershell
Get-ADOrganizationalUnit `
    -Identity "OU=Lab-Temporary,OU=Lab-IT,$DomainDN" `
    -ErrorAction SilentlyContinue
```

PowerShell returned:

```text
Get-ADOrganizationalUnit : Directory object not found
```

An independent filter check was also performed:

```powershell
Get-ADOrganizationalUnit -Filter "Name -eq 'Lab-Temporary'" |
Select-Object Name, DistinguishedName
```

No output was returned.

Therefore, `Lab-Temporary` was successfully deleted.

---

## 18. Re-validate the Permanent OUs

```powershell
Get-ADOrganizationalUnit -Filter "Name -like 'Lab-*'" |
Select-Object Name, DistinguishedName, ProtectedFromAccidentalDeletion |
Sort-Object DistinguishedName
```

**Final result:**

```text
Name          DistinguishedName                         ProtectedFromAccidentalDeletion
----          -----------------                         -------------------------------
Lab-Computers OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
Lab-IT        OU=Lab-IT,DC=corp,DC=lab
Lab-Users     OU=Lab-Users,OU=Lab-IT,DC=corp,DC=lab
```

Only the three intended permanent OUs remained.

---

## 19. Final User Verification

```powershell
Get-ADUser -Identity "labauto01" |
Select-Object Name, SamAccountName, Enabled, DistinguishedName
```

**Output:**

```text
Name                  SamAccountName Enabled DistinguishedName
----                  -------------- ------- -----------------
Lab Automation User01 labauto01         True CN=Lab Automation User01,OU=Lab-Users,OU=Lab-IT,DC=corp,DC=lab
```

**Final user state:**

```text
Lab Automation User01
SamAccountName: labauto01
Enabled: True
OU: Lab-Users
```

---

## 20. Final Computer Verification

```powershell
Get-ADComputer -Identity "LAB-TEST-PC01" |
Select-Object Name, Enabled, DistinguishedName
```

**Output:**

```text
Name          Enabled DistinguishedName
----          ------- -----------------
LAB-TEST-PC01   False CN=LAB-TEST-PC01,OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
```

**Final computer state:**

```text
LAB-TEST-PC01
Enabled: False
OU: Lab-Computers
```

---

## 21. Basic Automation Pass

After completing the administration manually, PowerShell logic was used to demonstrate repeatability and state validation.

```powershell
$DomainDN = (Get-ADDomain).DistinguishedName

$LabIT = Get-ADOrganizationalUnit `
    -Identity "OU=Lab-IT,$DomainDN" `
    -ErrorAction Stop

$LabComputers = Get-ADOrganizationalUnit `
    -Identity "OU=Lab-Computers,$($LabIT.DistinguishedName)" `
    -ErrorAction Stop

$LabComputers = Get-ADOrganizationalUnit `
    -Identity "OU=Lab-Computers,$LabIT" `
    -ErrorAction Stop

$User = Get-ADUser `
    -Identity "labauto01" `
    -ErrorAction Stop

if ($User.DistinguishedName -notlike "*,OU=Lab-Users,$LabIT") {
    Move-ADObject `
        -Identity $User.DistinguishedName `
        -TargetPath $LabUsers.DistinguishedName `
        -ErrorAction Stop
}

$Computer = Get-ADComputer `
    -Identity "LAB-TEST-PC01" `
    -ErrorAction Stop

[PSCustomObject]@{
    Domain           = $DomainDN
    ParentOU         = $LabIT.DistinguishedName
    UsersOU          = $LabUsers.DistinguishedName
    ComputersOU      = $LabComputers.DistinguishedName
    UserLocation     = $User.DistinguishedName
    ComputerLocation = $Computer.DistinguishedName
}
```

**Output:**

```text
Domain           : DC=corp,DC=lab
ParentOU         : OU=Lab-IT,DC=corp,DC=lab
UsersOU          : OU=Lab-Users,OU=Lab-IT,DC=corp,DC=lab
ComputersOU      : OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
UserLocation     : CN=Lab Automation User01,OU=Lab-Users,OU=Lab-IT,DC=corp,DC=lab
ComputerLocation : CN=LAB-TEST-PC01,OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
```

The automation completed successfully.

The important learning point is that the administration was first understood and performed manually before introducing automation to demonstrate repeatability.

---

## 22. Final Active Directory State

```text
corp.lab
└── Lab-IT
    ├── Lab-Users
    │   └── Lab Automation User01
    │       └── labauto01
    │
    └── Lab-Computers
        └── LAB-TEST-PC01
```

### Permanent OUs

```text
OU=Lab-IT,DC=corp,DC=lab
OU=Lab-Users,OU=Lab-IT,DC=corp,DC=lab
OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
```

All permanent OUs have:

```text
ProtectedFromAccidentalDeletion = True
```

### Temporary OU

```text
Lab-Temporary
```

**Status:**

```text
Deleted and independently verified absent.
```

### User

```text
labauto01
Enabled: True
Located in: Lab-Users
```

### Computer

```text
LAB-TEST-PC01
Enabled: False
Located in: Lab-Computers
```

---

## 23. Timezone Configuration Issue

During the lab, the Windows Server VM was found to be using Pacific Time instead of the appropriate timezone for Spain.

The configuration was checked with:

```powershell
Get-TimeZone
```

The incorrect configuration showed:

```text
Id          : Pacific Standard Time
DisplayName : (UTC-08:00) Pacific Time (US & Canada)
```

The VM timezone was corrected to the appropriate Windows timezone for Spain:

```powershell
Set-TimeZone -Id "Romance Standard Time"
```

The system time was then checked:

```powershell
Get-Date -Format "yyyy-MM-dd HH:mm:ss"
```

**Final recorded output:**

```text
2026-08-25 17:14:57
```

The timestamp was therefore recorded after correcting the VM timezone.

---

## 24. Skills Demonstrated

This lab provided practical experience with:

* Active Directory OU creation
* OU hierarchy design
* Distinguished Names (DNs)
* PowerShell Active Directory cmdlets
* User object administration
* Computer object administration
* Moving AD objects between containers
* OU accidental-deletion protection
* Controlled deletion
* Verification-driven administration
* Basic PowerShell automation
* State validation
* Troubleshooting a Windows Server timezone configuration

---

## 25. Administrative Methodology

The core methodology used throughout the exercise was:

> **Change → verify → understand → continue.**

This is preferable to treating successful command execution as proof that the desired configuration has actually been achieved.

For infrastructure administration, the important distinction is:

```text
Command succeeded
        ≠
Desired state confirmed
```

The lab therefore repeatedly queried Active Directory after changes to establish the resulting state.

---

## 26. Completion Status

**Lab 2.6 — Complete**

**Final state:**

```text
Domain:
corp.lab

Parent OU:
OU=Lab-IT,DC=corp,DC=lab

User OU:
OU=Lab-Users,OU=Lab-IT,DC=corp,DC=lab

Computer OU:
OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab

User:
labauto01
Enabled: True

Computer:
LAB-TEST-PC01
Enabled: False

Permanent OUs:
ProtectedFromAccidentalDeletion = True

Temporary OU:
Deleted and independently verified absent.

Server timezone:
Romance Standard Time
```
