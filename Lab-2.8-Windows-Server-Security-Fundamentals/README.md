# Lab 2.8 — Windows Server Security Fundamentals

## Objective

Practise fundamental Windows Server security administration in a controlled Active Directory environment:

* identify the Windows Server and its role
* inspect existing users and security groups
* create a disposable NTFS test environment
* configure group-based NTFS permissions
* verify ACL inheritance
* examine effective access prerequisites
* configure and verify File System auditing
* investigate the relationship between security configuration and audit telemetry
* troubleshoot a failed SACL configuration
* document an environment/security-token limitation
* clean up all temporary resources
* automate the verified parts of the workflow with PowerShell
* verify the automated configuration and cleanup

The lab was performed in the `corp.lab` test domain.

---

## Environment

* Windows Server 2025 Standard Evaluation
* Windows PowerShell 5.1
* Active Directory PowerShell module
* Domain: `corp.lab`
* Domain Controller: `WIN-URRN4NJRE9I`
* Current administrative account: `CORP\Administrator`

The server was confirmed to be a Domain Controller:

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object Name,Domain,DomainRole
```

Output:

```text
Name              Domain    DomainRole
----              ------    ----------
WIN-URRN4NJRE9I   corp.lab  5
```

`DomainRole 5` identifies the machine as a Primary Domain Controller.

The operating system was confirmed as:

```text
Windows Server 2025 Standard Evaluation
Build 26100
Server
```

The current security context was:

```powershell
whoami
```

Output:

```text
corp\administrator
```

---

## 1. Inspect the existing lab user

The existing automation account was inspected before making any changes:

```powershell
Get-ADUser -Identity labauto01 -Properties Enabled,MemberOf |
    Select-Object SamAccountName,Enabled,MemberOf
```

Output:

```text
SamAccountName Enabled MemberOf
-------------- ------- --------
labauto01      True    {CN=Lab-IT-Support,CN=Users,DC=corp,DC=lab}
```

The group membership was then independently verified:

```powershell
Get-ADGroupMember -Identity "Lab-IT-Support" |
    Select-Object Name,SamAccountName,ObjectClass
```

Output:

```text
Name                    SamAccountName ObjectClass
----                    -------------- -----------
Lab Automation User01   labauto01      user
```

The group itself was confirmed as a security group:

```powershell
Get-ADGroup -Identity 'Lab-IT-Support' |
    Select-Object Name,SamAccountName,GroupScope,GroupCategory
```

Output:

```text
Name            SamAccountName    GroupScope GroupCategory
----            --------------    ---------- -------------
Lab-IT-Support  Lab-IT-Support    Global     Security
```

---

## 2. Create the disposable NTFS test environment

A disposable test directory was created rather than modifying an existing system directory:

```powershell
New-Item -Path 'C:\Lab-Security-Test' -ItemType Directory
```

The protected test directory was then created:

```powershell
New-Item -Path 'C:\Lab-Security-Test\Protected-Data' -ItemType Directory
```

A harmless test file was later created inside it:

```powershell
New-Item `
    -Path 'C:\Lab-Security-Test\Protected-Data\Test-File.txt' `
    -ItemType File
```

The test environment was intentionally isolated under:

```text
C:\Lab-Security-Test
```

No existing system directory was used for the permission exercise.

---

## 3. Inspect the baseline NTFS permissions

The baseline ACL of the test directory was inspected:

```powershell
Get-Acl -Path 'C:\Lab-Security-Test\Protected-Data' |
    Format-List Path,Owner,Access
```

The ACL contained the expected inherited Windows permissions, including access for:

* `SYSTEM`
* `BUILTIN\Administrators`
* `BUILTIN\Users`
* `CREATOR OWNER`

The owner was:

```text
BUILTIN\Administrators
```

At this stage, `CORP\Lab-IT-Support` did not have an explicit NTFS permission on the test directory.

---

## 4. Configure group-based NTFS access

The security group `CORP\Lab-IT-Support` was granted `Modify` access to the disposable protected directory.

```powershell
$acl = Get-Acl -Path 'C:\Lab-Security-Test\Protected-Data'

$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    'CORP\Lab-IT-Support',
    'Modify',
    'ContainerInherit,ObjectInherit',
    'None',
    'Allow'
)

$acl.AddAccessRule($rule)

Set-Acl `
    -Path 'C:\Lab-Security-Test\Protected-Data' `
    -AclObject $acl
```

The resulting rule was independently verified:

```powershell
(Get-Acl -Path 'C:\Lab-Security-Test\Protected-Data').Access |
    Where-Object {
        $_.IdentityReference -eq 'CORP\Lab-IT-Support'
    } |
    Select-Object IdentityReference,
                  FileSystemRights,
                  AccessControlType,
                  IsInherited
```

Output:

```text
IdentityReference      FileSystemRights      AccessControlType IsInherited
-----------------      ----------------      ----------------- -----------
CORP\Lab-IT-Support    Modify, Synchronize   Allow             False
```

This demonstrated that the group had an **explicit** `Modify` permission on the protected directory.

---

## 5. Verify NTFS inheritance

The test file was created inside the protected directory and its ACL was inspected:

```powershell
(Get-Acl -Path 'C:\Lab-Security-Test\Protected-Data\Test-File.txt').Access |
    Where-Object {
        $_.IdentityReference -eq 'CORP\Lab-IT-Support'
    } |
    Select-Object IdentityReference,
                  FileSystemRights,
                  AccessControlType,
                  IsInherited
```

Output:

```text
IdentityReference      FileSystemRights      AccessControlType IsInherited
-----------------      ----------------      ----------------- -----------
CORP\Lab-IT-Support    Modify, Synchronize   Allow             True
```

This independently demonstrated that the `Modify` permission configured on the directory was inherited by the file.

---

## 6. Verify the identity-to-permission chain

The lab account's effective group memberships were inspected:

```powershell
Get-ADPrincipalGroupMembership -Identity labauto01 |
    Select-Object Name,GroupScope,GroupCategory
```

Output:

```text
Name              GroupScope GroupCategory
----              ---------- -------------
Domain Users      Global     Security
Lab-IT-Support    Global     Security
```

The relevant security chain was therefore:

```text
labauto01
    ↓
Lab-IT-Support
    ↓
NTFS Modify permission
    ↓
C:\Lab-Security-Test\Protected-Data
```

This established the configured identity and permission relationship.

---

## 7. Attempt to verify actual user access

An attempt was made to run a PowerShell process as `CORP\labauto01` to test actual access behaviour.

The account initially required a password reset because the existing password was not available.

The disposable lab account password was reset:

```powershell
Set-ADAccountPassword `
    -Identity labauto01 `
    -Reset `
    -NewPassword (Read-Host 'Enter new password' -AsSecureString)
```

The account was then verified:

```powershell
Get-ADUser -Identity labauto01 -Properties Enabled,LockedOut |
    Select-Object SamAccountName,Enabled,LockedOut
```

Output:

```text
SamAccountName Enabled LockedOut
-------------- ------- ---------
labauto01      True    False
```

An interactive `runas` attempt subsequently returned:

```text
RUNAS ERROR: Unable to run - powershell.exe
1385: Logon failure: the user has not been granted the requested logon type at this computer.
```

This was important evidence rather than an error to work around blindly.

The server is a Domain Controller, and `labauto01` was not permitted to perform the required interactive logon type on that machine.

The lab therefore did **not** modify Domain Controller user-rights policy merely to force an access test.

The result was documented as a limitation:

```text
Configured NTFS access: Verified
Identity/group relationship: Verified
Actual non-administrative access test: Not available on this Domain Controller
```

The configured permission was therefore not incorrectly represented as an observed access result.

---

## 8. Establish the baseline File System audit policy

The File System audit policy was inspected before modification:

```powershell
auditpol /get /subcategory:"File System"
```

Output:

```text
System audit policy
Category/Subcategory                      Setting
Object Access
  File System                             No Auditing
```

This established the original state.

---

## 9. Enable File System auditing

File System auditing was temporarily enabled for both successful and failed operations:

```powershell
auditpol /set /subcategory:"File System" /success:enable /failure:enable
```

Output:

```text
The command was successfully executed.
```

The effective policy was then verified:

```powershell
auditpol /get /subcategory:"File System"
```

Output:

```text
System audit policy
Category/Subcategory                      Setting
Object Access
  File System                             Success and Failure
```

This demonstrated that the audit policy itself had changed successfully.

---

## 10. Attempt to configure a SACL

A File System Audit Rule was created for `CORP\Lab-IT-Support`:

```powershell
$acl = Get-Acl -Path 'C:\Lab-Security-Test\Protected-Data'

$rule = New-Object System.Security.AccessControl.FileSystemAuditRule(
    'CORP\Lab-IT-Support',
    'CreateFiles,WriteData,Delete',
    'ContainerInherit,ObjectInherit',
    'None',
    'Success,Failure'
)

$acl.AddAuditRule($rule)

Set-Acl `
    -Path 'C:\Lab-Security-Test\Protected-Data' `
    -AclObject $acl
```

The command returned without an error.

However, the audit rules were independently checked:

```powershell
(Get-Acl -Path 'C:\Lab-Security-Test\Protected-Data').Audit |
    Select-Object IdentityReference,
                  FileSystemRights,
                  AuditFlags,
                  IsInherited
```

No audit entries were returned.

The SDDL was also inspected:

```powershell
(Get-Acl -Path 'C:\Lab-Security-Test\Protected-Data').Sddl
```

The resulting SDDL contained the DACL but no `S:` SACL section.

This demonstrated that the SACL had **not actually been configured**, despite `Set-Acl` returning without an error.

This distinction was important:

```text
Command completed
        ≠
Security configuration successfully applied
```

---

## 11. Investigate SeSecurityPrivilege

The privilege required to manage SACLs was inspected:

```powershell
whoami /priv |
    Where-Object { $_ -match 'SeSecurityPrivilege' }
```

Output:

```text
SeSecurityPrivilege    Manage auditing and security log    Disabled
```

The current process was confirmed to have an enabled Administrators group:

```powershell
whoami /groups |
    Select-String 'Administrators'
```

Output:

```text
BUILTIN\Administrators    Alias    S-1-5-32-544
Mandatory group,
Enabled by default, Enabled group, Group owner
```

The process integrity level was also checked:

```powershell
whoami /groups |
    Select-String 'Mandatory Label'
```

Output:

```text
Mandatory Label\High Mandatory Level
```

Therefore, the failure was not explained simply by the PowerShell process lacking administrative elevation.

---

## 12. Verify the local security policy assignment

The local security policy was exported:

```powershell
secedit /export /cfg C:\Windows\Temp\security-policy.cfg
```

The relevant privilege assignment was located:

```powershell
Select-String `
    -Path C:\Windows\Temp\security-policy.cfg `
    -Pattern 'SeSecurityPrivilege'
```

Output:

```text
SeSecurityPrivilege = *S-1-5-32-544
```

`S-1-5-32-544` is the SID for:

```text
BUILTIN\Administrators
```

Therefore the local security policy assigned `SeSecurityPrivilege` to the Administrators group.

The current identity was independently confirmed:

```powershell
whoami /user
```

Output:

```text
User Name          SID
================== ============================================
corp\administrator S-1-5-21-1656827716-2687331999-345050640-500
```

The account was also already a member of the Administrators group.

---

## 13. Investigate the privilege-token behaviour

A PowerShell/.NET helper was used to attempt to enable `SeSecurityPrivilege` in the current process token.

The first implementation did not report the underlying Windows API error correctly.

A corrected implementation was subsequently used to expose the actual Windows error.

The resulting error was:

```text
Not all privileges or groups referenced are assigned to the caller
```

This provided direct evidence that the current process token could not enable the requested privilege for the SACL operation.

The lab therefore did not continue by changing Domain Controller security policy or granting additional rights simply to force the exercise to succeed.

---

## 14. Document the SACL limitation

The SACL exercise was therefore recorded as:

```text
File System audit policy: Successfully configured and verified
SACL rule: Not applied
SeSecurityPrivilege: Present in privilege listing but could not be enabled for the current token
Actual audit event generation: Not demonstrated
```

This was treated as a troubleshooting result rather than a successful SACL configuration.

The important security relationship demonstrated was:

```text
Audit policy
    ↓
SACL
    ↓
Object access
    ↓
Security event
```

The lab successfully demonstrated the first component and investigated why the SACL component could not be completed in this environment.

---

## 15. Restore the audit policy

Because File System auditing was only enabled for the lab exercise, it was returned to its original state:

```powershell
auditpol /set /subcategory:"File System" /success:disable /failure:disable
```

The resulting policy was verified:

```powershell
auditpol /get /subcategory:"File System"
```

Output:

```text
System audit policy
Category/Subcategory                      Setting
Object Access
  File System                             No Auditing
```

The original audit-policy state was therefore restored.

---

# Automation

The lab followed the intended workflow:

```text
Understand manually
        ↓
Perform manually
        ↓
Verify manually
        ↓
Identify stable operations
        ↓
Automate
        ↓
Verify automation
        ↓
Clean up
```

The stable parts of the exercise were then repeated using PowerShell automation.

---

## 16. Define the automation variables

```powershell
$LabRoot = 'C:\Lab-Security-Test'
$ProtectedPath = Join-Path $LabRoot 'Protected-Data'
$TestFile = Join-Path $ProtectedPath 'Test-File.txt'
$SecurityGroup = 'CORP\Lab-IT-Support'

Write-Host "Lab 2.8 - Windows Server Security Fundamentals"
Write-Host "Target: $ProtectedPath"
Write-Host "Security group: $SecurityGroup"
```

Output:

```text
Lab 2.8 - Windows Server Security Fundamentals
Target: C:\Lab-Security-Test\Protected-Data
Security group: CORP\Lab-IT-Support
```

---

## 17. Automate creation of the disposable test environment

The automation included a safety check to prevent accidental reuse of an existing test directory:

```powershell
if (Test-Path $LabRoot) {
    throw "Lab root already exists: $LabRoot"
}

New-Item -Path $ProtectedPath -ItemType Directory -Force | Out-Null
New-Item -Path $TestFile -ItemType File | Out-Null

Write-Host "Created: $ProtectedPath"
Write-Host "Created: $TestFile"
```

Output:

```text
Created: C:\Lab-Security-Test\Protected-Data
Created: C:\Lab-Security-Test\Protected-Data\Test-File.txt
```

---

## 18. Automate the NTFS permission

```powershell
$acl = Get-Acl -Path $ProtectedPath

$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    $SecurityGroup,
    'Modify',
    'ContainerInherit,ObjectInherit',
    'None',
    'Allow'
)

$acl.AddAccessRule($rule)
Set-Acl -Path $ProtectedPath -AclObject $acl

Write-Host "Applied Modify permission for $SecurityGroup"
```

Output:

```text
Applied Modify permission for CORP\Lab-IT-Support
```

The actual ACL was then independently verified:

```powershell
(Get-Acl -Path $ProtectedPath).Access |
    Where-Object { $_.IdentityReference -eq $SecurityGroup } |
    Select-Object IdentityReference,
                  FileSystemRights,
                  AccessControlType,
                  IsInherited
```

Output:

```text
IdentityReference      FileSystemRights      AccessControlType IsInherited
-----------------      ----------------      ----------------- -----------
CORP\Lab-IT-Support    Modify, Synchronize   Allow             False
```

The file inheritance was also independently verified:

```powershell
(Get-Acl -Path $TestFile).Access |
    Where-Object { $_.IdentityReference -eq $SecurityGroup } |
    Select-Object IdentityReference,
                  FileSystemRights,
                  AccessControlType,
                  IsInherited
```

Output:

```text
IdentityReference      FileSystemRights      AccessControlType IsInherited
-----------------      ----------------      ----------------- -----------
CORP\Lab-IT-Support    Modify, Synchronize   Allow             True
```

The automated NTFS configuration therefore reproduced the manually verified result.

---

## 19. Automate File System auditing

```powershell
auditpol /set `
    /subcategory:"File System" `
    /success:enable `
    /failure:enable
```

Output:

```text
The command was successfully executed.
```

The actual resulting state was independently verified:

```powershell
auditpol /get /subcategory:"File System"
```

Output:

```text
System audit policy
Category/Subcategory                      Setting
Object Access
  File System                             Success and Failure
```

The automated audit-policy configuration therefore reproduced the manually verified result.

The automation explicitly reported the known SACL limitation rather than pretending that the SACL had been configured:

```powershell
Write-Host "SACL configuration: NOT APPLIED"
Write-Host "Reason: SeSecurityPrivilege could not be enabled in the current token."
```

Output:

```text
SACL configuration: NOT APPLIED
Reason: SeSecurityPrivilege could not be enabled in the current token.
```

---

## 20. Automated cleanup

The temporary audit policy was restored:

```powershell
auditpol /set `
    /subcategory:"File System" `
    /success:disable `
    /failure:disable
```

The resulting state was verified:

```powershell
auditpol /get /subcategory:"File System"
```

Output:

```text
System audit policy
Category/Subcategory                      Setting
Object Access
  File System                             No Auditing
```

The temporary filesystem was then removed:

```powershell
Remove-Item -Path $LabRoot -Recurse -Force
```

Cleanup was independently verified:

```powershell
Test-Path $LabRoot
```

Output:

```text
False
```

The automated test environment was therefore confirmed to be removed.

---

# Result

The Windows Server security administration exercise was completed with the following outcomes:

```text
Windows Server identified
        ↓
Existing AD identity/group structure inspected
        ↓
Disposable NTFS environment created
        ↓
Group-based Modify permission configured
        ↓
ACL independently verified
        ↓
Inheritance independently verified
        ↓
Actual non-admin access test attempted
        ↓
DC logon restriction identified
        ↓
File System auditing enabled
        ↓
Audit policy independently verified
        ↓
SACL configuration attempted
        ↓
SACL failure independently detected
        ↓
SeSecurityPrivilege investigated
        ↓
Token limitation identified
        ↓
Audit policy restored
        ↓
Temporary NTFS environment removed
        ↓
Automation performed
        ↓
Automated configuration verified
        ↓
Automated cleanup verified
```

Final state:

* The temporary `C:\Lab-Security-Test` directory does not exist.
* The temporary NTFS permissions no longer exist.
* File System auditing is restored to `No Auditing`.
* No permanent AD objects were deleted.
* No permanent group memberships were changed.
* No Domain Controller user-rights policy was modified to force an access test.
* The `labauto01` account remains present and enabled.
* The lab's temporary password reset of `labauto01` remains a known administrative change and should be documented as such because the original password was not available to restore.
* The attempted SACL configuration was not represented as successful.
* The SACL limitation was investigated and supported by actual Windows security-policy and token evidence.

---

# What I Learned

The main lesson of the lab was that Windows security administration is a chain of independently verifiable states rather than a collection of commands.

The practical chain was:

```text
Identity
    ↓
Group membership
    ↓
NTFS permission
    ↓
Inheritance
    ↓
Access behaviour
    ↓
Audit policy
    ↓
SACL
    ↓
Security event
    ↓
Detection / investigation
```

The lab also demonstrated the difference between:

```text
A command completing successfully
```

and:

```text
The intended security configuration actually existing
```

The SACL exercise was particularly useful because `Set-Acl` returned without an error, but subsequent inspection showed that no SACL had actually been created. Further investigation showed that `SeSecurityPrivilege` was assigned to the Administrators group by local security policy, but the current token could not enable it for the operation.

Rather than modifying Domain Controller security settings simply to force the exercise to succeed, the limitation was investigated, evidenced, and documented.

The lab therefore reinforced several Windows administration principles:

* least privilege
* group-based access control
* ACL inheritance
* auditability
* verification rather than assumption
* controlled change
* troubleshooting from evidence
* avoiding unnecessary security changes
* distinguishing configuration from effective behaviour
* automating only after the manual workflow is understood and verified

Most importantly, the lab followed the principle:

```text
Understand manually
→ Perform manually
→ Verify manually
→ Automate
→ Verify automation
→ Clean up
```

This provides a practical foundation for progressing from Windows Server administration into security monitoring, detection engineering, and investigation.
