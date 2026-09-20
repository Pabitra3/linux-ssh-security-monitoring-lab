# Networking Commands & Analysis

## 1. Overview

This document covers the networking commands used in the **Linux SSH Security Monitoring & Incident Investigation Lab**.

The purpose is to demonstrate how a SOC analyst can use network evidence to validate IP addresses, interfaces, routing paths, source/destination addresses, connectivity, SSH traffic paths, and security-investigation findings.

The lab uses **VirtualBox, Ubuntu Server, and Kali Linux**.

---

## 2. Lab Network Architecture

The lab uses two VirtualBox network types:

1. **NAT**
2. **Internal Network — SOC-LAB**

### NAT Network

NAT provides Internet access to the virtual machines.

| Machine | Interface | NAT IP |
|---|---|---|
| Ubuntu Server | `enp0s3` | `10.0.2.15/24` |
| Kali Linux | `eth0` | `10.0.2.6/24` |

NAT is primarily used for Internet connectivity, package installation, and operating-system updates.

### SOC-LAB Internal Network

The internal network provides isolated VM-to-VM communication.

| Machine | Interface | SOC-LAB IP |
|---|---|---|
| Ubuntu Server | `enp0s8` | `192.168.56.10/24` |
| Kali Linux | `eth1` | `192.168.56.20/24` |

The SOC-LAB network is used for Kali-to-Ubuntu communication, SSH testing, security-event generation, log analysis, and incident investigation.

### Logical Communication Flow

```text
                 Internet
                    |
              VirtualBox NAT
                    |
        +-----------+-----------+
        |                       |
   Ubuntu Server             Kali Linux
   10.0.2.15                 10.0.2.6
        |                       |
        +------ SOC-LAB --------+
          192.168.56.0/24

   Ubuntu: 192.168.56.10
   Kali:   192.168.56.20
```

---

# 3. `ip -br addr`

## Purpose

The `ip` command is the primary modern Linux networking utility.

The `addr` subcommand displays IP addresses assigned to network interfaces. The `-br` option produces a concise format.

### Command

```bash
ip -br addr
```

### Ubuntu Example

```text
enp0s3    UP    10.0.2.15/24
enp0s8    UP    192.168.56.10/24
```

### Kali Example

```text
eth0    UP    10.0.2.6/24
eth1    UP    192.168.56.20/24
```

## SOC Relevance

A SOC analyst can use this command to quickly determine:

- Which interfaces are active
- Which IP addresses are assigned
- Which interface belongs to the investigation network
- Whether an expected IP address is missing
- Whether the system is connected to multiple networks

---

# 4. Understanding the Lab Interfaces

## Ubuntu Server

### `enp0s3`

```text
10.0.2.15/24
```

Connected to VirtualBox NAT.

Primary purpose:

- Internet access
- Package installation
- Operating-system updates

### `enp0s8`

```text
192.168.56.10/24
```

Connected to the `SOC-LAB` Internal Network.

Primary purpose:

- Kali ↔ Ubuntu communication
- SSH testing
- Security monitoring

## Kali Linux

### `eth0`

```text
10.0.2.6/24
```

Connected to NAT.

### `eth1`

```text
192.168.56.20/24
```

Connected to the `SOC-LAB` Internal Network.

For the SSH investigation, the important source interface is `eth1`, with source IP `192.168.56.20`.

---

# 5. `ip route`

## Purpose

`ip route` displays the system's routing table.

The routing table tells Linux where packets should be sent.

### Command

```bash
ip route
```

The Ubuntu lab showed routes corresponding to both networks, including:

```text
192.168.56.0/24 dev enp0s8
```

and a NAT/default route through `enp0s3`.

## Important Concepts

### Destination Network

```text
192.168.56.0/24
```

This represents the SOC-LAB network.

### Interface

```text
enp0s8
```

Ubuntu uses this interface to communicate with Kali on the internal network.

### Default Route

The default route is used when a more specific route does not exist. Internet-bound traffic uses the NAT interface.

---

# 6. `ip route get`

## Purpose

`ip route get` shows how Linux would route traffic to a specific destination. It is particularly useful during security investigations.

### Command Used on Kali

```bash
ip route get 192.168.56.10
```

### Result

```text
192.168.56.10 dev eth1 src 192.168.56.20
```

## Interpretation

| Field | Meaning |
|---|---|
| `192.168.56.10` | Destination IP |
| `dev eth1` | Interface used |
| `src 192.168.56.20` | Source IP selected |

Therefore, Kali's SSH traffic to Ubuntu uses the isolated SOC-LAB interface and the expected source IP.

---

# 7. `ping`

## Purpose

`ping` tests basic IP connectivity between systems using ICMP Echo Request and Echo Reply messages.

### Kali → Ubuntu

```bash
ping -c 4 192.168.56.10
```

The lab produced:

```text
4 packets transmitted, 4 received, 0% packet loss
```

This establishes that Kali can reach Ubuntu over the SOC-LAB network.

However, **successful ping does not prove that SSH is available**. ICMP connectivity and TCP service availability are separate.

---

# 8. Testing the Opposite Direction

Ubuntu can test connectivity to Kali:

```bash
ping -c 4 192.168.56.20
```

If ICMP is permitted and the network is configured correctly, Ubuntu should receive replies.

The test should target the actual lab IP rather than an assumed gateway.

---

# 9. Why `192.168.56.1` Was Not Required

The following test was performed:

```bash
ping -c 4 192.168.56.1
```

It failed.

This did **not** mean the SOC-LAB network was broken. The VirtualBox Internal Network does not automatically provide a router or gateway at `192.168.56.1`.

The actual requirement was direct communication:

```text
Kali 192.168.56.20
        |
        |
Ubuntu 192.168.56.10
```

That communication worked successfully.

### SOC Lesson

Do not automatically treat an unreachable gateway as a network failure.

First determine:

1. What network architecture is intended?
2. Is a gateway actually required?
3. Does the required source-to-destination path work?
4. Does the routing table support that path?

---

# 10. NAT vs Internal Network

## NAT

NAT is used primarily for external connectivity:

```text
Ubuntu → NAT → Internet
```

## Internal Network

The `SOC-LAB` Internal Network provides isolated VM-to-VM communication:

```text
Kali → SOC-LAB → Ubuntu
```

This separation is useful in a security lab because testing can be kept isolated from the host's normal network.

---

# 11. SSH Network Path

The SSH test used:

```bash
ssh lexa@192.168.56.10
```

The network flow is:

```text
Kali
Source IP: 192.168.56.20
Interface: eth1
        |
        | TCP
        | Destination port 22
        |
        v
Ubuntu Server
Destination IP: 192.168.56.10
Interface: enp0s8
```

The SSH service listens on TCP port:

```text
22
```

This provides the network foundation for the authentication investigation.

---

# 12. Source and Destination Validation

When investigating a network security event, distinguish between:

- Source IP
- Destination IP
- Source interface
- Destination interface
- Destination port
- Protocol
- Timestamp

For this lab:

| Attribute | Value |
|---|---|
| Source system | Kali |
| Source IP | `192.168.56.20` |
| Source interface | `eth1` |
| Destination system | Ubuntu Server |
| Destination IP | `192.168.56.10` |
| Destination interface | `enp0s8` |
| Protocol | SSH over TCP |
| Destination port | `22` |

This information becomes useful when correlating network activity with authentication logs.

---

# 13. Network Evidence and SSH Logs

Network information alone does not explain what happened.

For example:

```text
192.168.56.20 → 192.168.56.10:22
```

establishes network communication.

SSH authentication logs provide additional context such as the username and authentication result.

Example:

```text
Failed password for lexa from 192.168.56.20
```

The source IP in the log matches the expected Kali source IP:

```text
192.168.56.20
```

This allows the analyst to correlate:

```text
Network Configuration
        +
Routing Evidence
        +
SSH Logs
        =
Higher-Confidence Investigation
```

---

# 14. Network Troubleshooting Methodology

A SOC analyst should troubleshoot network-related security alerts systematically.

## Step 1 — Check Interfaces

```bash
ip -br addr
```

Questions:

- Is the interface UP?
- Does it have an IP?
- Is the IP on the expected subnet?

## Step 2 — Check Routes

```bash
ip route
```

Questions:

- Is there a route to the destination?
- Which interface will be used?
- Is a default route available if Internet access is required?

## Step 3 — Check the Specific Path

```bash
ip route get 192.168.56.10
```

Questions:

- Which interface is selected?
- Which source IP will Linux use?
- Is the route consistent with the intended architecture?

## Step 4 — Check Connectivity

```bash
ping -c 4 192.168.56.10
```

Questions:

- Is the destination reachable?
- Is there packet loss?
- Is latency unexpectedly high?

A failed ping does not automatically mean the host is unreachable because ICMP may be blocked.

## Step 5 — Test the Application Service

For SSH:

```bash
sudo ss -lntp | grep ':22'
```

This determines whether TCP port 22 is listening on the server.

## Step 6 — Test Authentication

From Kali:

```bash
ssh lexa@192.168.56.10
```

If authentication succeeds, investigate the corresponding SSH log event.

If authentication fails, investigate `Failed password` events.

---

# 15. Common Networking Problems

## Problem: Interface Is Down

Check:

```bash
ip -br addr
```

If the expected interface is not `UP`, investigate the VirtualBox adapter configuration and Linux network configuration.

## Problem: Missing IP Address

Check:

```bash
ip -br addr
```

During the lab, Kali's SOC-LAB IP was manually configured with:

```bash
sudo ip addr add 192.168.56.20/24 dev eth1
```

This type of configuration is temporary and may not persist after reboot unless made persistent.

## Problem: Wrong Interface

Use:

```bash
ip route get 192.168.56.10
```

If Linux chooses an unexpected interface or source IP, investigate the routing table.

## Problem: Destination Unreachable

Check in order:

```bash
ip -br addr
```

```bash
ip route
```

```bash
ip route get 192.168.56.10
```

```bash
ping -c 4 192.168.56.10
```

Then verify the destination service if required.

---

# 16. SOC Investigation Workflow

A useful network investigation workflow is:

```text
Alert / Security Event
        |
        v
Identify Source IP
        |
        v
Identify Destination IP
        |
        v
Validate Interfaces
        |
        v
Validate Routing
        |
        v
Validate Connectivity
        |
        v
Identify Destination Service
        |
        v
Correlate With Logs
        |
        v
Determine Incident Context
        |
        v
Document Findings
```

This prevents an analyst from relying on a single data source.

---

# 17. Network Evidence in Incident Investigation

The networking work established that:

1. Ubuntu had the expected SOC-LAB IP:
   ```text
   192.168.56.10
   ```

2. Kali had the expected SOC-LAB IP:
   ```text
   192.168.56.20
   ```

3. Kali could reach Ubuntu over the internal network.

4. Kali's route to Ubuntu used:
   ```text
   eth1
   ```

5. Kali selected:
   ```text
   192.168.56.20
   ```
   as the source IP.

6. SSH traffic was directed toward:
   ```text
   192.168.56.10:22
   ```

7. SSH authentication logs recorded events from:
   ```text
   192.168.56.20
   ```

This correlation increased confidence that the observed SSH authentication events came from the intended lab source.

---

# 18. Commands Reference

## Display IP addresses

```bash
ip -br addr
```

## Display routing table

```bash
ip route
```

## Determine route to a destination

```bash
ip route get 192.168.56.10
```

## Test connectivity

```bash
ping -c 4 192.168.56.10
```

## Check SSH listening port

```bash
sudo ss -lntp | grep ':22'
```

## Check SSH service

```bash
sudo systemctl status ssh --no-pager
```

## View SSH service logs

```bash
sudo journalctl -u ssh --no-pager
```

## Find successful SSH authentication

```bash
sudo journalctl -u ssh --no-pager | grep "Accepted password"
```

## Find failed SSH authentication

```bash
sudo journalctl -u ssh --no-pager | grep "Failed password"
```

---

# 19. Key SOC Concepts Demonstrated

This networking exercise demonstrates practical understanding of:

- IPv4 addressing
- CIDR notation
- Network interfaces
- Routing tables
- Source IP selection
- Destination IP identification
- ICMP connectivity testing
- TCP service identification
- SSH network communication
- NAT
- Internal network segmentation
- Network-to-log correlation
- Security event validation
- Network troubleshooting

---

# 20. Interview Questions and Answers

## Q1. How did you identify the IP address of your Linux system?

I used:

```bash
ip -br addr
```

It provides a concise view of interfaces, interface state, and assigned IP addresses.

## Q2. How did you verify which interface Kali used to reach Ubuntu?

I used:

```bash
ip route get 192.168.56.10
```

The result showed:

```text
dev eth1 src 192.168.56.20
```

Therefore, Kali used `eth1` with source IP `192.168.56.20`.

## Q3. What is the difference between NAT and an Internal Network in your lab?

NAT provides Internet connectivity to the virtual machines, while the `SOC-LAB` Internal Network provides isolated VM-to-VM communication for the security lab.

## Q4. Does successful ping prove that SSH is working?

No.

Ping tests ICMP connectivity. SSH uses TCP, normally on port 22.

A host can respond to ping while SSH is unavailable, or ICMP can be blocked while SSH remains available.

## Q5. Why did you use `ip route get` during the investigation?

It allowed me to determine the exact interface and source IP Linux would use to reach the Ubuntu server.

That is useful for validating the origin of network activity and correlating it with security logs.

## Q6. Why is source IP validation important in a SOC investigation?

The source IP helps identify where an event originated. An analyst should validate that the IP belongs to the expected system before drawing conclusions about an incident.

In this lab, the SSH logs showed:

```text
192.168.56.20
```

which matched the Kali SOC-LAB address.

## Q7. Why should a SOC analyst correlate network evidence with logs?

A network connection tells you that communication occurred, while logs provide additional context such as:

- Username
- Authentication result
- Timestamp
- Process/service
- Session activity

Combining multiple evidence sources produces a more reliable investigation.

---

# 21. Final Investigation Summary

The networking portion of this SOC lab established the communication foundation required for SSH monitoring and incident investigation.

The final verified architecture was:

```text
Kali Linux
eth1
192.168.56.20
       |
       | SOC-LAB
       |
       v
Ubuntu Server
enp0s8
192.168.56.10
       |
       |
     SSH
    TCP/22
```

The NAT interfaces provide Internet connectivity separately:

```text
Kali       10.0.2.6
Ubuntu     10.0.2.15
```

The most important investigation evidence was the correlation between:

```text
Kali source IP
192.168.56.20
```

and the SSH authentication events recorded on:

```text
Ubuntu
192.168.56.10
```

This demonstrates a practical SOC methodology:

```text
Network Configuration
        ↓
Routing Validation
        ↓
Connectivity Testing
        ↓
Service Validation
        ↓
Log Correlation
        ↓
Security Investigation
        ↓
Incident Documentation
```

---

# 22. Level 2 SOC Skill Outcome

After completing this networking work, the project demonstrates practical ability to:

- Inspect Linux network configuration
- Understand interfaces and IP addressing
- Analyze routing information
- Validate source and destination addresses
- Troubleshoot basic connectivity
- Understand NAT versus isolated lab networking
- Identify the network path used by SSH
- Correlate network evidence with authentication logs
- Support an incident investigation with multiple evidence sources
- Document technical findings for a SOC workflow

The networking evidence is therefore not an isolated networking exercise. It directly supports the **SSH monitoring, authentication analysis, event correlation, and incident investigation** components of this SOC project.
