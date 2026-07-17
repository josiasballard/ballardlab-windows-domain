# BallardLab Windows Domain Infrastructure

A hands-on Windows Server 2022 infrastructure lab implementing Active Directory Domain Services, DNS, DHCP, Group Policy, domain-joined Windows clients, AGDLP-based access control, and SMB file services in an isolated Hyper-V environment.

The project models common Windows systems administration tasks including centralized identity management, client network configuration, role-based resource access, Group Policy deployment, and user access lifecycle changes.

> **Project Status:** In Progress — core domain infrastructure, access control, and role-based drive mapping are complete. Additional endpoint security policies are being implemented.

---

## Environment

| Component | Configuration |
|---|---|
| Hypervisor | Microsoft Hyper-V |
| Server OS | Windows Server 2022 |
| Client OS | Windows 11 Pro |
| Domain | `ballardlab.local` |
| Domain Controller | `DC01` |
| Domain Client | `CLIENT01` |
| Network | `10.10.10.0/24` |
| DNS / DHCP | `10.10.10.10` |
| DHCP Scope | `10.10.10.100 - 10.10.10.200` |

### Technologies

`Windows Server 2022` `Active Directory` `DNS` `DHCP` `Group Policy` `Hyper-V` `Windows 11` `SMB` `NTFS` `AGDLP`

---

## Architecture

```text
                    BALLARDLAB-LAN
                     10.10.10.0/24
                            |
              +-------------+-------------+
              |                           |
            DC01                       CLIENT01
         10.10.10.10                  DHCP Client
              |                       10.10.10.101
              |
     +--------+--------+
     |        |        |
   AD DS     DNS      DHCP
     |
 ballardlab.local
```

`BALLARDLAB-LAN` is an isolated Hyper-V Internal virtual network.

DC01 uses a static IPv4 address and provides Active Directory, DNS, and DHCP services. CLIENT01 receives network configuration through DHCP and is joined to the `ballardlab.local` domain.

The isolated subnet currently has no default gateway because no router is deployed in the lab.

[View network design](docs/01-network-design.md)

---

## Active Directory

A new Active Directory forest was deployed for:

```text
ballardlab.local
```

Directory objects are organized using departmental and administrative OUs.

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

Global security groups represent organizational roles:

```text
GG-Finance-Users
GG-Operations-Users
```

Domain Local groups represent access to specific resources:

```text
DL-Finance-Share-Modify
DL-Operations-Share-Modify
```

[View Active Directory design](docs/02-active-directory.md)

---

## Role-Based Access Control

Department file access uses the AGDLP model:

```text
Account
   |
Global Role Group
   |
Domain Local Resource Group
   |
NTFS Permission
```

Example:

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

Department users receive `Modify` rather than `Full Control`, allowing normal file operations without granting permission-management or ownership capabilities.

SMB share permissions are broad in the lab while granular authorization is enforced through NTFS ACLs.

Positive and negative access tests confirmed that authorized users could create, modify, and delete files while cross-department access was denied.

[View access control implementation](docs/04-access-control.md)

---

## DHCP and Active Directory DNS

DC01 provides DHCP configuration to CLIENT01.

```text
CLIENT01
IPv4:       10.10.10.101
DHCP Server: 10.10.10.10
DNS Server:  10.10.10.10
DNS Suffix:  ballardlab.local
```

Before DHCP was available, CLIENT01 received an APIPA `169.254.x.x` address.

After DHCP deployment and scope activation, the client successfully received valid BallardLab network configuration.

Active Directory DNS service discovery was validated by querying:

```text
_ldap._tcp.dc._msdcs.ballardlab.local
```

The SRV response identified:

```text
dc01.ballardlab.local
Port 389
10.10.10.10
```

This confirmed that CLIENT01 could use internal DNS to locate the domain controller providing LDAP services.

[View DHCP and DNS validation](docs/03-dhcp-dns.md)

---

## Group Policy and Drive Mapping

A Group Policy Object named:

```text
GPO-User-Department-DriveMaps
```

was linked to the `Users` OU.

Group Policy Preferences and item-level targeting automatically map departmental network drives based on Global security group membership.

| Role Group | Drive | Resource |
|---|---|---|
| `GG-Finance-Users` | `F:` | `\\DC01\Finance` |
| `GG-Operations-Users` | `O:` | `\\DC01\Operations` |

The same GPO provides different configurations to users based on their Active Directory role.

### Role Transfer Validation

Sarah Miller was transferred from Finance to Operations.

The change required:

```text
Remove: GG-Finance-Users
Add:    GG-Operations-Users
Move:   Finance OU -> Operations OU
```

No NTFS ACL changes were required.

Testing identified that the initial GPP `Update` action could leave a stale drive mapping after a role change.

The configuration was revised to use:

```text
Replace
+
Remove this item when it is no longer applied
```

After a complete sign-out and sign-in:

```text
Finance F:     Removed
Operations O:  Mapped
Finance Access: Denied
Operations Modify: Successful
```

The test also demonstrated that `gpupdate /force` reprocesses Group Policy but does not rebuild an existing Windows logon access token.

[View Group Policy and role-transfer validation](docs/05-group-policy.md)

---

## Troubleshooting and Validation

The project included hands-on diagnosis and validation of:

- APIPA addressing before DHCP availability
- Static addressing for domain infrastructure
- DHCP lease and client option assignment
- Active Directory DNS and LDAP SRV discovery
- Hyper-V Enhanced Session logon rights
- NTFS and SMB permission interaction
- Positive and negative authorization testing
- Group Policy processing with `gpupdate` and `gpresult`
- Windows access token refresh after group membership changes
- Stale GPP drive mappings during user role transitions

---

## Project Roadmap

- [x] Create isolated Hyper-V virtual network
- [x] Deploy Windows Server 2022
- [x] Configure static server addressing
- [x] Deploy Active Directory Domain Services
- [x] Configure AD-integrated DNS
- [x] Deploy and authorize DHCP
- [x] Configure DHCP client scope and options
- [x] Deploy and domain-join Windows 11 client
- [x] Implement OU and security group structure
- [x] Implement AGDLP-based access control
- [x] Configure departmental SMB shares and NTFS ACLs
- [x] Validate positive and negative resource access
- [x] Validate Active Directory DNS SRV records
- [x] Deploy Group Policy Preferences
- [x] Automate role-based departmental drive mapping
- [x] Validate user role-transfer lifecycle
- [x] Configure Windows security policies
- [ ] Configure Windows Defender Firewall policy
- [ ] Complete final infrastructure validation

---

## Documentation

1. [Network Design](docs/01-network-design.md)
2. [Active Directory](docs/02-active-directory.md)
3. [DHCP and DNS](docs/03-dhcp-dns.md)
4. [Access Control](docs/04-access-control.md)
5. [Group Policy and Role-Based Drive Mapping](docs/05-group-policy.md)
6. [Windows Security Policies](docs/06-security-policies.md)

---

## Skills Demonstrated

`Windows Server 2022` `Active Directory` `AD DS` `DNS` `DHCP` `Group Policy` `Group Policy Preferences` `Hyper-V` `Windows 11` `AGDLP` `NTFS Permissions` `SMB` `RBAC` `Least Privilege` `Identity and Access Management` `Infrastructure Troubleshooting`
