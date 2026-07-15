# Access Control

## Overview

BallardLab uses role-based security groups, the AGDLP model, SMB file sharing, and NTFS permissions to control access to department resources.

Two department file shares were implemented:

```text
\\DC01\Finance
\\DC01\Operations
```

The access-control design separates:

```text
User Identity
      |
      v
Business Role
      |
      v
Resource Access Group
      |
      v
Resource Permission
```

Permissions are assigned to security groups rather than directly to individual user accounts.

## Department Resources

The department folders are stored locally on DC01 under:

```text
C:\Shares
|
+-- Finance
+-- Operations
```

The folders are exposed to domain clients as SMB shares:

```text
C:\Shares\Finance
        |
        v
\\DC01\Finance
```

```text
C:\Shares\Operations
        |
        v
\\DC01\Operations
```

## AGDLP Permission Model

The environment uses the AGDLP model:

```text
A  = Accounts
G  = Global Groups
DL = Domain Local Groups
P  = Permissions
```

### Finance

```text
Sarah Miller
    |
    v
GG-Finance-Users
    |
    v
DL-Finance-Share-Modify
    |
    v
NTFS Modify
    |
    v
C:\Shares\Finance
```

### Operations

```text
Olivia Davis
    |
    v
GG-Operations-Users
    |
    v
DL-Operations-Share-Modify
    |
    v
NTFS Modify
    |
    v
C:\Shares\Operations
```

Global groups represent organizational roles.

Domain Local groups represent access to specific resources.

NTFS permissions are assigned to the Domain Local resource groups.

## NTFS Permission Configuration

NTFS inheritance was disabled on each department folder.

Existing inherited permissions were converted to explicit permission entries before broad user access entries were removed.

The resulting permission design retains administrative and system access while granting department access through a resource-specific Domain Local group.

### Finance ACL

Conceptually:

```text
SYSTEM                          -> Full Control
BALLARDLAB\Administrators       -> Full Control
DL-Finance-Share-Modify         -> Modify
```

### Operations ACL

Conceptually:

```text
SYSTEM                          -> Full Control
BALLARDLAB\Administrators       -> Full Control
DL-Operations-Share-Modify      -> Modify
```

No individual department user is directly assigned NTFS permission.

For example:

```text
smiller -> No direct ACL entry
odavis  -> No direct ACL entry
```

Access is inherited through security group membership.

## Why Modify Was Assigned

Department users were assigned the NTFS `Modify` permission rather than `Full Control`.

Modify allows the users to perform the required file operations:

```text
Read
Create
Write
Modify
Delete
```

Full Control would additionally allow users to change permissions and take ownership of resources.

Those administrative capabilities are not required for standard department users.

This follows the principle of least privilege.

## SMB Share Permissions

The SMB share permission layer was configured with:

```text
Everyone -> Full Control
```

The NTFS ACL is then used to enforce granular resource authorization.

The access path is:

```text
User connects to SMB share
          |
          v
Share permission evaluated
          |
          v
Everyone -> Full Control
          |
          v
NTFS permission evaluated
          |
          v
Applicable resource group?
       /       \
     Yes        No
      |          |
      v          v
   Access     No Allow
   Granted       |
                 v
            Access Denied
```

This design keeps the share permission layer broad while centralizing detailed authorization in the NTFS ACL.

The broad SMB permission does not override restrictive NTFS permissions.

When share and NTFS permissions both apply to network access, the effective result is constrained by both permission layers.

## Finance Access Validation

Sarah Miller authenticated to CLIENT01 using:

```text
BALLARDLAB\smiller
```

Her security group path was:

```text
smiller
   |
   v
GG-Finance-Users
   |
   v
DL-Finance-Share-Modify
```

Sarah accessed:

```text
\\DC01\Finance
```

The following operations were tested:

```text
Open existing file     -> Success
Read file              -> Success
Modify file            -> Success
Save changes           -> Success
Create new file        -> Success
Delete test file       -> Success
```

The successful create, edit, and delete operations validated the expected NTFS Modify permission.

## Operations Access Validation

Olivia Davis authenticated to CLIENT01 using:

```text
BALLARDLAB\odavis
```

Her security group path was:

```text
odavis
   |
   v
GG-Operations-Users
   |
   v
DL-Operations-Share-Modify
```

Olivia accessed:

```text
\\DC01\Operations
```

Validation included:

```text
Open share             -> Success
Create text document   -> Success
Modify document        -> Success
Duplicate document     -> Success
Delete duplicate       -> Success
```

The results validated Modify access to the Operations resource.

## Negative Access Testing

Positive access testing alone does not confirm that an authorization boundary is correctly enforced.

Cross-department access was therefore tested.

### Finance User Against Operations

Sarah Miller attempted to access:

```text
\\DC01\Operations
```

Her access path was:

```text
smiller
   |
   v
GG-Finance-Users
   |
   v
No membership in DL-Operations-Share-Modify
   |
   v
No applicable Allow entry on Operations ACL
   |
   v
Access Denied
```

The request returned:

```text
Access Denied
```

Sarah was not explicitly assigned a `Deny` permission.

Access was unavailable because her security group memberships did not map to an allowed security principal on the Operations NTFS ACL.

This distinction is important:

```text
No applicable Allow != Explicit Deny
```

## Windows Access Tokens

When Sarah authenticated to CLIENT01, Windows created an access token containing her user SID and applicable security group SIDs.

Conceptually:

```text
Sarah authenticates
        |
        v
Windows creates access token
        |
        v
Token contains group SIDs
        |
        +-- GG-Finance-Users
        |
        +-- DL-Finance-Share-Modify
```

When Sarah accessed the Finance folder, the NTFS ACL was evaluated against the security principals represented in her access token.

The ACL contained:

```text
DL-Finance-Share-Modify -> Modify
```

A matching allowed security principal was found and access was granted.

For the Operations folder, Sarah's token did not contain membership that mapped to:

```text
DL-Operations-Share-Modify
```

No applicable Allow permission granted access to the resource.

## User Transfer Scenario

The group-based access model simplifies user lifecycle changes.

Example ticket:

> Sarah Miller has transferred from Finance to Operations. Remove Finance access and provide standard Operations access.

The required Active Directory group changes are:

```text
Remove: GG-Finance-Users
Add:    GG-Operations-Users
```

Sarah's user object can also be moved:

```text
Finance OU
    |
    v
Operations OU
```

The Finance and Operations folder ACLs do not need to be modified.

The access paths automatically change through group membership:

```text
Old:
Sarah
  |
GG-Finance-Users
  |
DL-Finance-Share-Modify
  |
Finance Modify
```

```text
New:
Sarah
  |
GG-Operations-Users
  |
DL-Operations-Share-Modify
  |
Operations Modify
```

After a group membership change, Sarah may need to sign out and sign back in so Windows creates a new logon session and access token containing the updated group memberships.

## Security Principles Demonstrated

The access-control implementation demonstrates:

- Role-based access control
- AGDLP group nesting
- Least privilege
- Group-based authorization
- Separation of identity roles and resource permissions
- Explicit NTFS ACL management
- SMB and NTFS permission interaction
- Positive authorization testing
- Negative authorization testing
- User access lifecycle management

## Related Documentation

- [Active Directory](02-active-directory.md)
- [DHCP and DNS](03-dhcp-dns.md)
- [Network Design](01-network-design.md)
