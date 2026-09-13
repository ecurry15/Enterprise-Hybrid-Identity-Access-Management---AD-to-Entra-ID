# Enterprise Hybrid Identity & Access Management Lab — Active Directory to Entra ID

## Table of Contents

* [Business Scenario](#business-scenario)
* [Lab Architecture](#lab-architecture)
* [Active Directory Environment](#active-directory-environment)
* [Microsoft Entra Cloud Sync](#microsoft-entra-cloud-sync)
* [Moving Users to Cloud Management](#moving-users-to-cloud-management)
* [Entra ID Environment Setup](#entra-id-environment-setup)
* [Dynamic Groups & Administrative Units](#dynamic-groups--administrative-units)
* [Privileged Identity Management](#privileged-identity-management)
* [SAML SSO — Salesforce](#saml-sso--salesforce)
* [OAuth & Microsoft Graph](#oauth--microsoft-graph)
* [Conditional Access](#conditional-access)
* [Access Reviews](#access-reviews)
* [Log Analytics & Break-Glass Alerts](#log-analytics--break-glass-alerts)
* [Joiner, Mover, Leaver Process](#joiner-mover-leaver-process)
* [Real-World IAM Scenarios](#real-world-iam-scenarios)
* [Challenges & Troubleshooting](#challenges--troubleshooting)
* [Skills Demonstrated](#skills-demonstrated)

---
## Business Scenario

- Warriors is a company with 80 employees that has been acquired by Campbell Company.

- Before the acquisition, Warriors managed employee identities through an on-premises Windows Active Directory environment.

- As part of the acquisition, the 80 Warriors employees need to be moved into Campbell's Microsoft Entra environment.

- I was responsible for designing and testing an identity management process that could support the migration while also giving administrators a way to manage access, enforce security policies, and handle employee changes after the migration.

- The lab starts with an on-premises Windows Server environment and follows the identity lifecycle all the way through cloud synchronization, user management, access control, SSO, privileged access, access reviews, Conditional Access, and security monitoring.

---

### Technologies Used

| Category                     | Technologies                                                                |
| ---------------------------- | --------------------------------------------------------------------------- |
| **Identity & Directory**     | Microsoft Entra ID · Active Directory Domain Services · Windows Server 2022 |
| **Identity Synchronization** | Microsoft Entra Cloud Sync                                                  |
| **Access Management**        | RBAC · Dynamic Groups · Administrative Units · Role-Assignable Groups       |
| **Privileged Access**        | Privileged Identity Management (PIM)                                        |
| **Authentication & SSO**     | Conditional Access · SAML 2.0 · OAuth · Salesforce                          |
| **Identity Governance**      | Access Reviews                                                              |
| **Security Monitoring**      | Log Analytics · KQL                                                         |
| **Automation & APIs**        | PowerShell · Microsoft Graph PowerShell · Microsoft Graph Explorer          |

---

## Lab Architecture

### On-Premises

**Domain:**

`Warriors.com`

**Domain Controller:**

`Warriors-DC`

**Organizational Units:**

* `_Finance`
* `_HR`
* `_IT`
* `_Marketing`

Each department contains users and a corresponding security group.

### Microsoft Entra

The synced users are brought into the Campbell Entra tenant:

`campbell.onmicrosoft.com`

The Entra environment contains:

* Finance
* HR
* IT
* Marketing
* IAM administration
* Helpdesk administration
* Security operations
* Break-glass access
* Dynamic groups
* Dynamic Administrative Units
* PIM-controlled roles
* Enterprise applications

---

## Active Directory Environment

The first part of the lab was creating the on-premises identity environment that would represent the acquired Warriors organization.

### Active Directory Setup

I created a Windows Server 2022 domain controller named:

`Warriors-DC`

The server was configured with an Active Directory forest:

`Warriors.com`

I then created four Organizational Units:

* `_Finance`
* `_HR`
* `_IT`
* `_Marketing`

Each department also received a corresponding security group.

### User Provisioning with PowerShell

I used PowerShell to automate user creation.

Each department received a script that created 20 users and placed them into the correct OU and security group.

The script also populated the user's `department` attribute.

Script Used:

```powershell
$FirstNames = @(
    "James","Michael","Robert","David","William",
    "Richard","Joseph","Thomas","Christopher","Daniel",
    "Matthew","Anthony","Donald","Mark","Steven",
    "Paul","Andrew","Joshua","Kenneth","Kevin"
)

$LastNames = @(
    "Anderson","Bennett","Carter","Davis","Edwards",
    "Foster","Garcia","Harris","Jackson","Johnson",
    "Lewis","Mitchell","Morgan","Parker","Robinson",
    "Smith","Taylor","Thomas","Walker","Wilson"
)

for ($i = 0; $i -lt 20; $i++) {

    $FirstName = $FirstNames[$i]
    $LastName = $LastNames[$i]

    $Username = ($FirstName.Substring(0,1) + $LastName).ToLower()
    $DisplayName = "$FirstName $LastName"
    $Password = ConvertTo-SecureString "Password123!" -AsPlainText -Force

    New-ADUser `
Import-Module ActiveDirectory

# Configuration
$OU = "OU=_FINANCE,DC=Warriors,DC=com"
$Group = "Finance Users"
$Department = "Finance"

# Random first and last names

        -Name $DisplayName `
        -GivenName $FirstName `
        -Surname $LastName `
        -SamAccountName $Username `
        -UserPrincipalName "$Username@Warriors.com" `
        -DisplayName $DisplayName `
        -Department $Department `
        -Path $OU `
        -AccountPassword $Password `
        -Enabled $true

    Add-ADGroupMember -Identity $Group -Members $Username

    Write-Host "Created $DisplayName - $Username"
}

```
The `department` attribute becomes especially important later because Entra uses it to automatically manage dynamic group membership.

<img width="970" height="700" alt="Forest creation" src="https://github.com/user-attachments/assets/b47f9679-63d2-4908-aacc-0c385660ba11" />

<img width="828" height="603" alt="AD User Property" src="https://github.com/user-attachments/assets/c182e7ef-161f-4955-9e99-5da130f6a07c" />


---

## Microsoft Entra Cloud Sync

Once the on-premises environment was ready, I configured Microsoft Entra Cloud Sync to synchronize the Warriors users into Campbell's Entra environment.

### Configuration

I first enabled the **Hybrid Identity Administrator** role for the Cloud Sync administration account.

I then:

1. Installed the Microsoft Entra Cloud Sync agent.
2. Created a new Cloud Sync configuration.
3. Selected **AD to Microsoft Entra ID sync**.
4. Enabled password hash synchronization.
5. Added an OU-based scoping filter.
6. Enabled the synchronization configuration.

The OU scoping filter used the Distinguished Name of each department OU.

Example:

```text
OU=_FINANCE,DC=Warriors,DC=com
```

This allowed the synchronization scope to be controlled based on the OU structure in Active Directory.

### Result

The synchronization completed successfully and all **80 Warriors users** were created in the Campbell Entra environment.

<img width="1506" height="633" alt="Sync Scope Rule" src="https://github.com/user-attachments/assets/88325c7c-22e1-4bde-a6df-ce9996a93f72" />
<img width="1680" height="891" alt="Log_AD import" src="https://github.com/user-attachments/assets/53a830f1-37b4-49bf-b651-053146fd13bd" />
<img width="1563" height="903" alt="All Users" src="https://github.com/user-attachments/assets/ee0a12ed-3ef3-407e-a11c-92c72424cf42" />

---

## Moving Users to Cloud Management

After synchronization, the users were still considered managed by the on-premises directory.

This meant certain user properties could not be modified directly from Entra.

The next step was to move the users to cloud management.

### Checking the Management Source

I used Microsoft Graph Explorer to check whether a user was cloud managed.

```http
GET https://graph.microsoft.com/v1.0/users/{user-id}/onPremisesSyncBehavior?$select=isCloudManaged
```

The users initially showed that they were not cloud managed.

### Changing a User to Cloud Managed

I used Microsoft Graph to change the user's management source:

```http
PATCH
/users/{user-id}/onPremisesSyncBehavior

{
    "isCloudManaged": true
}
```

### Automating the Change

I connected Microsoft Graph to PowerShell and used a loop to update the target users, which saved time.

```powershell
foreach ($user in $TargetUsers) {

    $uri = "https://graph.microsoft.com/v1.0/users/$($user.Id)/onPremisesSyncBehavior"

    $body = @{
        isCloudManaged = $true
    } | ConvertTo-Json

    try {
        Invoke-MgGraphRequest `
            -Method PATCH `
            -Uri $uri `
            -Body $body `
            -ContentType "application/json"

        Write-Host "SUCCESS: $($user.UserPrincipalName)"
    }
    catch {
        Write-Host "FAILED: $($user.UserPrincipalName)"
        Write-Host $_.Exception.Message
    }
}
```
I then confirmed that the users were cloud-managed.

<img width="1439" height="656" alt="User is not cloud managed" src="https://github.com/user-attachments/assets/6ba8bf59-a4b7-42be-96ec-94c85e0546e6" />

<img width="727" height="427" alt="Successful Powershell Query" src="https://github.com/user-attachments/assets/0e2ac226-cc27-4efd-88c5-1707b00fc90c" />

---

## Entra ID Environment Setup

After the migration, I created the administrative structure needed to manage the cloud environment.

### Administrative Accounts

The following accounts were created:

* `IAMAdmin1`
* `SecOps1`
* `HelpdeskAdmin1`
* Break-glass account

### Role-Assignable Groups

three role-assignable groups:

* `UserAdmins`
* `Helpdesk Admins`
* `SecOps Admins`

These groups provide a controlled way to assign administrative roles without assigning privileged roles directly to individual users.

### Administrative Units

The IAM administrator created four Administrative Units:

* Finance
* HR
* IT
* Marketing

This allows administrative permissions to be scoped to specific areas of the organization.

<img width="1608" height="775" alt="All Groups" src="https://github.com/user-attachments/assets/af14feac-d24e-4382-805e-0426aa033a4a" />

---

## Dynamic Groups & Administrative Units

One of the main goals of the lab was to avoid manually maintaining access whenever an employee's department changes.

### Department-Based Dynamic Groups

I created four dynamic groups:

* Finance Users
* HR Users
* IT Users
* Marketing Users

Membership is based on the user's `department` attribute.

For example, when a user's department is set to `Finance`, the user is automatically added to the Finance Users group.

This attribute originated in Active Directory and was synchronized into Entra.

### Recently Hired Employees

I also created:

`Employees-Hired-Last-90-Days`

The group uses the employee hire date to identify recently hired users.

The rule used was:

```text
(user.employeeHireDate -ge system.now -minus P90D)
```

This group is later used as part of a Conditional Access scenario.

<img width="1687" height="484" alt="Dynamic AU creation" src="https://github.com/user-attachments/assets/d798a20f-913d-4c74-b614-4eaed95f9837" />

<img width="1100" height="885" alt="Dynamic group creation" src="https://github.com/user-attachments/assets/b548a069-26b3-4ef5-a00c-f018b56800f5" />

<img width="1474" height="910" alt="Users added to group" src="https://github.com/user-attachments/assets/346ba3d1-6563-4e74-8c5e-c3423beb8338" />

---

## Privileged Identity Management

PIM provides temporary administrative access instead of leaving privileged roles permanently active.

### PIM Assignments

The lab uses PIM to control access to the role-assignable groups.

Examples include:

* `IAMAdmin1` → `UserAdmins`
* `HelpdeskAdmin1` & `IT Users` → `Helpdesk Admins`
* `SecOps1` → `SecOps Admins`

The user must activate the appropriate access through PIM when the privilege is needed.

1. An administrative task is identified.
2. The administrator activates the required role through PIM.
3. The task is performed.
4. The elevated access expires.

<img width="1101" height="500" alt="Adding PIM assignment" src="https://github.com/user-attachments/assets/cefc958b-e61c-4520-9834-89222da8003d" />
<img width="1655" height="446" alt="PIM elligable" src="https://github.com/user-attachments/assets/e2b4a81a-2d9c-4c0b-860a-70a3fb7cb824" />

---

## SAML SSO — Salesforce

The next part of the lab simulated an IT department using Salesforce as a business application.

### Use Case

IT employees receive support tickets through Salesforce.

Only users assigned to the IT group should have access to the application.

The goal was to allow IT users to sign into Salesforce using their Entra credentials instead of maintaining a separate password.

### Configuration

1. Created the Salesforce enterprise application in Entra.
2. Assigned the IT group to the application.
3. Created a matching test user in Salesforce.
4. Configured Entra SSO.
5. Matched the Entity ID, ACS URL, and sign-on URL between Entra and Salesforce.
6. Used the XML metadata from Entra to configure Salesforce.
7. Tested the SSO flow.

### Result

The test user successfully federated into Salesforce using their Entra credentials.

<img width="1839" height="903" alt="SSO side by side" src="https://github.com/user-attachments/assets/46e1e4bc-c9d4-4549-ba50-6dc37b10d19e" />
<img width="946" height="841" alt="Login with EntraSSO" src="https://github.com/user-attachments/assets/77904ded-0813-4fa3-b037-7de99277dcd2" />
<img width="1678" height="897" alt="Successful SSO Log" src="https://github.com/user-attachments/assets/f52df75b-b192-4927-84f3-6b6946fd33c2" />

---

## OAuth & Microsoft Graph

The OAuth portion of the lab demonstrates how privileged security users can access Microsoft Graph to investigate identity information.

### Use Case

A security incident occurs, and the SecOps team needs to investigate an employee's access.

For example, SecOps may need to determine which groups an employee belongs to.

Instead of giving every security analyst broad permanent access, the lab uses PIM to control access to the investigation application.

### Application Setup

Created an enterprise application:

`SOC-Investigation-Tool`

Assignment was required so that users could not authenticate to the application unless they were assigned to it.

The application was configured with delegated Microsoft Graph permissions, including:

* `User.Read.All`
* `GroupMember.Read.All`

Admin consent was granted.

The `SecOps Admins` role-assignable group was then assigned to the application.


### Authentication

SecOps activates the appropriate PIM role and then authenticates to Microsoft Graph using PowerShell device authentication.

```powershell
$clientId = "YOUR-CLIENT-ID"
$tenantId = "YOUR-TENANT-ID"

Connect-MgGraph `
    -ClientId $clientId `
    -TenantId $tenantId `
    -Scopes "User.Read" `
    -UseDeviceAuthentication
```

> **Note:** Client and tenant IDs should be replaced with actual values.

### Investigating Group Membership

Once authenticated, the analyst can look up a specific user and review their group membership.

```powershell
$userUPN = "amitchell@campbell.onmicrosoft.com"

$user = Get-MgUser -UserId $userUPN

$groups = Get-MgUserMemberOf -UserId $user.Id |
    Where-Object {
        $_.AdditionalProperties["@odata.type"] -eq "#microsoft.graph.group"
    }

Write-Host "`nUser: $($user.DisplayName)"
Write-Host "UPN: $($user.UserPrincipalName)"
Write-Host "Groups: $($groups.Count)`n"

$groups | ForEach-Object {
    [PSCustomObject]@{
        GroupName = $_.AdditionalProperties["displayName"]
        GroupId   = $_.Id
    }
} | Format-Table -AutoSize
```

This gives SecOps a practical way to investigate identity and access information through Microsoft Graph.

<img width="899" height="890" alt="Oauth MS Graph permissions" src="https://github.com/user-attachments/assets/57483b8f-fbac-4e9f-9e33-d60618b27b88" />
<img width="1457" height="630" alt="Oauth Connected" src="https://github.com/user-attachments/assets/abd54167-446c-4624-9b30-2c9e0eaf9f01" />

---

## Conditional Access

Conditional Access was used to apply security controls based on the user's situation.

### MFA for Users

MFA was required for users across the environment.

The break-glass account was excluded from the MFA requirement so that emergency access remained available.

> In a production environment, a stronger break-glass design would use a separate authentication method such as a FIDO2 security key and tightly monitor the account.

A login alert was also configured for break-glass account sign-ins.

### Recently Hired Employees

The `Employees-Hired-Last-90-Days` dynamic group was created to prevent newly hired employees from logging in outside the office.

The policy:

* Targets the recently hired employee group.
* Applies to all cloud applications.
* Blocks access unless the sign-in is within the corporate network.

A named location was configured using the corporate (my home) IPv4 and IPv6 addresses.

A VPN was then used to simulate an employee signing in from outside the office (outside my home network).

The test user was denied access as expected.

<img width="749" height="882" alt="CA policy1" src="https://github.com/user-attachments/assets/144368c7-bed2-480e-ac80-6ede38acd805" />

<img width="1647" height="906" alt="BreakGlass excluded from policies" src="https://github.com/user-attachments/assets/01242478-599d-4ba6-a1a6-6f2155adb48a" />

<img width="705" height="585" alt="Dmorgan sign-in out of office" src="https://github.com/user-attachments/assets/eb29044f-d798-4a26-b3c0-05657e0d991f" />

---

## Access Reviews

Access Reviews were used to periodically review membership in the Finance Users group.

### Scenario

The Finance Users group should be reviewed quarterly to make sure users still belong to the Finance department.

Created a quarterly access review with `IAMAdmin1` as the reviewer.

Automatic changes were not applied because the group is dynamically managed.

> **Note:** If the user's `department` attribute still indicates Finance, the dynamic group will add the user back even after an automatic removal.

### Review Process

`IAMAdmin1` signs in to:

`https://myaccess.microsoft.com/`

1. Opens the pending access review.
2. Reviews the Finance Users membership.
3. Approves users who still require Finance access.
4. Denies users who should no longer have access.

Employee Kenneth Walker moved from Finance to HR.

IAMAdmin1 approved the 19 users who still belonged in Finance and denied Kenneth.

IAMAdmin1 then activated PIM and updated Kenneth's department from Finance to HR.

This allowed the dynamic group to update his membership.

<img width="1336" height="407" alt="Access review completed" src="https://github.com/user-attachments/assets/5dabcf1d-0665-4086-bf09-e21cfbf165d5" />

<img width="975" height="928" alt="Kennith walker denied in access review" src="https://github.com/user-attachments/assets/6ef71c3f-dbab-4314-be8a-9501b77760d8" />

---

## Log Analytics & Break-Glass Alerts

The lab also includes monitoring for the emergency break-glass account use.

### Log Analytics Workspace

I created a Log Analytics workspace:

`law-Warriors-Lab`

Diagnostic settings were configured to send:

* Sign-in logs
* Audit logs

to the workspace.

### KQL Investigation

After confirming logs were being received, I used KQL to search for break-glass account sign-ins.

```
SigninLogs
| where UserPrincipalName == "breakglass01@campbell400.onmicrosoft.com"
```

This allows the security team to quickly identify activity involving the emergency account.

### Alert Configuration

I then created an alert that:

* Checks the logs every 5 minutes.
* Triggers when at least one matching event is found.
* Uses an action group to send an email.
* Uses severity `0 - Critical`.
* Identifies the alert as a break-glass account sign-in.

A test sign-in was performed, and the alert successfully fired.

The SecOps team could then investigate the sign-in and determine why the emergency account was used.

### Why This Matters

Break-glass accounts should rarely be used.

Because of that, any sign-in should be treated as important and investigated.

<img width="804" height="883" alt="LAW creation and Diagnostic setting" src="https://github.com/user-attachments/assets/56cd0acb-04ae-4032-af30-114a2b18dee7" />

<img width="825" height="874" alt="BreakGlassAlert" src="https://github.com/user-attachments/assets/52cf3d31-ba33-4ee9-9fc4-29d814ba1bfc" />

<img width="1810" height="715" alt="Alert email" src="https://github.com/user-attachments/assets/d1e05652-8377-46e2-93aa-b9bb4af0930a" />

---

## Joiner, Mover, Leaver Process

### Joiner

When an employee joins the organization:

1. The user exists in the on-premises Active Directory environment.
2. The user is synchronized into Entra.
3. The user's department determines their dynamic group membership.
4. Additional access can be assigned based on their role.

Administrative accounts such as `IAMAdmin1`, `SecOps1`, and `HelpdeskAdmin1` were also created for administrative functions.

### Mover

Kenneth Walker moves from Finance to HR.

The IAM administrator:

1. Activates the required PIM role.
2. Changes Kenneth's department from Finance to HR.
3. Entra updates the user's dynamic group membership.

This removes him from Finance Users and adds him to HR Users.

### Leaver

Chloe Riely leaves the organization.

Her account is deactivated and her department is changed to `Disabled`.

This causes her to be removed from the Marketing dynamic group.


---

## Real-World IAM Scenarios

The lab was built around several scenarios that represent common IAM tasks.

### Scenario 1 — Helpdesk Password Reset

A Marketing employee, Nicole Wagner, forgets her password and submits a ticket.

Brandon Knight takes ownership of the ticket in Salesforce.

Knight then activates the required PIM role and performs the password reset.

Before PIM:
<img width="1904" height="618" alt="PW reset before PIM" src="https://github.com/user-attachments/assets/11c90574-0789-4da3-bfcd-5a81d9567ae9" />

PIM Activation Log:

<img width="1672" height="867" alt="Bknight PIM log" src="https://github.com/user-attachments/assets/92a4b539-e8e2-4b73-abcd-b48c0be750ae" />

Result:

<img width="1844" height="721" alt="pw-reset-after PIM" src="https://github.com/user-attachments/assets/363feeda-4952-4c37-adc1-920b0e8ff628" />

---

### Scenario 2 — Security Investigation

A security incident occurs and SecOps needs to investigate employee Anthony Mitchell.

The analyst needs to determine which groups the user belongs to.

Before PIM activation, access to the investigation application is denied.

The SecOps analyst then:

1. Activates the PIM role.
2. Gains access to the assigned application.
3. Authenticates through OAuth.
4. Uses Microsoft Graph PowerShell.
5. Queries Anthony's group membership.

Before PIM:
<img width="853" height="854" alt="SecOps blocked from Oauth" src="https://github.com/user-attachments/assets/5551c5eb-45d3-4f47-895d-b534fa8d8860" />

PIM Activation:
<img width="1898" height="944" alt="SecOpsPIM Activation" src="https://github.com/user-attachments/assets/089212b7-3684-41b7-8e39-2b19214e527f" />

Results:
<img width="1107" height="622" alt="MS-graph successful query" src="https://github.com/user-attachments/assets/a30cfdc8-1d81-4b4b-b4c0-ef44d38f6a5e" />

<img width="1684" height="902" alt="Log_Oauth" src="https://github.com/user-attachments/assets/1d0a959d-af31-471a-abe0-a94eb6b084be" />


---

### Scenario 3 — Employee Department Change

Kenneth Walker changes departments from Finance to HR.

The IAM administrator activates the appropriate PIM role and updates his department.

Because the department-based groups are dynamic, Entra can automatically update his group membership.

Before PIM: 

<img width="976" height="936" alt="IAM before PIM" src="https://github.com/user-attachments/assets/4bb26259-2490-4d6f-97cb-7f824153ea8d" />

PIM Activation Log:
<img width="1680" height="905" alt="IAM PIM log" src="https://github.com/user-attachments/assets/9fc6a05c-c10f-48de-8f08-0c288a93894f" />

Results:

<img width="1901" height="929" alt="IAM after PIM" src="https://github.com/user-attachments/assets/4d451327-0f32-4f97-b133-85eac617f64e" />

---

## Challenges & Troubleshooting

### OAuth Device Authentication

I initially received:

```text
DeviceCodeCredential authentication failed
```

The fix was to allow public client flows because PowerShell device-code authentication uses a public client.

I also encountered the same error after making the configuration change.

Installing a newer version of PowerShell resolved the issue.

### Group Membership Query

```text
Status: 403 (Forbidden)
Authorization_RequestDenied
```

When attempting to retrieve another user's group membership:

```powershell
Get-MgUserMemberOf -UserId $user.Id
```

The issue was resolved by adding the required permission:

`Directory.Read.All`

I thought that `Group.Read.All` was sufficient, but the ability to read directories was also required.

### Application Assignment

The OAuth application was initially accessible without requiring users to be assigned to it.

I enabled:

**Assignment required = Yes**

This ensured that only users assigned to the application could authenticate.

---

## Skills Demonstrated

| Category                                          | Skills                                                                                                                                                                                                                          |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Identity & Access Management**                  | User lifecycle management · Joiner/Mover/Leaver processes · RBAC · Group-based access · Attribute-based access · Administrative Units · Dynamic groups · Role-assignable groups · Privileged access management · Access Reviews |
| **Microsoft Entra ID**                            | Entra ID administration · Cloud-managed users · Enterprise applications · Conditional Access · PIM · Application assignments · Named locations · MFA · Break-glass account management                                           |
| **Active Directory**                              | Windows Server 2022 · Active Directory Domain Services · Organizational Units · Security groups · User provisioning · PowerShell automation · Department attributes                                                             |
| **Identity Federation & Application Integration** | SAML 2.0 · SSO · Salesforce integration · Enterprise applications · OAuth · Microsoft Graph · Delegated permissions                                                                                                             |
| **Security & Monitoring**                         | Log Analytics · Diagnostic settings · KQL · Sign-in log investigation · Audit logs · Alert rules · Action groups · Break-glass account monitoring                                                                               |
| **Automation**                                    | PowerShell · Active Directory user provisioning · Microsoft Graph PowerShell · Bulk cloud-management changes · KQL queries                                                                                                      |
