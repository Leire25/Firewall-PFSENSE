# pfSense Firewall Lab — Network Segmentation & ICMP flood Defense

# Overview

This project is a home lab built with VirtualBox on my Ubuntu Desktop. I created virtual machines for pfSense, Kali Linux, and Ubuntu Desktop to simulate a segmented network environment.

The objective was to create a small segmented network where pfSense acts as the firewall between an external/WAN network and an isolated internal/LAN network.

The lab was then used to simulate an attack from Kali Linux against an Ubuntu host using hping3, observe the traffic with Wireshark, and finally mitigate the traffic using a pfSense firewall rule.

As an additional challenge, I decided to build and configure the VirtualBox machines from the Linux terminal using VBoxManage instead of relying on the VirtualBox graphical interface.

This gave me practical experience with:

- VirtualBox administration
- Linux command-line tools
- Virtual networking
- Routing
- Firewall configuration
- Packet analysis
- Troubleshooting

---

## Objectives

- Simulate a network using VirtualBox.
- Create and configure the VMs from the Linux terminal.
- Configure pfSense as firewall/router.
- Configure DHCP on the pfSense LAN interface.
- Configure Kali Linux as an external/attacker host.
- Configure Ubuntu as the protected internal host.
- Configure routing between the WAN and internal network.
- Generate controlled traffic with `hping3`.
- Capture and analyze traffic using Wireshark.
- Create a pfSense firewall rule to block the traffic and stop the simulated attack.
- Verify the mitigation using Wireshark and pfSense logs.
  
---

# 1. Lab Architecture

The lab consists of three virtual machines:

| Machine | Role | Network |
|---|---|---|
| pfSense | Firewall / Router | WAN + LAN |
| Kali Linux | Attacker | WAN |
| Ubuntu Desktop | Protected Host | LAN |

### Network Topology

```text
                    Home Network
                  192.168.1.0/24
                         |
                         |
                 +-------+-------+
                 |   VirtualBox  |
                 +-------+-------+
                         |
              +----------+----------+
              |                     |
            Kali                 pfSense
          Attacker             Firewall/Router
        192.168.1.x            WAN: 192.168.1.x
                                    |
                                    |
                              LAN: 192.168.50.1
                                    |
                              Internal Network
                                    "int"
                                    |
                                 Ubuntu
                               192.168.50.x
(Taking a value from the configured pool 192.168.50.10 - 192.168.50.100)
```
The lab is divided into two networks:
```text
      WAN
192.168.1.0/24
       |
    pfSense
       |
      LAN
192.168.50.0/24
       |
    Ubuntu
```
With the goal of having Kali connected to the WAN side while Ubuntu is isolated behind the pfSense LAN interface.

---

# 2. Virtual Machines

The following table shows the specifications and network configuration used for each virtual machine in this lab.

## pfSense

| Setting   | Configuration            |
| --------- | ------------------------ |
| OS        | pfSense CE               |
| CPU       | 2 vCPU                   |
| RAM       | 2 GB                     |
| Disk      | 16 GB                    |
| Adapter 1 | Bridged — WAN            |
| Adapter 2 | Internal Network — `int` |
| Role      | Firewall / Router        |

## Ubuntu

| Setting   | Configuration            |
| --------- | ------------------------ |
| OS        | Ubuntu Desktop           |
| CPU       | 2 vCPU                   |
| RAM       | 4 GB                     |
| Disk      | 40 GB                    |
| Adapter 1 | Internal Network — `int` |
| Role      | Protected Host           |

## Kali Linux

| Setting   | Configuration     |
| --------- | ----------------- |
| OS        | Kali Linux        |
| CPU       | 2 vCPU            |
| RAM       | 4 GB              |
| Disk      | 40 GB             |
| Adapter 1 | Bridged — WAN     |
| Role      | Attacker          |

---

# 3. VirtualBox Configuration from the Terminal

One of the main challenges I added to the original lab was creating and configuring the virtual machines without using the VirtualBox GUI.

I used `VBoxManage` from the Ubuntu host to:

- Create and register the VMs.
- Configure CPU and memory.
- Configure network adapters.
- Configure the internal network.
- Create virtual disks.
- Attach storage controllers.
- Attach ISO images.
- Start and stop the virtual machines.
- Inspect the resulting configuration.

>**Note:** Network interface names are replaced with placeholders to avoid exposing details about my local network.

The following are the steps used to create pfSense, I simply used the same steps for the rest of the VMs with the necessary changes.

pfSense is FreeBSD-based and needs **two NICs**: one bridged for WAN, one internal for LAN.

### Step 1 — Create and register the VM

```bash
VBoxManage createvm --name "pfSense" --ostype "FreeBSD_64" --register
```

### Step 2 — Configure the hardware

```bash
VBoxManage modifyvm "pfSense" --memory 2048 --cpus 2 --vram 32 --firmware bios --boot1 dvd --boot2 disk --boot3 none --boot4 none
```

### Step 3 — Configure NIC 1 (WAN, bridged in this case)

```bash
VBoxManage modifyvm "pfSense" --nic1 bridged --bridgeadapter1 "HOST_NIC"
```

### Step 4 — Configure NIC 2 (LAN, internal network `int`)

```bash
VBoxManage modifyvm "pfSense" --nic2 intnet --intnet2 "int"
```

### Step 5 — Create the virtual disk (16 GB)

```bash
VBoxManage createmedium disk --filename "$HOME/VirtualBox VMs/pfSense/pfSense.vdi" --size 16384 --format VDI
```

### Step 6 — Add an IDE controller

```bash
VBoxManage storagectl "pfSense" --name "IDE" --add ide --controller PIIX4 --bootable on
```

### Step 7 — Attach the hard disk

```bash
VBoxManage storageattach "pfSense" --storagectl "IDE" --port 0 --device 0 --type hdd --medium "$HOME/VirtualBox VMs/pfSense/pfSense.vdi"
```

### Step 8 — Attach the pfSense ISO 

```bash
VBoxManage storageattach "pfSense" --storagectl "IDE" --port 1 --device 0 --type dvddrive --medium "$HOME/Downloads/pfSense.iso"
```

Ubuntu was connected to the same internal network:

```bash
VBoxManage modifyvm "Ubuntu" \
    --nic1 intnet \
    --intnet1 "int"
```

Kali was configured with a bridged adapter:

```bash
VBoxManage modifyvm "Kali" \
    --nic1 bridged \
    --bridgeadapter1 "YOUR_HOST_NIC"
```

I also used commands such as:

```bash
VBoxManage list vms
```

and:

```bash
VBoxManage showvminfo "pfSense"
```

to verify that the VMs and their network configuration were set up correctly.

### Why I did this

This additional challenge was intentional.

Instead of only learning how to configure a virtual network through a graphical interface, I wanted to understand how the VMs and their networking were actually configured and become more comfortable managing virtualization from the Linux command line.

## Summary of Networking

| VM | Adapter 1 | Adapter 2 | Internal Network Name |
|---|---|---|---|
| **pfSense** | `bridged` (WAN) | `intnet` (LAN) | `int` |
| **Ubuntu** | `intnet` | — | `int` |
| **Kali** | `bridged` | — | — |

## Problems Encountered

### VM Creation

One of the first errors I encountered was forgetting to use the `--register` flag when creating a virtual machine.

Without this flag, the VM was created but was not registered with VirtualBox, so it did not appear in the list of available VMs.

I fixed this by using:

```bash
VBoxManage createvm --name "VM_NAME" --ostype "OS_TYPE" --register
```

### pfSense ISO

I also encountered an issue when trying to decompress the pfSense ISO by GUI so I decompressed it from the terminal using:

```bash
gunzip pfSense-CE-2.7.2-RELEASE-amd64.iso.gz
```

---

# 4. pfSense Configuration

After creating the virtual machines, I installed pfSense using the default installation options. Once the installation was complete, I detached the ISO image from the virtual machine so that pfSense would boot from the virtual disk.

I assigned the network interfaces as follows:

```text
vtnet0 → WAN
vtnet1 → LAN
```

The WAN interface obtained an IP address from my home router using DHCP. I have omitted the actual WAN address from my host from this documentation to avoid exposing details about my local network.

For the purpose of this lab, I configured access to the pfSense web interface through the WAN side.

The pfSense WAN IP was:

192.168.1.139

I then accessed the web interface from:

`https://192.168.1.139`

I configured the necessary WAN firewall rule to allow access from my home network.

> Security note: Allowing management access to the firewall from the WAN interface is not recommended for a normal production deployment. This was done here specifically as part of the isolated home-lab exercise.

The initial LAN configuration used:

```text
192.168.1.1/24
```

### Troubleshooting the Network Connection

One of the first problems I encountered was being unable to connect to the pfSense web interface from my host machine.

My first assumption was that the problem was related to using a Wi-Fi connection with a bridged VirtualBox adapter. I thought that bridged networking over Wi-Fi might be blocking some of the traffic required for the lab.

Since my PC does not have an Ethernet port, I bought a USB Ethernet adapter to test this. This delayed the lab by a day, but after testing with the Ethernet connection I found that Wi-Fi was not actually the problem.

I then looked more closely at the IP configuration and realised that my home network was also using `192.168.1.1` as its gateway. The pfSense LAN interface was initially configured with the same address:

```text
Home network gateway: 192.168.1.1
pfSense LAN:          192.168.1.1
```

This caused a network addressing conflict.

I changed the pfSense LAN address to a different private subnet:

```text
pfSense LAN: 192.168.50.1/24
```

After making this change, the connectivity problem was resolved.

This was one of the most useful troubleshooting steps in the lab because it showed me the importance of checking the existing network configuration before assigning addresses to a new network.

### Initial Firewall Access

During the initial setup, I also needed to allow my host machine to communicate with the pfSense web interface without repeatedly disabling the firewall.

Instead of continuing to disable and re-enable the firewall from the terminal, I created a specific firewall rule allowing the required traffic from my host machine.

This allowed me to access the pfSense web interface while keeping the firewall enabled.

### LAN DHCP

After resolving the network conflict, I configured DHCP on the LAN interface.

```text
Network: 192.168.50.0/24
DHCP range: 192.168.50.10 - 192.168.50.100
```

The remaining pfSense configuration was completed after setting up the Kali Linux and Ubuntu virtual machines, since some of the firewall and connectivity settings depended on the final network configuration of those machines.

### Evidence

Disabling the firewall to get access:

<img width="674" height="405" alt="image" src="https://github.com/user-attachments/assets/cb7e18a0-be8e-454f-ab91-015ec27324e0" />


Pinging from host machine:

<img width="528" height="207" alt="image" src="https://github.com/user-attachments/assets/0e1ade31-c178-4b54-9a1c-4dfa328d32a2" />


Setting Firewall rule:

<img width="1275" height="814" alt="image" src="https://github.com/user-attachments/assets/a76c2dc4-159c-4371-9576-87e15e700da2" />

---

# 5. Ubuntu — Protected Host

Ubuntu was connected only to the internal VirtualBox network.

Its network configuration was assigned through the DHCP server configured on pfSense, I verified the configuration with:

```bash
ip addr
```

### DHCP Configuration

I initially configured and tested the DHCP settings from the pfSense terminal. Then, I reset the configuration and I configured DHCP again through the pfSense web interface.

The final DHCP configuration was:

```text
Network: 192.168.50.0/24
Gateway: 192.168.50.1
DHCP range: 192.168.50.10 - 192.168.50.100
```

Ubuntu was then able to obtain an IP address automatically from the pfSense DHCP server.

I verified the assigned address from Ubuntu with:

```bash
ip addr
```

I also tested external connectivity from Ubuntu and DNS resolution:

```bash
ping 8.8.8.8
ping google.com
```

Finally, I verified connectivity between pfsense and Ubuntu:

```bash
ping 192.168.50.1
```

This confirmed that Ubuntu was connected to the internal network.

### Evidence

Configuration on terminal:

<img width="633" height="396" alt="image" src="https://github.com/user-attachments/assets/8a0e1343-5a8c-4d76-93e1-ce2bb00c8d0c" />

Configuration on browser:

<img width="1299" height="834" alt="image" src="https://github.com/user-attachments/assets/32b8777d-22ff-4c71-9b89-d86716aa88dc" />

DHCP test:

<img width="816" height="579" alt="image" src="https://github.com/user-attachments/assets/577baeb3-9d96-489f-8d18-e876a88cef1c" />

---

# 6. Kali Linux — Attacker

Kali Linux was connected to the same bridged network as the pfSense WAN interface.

I checked the network configuration using:

```bash
ip addr
```

and, when needed:

```bash
ifconfig
```
I added the necessary pfSense firewall rule to allow ICMP traffic to the pfSense WAN interface for testing. I then added a static route on Kali for the internal 192.168.50.0/24 network through the pfSense WAN address:

```bash
sudo ip route add 192.168.50.0/24 via 192.168.1.139
```

I then verified connectivity to the Ubuntu host and the pfsense:

```bash
ping 192.168.1.139
ping 192.168.50.10
```

This confirmed that Kali could reach the internal network through pfSense before moving on to the traffic-generation and firewall testing.

### Evidence

<img width="1308" height="544" alt="image" src="https://github.com/user-attachments/assets/a443100e-6bb8-4864-a33b-b31fda272906" />

Kali rule:

<img width="1300" height="750" alt="image" src="https://github.com/user-attachments/assets/bf06f589-b2c5-470d-a7d5-b48cd4004691" />


Kali-pfsense ping test:

<img width="811" height="313" alt="image" src="https://github.com/user-attachments/assets/af9ca6c0-0dd2-4e19-957b-e50d4f83410f" />


Ubuntu ping test: 

<img width="583" height="147" alt="image" src="https://github.com/user-attachments/assets/4e6fce4b-fbb0-48f0-b6f0-4a4eae66f433" />



---

# 7. Traffic Generation with hping3

The next stage was to generate controlled traffic from Kali toward the Ubuntu host.

I used `hping3` to generate an ICMP flood:

```bash
sudo hping3 -1 --flood 192.168.50.10
```

I also experimented with SYN traffic:

```bash
sudo hping3 --flood -S -p 80 192.168.50.10
```

> **Important:** These commands were executed only against my own virtual machines inside the lab environment.

---

# 8. Packet Analysis with Wireshark

To observe the generated traffic, I installed Wireshark on the Ubuntu machine.

During the installation, I encountered a problem with missing libraries/dependencies. I had to install the required libraries before I could complete the Wireshark installation and run it successfully.

Once Wireshark was installed, I started a packet capture on Ubuntu while generating traffic from Kali.

The packet capture showed a significant increase in incoming traffic during the flood.

This provided a visual representation of the traffic generated by `hping3` and allowed me to observe the packets reaching the Ubuntu host.

### Evidence

Screenshot:

Attack generated from Kali:

<img width="722" height="224" alt="image" src="https://github.com/user-attachments/assets/b5978a25-5519-4f66-a4ff-67c3c258e3dd" />


Screenshot showing the traffic observed during the test:

<img width="1063" height="632" alt="image" src="https://github.com/user-attachments/assets/5b892528-5b1e-40c1-826a-8278eac114a2" />

> The original packet capture file is censored because the capture also contained unrelated network traffic from the host environment.

---

# 9. Blocking the Traffic with pfSense

After confirming that the traffic was reaching Ubuntu, I created a firewall rule in pfSense to block traffic from the Kali IP address to the Ubuntu host.

The rule was configured approximately as:

```text
Action: Block
Protocol: Any
Source: [KALI_IP]
Destination: [UBUNTU_IP]
Logging: Enabled
Description: Block Kali ICMP flood
```

The blocking rule was placed above the rule that previously allowed the traffic.

I enabled logging on the rule so that I could verify that pfSense was detecting and blocking the traffic.

### Evidence

<img width="761" height="342" alt="image" src="https://github.com/user-attachments/assets/4911cf4a-991a-4cd9-9e6d-676ef996b620" />

## 10. What I Learned

This lab helped me understand several concepts that are difficult to fully appreciate through theory alone.

### Virtual Networking

It helped me review how different VirtualBox networking modes affect connectivity (had experience through college):

- Bridged networking
- Internal networking
- WAN/LAN separation
- Virtual network interfaces

The most important concept was understanding that the VirtualBox internal network is isolated from the physical network and that pfSense can be used to connect the two segments.

### Routing

I learned how to set up the connectivity between two different subnets.

In this lab:

```text
192.168.1.0/24
        |
    pfSense
        |
192.168.50.0/24
```
Kali therefore needed a route toward the internal subnet through pfSense.

The route was configured so that traffic destined for the `192.168.50.0/24` internal network was sent through the pfSense WAN interface.

### DHCP

I configured pfSense as the DHCP server for the internal LAN.

This allowed Ubuntu to automatically receive:

- An IP address
- A gateway
- DNS configuration

### Firewall Configuration

I learned how firewall rules determine whether traffic is allowed to pass between interfaces and hosts.

I also learned that rule order matters, because a more specific blocking rule needs to be evaluated before a broader allow rule if both could match the same traffic.

### Packet Analysis

Using Wireshark allowed me to see the difference between normal traffic and the traffic generated during the flood.

This helped me visually connect the firewall configuration with actual network packets.

### Security Monitoring

Enabling logging on the blocking rule also demonstrated how firewall events can provide useful security telemetry.

This is relevant to later security operations work because network security tools generate logs that can be used to identify suspicious activity and investigate incidents.

### Linux Command Line

Creating the entire VirtualBox environment from the terminal gave me additional practice with `VBoxManage` and general Linux troubleshooting.

---

## 11. Troubleshooting Approach

One of the most useful parts of this project was learning to troubleshoot the network systematically.

When something did not work, I tried to determine where the communication stopped rather than immediately changing multiple settings.

For example:

```text
Kali
 ↓
Kali routing
 ↓
pfSense WAN
 ↓
pfSense firewall
 ↓
pfSense LAN
 ↓
Ubuntu
```

This helped me narrow down the actual cause of the issue.

---

## 12. Security Considerations

This lab was intentionally designed as a controlled environment.

The attack traffic was generated only between virtual machines that I controlled.

The WAN-side pfSense management access was also configured specifically for the exercise and should not be treated as a recommended production configuration.

In a real environment, firewall management interfaces should be restricted to trusted management networks or other secure administration mechanisms.

---

## 13. Improvements / Next Steps

Possible extensions for this lab include:

- Send pfSense logs to a SIEM such as Wazuh
- Create more granular firewall rules
- Analyze additional attack traffic
- Experiment with VLAN segmentation
- Automate the entire VirtualBox deployment with a Bash script
- Add monitoring and alerting
- Document firewall events as incident-response evidence

---

## 14. Reference

This lab was inspired by the following pfSense home-lab walkthrough:

**Royden Rebello (The Social Dork) — Home-Lab Walk-Through: pfSense ↔ Kali ↔ Ubuntu (ICMP flood Defense Demo)**

Original walkthrough:

https://youtu.be/-yRvfbElT7M

The environment in this repository was rebuilt independently, with additional experimentation and a command-line VirtualBox configuration challenge using `VBoxManage`.

---

## 15. Final Result

The completed lab demonstrates the following workflow:

    EXTERNAL NETWORK
          |
        Kali
       Attacker
          |
          | Attack traffic
          v
    +-------------+
    |   pfSense   |
    |   Firewall  |
    +-------------+
          |
          | LAN
          |
    +-------------+
    |   Ubuntu    |
    |    Protected Host   |
    +-------------+

    1. Generate traffic
    2. Observe with Wireshark
    3. Create firewall rule
    4. Block traffic
    5. Verify in Wireshark
    6. Verify in pfSense logs

This project gave me practical experience with network segmentation, routing, DHCP, firewall configuration, packet analysis, Linux administration, VirtualBox networking, and basic attack detection and mitigation.
