<h1>DoubleLion Home SOC Lab</h1>

<h2>Project Goal</h2>

I am build a fully functional Security Operations Centre environment on as a home lab for continous practice, something that actually simulates how enterprise SOC teams monitor real networks. The end goal is to have a lab where i can generate real network traffic, detect it using a proper SIEM stack, capture packets on a dedicated sensor-node or machine, and practice the analyst workflow from end to end.


<h2>Environment & Resources</h2>

Everything runs on my physical machine - It's a Parrot OS host using KVM/QEMU managed through virt-manager with no cloud involvement Just local virtualisation.

<h2>Virtual machines I built:</h2>

| Machine | OS | Role |
|---|---|---|
| doublelion-vyos-router | VyOS | Core router handling NAT, VLAN segmentation, DHCP |
| doublelion-ubuntu-server | Ubuntu Server | Wazuh SIEM stack — manager, dashboard, indexer, filebeat |
| sensor-node | Kali Linux | Dedicated network sensor running in promiscuous mode |
| doublelion-win-server | Windows Server | Asset — target machine for detection exercises |
| win11 | Windows 11 | Asset — workstation target |
| doublelion-firewall | pfSense | Firewall — integration in progress |
| doublelion-sec-onion | Security Onion | Attempted — shelved due to host resource limits |


<img width="849" height="631" alt="2026-09-26_16-09" src="https://github.com/user-attachments/assets/12092971-c03a-4c00-b8af-183244456109" />



<h2>Network Design</h2>

I split the lab into two VLANs to simulate a real network where your SOC machines and your monitored assets are on separate segments.


<h3>VLAN 10 - ASSET network (192.168.10.0/24)</h3>
This is where the target machines live - Windows Server, Windows 11 workstation, and the Wazuh server itself at 192.168.10.60.

<h3>VLAN 20 - SOC network (192.168.20.0/24)</h3>
This is where the sensor node (Kali) lives, receiving mirrored traffic from the asset network.

The VyOS router sits at the centre, trunking both VLANs and handling NAT/DNAT so everything can reach the internet through the host bridge.

<h3>VyOS interface config:</h3>

Interface     IP Address            Description
eth1          —                     LAN trunk (VLAN parent)
eth1.10       192.168.10.1/24       ASSET VLAN gateway
eth1.20       192.168.20.1/24       SOC VLAN gateway
eth0          192.168.122.246/24    WAN — DHCP from host bridge



<img width="843" height="571" alt="2026-09-26_16-23" src="https://github.com/user-attachments/assets/c1cb97e8-f3df-42ee-a7d5-f8f4fffd01c9" />



<h3>VyOS NAT — forwarding Wazuh ports to the host:</h3>

Port 443   → 192.168.10.60 (Wazuh Dashboard HTTPS)
Port 8443  → 192.168.10.60 (Wazuh API)
Port 8000  → 192.168.10.60 (web interface)
Port 22    → 192.168.10.60 (SSH)


<img width="842" height="572" alt="image" src="https://github.com/user-attachments/assets/55c3ef53-fb19-4153-8891-584ea5387a65" />


<h3>Routing verified on both machines:<h/3>

  
<img width="1344" height="589" alt="2026-09-26_16-29" src="https://github.com/user-attachments/assets/5a2a32ab-90dc-444f-9a38-1395f30e92e0" />



<h2>Traffic Mirroring - How the Sensor Node Sees Everything</h2>

This was the most technically interesting part of the setup. I needed the Kali sensor node to passively see all traffic on the ASSET VLAN without being a man-in-the-middle and disrupting anything.

The solution was Open vSwitch port mirroring on the host. I wrote a script that automatically detects the sniffing port and creates the mirror:

```bash
sudo /usr/local/bin/setup-asset-mirror.sh

Found sniffing port: vnet5
Mirror created successfully

_uuid        : 88c327f8-ff94-4582-bdc6-0b231be8e01a
name         : asset-mirror
output_port  : 072af56c-69e7-4438-a81e-c73fe8ba2713
select_all   : true
```

With `select_all: true`, every packet crossing the ASSET VLAN bridge is cloned and sent to the sensor node's eth1 interface. This is how enterprise TAP and SPAN configurations work in production SOC environments, other alternatives exist which will be covered next time


<img width="727" height="485" alt="2026-10-01_09-28" src="https://github.com/user-attachments/assets/a84f0b8b-7c2b-4cf8-81d5-7bb5c0a00dc5" />


<h3>The Script</h3>


<img width="668" height="438" alt="image" src="https://github.com/user-attachments/assets/04792834-8e75-4d56-9f36-aa02ce94d1a2" />


On the sensor node, I then set eth1 to promiscuous mode and ran tcpdump to confirm traffic was arriving:

<img width="500" height="458" alt="image" src="https://github.com/user-attachments/assets/4b16df73-9058-442e-b4ff-669be2f66f09" />


image showing promiscous mode up on eth1


```bash
sudo ip addr flush dev eth1
sudo ip link set eth1 promisc on
sudo ip link set eth1 up
sudo tcpdump -i eth1 -c 30 -nn tcp
```


<img width="1359" height="667" alt="2026-09-26_16-57" src="https://github.com/user-attachments/assets/8a80b221-56a7-4a92-903d-fd04a53c053f" />


as you can see above i had created a user on the ubuntu server named mrjosh so i ssh into that user and all traffic were mirrored to my SOC sensor machine



<h2>Wazuh SIEM Stack</h2>

The Wazuh stack runs entirely on the Ubuntu server at 192.168.10.60. an alternative will be to deploy the stack on the sock machine or server and have an agent sit on the ubuntu server or any asset device you iintend to monitor.


```bash
# Manager
sudo systemctl status wazuh-manager | grep running
Active: active (running) since Sat 2026-09-26 15:31:48 UTC

# Dashboard
sudo systemctl status wazuh-dashboard | grep running
Active: active (running) since Sat 2026-09-26 15:10:20 UTC

# Indexer
sudo systemctl status wazuh-indexer | grep running
Active: active (running) since Sat 2026-09-26 15:14:02 UTC

# Filebeat
sudo systemctl status filebeat | grep running
Active: active (running) since Sat 2026-09-26 15:12:19 UTC
```

<img width="596" height="375" alt="image" src="https://github.com/user-attachments/assets/5178d36a-6a9e-4bd4-805b-9a7fdd65bf99" />


<h3>SSH Access Verified - Lateral Movement Test</h3>

To verify the Wazuh server was accessible from outside the lab and to simulate what an attacker with access to the WAN-side would see, I SSH'd in through the VyOS NAT rule on port 22. Successfully landed as user `mrjosh`:

```bash
mrjosh@doublelion:~$ pwd
/home/mrjosh

mrjosh@doublelion:~$ mkdir here
mrjosh@doublelion:~$ exit
Logout
Connection to 192.168.122.246 closed.
```

The traffic from this SSH session was visible in the tcpdump output on the sensor node - confirming the mirror was working correctly and traffic from the host to the ASSET VLAN was being captured.


<img width="1359" height="667" alt="2026-09-26_16-57" src="https://github.com/user-attachments/assets/8a80b221-56a7-4a92-903d-fd04a53c053f" />



<h2>Challenges I Encountered</h2>

<b>Security Onion could not run on my hardware due to limited resource.</b> I attempted to deploy Security Onion as an alternative SIEM but my host machine did not have sufficient RAM to run it alongside the other VMs. I had documented this honestly and pivoted to Wazuh which has a lighter footprint while covering all the core SIEM functions i needed.

<b>OVS port mirroring required custom scripting.</b> Getting the traffic mirror to work reliably was not straightforward. libvirt's default networking does not expose easy port mirror controls, so I wrote the setup-asset-mirror.sh script to automate the OVS bridge mirror creation on every boot.

OVS gave me a lot of trouble since it wasn't the usual cisco or huawie switch i was used to especially commands i had to read alot about their documentation. you can check it our if you have limited resource to deploy move machines

link: https://docs.openvswitch.org/en/latest/tutorials/

<br>
<h2>What This Lab Proves</h2>

Every piece of this lab maps directly to real SOC and network engineering work:

- Deploying and operating a SIEM (Wazuh) - what most SOC analyst do on day one for those involved in setting up the environment, others just use what has been deployed.
- Configuring VLANs, NAT/DNAT and inter-VLAN routing - core CCNA knowledge applied in practice
- Setting up passive traffic capture via port mirroring - how enterprise TAP/SPAN works
- Writing automation scripts for repeatable lab configuration
- Troubleshooting multi-VM networking across bridged and OVS interfaces

<br>

<h1>What Is Next</h1>

- Bring pfSense online and integrate it between the WAN and the router for stateful firewall inspection
- Deploy Suricata on the sensor node for signature-based IDS alerts feeding into Wazuh
- Deploy Zeek alongside Suricata for protocol-level connection analysis and DNS logging
- Enrol Windows 11 and Windows Server as Wazuh agents so host-based logs appear in the dashboard
- Run structured attack simulations — port scans, brute force, lateral movement — and document the detections
- Produce full incident response reports from Wazuh alert data
- More and More Practice

