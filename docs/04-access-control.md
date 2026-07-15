# Access Control

## Overview

BallardLab uses role-based security groups, the AGDLP model, SMB file sharing, and NTFS permissions to control access to departmental resources.

Two department file shares were implemented:

```text
\\DC01\Finance
\\DC01\Operations
```

Permissions are assigned to security groups rather than directly to individual user accounts.

The authorization design follows:

```text
User Account
     |
     v
Global Role Group
     |
     v
Domain Local Resource Group
     |
     v
NTFS Permission
```

---

## Department Resources

Department folders are hosted on DC01:

```text
C:\Shares
|
+-- Finance
+-- Operations
```

The folders are exposed as SMB shares:

```text
C:\Shares\Finance    -> \\DC01\Finance
C:\Shares\Operations -> \\DC01\Operations
```

---

## AGDLP Permission Model

BallardLab uses the AGDLP model:

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
GG-Finance-Users
     |
DL-Finance-Share-Modify
     |
NTFS Modify
     |
\\DC01\Finance
```

### Operations

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

Global groups represent organizational roles.

Domain Local groups represent access to specific resources.

This separates:

```text
Who the user is        -> Global Group
What access is granted -> Domain Local Group
```

Resource permissions can therefore remain unchanged when users move between organizational roles.

---

## NTFS Permission Configuration

NTFS inheritance was disabled on each departmental folder.

Existing inherited permissions were converted to explicit entries before broad user access entries were removed.

The final ACLs retain administrative and system access while granting departmental access through resource-specific Domain Local groups.

### Finance ACL

```text
SYSTEM                            -> Full Control
BALLARDLAB\Administrators         -> Full Control
DL-Finance-Share-Modify           -> Modify
```

![Finance NTFS ACL](../screenshots/21-finance-ntfs-acl.png)

### Operations ACL

```text
SYSTEM                            -> Full Control
BALLARDLAB\Administrators         -> Full Control
DL-Operations-Share-Modify        -> Modify
```

![Operations NTFS ACL](../screenshots/22-operations-ntfs-acl.png)

Individual department users do not receive direct ACL entries.

```text
smiller -> No direct ACL entry
odavis  -> No direct ACL entry
```

Access is provided through nested security group membership.

---

## Least-Privilege Permission Design

Department users receive NTFS `Modify` rather than `Full Control`.

Modify permits the required file operations:

```text
Read
Create
Write
Modify
Delete
```

Full Control would additionally allow users to change permissions and take ownership of the resource.

Those administrative capabilities are not required for standard department users.

The permission model therefore provides the operational access required by each department without granting administrative control over the resource ACL.

---

## SMB and NTFS Permission Model

The SMB share permission layer was configured with:

```text
Everyone -> Full Control
```

Granular authorization is enforced through NTFS permissions.

```text
User connects to SMB share
          |
          v
SMB permission evaluated
          |
          v
Everyone -> Full Control
          |
          v
NTFS ACL evaluated
          |
          v
Matching resource group?
       /       \
     Yes        No
      |          |
      v          v
   Access     No applicable
   Granted    Allow entry
                  |
                  v
             Access Denied
```

The broad SMB permission does not override restrictive NTFS permissions.

For network access, both permission layers apply. The effective access is constrained by the permissions available through both layers.

This design keeps the SMB layer simple while using NTFS as the primary departmental authorization boundary.

---

## Positive Access Validation

### Finance

Sarah Miller initially authenticated to CLIENT01 as:

```text
BALLARDLAB\smiller
```

Her Finance authorization path was:

```text
smiller
   |
GG-Finance-Users
   |
DL-Finance-Share-Modify
```

Sarah successfully accessed:

```text
\\DC01\Finance
```

Validation included:

```text
Open share       -> Success
Read file        -> Success
Modify file      -> Success
Save changes     -> Success
Create file      -> Success
Delete test file -> Success
```

![Finance Modify access validation](../screenshots/10-finance-modify-access-success.png)

The successful create, modify, and delete operations validated the expected NTFS `Modify` permission.

### Operations

Olivia Davis authenticated as:

```text
BALLARDLAB\odavis
```

Her authorization path was:

```text
odavis
   |
GG-Operations-Users
   |
DL-Operations-Share-Modify
```

Olivia successfully accessed:

```text
\\DC01\Operations
```

Testing confirmed that she could create, modify, and delete files within the Operations resource.

---

## Negative Access Validation

Cross-department access was tested to verify the authorization boundary.

While assigned to the Finance role, Sarah attempted to access:

```text
\\DC01\Operations
```

Her security group memberships did not map to:

```text
DL-Operations-Share-Modify
```

The Operations NTFS ACL therefore contained no applicable Allow entry for her authorization path.

The result was:

```text
Access Denied
```

![Operations access denied to Finance user](../screenshots/11-operations-access-denied.png)

Sarah was not assigned an explicit `Deny` permission.

The access failure occurred because no applicable Allow entry granted access.

```text
No applicable Allow != Explicit Deny
```

This confirmed that the departmental authorization boundary was functioning as intended.

---

## Role-Based Access Lifecycle

The group-based design was also tested through a departmental transfer scenario.

Sarah Miller was transferred from Finance to Operations.

The Active Directory changes were:

```text
Remove: GG-Finance-Users
Add:    GG-Operations-Users
Move:   Finance OU -> Operations OU
```

No Finance or Operations folder ACL changes were required.

The authorization path changed through group membership:

```text
Previous:

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

After a new Windows logon session refreshed Sarah's access token, Operations access succeeded and Finance access was denied.

The detailed access-token refresh, Group Policy processing, and departmental drive mapping behavior are documented in:

[Group Policy and Role-Based Drive Mapping](05-group-policy.md)

---

## Security Principles Demonstrated

The implementation demonstrates:

- Role-based access control
- AGDLP group nesting
- Least privilege
- Group-based authorization
- Separation of user roles and resource permissions
- Explicit NTFS ACL management
- SMB and NTFS permission interaction
- Positive authorization testing
- Negative authorization testing
- User access lifecycle management
- Access changes without direct ACL modification

---

## Validation Summary

The access-control design was validated by:

- Creating separate Finance and Operations resources
- Implementing resource-specific Domain Local groups
- Nesting role-based Global groups into resource groups
- Removing broad inherited NTFS user access
- Assigning NTFS Modify permission through Domain Local groups
- Verifying department users were not granted Full Control
- Testing successful create, modify, and delete operations
- Testing unauthorized cross-department access
- Transferring a user between departmental roles
- Updating effective access without modifying resource ACLs

---

## Related Documentation

- [Active Directory](02-active-directory.md)
- [DHCP and DNS](03-dhcp-dns.md)
- [Network Design](01-network-design.md)
- [Group Policy and Role-Based Drive Mapping](05-group-policy.md)
