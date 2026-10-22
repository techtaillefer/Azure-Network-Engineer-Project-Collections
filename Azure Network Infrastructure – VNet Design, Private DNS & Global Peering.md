# AZ-700 | Azure Network Infrastructure – VNet Design, Private DNS & Global Peering

**Lab Topics:** Virtual Networks and Subnetting, Private DNS Configuration, and Global VNet Peering  
**Platform:** Microsoft Azure  
**Environment:** Azure for Students Subscription  
**Tools:** Azure Portal, Azure Cloud Shell, PowerShell, ARM Templates

## Overview

For this module, I worked through three Azure networking labs building off another focused on building and understanding a basic cloud network infrastructure.

The goal was to get hands-on experience with how Azure Virtual Networks are designed, how IP address spaces are divided into subnets, how DNS resolution works for private resources, and how separate virtual networks can communicate across different Azure regions.

These labs were based on a fictional organization called Contoso, which needed a network environment for its core services, manufacturing operations, and research department.

The three areas I focused on were:

1. Designing and implementing Azure Virtual Networks and subnets.
2. Configuring private DNS for internal name resolution.
3. Connecting virtual networks in different regions through global VNet peering.

---

## Lab 1: Design and Implement a Virtual Network in Azure

### Objective

The objective of this lab was to create three virtual networks across different Azure regions and configure their respective subnets based on the organization's networking requirements.

### Network Configuration

I followed the Contoso network design using the following address spaces:

| Virtual Network | Azure Region | Address Space |
|---|---|---|
| CoreServicesVnet | East US | 10.20.0.0/16 |
| ManufacturingVnet | West Europe | 10.30.0.0/16 |
| ResearchVnet | Southeast Asia | 10.40.0.0/16 |

The resources were organized under a resource group named `ContosoResourceGroup`.

Each VNet was assigned its own non-overlapping IPv4 address space to avoid addressing conflicts and support future connectivity between networks.

### Subnet Configuration

**CoreServicesVnet – 10.20.0.0/16**

| Subnet | Address Range | Purpose |
|---|---|---|
| GatewaySubnet | 10.20.0.0/27 | Reserved for a virtual network gateway |
| SharedServicesSubnet | 10.20.10.0/24 | Shared infrastructure and services |
| DatabaseSubnet | 10.20.20.0/24 | Database resources |
| PublicWebServiceSubnet | 10.20.30.0/24 | Web-related workloads |

For instance here:
<img width="668" height="521" alt="image" src="https://github.com/user-attachments/assets/ef6d92d2-63a5-4305-a1b2-bc0a42ef3a15" />


**ManufacturingVnet – 10.30.0.0/16**

| Subnet | Address Range |
|---|---|
| ManufacturingSystemSubnet | 10.30.10.0/24 |
| SensorSubnet1 | 10.30.20.0/24 |
| SensorSubnet2 | 10.30.21.0/24 |
| SensorSubnet3 | 10.30.22.0/24 |

**ResearchVnet – 10.40.0.0/16**

| Subnet | Address Range |
|---|---|
| ResearchSystemSubnet | 10.40.0.0/24 |

### Understanding the Subnetting

One of the main things I focused on during this lab was understanding CIDR notation and how Azure divides networks into smaller subnet ranges.

For example, `10.20.0.0/16` uses 16 bits for the network prefix, leaving another 16 bits for addressing within the VNet.

A `/24` subnet uses 24 bits for its network prefix and leaves 8 bits for host addresses.

That means:

- `/16` = 65,536 total IPv4 addresses.
- `/24` = 256 total IPv4 addresses.
- `/27` = 32 total IPv4 addresses.

I also learned that Azure reserves five IP addresses within each subnet.

Therefore, a `/24` subnet provides 251 allocatable addresses, while a `/27` subnet provides 27.

This helped me understand why planning the IP address space is important before deploying resources. Separating workloads into different subnets provides better organization and allows network security and routing controls to be applied to specific parts of the environment.

### Key Takeaway

This lab helped me connect subnetting concepts with actual cloud infrastructure.

Instead of only calculating subnet masks and IP ranges, I could see how networks are organized inside Azure and why an organization might separate databases, web applications, shared services, and manufacturing systems.

I also gained a better understanding of how CIDR notation determines the number of addresses available in each subnet.

---

## Lab 2: Configure DNS Settings in Azure

### Objective

The purpose of this lab was to understand how Azure Private DNS allows virtual machines and other private resources to resolve domain names within Azure networks.

The lab uses a private DNS zone named `contoso.com`, which is linked to `CoreServicesVnet`.

### Private DNS Configuration

The intended configuration was:

| Setting | Value |
|---|---|
| DNS Zone Type | Azure Private DNS |
| Zone Name | contoso.com |
| Resource Group | ContosoResourceGroup |
| Linked VNet | CoreServicesVnet |
| VNet Link Name | CoreServicesVnetLink |
| Automatic Registration | Enabled |

The virtual network link is important because it allows resources within the linked network to resolve records in the private DNS zone.

When automatic registration is enabled, Azure can automatically manage DNS A records for supported virtual machines in that network.

### Understanding Private vs. Public DNS

One of the concepts I spent time reviewing was the difference between public and private DNS zones.

Public DNS is intended for domain names that need to be resolved through public DNS infrastructure, while Azure Private DNS is designed for private name resolution within connected Azure networks.

For example, a virtual machine could have a private IP address such as `10.20.20.4` and a DNS name like:

`TestVM1.contoso.com`

Instead of remembering the private IP address, another VM in the linked VNet could use the hostname to locate that resource.

The DNS service would resolve the hostname to its corresponding private IP address.

### DNS Testing

The lab uses two virtual machines, TestVM1 and TestVM2, to demonstrate internal DNS resolution.

<img width="481" height="140" alt="image" src="https://github.com/user-attachments/assets/1b561a51-fbc5-4b3f-9b52-52f400a018b7" />


The main commands used for verification are:

```powershell
ipconfig /all
```

This displays the virtual machine's network configuration, including its IP address and DNS information.

```powershell
nslookup TestVM2.contoso.com
```

This tests whether the DNS hostname resolves to the correct private IP address.

The lab also includes a ping test. However, an unsuccessful ping does not necessarily mean DNS resolution has failed. The Windows Firewall on the virtual machines can block ICMP traffic even when name resolution is working correctly.

<img width="463" height="271" alt="image" src="https://github.com/user-attachments/assets/b4c9b461-b5b1-4379-b048-6773458075c4" />


### Subscription Considerations

Because I am using an Azure for Students subscription, I also had to consider service availability, permissions, and the potential costs associated with creating additional resources.

I reviewed the required Private DNS configuration and the difference between Azure Private DNS and public DNS. Any restricted deployment or uncompleted VM testing needs to be documented separately rather than treated as a successful test.

### Key Takeaway

Private DNS is also important within cloud environments because applications and virtual machines need a reliable way to locate internal resources.

I learned how VNet linking and automatic DNS registration can simplify internal network management without relying on manually maintained IP-to-hostname mappings.

---

## Lab 3: Connect Two Azure Virtual Networks Using Global VNet Peering

### Objective

The objective of this lab was to understand how two Azure Virtual Networks in different geographic regions can communicate through global VNet peering.

The networks involved were:

| Virtual Network | Region | Address Space |
|---|---|---|
| CoreServicesVnet | East US | 10.20.0.0/16 |
| ManufacturingVnet | West Europe | 10.30.0.0/16 |

By default, creating two separate Azure VNets does not automatically establish connectivity between them.

VNet peering provides private connectivity between these networks through Microsoft's backbone infrastructure.

### Peering Configuration

The lab uses two peering links:

- `CoreServicesVnet-to-ManufacturingVnet`
- `ManufacturingVnet-to-CoreServicesVnet`

The configuration enables virtual network access between the two networks and permits forwarded traffic according to the lab settings.

Once both peering links are configured and connected, resources in the two VNets can communicate through their private IP addresses, subject to applicable security rules and routing.

Example:

<img width="482" height="155" alt="image" src="https://github.com/user-attachments/assets/fd47a6a6-018a-418f-8b63-d9035936869c" />


### Connectivity Testing

To understand the purpose of the peering connection, the lab compares connectivity before and after the peering is configured.

The test involves two virtual machines:

- **TestVM1:** Located in CoreServicesVnet.
- **ManufacturingVM:** Located in ManufacturingVnet.

From ManufacturingVM, the lab tests TCP connectivity to TestVM1 on the Remote Desktop Protocol port, 3389.

```powershell
Test-NetConnection 10.20.20.4 -Port 3389
```

The IP address should be replaced with TestVM1's actual private address if it differs.

**Before VNet peering:**

The connection is expected to fail because the two virtual networks do not yet have a route to communicate with each other through VNet peering.

<img width="190" height="32" alt="image" src="https://github.com/user-attachments/assets/ff94138f-26f4-4fe5-a170-163bbcfc9e91" />

**After VNet peering:**

With the peering connections established, the same TCP connectivity test is expected to succeed, assuming the relevant firewall and network security rules allow the traffic.

A successful test would display:

```text
TcpTestSucceeded : True
```

As seen here:

<img width="211" height="60" alt="image" src="https://github.com/user-attachments/assets/ad4f88ff-0308-4817-9406-822e11125a4c" />


This demonstrates how global VNet peering allows workloads deployed in different Azure regions to communicate privately.

### Understanding Global VNet Peering

One important concept I learned is that global VNet peering does not require traffic to travel across the public internet.

Azure uses Microsoft's backbone network to carry traffic between peered virtual networks.

This is useful when an organization has resources deployed in multiple geographic regions but still needs private connectivity between them.

I also learned that VNet peering is not automatically transitive. For example, if VNet A is peered with VNet B, and VNet B is peered with VNet C, that does not automatically mean VNet A can communicate with VNet C through those peerings.

### Key Takeaway

This lab helped me understand how private connectivity works between Azure networks.

It also reinforced the importance of checking network connectivity through testing rather than assuming two resources can communicate just because they exist within the same cloud environment.

Global VNet peering is particularly useful for organizations with workloads distributed across different Azure regions.

---

## Overall Learning and Reflection

Working through these three labs gave me a better understanding of how different Azure networking components fit together.

The first lab focused on building the network foundation using VNets, subnets, IP address ranges, and CIDR notation.

The second lab introduced the role of private DNS in allowing cloud resources to locate one another by hostname.

The third lab showed how separate virtual networks can be connected through global VNet peering, making private communication possible between different geographic regions.

The main concepts I took away from this module were:

- **IP addressing and subnetting:** Planning non-overlapping address ranges and understanding how CIDR notation affects the number of available IP addresses.
- **Network organization:** Separating workloads into appropriate subnets based on their roles.
- **Private DNS:** Using DNS zones and VNet links to support name resolution for private Azure resources.
- **Global VNet peering:** Connecting virtual networks across different Azure regions using Microsoft's private backbone.
- **Troubleshooting:** Using commands such as `ipconfig`, `nslookup`, and `Test-NetConnection` to examine networking configurations and connectivity.

Overall, this module helped me better understand the relationship between traditional networking concepts and their implementation in Microsoft Azure.

It reinforced that cloud networking still relies on familiar fundamentals such as IP addressing, subnetting, DNS, routing, and connectivity testing, but those concepts are implemented and managed through Azure networking services.

## References

**Lab Source:** [MicrosoftLearning AZ-700 – Designing and Implementing Microsoft Azure Networking Solutions.](https://learn.microsoft.com/en-us/training/paths/design-implement-microsoft-azure-networking-solutions-az-700/)
