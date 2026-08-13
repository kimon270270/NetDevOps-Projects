# Containerlab + Ansible Spine-Leaf Network

A hands-on NetDevOps project that builds and configures a routed **spine-leaf network fabric** using **Containerlab, Arista cEOS, and Ansible**.

The project was built to explore how a modern Layer 3 spine-leaf topology works underneath the architecture, while also learning how network infrastructure can be deployed, configured, and verified through automation.

The project intentionally started from a simple design and evolved through multiple iterations as architectural and implementation problems were discovered and resolved.

---

## Project Goals

The primary goals of this project were to:

* Build a reproducible spine-leaf topology using Containerlab.
* Use Arista cEOS as the network operating system for the spine and leaf switches.
* Configure a Layer 3 spine-leaf underlay.
* Use point-to-point networks between spine and leaf switches.
* Use OSPF for dynamic routing.
* Understand and observe ECMP behavior.
* Configure server-facing networks and default gateways on the leaf switches.
* Use Ansible to configure the network after the devices are staged.
* Use Ansible to gather operational information from the network.
* Troubleshoot both networking and automation problems.
* Validate east-west connectivity between hosts in different subnets.

---

# Architecture

The topology consists of:

* 2 spine switches
* 4 leaf switches
* 8 Linux servers
* 8 routed spine-to-leaf connections


<img width="663" height="265" alt="Screenshot 2026-08-12 211830" src="https://github.com/user-attachments/assets/e307f054-9849-43ee-a1e6-5ff29dccbf0c" />


Each leaf is connected to both spines using independent Layer 3 point-to-point links.

The final fabric therefore provides multiple equal-cost paths between leaf switches.

---

# Technology Stack

| Technology   | Purpose                                  |
| ------------ | ---------------------------------------- |
| Containerlab | Network topology/container orchestration |
| Arista cEOS  | Spine and leaf network operating system  |
| Alpine Linux | End-host/server simulation               |
| Ansible      | Network configuration and verification   |
| OSPF         | Dynamic routing protocol                 |
| ECMP         | Multipath forwarding                     |
| Linux / WSL  | Lab and automation environment           |
| SSH          | Ansible network connectivity             |

---

# IP Addressing

The project uses the `192.168.10.0/24` address space.

## Server Networks

Each leaf has its own `/29` server-facing network.

| Leaf  | Network            | Default Gateway | Server 1        | Server 2        |
| ----- | ------------------ | --------------- | --------------- | --------------- |
| Leaf1 | `192.168.10.0/29`  | `192.168.10.1`  | `192.168.10.2`  | `192.168.10.3`  |
| Leaf2 | `192.168.10.8/29`  | `192.168.10.9`  | `192.168.10.10` | `192.168.10.11` |
| Leaf3 | `192.168.10.16/29` | `192.168.10.17` | `192.168.10.18` | `192.168.10.19` |
| Leaf4 | `192.168.10.24/29` | `192.168.10.25` | `192.168.10.26` | `192.168.10.27` |

The leaf switch provides the default gateway for the two servers directly connected to it.

---

## Spine-Leaf Point-to-Point Networks

Each spine-to-leaf connection uses a dedicated `/30`.

| Connection     | Network            |
| -------------- | ------------------ |
| Spine1 ↔ Leaf1 | `192.168.10.32/30` |
| Spine1 ↔ Leaf2 | `192.168.10.36/30` |
| Spine1 ↔ Leaf3 | `192.168.10.40/30` |
| Spine1 ↔ Leaf4 | `192.168.10.44/30` |
| Spine2 ↔ Leaf1 | `192.168.10.48/30` |
| Spine2 ↔ Leaf2 | `192.168.10.52/30` |
| Spine2 ↔ Leaf3 | `192.168.10.56/30` |
| Spine2 ↔ Leaf4 | `192.168.10.60/30` |

The point-to-point links are routed interfaces rather than Layer 2 switchports.

---

# Routing Design

The spine-leaf fabric uses **OSPF Area 0**.

Each leaf forms OSPF adjacencies with both spines.

The leaf advertises its directly connected server network into OSPF.

The spines learn the server networks and advertise the fabric routes.

Because each leaf has two equal-cost paths through the two spines, the network can use ECMP.

---

# Why Layer 3?

The spine-leaf portion of the topology is intentionally routed.

The original design initially considered using a common Layer 2 network across the fabric. During development, this approach was abandoned because it does not provide the desired routed fabric behavior and creates Layer 2 loop/multipath complications.

The final design therefore uses:

```text
Server
   |
   | Layer 2
   |
 Leaf
   |
   | Layer 3 /30
   |
 Spine
```

The server-facing side operates as Layer 2, while the spine-leaf fabric operates as Layer 3.

This allows OSPF to calculate routes through the fabric and allows redundant spine-leaf paths to participate in ECMP.

---

# Containerlab

Containerlab is responsible for creating the topology and starting the containers.

The topology definition contains:

* 2 Arista cEOS spine switches
* 4 Arista cEOS leaf switches
* 8 Alpine Linux servers
* All physical/logical links between the devices

The cEOS devices use:

```text
ceos:4.36.1F
```

The topology is defined in the Containerlab YAML file.

---

# Startup Configuration / Device Staging

The startup configurations should **not** be interpreted as the complete final configuration of the network.

They are used to stage the cEOS devices so that the devices can start and become accessible to the automation environment.

The general workflow is:

```text
Containerlab
     |
     v
Start cEOS devices
     |
     v
Startup / bootstrap configuration
     |
     v
Devices become accessible
     |
     v
Ansible connects to devices
     |
     v
Ansible applies network configuration
     |
     v
Ansible performs verification
```

The actual network configuration is performed through the Ansible playbooks.

---

# Ansible

Ansible is used to configure and verify the network after the devices have been staged.

The playbook contains separate configuration sections for:

* All devices
* Leaf switches
* Spine switches
* Information gathering / verification

---

## Configuration Performed by Ansible

### All Devices

The common configuration includes:

* MOTD banner
* OSPF process initialization
* OSPF reference bandwidth
* Configuration saving

Example OSPF configuration:

```text
router ospf 1
   area 0.0.0.0 default-cost 10
   auto-cost reference-bandwidth 10000
```

---

## Leaf Configuration

The leaf play configures:

* OSPF on the server-facing VLAN interface
* OSPF on the spine-facing routed interfaces
* Point-to-point OSPF network type
* Removal of the previous default route
* Server-facing interfaces as access ports
* VLAN 10

The leaf therefore performs both:

```text
Layer 2
Server-facing connectivity
```

and:

```text
Layer 3
Routing toward the spine fabric
```

---

## Spine Configuration

The spine configuration includes:

* Default route
* OSPF on spine-to-leaf interfaces
* Point-to-point OSPF network type
* OSPF default route advertisement

The spines therefore act as the upstream portion of the routed fabric.

---

# Verification

Ansible is also used for operational information gathering.

The project uses commands such as:

```text
show ip route
```

and:

```text
show ip ospf neighbor summary
```

to verify routing and OSPF operation.

During development, `cli_parse` was investigated as a way of converting CLI output into structured data. Due to implementation issues encountered with the parser environment, the project ultimately used `ansible.netcommon.cli_command` for operational information gathering.

---

# Server Configuration

The servers use Alpine Linux containers.

Each server receives:

* An IP address
* A subnet mask
* A default gateway
* An active Ethernet interface

For example:

```text
Server1-1
IP:      192.168.10.2/29
Gateway: 192.168.10.1
```

and:

```text
Server2-1
IP:      192.168.10.10/29
Gateway: 192.168.10.9
```

The server configuration is defined in the Containerlab topology.

The interface is brought up before assigning the IP address and default route.

---

# Expected Traffic Flow

For traffic between servers located on different leaf switches, the expected path is:

```text
Server
   |
   v
Leaf
   |
   v
Spine
   |
   v
Leaf
   |
   v
Server
```

For example:

```text
Server1-1
192.168.10.2/29
       |
       v
     Leaf1
       |
       v
    Spine1
       |
       v
     Leaf2
       |
       v
Server2-1
192.168.10.10/29
```

The alternate spine provides another equal-cost path:

```text
Server1-1
    |
  Leaf1
    |
  Spine2
    |
  Leaf2
    |
Server2-1
```

---

# ECMP

Because each leaf has two routed paths toward the spine layer, OSPF can install equal-cost routes.

For example, Leaf1 can reach networks behind Leaf2 through:

```text
Leaf1 → Spine1 → Leaf2
```

or:

```text
Leaf1 → Spine2 → Leaf2
```

The routing table can therefore contain multiple equal-cost next hops.

ECMP behavior was verified during the project using routing-table inspection and traceroute.

---

# Validation

After the final topology was deployed, east-west connectivity was tested between servers in different subnets.

The validation included:

* Server-to-server connectivity
* OSPF neighbor establishment
* OSPF route advertisement
* Routing table inspection
* Traceroute
* Observation of multiple paths
* Verification of the expected `Server → Leaf → Spine → Leaf → Server` forwarding path

The final topology successfully provided east-west connectivity between the server networks.

---

# Troubleshooting Performed

This project intentionally involved significant troubleshooting rather than being built entirely from a predetermined configuration.

Some of the problems encountered included:

### 1. Initial Layer 2 design

The original topology attempted to use a common network across the spine-leaf architecture.

This was redesigned into a routed L3 fabric.

### 2. Overlapping subnets

The first L3 redesign still reused the same address space across different Layer 3 segments.

This resulted in overlapping IP networks and prevented routing from working as intended.

The addressing plan was redesigned using separate `/29` server networks and `/30` point-to-point networks.

### 3. OSPF not advertising server networks

OSPF initially advertised the point-to-point networks but not the server-facing networks.

The issue was traced to the state of VLAN 10 and the server-facing access interfaces.

After configuring the access interfaces for VLAN 10, the corresponding SVI networks became active and were advertised through OSPF.

### 4. Alpine Linux interface behavior

The Alpine Linux container environment required the interface to be brought up before assigning the IP address.

The final sequence was:

```text
ip link set eth1 up
ip addr add <address>/<prefix> dev eth1
ip route replace default via <gateway> dev eth1
```

### 5. Ansible SSH/session problems

The project encountered:

* SSH permission problems
* Incorrect credential file paths
* SSH session errors
* Maximum session limits
* Python environment issues
* Ansible virtual environment issues
* pyATS/Genie dependency issues
* `cli_parse` problems

These issues were resolved or worked around during development.

---

# Prerequisites

Before deploying the project, install:

* Docker
* Containerlab
* Ansible
* SSH client
* Linux/WSL environment
* Arista cEOS image

You will also need access to the cEOS image referenced by the Containerlab topology.

The project uses:

```text
ceos:4.36.1F
```

---

# Security Notice

**Do not commit real credentials to GitHub.**

The project uses credential variables for Ansible authentication.

Before publishing the repository:

1. Remove any real passwords.
2. Do not commit private keys.
3. Do not commit sensitive tokens.
4. Replace credentials with example values.
5. Use Ansible Vault or another secure secret-management method for real credentials.

If credentials have ever been committed to a public repository, rotate them.

---

# Deployment

## 1. Clone the repository

```bash
git clone <repository-url>
cd <repository-directory>
```

## 2. Verify Containerlab

```bash
containerlab version
```

## 3. Verify Ansible

```bash
ansible --version
```

## 4. Verify the cEOS image

Make sure the required cEOS image is available to Containerlab:

```text
ceos:4.36.1F
```

## 5. Deploy the topology

Run Containerlab using the topology file:

```bash
sudo containerlab deploy -t topology.yml
```

Containerlab will create the spine switches, leaf switches, servers, and links defined in the topology.

---

# Configure the Network with Ansible

Once the topology is running, verify that the devices are reachable through the management network.

Then run the Ansible playbook from the repository.

For example:

```bash
ansible-playbook -i <inventory> <playbook>
```

Replace `<inventory>` and `<playbook>` with the files included in the repository.

The playbook will configure the devices and perform the configured verification tasks.

---

# Verify the Network

After Ansible completes, verify OSPF:

```text
show ip ospf neighbor
```

Verify the routing table:

```text
show ip route
```

Verify a specific learned network:

```text
show ip route <network>
```

Then test connectivity from the Linux servers.

For example:

```bash
ping 192.168.10.10
```

and:

```bash
traceroute 192.168.10.10
```

The expected path between servers on different leaf switches should traverse:

```text
Server → Leaf → Spine → Leaf → Server
```

---

# Destroy the Lab

When finished, remove the Containerlab topology:

```bash
sudo containerlab destroy -t topology.yml
```

---

# What This Project Demonstrates

This project demonstrates practical experience with:

### Networking

* IPv4 subnetting
* `/29` host networks
* `/30` point-to-point networks
* Layer 2 vs Layer 3 design
* Routed interfaces
* SVIs
* Default gateways
* OSPF
* OSPF Area 0
* ECMP
* Routing tables
* East-west traffic
* Network troubleshooting

### Network Automation

* Ansible inventory
* Ansible playbooks
* Network device automation
* `ansible.netcommon.cli_config`
* `ansible.netcommon.cli_command`
* SSH-based network automation
* Credential variables
* Operational verification

### Network Simulation / Lab Infrastructure

* Containerlab
* Arista cEOS
* Alpine Linux containers
* Linux networking
* Reproducible topology deployment

---

# Lessons Learned

One of the primary lessons from this project was that a spine-leaf topology is not simply a collection of switches connected in a particular shape.

The topology depends on an underlying routing architecture.

The final design uses:

```text
Layer 2
   ↓
Server-facing network

Layer 3
   ↓
Leaf-to-spine fabric

OSPF
   ↓
Dynamic route exchange

ECMP
   ↓
Multiple equal-cost paths
```

Another major lesson was that automation introduces its own engineering problems.

A network can be correctly designed while the automation environment is still broken because of:

* SSH problems
* Python environments
* dependencies
* credentials
* session limits
* incorrect inventory/configuration
* module behavior

Therefore, NetDevOps requires understanding both the network and the automation environment used to manage it.

---

# Current Limitations

This project is intentionally a relatively simple Layer 3 spine-leaf underlay.

It does **not** currently implement:

* VXLAN
* EVPN
* BGP EVPN
* MLAG
* EVPN multihoming
* VRFs
* Distributed anycast gateways
* Automated rollback
* CI/CD pipeline
* Automated failure/convergence testing
* Full configuration templating/data-driven configuration
* Redundant server connections

The project should therefore be viewed as a learning implementation of a **routed spine-leaf underlay**, rather than a complete production data-center fabric.

---

# Future Improvements

Potential future iterations could include:

* Convert hard-coded configuration into Jinja2 templates.
* Move topology-specific values into structured Ansible variables.
* Improve idempotency.
* Add automated pre-deployment validation.
* Add automated post-deployment validation.
* Test spine/leaf link failures.
* Measure OSPF convergence.
* Implement automated rollback strategies.
* Add Git-based version control workflows.
* Add CI validation.
* Introduce BGP as an alternative underlay protocol.
* Explore VXLAN/EVPN after the L3 underlay is fully understood.
* Add telemetry and structured operational data collection.

---

# Project Status

**Completed**

The final topology successfully deployed the intended routed spine-leaf network and provided east-west connectivity between servers across different leaf switches.

The project also served as a practical introduction to network automation using Ansible and Containerlab.

---

# Author

**Kimon Pandit Chhetri**

Network Engineering / NetDevOps Learning Project

---

## Disclaimer

This project is intended for learning and laboratory use.

The configuration should not be deployed directly into a production network without appropriate design review, security review, testing, change management, and adaptation to the target environment.
