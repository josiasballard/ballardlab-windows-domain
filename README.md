# BallardLab Windows Domain Infrastructure

A Windows Server 2022 infrastructure lab implementing Active Directory Domain Services, DNS, DHCP, domain-joined Windows endpoints, role-based access control, and SMB file services in an isolated Hyper-V environment.

The environment was built to model common Windows infrastructure administration tasks including domain deployment, centralized identity management, client network configuration, Active Directory service discovery, and least-privilege resource access.

> **Project Status:** In Progress — Group Policy and additional security controls are currently being implemented.

---

## Environment Overview

| Component | Configuration |
|---|---|
| Hypervisor | Microsoft Hyper-V |
| Server | Windows Server 2022 |
| Domain | `ballardlab.local` |
| Domain Controller | `DC01` |
| Domain Client | `CLIENT01` |
| Client OS | Windows 11 Pro |
| Network | `10.10.10.0/24` |
| DNS Server | `10.10.10.10` |
| DHCP Server | `10.10.10.10` |
| DHCP Scope | `10.10.10.100 - 10.10.10.200` |

### Core Services

- Active Directory Domain Services (AD DS)
- AD-integrated DNS
- DHCP
- SMB file sharing
- NTFS access control
- AGDLP-based security group design

---

## Architecture

The lab operates on an isolated Hyper-V virtual network named `BALLARDLAB-LAN`.

```text
                    BALLARDLAB-LAN
                     10.10.10.0/24
                            |
              +-------------+-------------+
              |                           |
            DC01                       CLIENT01
         10.10.10.10                 DHCP Client
              |                       10.10.10.x
              |
     +--------+--------+
     |        |        |
   AD DS     DNS      DHCP
     |
 ballardlab.local```

`DC01` uses a static IPv4 address and provides centralized identity, DNS, and DHCP services.

`CLIENT01` is a Windows 11 Pro workstation configured as a DHCP client and joined to the `ballardlab.local` domain.

The network is intentionally isolated and currently has no default gateway because communication is limited to the local `10.10.10.0/24` subnet.

[View detailed network design](docs/01-network-design.md)

---

## Active Directory Design

A new Active Directory forest was deployed for:

```text
ballardlab.local
```

The directory structure separates users and security groups by organizational and administrative function.

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

### Role-Based Global Groups

```text
GG-Finance-Users
GG-Operations-Users
```

Global groups represent user roles within the organization.

### Resource Permission Groups

```text
DL-Finance-Share-Modify
DL-Operations-Share-Modify
```

Domain Local groups represent access levels to specific resources.

[View Active Directory documentation](docs/02-active-directory.md)

---

## AGDLP Access Control

File share permissions were implemented using the AGDLP model:

```text
Accounts
   |
Global Groups
   |
Domain Local Groups
   |
Permissions
```

### Finance Access Path

```text
Sarah Miller (smiller)
        |
GG-Finance-Users
        |
DL-Finance-Share-Modify
        |
NTFS Modify
        |
\\DC01\Finance
```

### Operations Access Path

```text
Olivia Davis (odavis)
        |
GG-Operations-Users
        |
DL-Operations-Share-Modify
        |
NTFS Modify
        |
\\DC01\Operations
```

Permissions are assigned to resource-specific Domain Local groups rather than directly to individual user accounts.

This design allows access to be managed through role group membership and avoids maintaining individual user ACL entries.

[View access control documentation](docs/04-access-control.md)

---

## SMB and NTFS Permission Model

Department resources are exposed through SMB shares:

```text
\\DC01\Finance
\\DC01\Operations
```

The SMB share layer permits broad access while NTFS permissions enforce granular resource authorization.

```text
SMB Share Permission
Everyone -> Full Control
          |
          v
NTFS ACL
Resource Domain Local Group -> Modify
```

Broad inherited NTFS entries were removed from the department folders after inheritance was disabled and existing permissions were converted to explicit entries.

The resulting ACL design retains administrative access while assigning user access through the appropriate Domain Local security group.

---

## DHCP Configuration

`DC01` was configured as an authorized Windows DHCP server in Active Directory.

The client scope uses:

```text
Network:      10.10.10.0/24
Scope Start:  10.10.10.100
Scope End:    10.10.10.200
DNS Server:   10.10.10.10
DNS Domain:   ballardlab.local
```

Lower addresses are reserved for statically addressed servers and future network infrastructure.

Before DHCP was configured, CLIENT01 received an APIPA address in the `169.254.0.0/16` range.

After DHCP deployment and scope activation, CLIENT01 successfully received an address in the configured client range along with the internal DNS server and domain suffix.

[View DHCP and DNS documentation](docs/03-dhcp-dns.md)

---

## Active Directory DNS Validation

DNS configuration was validated from CLIENT01.

### Forward Resolution

```text
dc01.ballardlab.local
        |
        v
10.10.10.10
```

### Active Directory Service Discovery

The LDAP Domain Controller SRV record was queried:

```text
_ldap._tcp.dc._msdcs.ballardlab.local
```

The DNS server returned:

```text
Service Target: dc01.ballardlab.local
Port:           389
IP Address:     10.10.10.10
```

This validated that the domain client could use DNS to discover the domain controller providing LDAP services for `ballardlab.local`.

---

## Access Validation

Access controls were validated using domain user accounts from separate departments.

### Finance User

`BALLARDLAB\smiller`

Expected access:

```text
\\DC01\Finance    -> Modify
\\DC01\Operations -> No Access
```

Validation included:

- Opening the Finance share
- Reading an existing file
- Modifying and saving the file
- Creating a new file
- Deleting the test file

All expected Modify operations completed successfully.

### Cross-Department Access Test

While authenticated as `BALLARDLAB\smiller`, access to:

```text
\\DC01\Operations
```

returned:

```text
Access Denied
```

This confirmed that the Finance user's security group memberships did not map to an allowed security principal on the Operations NTFS ACL.

The same positive access testing was completed for the Operations user against the Operations share.

---

## Troubleshooting and Validation

Several configuration issues were diagnosed through the build process.

### APIPA Address Assignment

CLIENT01 initially received a `169.254.x.x` address.

The client NIC was configured for DHCP, but no DHCP server was available on the virtual network.

After the DHCP role was configured, authorized, and provided an active scope, the client successfully received valid network configuration.

### Active Directory DNS

CLIENT01 was configured to use:

```text
10.10.10.10
```

as its DNS server.

Using an external resolver such as a public DNS service would prevent the client from discovering the private `ballardlab.local` Active Directory DNS records.

Forward and SRV record lookups were used to validate internal DNS and AD service discovery.

### Hyper-V Enhanced Session Authentication

A domain user was initially unable to sign in through Hyper-V Enhanced Session Mode because the account did not have Remote Desktop Services logon rights.

Rather than granting unnecessary RDP permissions to a standard Finance user, Enhanced Session Mode was disabled and the client was accessed through the standard VM console.

This preserved the intended least-privilege access model.

---

## Project Roadmap

- [x] Create isolated Hyper-V virtual network
- [x] Deploy Windows Server 2022
- [x] Configure static server IPv4 addressing
- [x] Deploy Active Directory Domain Services
- [x] Create `ballardlab.local` forest
- [x] Configure AD-integrated DNS
- [x] Deploy and authorize DHCP
- [x] Configure DHCP client scope and options
- [x] Deploy Windows 11 Pro client
- [x] Join CLIENT01 to the domain
- [x] Implement organizational unit structure
- [x] Create role-based Global security groups
- [x] Create resource-based Domain Local security groups
- [x] Implement AGDLP access control
- [x] Configure department SMB shares
- [x] Configure explicit NTFS ACLs
- [x] Validate positive user access
- [x] Validate cross-department access restrictions
- [x] Validate Active Directory DNS SRV records
- [ ] Deploy Group Policy Objects
- [ ] Automate department drive mapping
- [ ] Configure Windows security policies
- [ ] Configure Windows Defender Firewall policy
- [ ] Complete final infrastructure validation

---

## Documentation

- [Network Design](docs/01-network-design.md)
- [Active Directory](docs/02-active-directory.md)
- [DHCP and DNS](docs/03-dhcp-dns.md)
- [Access Control](docs/04-access-control.md)

---

## Skills Demonstrated

`Windows Server 2022` `Active Directory` `AD DS` `DNS` `DHCP` `Hyper-V` `Windows 11` `AGDLP` `NTFS Permissions` `SMB` `RBAC` `Least Privilege` `Infrastructure Troubleshooting`
