# Lab 2.7 — Active Directory Group Policy Administration Fundamentals

## Objective

Practise the basic Group Policy lifecycle in an Active Directory domain:

* inspect the existing GPOs
* create/use a test GPO
* link it to a specific OU
* configure a harmless computer setting
* verify the setting and scope
* check actual Group Policy processing
* remove the link and delete the test GPO
* verify the final state

The lab was performed in the `corp.lab` test domain.

---

## Environment

* Windows Server 2025
* Windows PowerShell 5.1
* Active Directory PowerShell module
* GroupPolicy PowerShell module
* Domain: `corp.lab`

Existing lab structure:

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

`LAB-TEST-PC01` is an AD computer object and was disabled. It was not a live endpoint.

---

## 1. Confirm the domain

```powershell
$Domain = Get-ADDomain
$Domain | Select-Object DNSRoot, DistinguishedName
```

Output:

```text
DNSRoot  DistinguishedName
-------  -----------------
corp.lab DC=corp,DC=lab
```

---

## 2. Check the lab OUs

```powershell
Get-ADOrganizationalUnit -Filter "Name -like 'Lab-*'" |
    Select-Object Name, DistinguishedName, ProtectedFromAccidentalDeletion |
    Sort-Object DistinguishedName
```

Output:

```text
Lab-Computers OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
Lab-IT        OU=Lab-IT,DC=corp,DC=lab
Lab-Users     OU=Lab-Users,OU=Lab-IT,DC=corp,DC=lab
```

The target OU for this lab was:

```text
OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
```

---

## 3. Confirm the GroupPolicy module

```powershell
Get-Module -ListAvailable -Name GroupPolicy |
    Select-Object Name, Version, Path
```

Output:

```text
GroupPolicy 1.0.0.0 C:\WINDOWS\system32\WindowsPowerShell\v1.0\Modules\GroupPolicy\GroupPolicy.psd1
```

---

## 4. Check the existing GPOs

```powershell
Get-GPO -All |
    Select-Object DisplayName, Id, GpoStatus |
    Sort-Object DisplayName
```

Output:

```text
Default Domain Controllers Policy 6ac1786c-016f-11d2-945f-00c04fb984f9 AllSettingsEnabled
Default Domain Policy             31b2f340-016d-11d2-945f-00c04fb984f9 AllSettingsEnabled
```

I did not modify either of the default GPOs.

---

## 5. Resolve the target OU

```powershell
$LabComputersOU = Get-ADOrganizationalUnit `
    -Identity "OU=Lab-Computers,OU=Lab-IT,$($Domain.DistinguishedName)" `
    -ErrorAction Stop

$LabComputersOU |
    Select-Object Name, DistinguishedName, ProtectedFromAccidentalDeletion
```

Output:

```text
Name          DistinguishedName
Lab-Computers OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
```

---

## 6. Check the test computer object

```powershell
Get-ADComputer -Identity "LAB-TEST-PC01" |
    Select-Object Name, Enabled, DNSHostName, DistinguishedName
```

Output:

```text
Name            Enabled DNSHostName DistinguishedName
----            ------- ----------- -----------------
LAB-TEST-PC01   False               CN=LAB-TEST-PC01,OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
```

There was no live computer associated with this object.

---

## 7. Test GPO creation

I attempted to create the planned test GPO:

```powershell
$TestGPO = New-GPO -Name "Lab-IT-Test-Policy" -ErrorAction Stop
```

PowerShell returned:

```text
New-GPO : The command cannot be completed because a "Lab-IT-Test-Policy" GPO already exists in the corp.lab domain.
FullyQualifiedErrorId : GpoWithNameAlreadyExists
```

So the GPO already existed. I did not recreate it.

I inspected the existing object instead:

```powershell
Get-GPO -Name "Lab-IT-Test-Policy" |
    Select-Object DisplayName, Id, GpoStatus
```

Output:

```text
DisplayName        Id                                            GpoStatus
-----------        --                                            ---------
Lab-IT-Test-Policy 7d84da3d-02f9-40db-adb5-a9e24670ea9f          AllSettingsEnabled
```

Its recorded creation time was:

```text
CreationTime     : 08/09/2026 18:43:15
ModificationTime : 08/09/2026 18:43:14
```

The original creator was not established during the lab.

---

## 8. Check the GPO before configuration

```powershell
Get-GPOReport -Name "Lab-IT-Test-Policy" -ReportType Xml
```

The report showed:

```text
Name: Lab-IT-Test-Policy
GUID: {7D84DA3D-02F9-40DB-ADB5-A9E24670EA9F}
Domain: corp.lab
Owner: CORP\Domain Admins
```

The Computer Configuration version was initially:

```text
VersionDirectory: 0
VersionSysvol:    0
```

There were no substantive test settings configured at this point.

---

## 9. Check existing GPO inheritance

Before linking the test GPO:

```powershell
Get-GPInheritance -Target $LabComputersOU.DistinguishedName
```

Output:

```text
Name                  : lab-computers
ContainerType         : OU
Path                  : ou=lab-computers,ou=lab-it,dc=corp,dc=lab
GpoInheritanceBlocked : No
GpoLinks              : {}
InheritedGpoLinks     : {Default Domain Policy}
```

The target OU therefore had no direct GPO links before the test.

---

## 10. Link the test GPO

```powershell
New-GPLink `
    -Name "Lab-IT-Test-Policy" `
    -Target "OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab" `
    -LinkEnabled Yes
```

Output:

```text
GpoId       : 7d84da3d-02f9-40db-adb5-a9e24670ea9f
DisplayName : Lab-IT-Test-Policy
Enabled     : True
Enforced    : False
Target      : OU=Lab-Computers,OU=Lab-IT,DC=corp,DC=lab
Order       : 1
```

Verify:

```powershell
Get-GPInheritance -Target $LabComputersOU.DistinguishedName
```

Output:

```text
GpoLinks              : {Lab-IT-Test-Policy}
InheritedGpoLinks     : {Lab-IT-Test-Policy, Default Domain Policy}
GpoInheritanceBlocked : No
```

The GPO was now linked directly to `Lab-Computers`.

---

## 11. Configure a harmless test setting

I used a registry value under a lab-specific key rather than changing an actual Windows security setting.

```powershell
Set-GPRegistryValue `
    -Name "Lab-IT-Test-Policy" `
    -Key "HKLM\Software\Lab-IT" `
    -ValueName "PolicyTest" `
    -Type String `
    -Value "Lab-IT-Test-Policy Applied"
```

The GPO version changed from:

```text
ComputerVersion : AD Version: 0, SysVol Version: 0
```

to:

```text
ComputerVersion : AD Version: 1, SysVol Version: 1
```

---

## 12. Verify the GPO setting directly

```powershell
Get-GPRegistryValue `
    -Name "Lab-IT-Test-Policy" `
    -Key "HKLM\Software\Lab-IT"
```

Output:

```text
KeyPath     : Software\Lab-IT
FullKeyPath : HKEY_LOCAL_MACHINE\Software\Lab-IT
Hive        : LocalMachine
PolicyState : Set
Value       : Lab-IT-Test-Policy Applied
Type        : String
ValueName   : PolicyTest
HasValue    : True
```

This confirmed that the setting existed in the GPO itself.

---

## 13. Check the GPO report after configuration

A second XML report showed:

```text
Name: Lab-IT-Test-Policy
GUID: {7D84DA3D-02F9-40DB-ADB5-A9E24670EA9F}
Domain: corp.lab
Owner: CORP\Domain Admins
```

Computer Configuration:

```text
VersionDirectory: 1
VersionSysvol:    1
Enabled:          true
```

Configured registry setting:

```text
Software\Lab-IT\PolicyTest = Lab-IT-Test-Policy Applied
```

The report also showed the link to `Lab-Computers` as enabled.

---

## 14. Find a live domain-joined computer

I checked the machine on which the commands were being run:

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object Name, Domain, PartOfDomain
```

Output:

```text
Name            Domain   PartOfDomain
----            ------   ------------
WIN-URRN4NJRE9I corp.lab True
```

I then checked its AD object:

```powershell
Get-ADComputer -Identity "WIN-URRN4NJRE9I" |
    Select-Object Name, Enabled, DistinguishedName
```

Output:

```text
Name            Enabled DistinguishedName
----            ------- -----------------
WIN-URRN4NJRE9I True    CN=WIN-URRN4NJRE9I,OU=Domain Controllers,DC=corp,DC=lab
```

This machine is a Domain Controller.

I therefore did **not** move it into `Lab-Computers` just to test the GPO.

---

## 15. Check actual Group Policy processing

```powershell
gpresult /r /scope computer
```

Relevant output:

```text
RSOP data ... on WIN-URRN4NJRE9I

OS Configuration: Primary Domain Controller

COMPUTER SETTINGS

CN=WIN-URRN4NJRE9I,OU=Domain Controllers,DC=corp,DC=lab

Applied Group Policy Objects
    Default Domain Controllers Policy
    Default Domain Policy
```

`Lab-IT-Test-Policy` was not applied to this machine because the machine is located in the `Domain Controllers` OU, not `Lab-Computers`.

This was useful because it demonstrated the difference between:

```text
GPO exists
    ↓
GPO is linked
    ↓
Object is in the linked OU
    ↓
GPO is in scope
    ↓
GPO is processed
    ↓
Setting appears on the computer
```

The first two conditions were true for the test GPO, but the live Domain Controller was outside its scope.

---

## 16. Verify the setting directly on the live machine

```powershell
Get-ItemProperty `
    -Path "HKLM:\Software\Lab-IT" `
    -Name "PolicyTest" `
    -ErrorAction SilentlyContinue
```

There was no output.

This independently confirmed that the test registry value had not been applied to the Domain Controller.

---

## 17. Remove the GPO link

Before cleanup, the link was still present:

```powershell
Get-GPInheritance -Target $LabComputersOU.DistinguishedName
```

Output included:

```text
GpoLinks : {Lab-IT-Test-Policy}
```

Remove the link:

```powershell
Remove-GPLink `
    -Name "Lab-IT-Test-Policy" `
    -Target $LabComputersOU.DistinguishedName
```

Verify:

```powershell
Get-GPInheritance -Target $LabComputersOU.DistinguishedName
```

Output:

```text
GpoLinks              : {}
InheritedGpoLinks     : {Default Domain Policy}
GpoInheritanceBlocked : No
```

The direct link was removed.

---

## 18. Delete the test GPO

```powershell
Remove-GPO -Name "Lab-IT-Test-Policy"
```

Verify that it no longer exists:

```powershell
Get-GPO -Name "Lab-IT-Test-Policy" -ErrorAction SilentlyContinue
```

No output was returned.

The temporary GPO had been deleted.

---

## 19. Final verification

```powershell
Get-GPInheritance -Target $LabComputersOU.DistinguishedName
```

Final output:

```text
Name                  : lab-computers
ContainerType         : OU
Path                  : ou=lab-computers,ou=lab-it,dc=corp,dc=lab
GpoInheritanceBlocked : No
GpoLinks              : {}
InheritedGpoLinks     : {Default Domain Policy}
```

The test GPO was also confirmed absent with:

```powershell
Get-GPO -Name "Lab-IT-Test-Policy" -ErrorAction SilentlyContinue
```

No output.

---

## Result

The GPO lifecycle was completed:

```text
Existing GPO
    ↓
Inspected
    ↓
Linked to Lab-Computers
    ↓
Configured with a harmless registry value
    ↓
Verified in the GPO
    ↓
Checked against a live Domain Controller
    ↓
Confirmed out of scope
    ↓
GPO link removed
    ↓
GPO deleted
    ↓
Final state verified
```

Final state:

* `Lab-IT-Test-Policy` does not exist.
* `Lab-Computers` has no direct GPO links.
* `Default Domain Policy` remains inherited.
* GPO inheritance is not blocked.
* The live Domain Controller was not moved or modified.
* `LAB-TEST-PC01` remains an AD computer object in `Lab-Computers`.
* No temporary GPO remains.

## What I learned

The main point of the lab was not just creating a GPO. It was understanding **scope and processing**.

A GPO can exist and be linked correctly without being applied to a particular computer. The location of the computer object in AD determines whether the linked GPO is in scope.

The lab also reinforced the importance of verifying each stage independently rather than assuming that a successful PowerShell command means the intended configuration was actually applied.
