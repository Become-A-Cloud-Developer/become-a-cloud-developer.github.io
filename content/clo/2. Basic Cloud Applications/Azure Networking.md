+++
title = "Azure Networking"
weight = 3
date = 2024-03-03
draft = false
+++

# Introduction

Deploying ASP.NET applications on Azure requires an understanding of Azure Virtual Networks (VNets), subnets, Network Interface Cards (NICs), and firewall rules, including Network Security Groups (NSGs), Application Security Groups (ASGs), and Service Tags. These elements are critical for building a secure network architecture.

## Azure Virtual Networks and Subnets

Azure Virtual Networks (VNets) provide the foundational isolation in Azure needed for secure communication between resources. Subnets within VNets allow for further segmentation, offering control over IP address allocation and resource communication. For applications, organizing resources into subnets based on their roles (e.g., web tier vs. data tier) can improve both security and performance.

### Non-Routable IP Ranges

Non-routable IP ranges are used within VNets and subnets to ensure internal traffic is isolated from the public internet. The non-routable IP ranges are:

- 10.0.0.0 to 10.255.255.255
- 172.16.0.0 to 172.31.255.255
- 192.168.0.0 to 192.168.255.255

## Network Interface Cards (NICs)

NICs connect Azure resources like VMs to VNets, assigning IP addresses from the subnet they're connected to. NICs can have NSGs applied for detailed traffic control.

## Firewall Rule Concepts

### Network Security Groups (NSGs)

NSGs control inbound and outbound traffic to resources in a VNet. They can be applied at both the subnet and NIC levels, allowing developers to define detailed security rules based on protocol, source and destination IP address, and port number.

### Application Security Groups (ASGs)

ASGs label VMs by function, allowing NSG rules to be applied to the group rather than individual VMs. This is useful for managing dynamic environments where VMs are frequently added or removed.

### Service Tags

Service Tags simplify NSG rule creation by representing Azure services' IP address ranges. This allows developers to create rules targeting specific Azure services without listing individual IP addresses.

## Best Practices

1. **Network Segmentation**: Use subnets for network segmentation based on the application's architecture to isolate different application tiers.
2. **Least Privilege**: Implement NSGs and ASGs following the principle of least privilege, restricting communication to what is necessary.
3. **Dynamic Policies**: Use ASGs for security policy management in dynamic environments.
4. **Service Tags**: Use service tags in NSG rules for simplified management to Azure services.
5. **Monitoring**: Continuously monitor network security settings and logs to identify and address potential security issues.










## Application Security Groups (ASGs)

ASGs allow developers and network administrators to label virtual machines (VMs) based on their roles or functions within a network. This logical grouping enables easier configuration of NSG rules, without the need to specify individual IP addresses. ASGs are particularly useful in dynamic environments where VMs are frequently added or removed, as they allow security rules to automatically apply to all members of a group.

## Practical Example: Securing an ASP.NET Application

Consider a scenario where you have an ASP.NET application deployed in Azure, with the following architecture:

- A Linux VM serving as a **Bastion Host** for secure SSH access.
- Another Linux VM functioning as a **Reverse Proxy** to direct web traffic to the application server.
- An application server running the **ASP.NET Web App** on port 5000.

To secure this setup, we'll use ASGs to group these VMs based on their roles and apply NSG rules to manage traffic flow efficiently.

### Step 1: Define ASGs

We start by creating three ASGs:

- **BastionASG**: Includes the Bastion Host VM.
- **ReverseProxyASG**: Contains the Reverse Proxy VM.
- **WebAppASG**: Groups VMs running the ASP.NET application.

### Step 2: Attach NSG Rules to Subnet

Next, we attach an NSG to the subnet hosting our VMs and define the following rules:

| Priority | Name                       | Port | Protocol | Source            | Destination       | Action | Description                                                                 |
|----------|----------------------------|------|----------|-------------------|-------------------|--------|-----------------------------------------------------------------------------|
| 100      | Allow-SSH-Bastion          | 22   | TCP      | Internet          | BastionASG        | Allow  | Allows SSH access to the Bastion Host.                                      |
| 200      | Allow-HTTP-ReverseProxy    | 80   | TCP      | Internet          | ReverseProxyASG   | Allow  | Allows HTTP traffic to the Reverse Proxy.                                   |
| 210      | Allow-HTTPS-ReverseProxy   | 443  | TCP      | Internet          | ReverseProxyASG   | Allow  | Allows HTTPS traffic to the Reverse Proxy.                                  |
| 300      | Allow-ReverseProxy-To-WebApp     | 5000 | TCP      | ReverseProxyASG   | WebAppASG         | Allow  | Allows traffic from the Reverse Proxy to the ASP.NET Web App.               |
| 400      | Deny-Internet-To-WebApp    | *    | Any      | Internet          | WebAppASG         | Deny   | Blocks direct internet access to the ASP.NET Web App.                       |
|    | Other rules           |     |      |               |               |   |      |

### Key Benefits of Using ASGs in This Scenario

- **Simplified Management**: By grouping VMs into ASGs, we can apply NSG rules more broadly, without targeting individual VMs. This simplifies rule management, especially as the environment changes.
- **Dynamic Security Posture**: Adding or removing VMs from an ASG automatically updates the security posture without requiring manual NSG rule adjustments.
- **Enhanced Security**: ASGs help ensure that only the necessary traffic is allowed to each part of the application architecture, minimizing the attack surface.


## Conclusion

Understanding and utilizing VNets, subnets, NICs, NSGs, ASGs, and service tags are essential for protecting applications from external threats and ensuring they operate securely within Azure's cloud environment. Regular monitoring and management of these settings are key to maintaining security.