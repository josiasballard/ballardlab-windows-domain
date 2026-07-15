# Active Directory

## Overview

BallardLab uses Active Directory Domain Services (AD DS) to provide centralized identity and directory services for the Windows domain environment.

A new Active Directory forest was deployed with the domain:

```text
ballardlab.local
```

DC01 was promoted as the first domain controller in the forest and hosts the Active Directory database and domain services for the lab.

```text
Forest:            ballardlab.local
Domain:            ballardlab.local
NetBIOS Name:      BALLARDLAB
Domain Controller: DC01
```

## Domain Controller Deployment

The Active Directory Domain Services role was installed on DC01 before the server was promoted to a domain controller.

A new forest was created using:

```text
Domain Name:        ballardlab.local
NetBIOS Name:       BALLARDLAB
DNS Installation:   Enabled
```

The domain controller promotion also installed DNS services required for Active Directory name resolution and service discovery.

After promotion and restart, domain administrative authentication used the BALLARDLAB domain context rather than the original standalone server account context.

## Organizational Unit Structure

Organizational Units were created to organize directory objects by administrative and business function.

```text
BallardLab
|
+-- Users
|   +-- Finance
|   +-- Operations
|
+-- Groups
    +-- Global
    +-- Domain Local
```

The Finance and Operations OUs contain user objects associated with each department.

Security groups are organized separately by group scope.

This structure separates object organization from resource authorization.

An OU determines where an object is administratively organized and can affect Group Policy application.

Security group membership determines access paths to protected resources.

## Domain User Accounts

Two domain user accounts were created to validate department-based access control.

### Finance

```text
Name:       Sarah Miller
Username:   smiller
Department: Finance
```

### Operations

```text
Name:       Olivia Davis
Username:   odavis
Department: Operations
```

The accounts were placed in their corresponding department OUs.

These users were used to perform both positive and negative access testing against department file shares.

## Global Security Groups

Role-based Global security groups were created:

```text
GG-Finance-Users
GG-Operations-Users
```

User accounts are assigned to a Global group based on their organizational role.

```text
Sarah Miller
      |
GG-Finance-Users
```

```text
Olivia Davis
      |
GG-Operations-Users
```

Global groups answer the administrative question:

> Who are these users?

For example, `GG-Finance-Users` represents users performing the Finance role.

Resource permissions are not assigned directly to the individual user accounts.

## Domain Local Security Groups

Resource-specific Domain Local groups were created:

```text
DL-Finance-Share-Modify
DL-Operations-Share-Modify
```

These groups represent access to specific resources.

Domain Local groups answer the administrative question:

> What access does this group receive?

The role-based Global groups were nested into the appropriate resource groups.

```text
GG-Finance-Users
        |
DL-Finance-Share-Modify
```

```text
GG-Operations-Users
        |
DL-Operations-Share-Modify
```

The Domain Local groups are then assigned permissions on the corresponding NTFS resource.

## AGDLP Design

BallardLab uses the AGDLP access-control model:

```text
A - Accounts
G - Global Groups
DL - Domain Local Groups
P - Permissions
```

The complete Finance access path is:

```text
Sarah Miller
    |
GG-Finance-Users
    |
DL-Finance-Share-Modify
    |
NTFS Modify
    |
\\DC01\Finance
```

The Operations access path is:

```text
Olivia Davis
    |
GG-Operations-Users
    |
DL-Operations-Share-Modify
    |
NTFS Modify
    |
\\DC01\Operations
```

This design separates user role membership from resource permission assignment.

For example, if Sarah Miller transfers from Finance to Operations, the resource ACL does not need to be modified.

Her Active Directory group memberships can be changed:

```text
Remove: GG-Finance-Users
Add:    GG-Operations-Users
```

Her user object can also be moved from the Finance OU to the Operations OU to reflect the administrative change and allow department-specific Group Policy to apply appropriately.

## Access Tokens and Group Membership

When a domain user authenticates to CLIENT01, Windows builds an access token containing the user's security identifiers and applicable security group memberships.

For a Finance user, the authorization path includes membership through:

```text
GG-Finance-Users
        |
DL-Finance-Share-Modify
```

When the user accesses the Finance resource, Windows evaluates the security principals in the user's token against the resource access control list.

The NTFS ACL contains:

```text
DL-Finance-Share-Modify -> Modify
```

Because the user's group membership maps to an allowed security principal, access is granted.

After security group membership changes, an existing user session may continue using an access token created before the change.

Signing out and signing back in causes Windows to create a new logon session and access token using the updated group membership.

## Domain Client

CLIENT01 was joined to:

```text
ballardlab.local
```

The domain join process depended on internal DNS to locate Active Directory domain controller services.

After the domain join, users authenticated using domain identities such as:

```text
BALLARDLAB\smiller
BALLARDLAB\odavis
```

rather than the original CLIENT01 local administrator account.

The successful domain join also created a CLIENT01 computer object in Active Directory and established a trust relationship between the workstation and the domain.

## Validation

The Active Directory design was validated by:

- Authenticating domain users on CLIENT01
- Confirming department security group membership
- Validating nested Global and Domain Local group relationships
- Accessing SMB resources using domain identities
- Testing authorized department access
- Testing unauthorized cross-department access

The detailed NTFS and SMB authorization implementation is documented in:

[Access Control](04-access-control.md)
