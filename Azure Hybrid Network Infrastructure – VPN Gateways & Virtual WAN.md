

# AZ-700 | Design and Implement Hybrid Networking

**Project Title:** Azure Hybrid Network Infrastructure – VPN Gateways & Virtual WAN

**Platform:** Microsoft Azure
**Environment:** Azure for Students Subscription
**Tools:** Azure Portal, Azure Cloud Shell, PowerShell, ARM Templates
**Focus Areas:** Hybrid Connectivity, VPN Gateways, IPsec/IKE Tunnels, Virtual WAN, Hub-and-Spoke Networking, and Routing



## Project Description

This project focuses on designing and configuring hybrid networking infrastructure in Microsoft Azure to establish secure connectivity between virtual networks across different geographic regions. Building on the virtual network configurations from Module 1, I explored Azure VPN Gateways, VNet-to-VNet VPN connections, and Azure Virtual WAN to understand different approaches to connecting distributed cloud environments. The labs cover deploying network gateways, establishing encrypted VPN tunnels, configuring centralized virtual hubs, and connecting VNets through a hub-and-spoke architecture. These implementations demonstrate how Azure networking services support secure communication, centralized connectivity management, and scalable enterprise network designs.

<img width="1184" height="492" alt="image" src="https://github.com/user-attachments/assets/ad476f40-d6eb-4581-b4e3-9aff83438773" />


## Overview

For this project, I focused on understanding how Microsoft Azure supports connectivity between separate networks using VPN gateways and centralized network architectures.

Building on the virtual networks created in the prior project, the goal was to explore how Azure can securely connect network environments across different regions without requiring resources to communicate directly over the public internet.

The module consisted of two hands-on labs:

1. **Create and Configure a Virtual Network Gateway**
2. **Create a Virtual WAN Using the Azure Portal**

These exercises focused on the differences between connecting networks through individual VPN gateways and managing multiple network connections through a centralized Virtual WAN architecture.

---

# Lab 1: Create and Configure a Virtual Network Gateway

## Objective

The objective of this lab was to configure Azure VPN Gateways to establish a secure connection between two virtual networks located in different Azure regions.

The scenario involved connecting Contoso's Core Services network in East US with its Manufacturing network in West Europe.

Unlike the global VNet peering configuration covered in Module 1, this lab demonstrates how Azure VPN Gateways establish encrypted connections using IPsec/IKE VPN tunnels.

## Network Configuration

The environment uses two virtual networks with separate, non-overlapping IP address spaces.

| Resource          | Azure Region | Address Space |
| ----------------- | ------------ | ------------- |
| CoreServicesVnet  | East US      | 10.20.0.0/16  |
| ManufacturingVnet | West Europe  | 10.30.0.0/16  |

Both networks are deployed under the resource group `ContosoResourceGroup`.

A virtual machine is deployed into each network to test connectivity before and after establishing the VPN connection.

| Virtual Machine | Virtual Network   | Purpose                                 |
| --------------- | ----------------- | --------------------------------------- |
| CoreServicesVM  | CoreServicesVnet  | Destination VM for connectivity testing |
| ManufacturingVM | ManufacturingVnet | Source VM for connectivity testing      |

### Deploying the Network Infrastructure

The lab uses Azure Resource Manager (ARM) templates to provision the virtual networks and virtual machines.

These templates are deployed through PowerShell in Azure Cloud Shell.

The initial deployment uses the following commands:

```powershell
$RGName = "ContosoResourceGroup"

New-AzResourceGroup -Name $RGName -Location "eastus"

New-AzResourceGroupDeployment `
    -ResourceGroupName $RGName `
    -TemplateFile azuredeploy.json `
    -TemplateParameterFile azuredeploy.parameters.json
```

Using ARM templates helped me understand how Azure resources can be provisioned through Infrastructure as Code rather than configuring every component individually through the portal.

The templates define the required infrastructure and allow deployments to be repeated more consistently.

## Configuring the VPN Gateways

The next part of the lab involves creating a VPN gateway for each virtual network.

Each gateway requires a dedicated subnet named `GatewaySubnet`, which Azure uses to host the managed gateway infrastructure.

### CoreServices VPN Gateway

| Setting            | Configuration           |
| ------------------ | ----------------------- |
| Gateway Name       | CoreServicesVnetGateway |
| Region             | East US                 |
| Gateway Type       | VPN                     |
| SKU                | VpnGw1AZ                |
| Generation         | Generation1             |
| Virtual Network    | CoreServicesVnet        |
| Gateway Subnet     | 10.20.0.0/27            |
| Public IP Type     | Standard                |
| Active-Active Mode | Disabled                |
| BGP                | Disabled                |

For example:

<img width="269" height="167" alt="image" src="https://github.com/user-attachments/assets/2f5ffe7e-49e1-4783-ba72-e33f9ea36b5e" />


### Manufacturing VPN Gateway

| Setting            | Configuration               |
| ------------------ | --------------------------- |
| Gateway Name       | ManufacturingVnetGateway    |
| Region             | West Europe                 |
| Gateway Type       | VPN                         |
| SKU                | VpnGw1AZ                    |
| Generation         | Generation1                 |
| Virtual Network    | ManufacturingVnet           |
| Gateway Subnet     | Dedicated /27 GatewaySubnet |
| Public IP Type     | Standard                    |
| Active-Active Mode | Disabled                    |
| BGP                | Disabled                    |

And as seen here:

<img width="241" height="167" alt="image" src="https://github.com/user-attachments/assets/8663eeb4-3138-438c-b752-a52482574940" />


One important concept I learned was that a VPN gateway is not simply a virtual machine running VPN software.

Azure VPN Gateway is a managed networking service that provides VPN connectivity between Azure virtual networks and other network environments.

The `GatewaySubnet` is specifically reserved for this service and should not be used for ordinary application workloads.

### Understanding the Gateway Subnet

The CoreServices gateway uses:

`10.20.0.0/27`

The `/27` prefix means that 27 bits are used for the network portion, leaving five bits for host addressing.

This provides:

- 32 total IPv4 addresses.
- 27 allocatable IP addresses after Azure reservations.
- A dedicated address range for the managed VPN gateway infrastructure.

The gateway subnet must exist within the VNet's overall address space and must not overlap with any other subnet.

This reinforced the importance of allocating address space for network infrastructure during the initial design process.

---

## Establishing the VNet-to-VNet VPN Connection

After deploying both VPN gateways, the next objective is to establish connectivity between them.

The lab creates two connection resources, one from each gateway.

| Setting         | Configuration                           |
| --------------- | --------------------------------------- |
| Connection 1    | CoreServicesGW-to-ManufacturingGW       |
| Connection 2    | ManufacturingGW-to-CoreServicesGW       |
| Connection Type | VNet-to-VNet                            |
| VPN Protocol    | IKEv2                                   |
| Authentication  | Pre-Shared Key (PSK)                    |
| Shared Key      | Same key configured on both connections |
| BGP             | Disabled                                |

The connection uses IPsec for protecting network traffic and IKEv2 for negotiating the security association between the VPN gateways.

Both gateways must be configured with matching authentication information for the VPN connection to establish successfully.

The Microsoft lab uses `abc123` as its example pre-shared key. In a production environment, I would use a strong, randomly generated key stored securely.

Example:

<img width="312" height="188" alt="image" src="https://github.com/user-attachments/assets/0730a59c-ca54-4e4a-9469-6e1ec281485b" />


### How the Connection Works

The intended traffic flow is:

```text
                 MICROSOFT AZURE

  EAST US                         WEST EUROPE
  ───────                         ───────────

  CoreServicesVnet                ManufacturingVnet
  10.20.0.0/16                    10.30.0.0/16
         |                               |
         |                               |
  CoreServicesVM                  ManufacturingVM
         |                               |
         |                               |
  CoreServicesVnetGateway         ManufacturingVnetGateway
         |                               |
         |                               |
         +------ IPsec/IKEv2 VPN --------+
                  Encrypted Tunnel
```

The VPN gateways establish a secure logical tunnel between the two networks.

Although the gateways use public IP addresses for their VPN endpoints, the virtual machines communicate using their private IP addresses.

The traffic is encrypted as it passes between the VPN gateway endpoints.

---

## Testing Network Connectivity

The lab demonstrates the importance of verifying connectivity both before and after configuring the VPN.

### Before Establishing the VPN

From ManufacturingVM, the following PowerShell command tests connectivity to CoreServicesVM:

```powershell
Test-NetConnection 10.20.20.4 -Port 3389
```

Port `3389` is used by Remote Desktop Protocol (RDP).

The expected result before the VPN connection is established is:

```text
TcpTestSucceeded : False
```

This indicates that the virtual machines cannot establish the requested TCP connection.

The test demonstrates that separate virtual networks do not automatically have connectivity simply because they exist within Azure.

### After Establishing the VPN

Once both VPN connections are configured, their status should display:

`Connected`

Repeating the connectivity test should produce:

```text
TcpTestSucceeded : True
```

A successful result demonstrates that the networks have a functioning communication path through their VPN gateways, provided the necessary firewall and network security rules permit the traffic.

<img width="284" height="221" alt="image" src="https://github.com/user-attachments/assets/749fdbb7-b9ed-4356-af82-788d9ccf5173" />


## Key Takeaways

This lab helped me understand how VPN gateways provide encrypted communication between separate Azure network environments.

The main concepts I focused on were:

- **VPN Gateways:** Azure-managed services used to establish secure VPN connections.
- **GatewaySubnet:** A dedicated subnet reserved for VPN gateway infrastructure.
- **IPsec/IKEv2:** Protocols used to negotiate and protect VPN communication.
- **VNet-to-VNet VPN:** A method for establishing encrypted connections between Azure virtual networks.
- **Network Testing:** Verifying connectivity with PowerShell before and after making network changes.

I also gained a better understanding of how VPN gateways differ from VNet peering.

While VNet peering provides direct private connectivity over Microsoft's backbone network, VPN gateways establish IPsec-encrypted tunnels and can also support connectivity with external, on-premises networks.

---

# Lab 2: Create a Virtual WAN Using the Azure Portal

## Objective

The objective of this lab was to understand how Azure Virtual WAN can simplify connectivity management by combining networking and routing capabilities within a centralized architecture.

Instead of configuring individual connections between multiple virtual networks, Azure Virtual WAN provides a hub-and-spoke model designed to support larger, geographically distributed environments.

The exercise focuses on creating a Virtual WAN, deploying a virtual hub, and connecting an existing Azure VNet to that hub.

## Understanding Azure Virtual WAN

Azure Virtual WAN is a managed networking service that integrates several networking capabilities, including:

- Site-to-Site VPN connectivity.
- Point-to-Site VPN connectivity.
- ExpressRoute connectivity.
- Virtual network connections.
- Centralized routing and connectivity management.

The architecture uses virtual hubs as central connectivity points for connected networks.

This approach reduces the need to manage large numbers of individual network-to-network connections.

## Creating the Virtual WAN

The first step in the lab is to create a Standard Azure Virtual WAN resource.

The configuration uses the following settings:

| Setting              | Configuration        |
| -------------------- | -------------------- |
| Resource Name        | ContosoVirtualWAN    |
| Resource Group       | ContosoResourceGroup |
| Virtual WAN Type     | Standard             |
| Management Interface | Azure Portal         |

One important concept I learned is that a Virtual WAN resource is global.

Although Azure requires a resource location when creating it, the Virtual WAN itself can contain virtual hubs deployed across different regions.

This allows organizations to centrally manage geographically distributed network connectivity.

---

## Creating the Virtual Hub

After creating the Virtual WAN, the next step is to deploy a virtual hub.

The virtual hub acts as the central connectivity and routing component within the Virtual WAN architecture.

### Virtual Hub Configuration

| Setting                  | Configuration                  |
| ------------------------ | ------------------------------ |
| Hub Name                 | ContosoVirtualWANHub-WestUS    |
| Azure Region             | West US                        |
| Address Space            | 10.60.0.0/24                   |
| Virtual Hub Capacity     | 2 Routing Infrastructure Units |
| Site-to-Site VPN Gateway | Enabled                        |
| VPN Gateway Scale Units  | 1                              |
| Hub Routing Preference   | Default                        |

The hub uses the private address space:

`10.60.0.0/24`

This provides a distinct, non-overlapping address range for the virtual hub infrastructure.

Unlike the ordinary subnets created in Module 1, the virtual hub address space is reserved for the managed Virtual WAN hub infrastructure rather than application virtual machines.

### Understanding Hub-and-Spoke Networking

The main concept behind Virtual WAN is hub-and-spoke architecture.

In this design:

**Hub:** Central networking infrastructure responsible for providing connectivity and routing.

**Spokes:** Networks or other environments connected to the central hub.

A simplified example looks like this:

```text
                  AZURE VIRTUAL WAN
                    ContosoVirtualWAN
                           |
                           |
                +--------------------+
                |                    |
                |  Virtual WAN Hub   |
                |                    |
                |     West US        |
                |    10.60.0.0/24    |
                |                    |
                +--------------------+
                           |
                           |
                    VNet Connection
                           |
                           |
                +--------------------+
                |                    |
                |    ResearchVnet    |
                |                    |
                |    Southeast Asia  |
                |    10.40.0.0/16    |
                |                    |
                +--------------------+
```

The lab only connects one VNet to the hub, but additional networks could be connected as the environment grows.

This helped me understand why centralized network architectures are useful for organizations operating across multiple regions.

---

## Connecting ResearchVnet to the Virtual Hub

The final configuration task involves connecting an existing virtual network to the Virtual WAN hub.

The lab uses the ResearchVnet from Module 1.

### Research Virtual Network

| Setting         | Configuration        |
| --------------- | -------------------- |
| Virtual Network | ResearchVnet         |
| Region          | Southeast Asia       |
| Address Space   | 10.40.0.0/16         |
| Resource Group  | ContosoResourceGroup |

If the virtual network was removed after completing an earlier exercise, it can be recreated using Azure PowerShell.

```powershell
$existingVnet = Get-AzVirtualNetwork `
    -Name "ResearchVnet" `
    -ResourceGroupName "ContosoResourceGroup" `
    -ErrorAction SilentlyContinue

if (-not $existingVnet) {
    New-AzVirtualNetwork `
        -Name "ResearchVnet" `
        -ResourceGroupName "ContosoResourceGroup" `
        -Location "southeastasia" `
        -AddressPrefix "10.40.0.0/16"
}
```

This script first checks whether ResearchVnet already exists.

If Azure cannot find the network, the script creates it using the specified address space and location.

### Virtual Network Connection

The following settings are used to connect ResearchVnet to the virtual hub:

| Setting                | Configuration                     |
| ---------------------- | --------------------------------- |
| Connection Name        | ContosoVirtualWAN-to-ResearchVNet |
| Virtual WAN            | ContosoVirtualWAN                 |
| Virtual Hub            | ContosoVirtualWANHub-WestUS       |
| Connected VNet         | ResearchVnet                      |
| Resource Group         | ContosoResourceGroup              |
| Propagate to None      | Yes                               |
| Associated Route Table | Default                           |

The connection attaches ResearchVnet to the Virtual WAN hub.

One of the more important networking concepts introduced here is **route propagation**.

### Understanding Route Propagation

In Azure Virtual WAN, route propagation determines which route tables receive routes advertised by a connection.

The lab sets:

`Propagate to None: Yes`

This means the connection's routes are not propagated to other hub route tables through the normal propagation configuration.

The connection is still associated with the default route table, but route association and route propagation are separate settings.

This distinction matters because simply connecting a VNet to a hub does not mean every connected network will automatically be able to communicate with it.

Routing policies, route tables, and security configurations all influence which networks can exchange traffic.

## Key Takeaways

This lab helped me understand how Azure Virtual WAN provides an alternative to managing separate network connections individually.

The concepts I focused on included:

- **Azure Virtual WAN:** Centralized management of networking, connectivity, and routing capabilities.
- **Virtual Hubs:** Managed connectivity points that support gateways and VNet connections.
- **Hub-and-Spoke Architecture:** A network design that centralizes connectivity rather than relying on many individual connections.
- **Route Propagation:** Controlling how network routes are distributed between connected environments.
- **Scalability:** Designing network connectivity that can expand to multiple Azure regions and external networks.

The exercise also introduced the difference between Basic and Standard Virtual WAN.

Standard Virtual WAN supports a broader range of connectivity options, including VNet connectivity and ExpressRoute, making it suitable for more complex networking environments.

---

# Overall Learning and Reflection

Module 2 expanded on the Azure networking concepts introduced in Module 1 by focusing on how separate networks can be securely connected and centrally managed.

The first lab demonstrated the configuration of Azure VPN Gateways and the use of IPsec/IKEv2 to establish encrypted connectivity between virtual networks.

The second lab introduced Azure Virtual WAN and its hub-and-spoke architecture, showing how organizations can centralize network connectivity and routing rather than configuring every network connection independently.

One of the biggest concepts I took away from this module was understanding that **network connectivity can be implemented in several ways depending on the environment's requirements**.

For example:

| Technology        | Primary Purpose                                                             |
| ----------------- | --------------------------------------------------------------------------- |
| VNet Peering      | Private connectivity between Azure VNets                                    |
| Azure VPN Gateway | Encrypted VPN connectivity between networks                                 |
| Azure Virtual WAN | Centralized connectivity and routing across multiple networks               |
| ExpressRoute      | Private connectivity between external networks and Microsoft cloud services |

Each approach serves different architectural requirements involving security, scalability, performance, and operational complexity.

I also gained a better understanding of how important IP address planning remains when implementing hybrid networking.

VPN gateways, virtual hubs, and connected networks all require properly planned, non-overlapping address spaces to avoid routing conflicts.

Overall, this module strengthened my understanding of how enterprise cloud environments connect resources across multiple networks and geographic regions.

It also reinforced the relationship between traditional networking concepts, such as VPN tunnels, routing, CIDR addressing, and hub-and-spoke topology, and how those technologies are implemented through Microsoft Azure's managed networking services.

---

## References

**Microsoft Learning – AZ-700: Designing and Implementing Microsoft Azure Networking Solutions**
[Microsoft Learn Intermediate Course – AZ-700](https://learn.microsoft.com/en-us/training/paths/design-implement-microsoft-azure-networking-solutions-az-700/)
