# Network-Penetration-Testing
My hands-on notes from Section 3: recon, Nmap, and enumeration against a live lab target

<img width="2760" height="1464" alt="image" src="https://github.com/user-attachments/assets/8adbbe85-7c70-4593-9f66-3b47c3252d9f" />

---

# Section 3 – Security

### Course 1: Network Penetration Testing — eCPPTv3

This write-up covers section 3 of the eCPPTv3 training path, focused on network penetration testing from an active reconnaissance and enumeration perspective.

The goal of this section is to build a solid understanding of how hosts communicate at the protocol level, how tools like Nmap map and fingerprint a network in practice, and how penetration testers enumerate Windows services like SMB, NetBIOS, and SNMP to surface accounts, shares, and misconfigurations worth pursuing.

Alongside the theory, this write-up includes hands-on Nmap scans run against a live lab target to demonstrate each concept in practice.

#### Table of Contents

- [Active Information Gathering](#active-information-gathering--network-penetration-testing)
    - [Penetration Testing Methodology](#penetration-testing-methodology)
    - [Active Information Gathering](#active-information-gathering)
- [Networking Fundamentals](#networking-fundamentals--network-penetration-testing)
    - [Network Protocols & Packets](#network-protocols)
    - [The OSI Model](#the-osi-model)
    - [Network Layer (Layer 3)](#network-layer-layer-3)
    - [Transport Layer (Layer 4)](#transport-layer-layer-4)
    - [Network Mapping](#network-mapping)
    - [Nmap](#nmap)
    - [Ping Sweeps](#ping-sweeps)
- [Introduction to Enumeration](#introduction-to-enumeration--network-penetration-testing)
    - [SMB and NetBIOS Enumeration](#smb-and-netbios-enumeration)
    - [SNMP Enumeration](#snmp-enumeration)
    - [SMB Relay Attack](#smb-relay-attack)
    - [Dumping and Cracking NTLM Hashes](#dumping-and-cracking-ntlm-hashes)

---

### Active Information Gathering — Network Penetration Testing

#### Penetration Testing Methodology

Before getting into active techniques, it's worth laying out where they sit inside the overall penetration testing methodology. A standard engagement moves through five phases.

Everything starts with information gathering, which splits into two tracks. Passive information gathering relies on OSINT (domain and IP analysis, social media, Google dorks, search engines, DNS recon) without ever touching the target directly. Active information gathering, on the other hand, involves network mapping, host discovery, port scanning, and service/OS detection, all of which require direct interaction with the target.

Once hosts and services have been identified, the tester moves into enumeration. This phase digs deeper into what's actually running: service enumeration, user enumeration, and share enumeration all fall under this umbrella.

With enough data collected, exploitation (or initial access) becomes possible. This is where vulnerability analysis, threat modeling, and exploit development or modification come into play, using everything gathered so far to identify and leverage weaknesses.

Getting a foothold is only the beginning, though. Post-exploitation covers what happens after initial access: local enumeration, privilege escalation, credential access, persistence, defense evasion, and lateral movement across the network.

Finally, everything gets pulled together into a report. Findings need to be structured clearly, with actionable remediation steps, or the technical work loses most of its value to the client.

#### Active Information Gathering

So what does "active" actually mean here? It means the tester is no longer just observing. They're directly interacting with the target system or network to collect data and surface potential vulnerabilities. Scanning, probing, direct connections: all of it leaves a trace, which is the key difference from passive recon. A well-configured IDS can, in theory, catch this activity happening.

This phase breaks down into two main branches. Scanning and network mapping cover host discovery, port scanning, and service/OS fingerprinting, basically figuring out what's alive and what's open. Enumeration then goes further, pulling usernames, shares, and configuration details out of whatever services were discovered.

#### Wrapping Up

This article laid out the foundational structure of a penetration test and where active information gathering fits within it: the five phases from information gathering to reporting, the split between passive and active recon, and the two branches that make up active information gathering.

Active information gathering sets the direction for everything that follows it. The quality of the recon phase tends to directly determine how far the rest of the engagement can go.

---

### Networking Fundamentals — Network Penetration Testing

#### Network Protocols

Hosts on a network talk to each other through network protocols. These protocols exist so that different systems, regardless of hardware or software, can exchange information without needing to know anything about each other's internals. That communication happens through packets.

#### Packets

A packet is the basic unit of data transmission on a network. At the physical layer, packets are really just streams of bits traveling as electrical signals over Ethernet cables, Wi-Fi, or whatever medium is in use, later interpreted as the 1s and 0s that make up the actual data.

Every packet follows the same general structure regardless of protocol. The header carries protocol-specific metadata, such as source and destination addresses, TTL, and protocol type, telling the receiving host how to interpret and handle the rest of the communication. The payload is the actual content being sent: part of an email, a chunk of a downloaded file, whatever the packet was created to carry.

#### The OSI Model

The OSI (Open Systems Interconnection) model is a conceptual framework from ISO that breaks network communication down into seven abstraction layers. It exists to give engineers a shared language for reasoning about how communication actually works, and to make interoperability between different systems and technologies possible in the first place.

Worth keeping in mind: the OSI model isn't a strict implementation blueprint. Real protocols don't always map onto it cleanly, but it's still the reference point most people use when designing or explaining network architecture.

|Layer|Name|Function|
|---|---|---|
|7|Application|Provides network services directly to end-users or applications|
|6|Presentation|Data format translation, encryption, and compression|
|5|Session|Manages sessions between applications, handles synchronization and dialog control|
|4|Transport|End-to-end communication, flow control, and segmentation|
|3|Network|Logical addressing and routing|
|2|Data Link|Physical addressing, framing, and error detection|
|1|Physical|Physical connection between devices (Ethernet cables, Wi-Fi, coax)|

#### Network Layer (Layer 3)

The Network Layer handles logical addressing, routing, and forwarding packets across different networks. Its main job is finding the optimal path from source to destination, even when the two endpoints sit on completely separate networks. It abstracts away the underlying physical infrastructure, which is what lets different network types get stitched together into one connected internetwork.

Two protocols dominate this layer. IP (Internet Protocol) handles logical addressing, routing, and the fragmentation and reassembly of packets. It's the foundation everything else on the internet is built on top of. ICMP works alongside it for error reporting and diagnostics; it's the protocol behind tools like `ping` and `traceroute`, and it plays a direct role in host discovery during a pentest.

##### IP Versions

IPv4 uses 32-bit addresses written as four dot-separated octets, like `192.168.0.1`. Each octet is a single byte, giving IPv4 a theoretical address space of roughly 4.3 billion addresses, a number that sounded huge decades ago and turned out to be nowhere near enough. Despite that exhaustion problem, it's still by far the most widely deployed version today.

IPv6 was built to fix exactly that limitation. Its addresses are 128 bits long, written in hexadecimal, like `2001:0db8:85a3:0000:0000:8a2e:0370:7334`, which pushes the available address space into territory that's effectively inexhaustible for the foreseeable future.

##### What IP Actually Handles

Beyond just addressing, IP is responsible for a few other things worth knowing well.

Logical addressing assigns hierarchical addresses to interfaces using network classes, subnets, and CIDR notation, giving every device on a network a unique identifier. Packet fragmentation and reassembly comes into play when a packet needs to cross networks with different MTU sizes. It gets broken into smaller fragments along the way and reassembled back into the original packet once it reaches its destination.

IP also defines how packets actually get delivered. Unicast sends from one host straight to one specific destination. Broadcast sends to every host on a subnet at once. Multicast sits in between, sending to a defined group of devices that opted in to receive it.

Subnetting is the practice of splitting a large network into smaller, more manageable sub-networks, which helps with both efficiency and security.

##### IPv4 Header Fields

Every field in the IPv4 header exists to help route and deliver the packet correctly. For a pentester, this matters beyond theory. Tools like Nmap manipulate several of these fields directly for scanning and evasion purposes, so understanding what each one does isn't optional if you want to know what your scanner is actually doing under the hood.

|Field|Size|Description|
|---|---|---|
|Version|4 bits|IP version, value is 4 for IPv4|
|Header Length|4 bits|Header size in 32-bit words. Minimum is 5 (20 bytes), maximum is 15 (60 bytes)|
|Type of Service|8 bits|Packet priority and congestion control via DSCP and ECN|
|Total Length|16 bits|Total size of the packet including header and payload, max 65,535 bytes|
|Identification|16 bits|Shared by all fragments of the same packet, used during reassembly|
|Flags|3 bits|Reserved (always 0), Don't Fragment (DF), More Fragments (MF)|
|Fragment Offset|13 bits|Indicates where a given fragment belongs within the original unfragmented packet, measured in units of 8 bytes|
|TTL|8 bits|Maximum hops before the packet is discarded, decremented by 1 at each router|
|Protocol|8 bits|Identifies the upper-layer protocol: 6 (TCP), 17 (UDP), 1 (ICMP)|
|Header Checksum|16 bits|Verifies the integrity of the header only (not the payload); recalculated at every hop since TTL changes|
|Source IP|32 bits|IPv4 address of the sender|
|Destination IP|32 bits|IPv4 address of the intended recipient|

The Fragment Offset field is what makes fragmentation actually work in practice. Without it, a receiving host would have no way to reassemble scattered fragments back into the correct order. The Header Checksum only protects the header itself, not the payload, and it gets recalculated by every router along the path since fields like TTL change hop by hop.

##### Reserved IPv4 Addresses

A handful of IPv4 ranges are set aside and can't be used for general host communication:

- `0.0.0.0 – 0.255.255.255`: represents "this" network.
- `10.0.0.0 – 10.255.255.255`: a full Class A block reserved for private networks (RFC1918). This is the range you'll see most often in large corporate networks and cloud environments.
- `127.0.0.0 – 127.255.255.255`: the loopback range, representing the local host.
- `172.16.0.0 – 172.31.255.255`: a Class B block reserved for private networks (RFC1918), common on medium-sized internal networks.
- `192.168.0.0 – 192.168.255.255`: a Class C block reserved for private networks (RFC1918), the one most people recognize from home routers.

These three ranges (`10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16`) are the full RFC1918 private address space. Full details are documented in RFC5735.

#### Transport Layer (Layer 4)

Sitting above the Network Layer, the Transport Layer handles end-to-end communication between two devices. Where the Network Layer just gets packets to the right machine, the Transport Layer makes sure the data lands on the right application on that machine, correctly and in order. Error detection, flow control, and segmentation into transmittable units all happen here.

Two protocols dominate this layer, and they take almost opposite approaches to the same problem.

##### TCP (Transmission Control Protocol)

TCP is connection-oriented and reliable. A connection has to be established before any data moves, and once it is, TCP guarantees the data arrives accurately and in order, retransmitting anything lost or corrupted along the way. That reliability is exactly why it's the protocol behind web browsing, email, and file transfers, where losing or scrambling data isn't acceptable.

**The 3-Way Handshake**

Every TCP connection starts with the same three-step exchange. The client sends a segment with the SYN flag set, along with a randomly generated Initial Sequence Number, signaling intent to connect. The server responds with SYN and ACK both set, acknowledging the client's ISN while providing its own. The client then sends back an ACK, acknowledging the server's ISN. At that point the connection is live and data can start flowing.

This handshake matters a lot in pentesting, because Nmap's SYN scan (`-sS`) deliberately skips the third step. Instead of completing the handshake, it sends a RST. That makes the scan faster and stealthier, since no full connection ever gets logged on the target side.

**TCP Control Flags**

|State|SYN|ACK|FIN|
|---|---|---|---|
|Initiating a connection|Set|Clear|Clear|
|Responding to a connection|Set|Set|Clear|
|Terminating a connection|Clear|Set|Set|

**TCP Port Ranges**

Ports are 16-bit unsigned integers, from 0 to 65,535, split into three categories.

Well-known ports (0–1023) are standardized by IANA for common services:

|Port|Service|
|---|---|
|21|FTP|
|22|SSH|
|25|SMTP|
|80|HTTP|
|110|POP3|
|443|HTTPS|

Registered ports (1024–49151) get assigned to specific applications by IANA:

|Port|Service|
|---|---|
|3306|MySQL|
|3389|RDP|
|8080|HTTP Alternate|
|27017|MongoDB|

Dynamic/private ports (49152–65535) are used for ephemeral connections that clients initiate on the fly.

##### UDP (User Datagram Protocol)

UDP takes the opposite approach: connectionless and lightweight. Packets go out without any connection setup, and there's no guarantee any of them arrive. Each datagram stands alone: no sequencing, no acknowledgment, no retransmission if something's lost along the way.

What it gives up in reliability, it makes up for in speed. No handshake, no acknowledgment overhead, smaller headers, which is exactly why it's the protocol of choice for anything real-time, where low latency matters more than perfect delivery: VoIP, video streaming, DNS, online gaming.

##### TCP vs UDP

|Feature|TCP|UDP|
|---|---|---|
|Connection|3-way handshake required|Connectionless|
|Reliability|Guaranteed delivery and ordering, retransmission supported|No guarantees, no retransmission|
|Header Size|Larger due to control fields|Smaller, lower overhead|
|Speed|Slower due to connection management|Faster|
|Use Cases|HTTP, FTP, SMTP, HTTPS, Telnet|DNS, DHCP, SNMP, VoIP, online gaming|

#### Network Mapping

Once passive recon wraps up, the next move is into active territory, and the first real task there is network mapping: finding hosts, open ports, services, and infrastructure across the target environment.

Take a realistic scenario: a company scopes an engagement to the block `200.200.0.0/16`. That /16 netmask covers up to 65,536 possible hosts, from `200.200.0.0` to `200.200.255.255`. Before anything else can happen, the tester needs to figure out which of those addresses actually belong to live systems. Network mapping answers that, and builds the foundational picture everything else gets built on.

A thorough network mapping phase covers several things at once: discovering which hosts are actually live and responding, identifying open ports and services (which directly defines the attack surface), building out the network topology (routers, switches, firewalls, and how it all connects), fingerprinting operating systems to enable OS-specific attack strategies, detecting exact service versions (since a specific version often maps straight to a known CVE), and spotting whatever filtering or security measures (firewalls, IDS/IPS) will need to be understood or worked around later.

#### Nmap

Nmap is the industry-standard open-source tool for network mapping, and it's usually the first thing reached for in active recon. It covers host discovery, port scanning, service version detection, OS fingerprinting, and scripted service interaction through its built-in scripting engine.

**Host Discovery**

```bash
nmap 192.168.1.1-3 -sL         # List targets only, no scan
nmap 192.168.1.0/24 -sn        # Host discovery only, disable port scan
nmap 192.168.1.1-5 -Pn         # Skip host discovery, port scan only
nmap 192.168.1.0/24 -PR        # ARP discovery on local network
nmap 192.168.1.1 -n            # Disable DNS resolution
```

<img width="1396" height="1042" alt="image" src="https://github.com/user-attachments/assets/530bf8f1-b4b8-4e8f-bfaf-aefa3f70eec0" />

1. Host Discovery

Ran nmap 192.168.1.0/24 -sn to perform a ping sweep across the entire subnet without touching any ports. The scan identified 5 live hosts out of 256 possible addresses, with Nmap resolving the vendor for each host from its MAC address. The goal here was to narrow down which hosts were actually alive before moving into any deeper scanning.


**Port Scanning**

```bash
nmap 192.168.1.1 -p 21                      # Scan a single port
nmap 192.168.1.1 -p 21-100                  # Scan a port range
nmap 192.168.1.1 -p U:53,T:21-25,80         # Scan specific TCP and UDP ports
nmap 192.168.1.1 -p-                        # Scan all 65535 ports
nmap 192.168.1.1 -F                         # Fast scan, top 100 ports
nmap 192.168.1.1 --top-ports 2000           # Scan top 2000 ports
```

<img width="1396" height="1714" alt="image" src="https://github.com/user-attachments/assets/98c68803-1411-49fa-9186-2d0ef9e395e9" />


2. Port Scan

With the target identified (192.168.1.3), ran nmap --top-ports 25 to check the most common 25 ports. Several ports came back open, including FTP, SSH, Telnet, SMB (445), and MySQL, giving an early read on the attack surface before drilling down into service versions.


**Scan Types**

```bash
nmap 192.168.1.1 -sS    # TCP SYN scan (default, requires root)
nmap 192.168.1.1 -sT    # TCP connect scan (no root required)
nmap 192.168.1.1 -sU    # UDP scan
nmap 192.168.1.1 -sA    # TCP ACK scan
```

The SYN scan is the default and the most widely used for good reason: it sends a SYN and waits for a response without ever completing the handshake, which makes it both faster and stealthier since the target never logs a full connection.

**Service Version and OS Detection**

```bash
nmap 192.168.1.1 -sV                 # Detect service versions
nmap 192.168.1.1 -sV --version-all   # Maximum detection intensity, slower but more accurate
nmap 192.168.1.1 -O                  # OS detection via TCP/IP stack fingerprinting
nmap 192.168.1.1 -O --osscan-guess   # More aggressive OS guessing
nmap 192.168.1.1 -A                  # OS detection + versions + scripts + traceroute
```

<img width="2048" height="1714" alt="image" src="https://github.com/user-attachments/assets/bdd8e549-d5d1-4c20-ad2c-b368382982cb" />

3. Service/Version Detection

Ran nmap 192.168.1.3 -sV to pin down the exact version running on each open service. The results surfaced outdated versions with well-documented vulnerabilities, most notably vsftpd 2.3.4 (which has a known, CVE-documented backdoor) and UnrealIRCd (which also has a known backdoored release). This is a concrete example of how a version number alone can point straight to a viable exploitation path.


**Output Formats**

```bash
nmap 192.168.1.1 -oN scan.txt                 # Normal text output
nmap 192.168.1.1 -oX scan.xml                 # XML output
nmap 192.168.1.1 -oG scan.gnmap               # Grepable output
nmap 192.168.1.1 -oN scan.txt --append-output # Append to existing file
nmap --iflist                                 # Show host interfaces and routes
```

**Firewall Detection and IDS Evasion**

```bash
nmap 192.168.1.1 -f                                             # Fragment packets
nmap 192.168.1.1 --mtu 32                                       # Custom MTU offset
nmap -D 192.168.1.101,192.168.1.102,192.168.1.103 192.168.1.1   # Decoy scan
nmap -g 53 192.168.1.1                                          # Spoof source port
nmap --data-length 200 192.168.1.1                              # Append random data to packets
```

<img width="1396" height="1638" alt="image" src="https://github.com/user-attachments/assets/23cf261f-4570-45c9-a8ec-b52c8e38d735" />

5. Evasion

Ran nmap -sS -D 192.168.1.101,192.168.1.102,ME 192.168.1.3 to demonstrate the decoy scan technique. The command made the scan appear to originate from multiple sources at once, so anyone monitoring logs would see several possible sources instead of being able to immediately pin down the real one.

Packet fragmentation splits scan packets into smaller pieces that basic packet filters struggle to inspect properly. Decoy scanning makes the traffic appear to come from several addresses at once, which muddies any attempt to identify the real source.


**Nmap Scripting Engine (NSE)**

```bash
nmap 192.168.1.1 -sC                                                       # Run default scripts
nmap 192.168.1.1 --script default                                          # Same as -sC
nmap --script snmp-sysdescr --script-args snmpcommunity=admin 192.168.1.1 # Script with arguments
```

<img width="2048" height="1416" alt="image" src="https://github.com/user-attachments/assets/bc624c79-6d32-424b-a104-462655cda9a0" />

5. Nmap Scripting Engine (NSE)

After the -sV scan surfaced vsftpd 2.3.4, I ran the dedicated NSE script ftp-vsftpd-backdoor to confirm the vulnerability. The result confirmed the service is VULNERABLE, referencing the documented CVE (CVE-2011-2523) and disclosure date, and even showing the output of an id command executed on the target confirming root-level access. This is a direct example of how a version number surfaced during routine scanning can point to an actual exploitable path.


**Timing Templates**

```bash
nmap 192.168.1.1 -T0    # Paranoid, extremely slow, maximum IDS evasion
nmap 192.168.1.1 -T3    # Normal, default speed
nmap 192.168.1.1 -T5    # Insane, maximum speed, assumes a fast, reliable network
```

**Other Useful Commands**

```bash
nmap 192.168.1.1                     # Scan a single IP
nmap 192.168.1.1 192.168.2.1         # Scan specific IPs
nmap 192.168.1.1-254                 # Scan a range
nmap scanme.nmap.org                 # Scan a domain
nmap 192.168.1.0/24                  # Scan using CIDR notation
nmap -iL targets.txt                 # Scan targets from a file
nmap -iR 100                         # Scan 100 random hosts
nmap --exclude 192.168.1.1           # Exclude specific hosts
nmap -6 2607:f0d0:1002:51::4         # IPv6 scanning
nmap -h                              # Help screen
```

#### Ping Sweeps

A ping sweep is one of the simplest host discovery techniques out there, and often the first thing run against a range. It sends ICMP Echo Request packets across a set of IP addresses and just listens for what comes back.

An ICMP Echo Request is a Type 8, Code 0 packet. If the target is alive and not blocking ICMP, it answers with an Echo Reply (Type 0, Code 0), and that reply is enough to confirm the host is up.

No reply doesn't necessarily mean the host is dead, though. A firewall configured to drop ICMP will silently discard the request, making a perfectly live host look offline. That's why ping sweeps work best as a starting point rather than a final answer; pairing them with TCP SYN pings or ARP scanning covers the gap whenever ICMP is filtered.

#### Wrapping Up

This article covered the core networking concepts underpinning every technique used during active information gathering: how hosts communicate through protocols and packets, the seven-layer OSI model, the Network Layer with IP addressing and header fields, the Transport Layer with TCP vs UDP and the three-way handshake, network mapping and its objectives, Nmap across host discovery through evasion, and ping sweeps as a discovery technique.

None of the tools covered later in this series make much sense without this foundation. Knowing why a SYN scan behaves the way it does, or why ICMP sometimes just disappears, is what separates a tester who understands their tools from one who's just running commands they memorized.

---

### Introduction to Enumeration — Network Penetration Testing

#### What Is Enumeration?

Once host discovery and port scanning are done, the next phase is service enumeration. Scanning tells you what's open; enumeration tells you what's actually sitting behind it: account names, shared resources, misconfigurations, and other details that become useful once exploitation starts.

Like scanning, this phase involves active connections to remote systems. Plenty of protocols found on networked systems become targets the moment they're misconfigured or left enabled without a real reason, and finding exactly that is the point of this phase.

#### SMB and NetBIOS Enumeration

NetBIOS and SMB show up together constantly on Windows networks, but they're two distinct technologies.

NetBIOS is an API and a set of network protocols that lets applications on different machines find and talk to each other over a local network. It offers three services: name registration and resolution (NetBIOS-NS, port 137), connectionless communication and broadcasting (NetBIOS-DGM, port 138), and connection-oriented, reliable data transfer (NetBIOS-SSN, port 139).

SMB, meanwhile, is the primary file-sharing protocol on Windows. It lets users access files, printers, and other resources on remote machines as though they were local, and it also supports named pipes and inter-process communication.

SMB has evolved through several versions over the years:

|Version|Introduced With|Notes|
|---|---|---|
|SMB 1.0|Windows XP|Original version, known security vulnerabilities|
|SMB 2.0/2.1|Windows Vista / Server 2008|Improved performance and security|
|SMB 3.0+|Windows 8 / Server 2012|Added encryption, multichannel support, virtualization improvements|

It uses port 445 for direct traffic, and port 139 when running over NetBIOS.

Modern Windows networks lean on SMB for file and printer sharing, with DNS handling name resolution instead of NetBIOS. Still, NetBIOS over TCP/IP (port 139) tends to stay enabled for backward compatibility, so in practice both protocols are usually running side by side, and both are worth enumerating.

#### SNMP Enumeration

SNMP (Simple Network Management Protocol) is an application-layer protocol built to monitor and manage networked devices: routers, switches, printers, servers, and anything else that needs remote visibility. It runs over UDP and involves three components: the manager, which queries and interacts with agents across the network; the agent, software running on the managed device that responds to queries and sends traps; and the MIB, a hierarchical database defining the structure of available data, where every data point has its own Object Identifier.

SNMP has three versions, and the security gap between them is significant. SNMPv1 uses community strings for authentication with no encryption at all. SNMPv2c also relies on community strings but adds bulk transfer support. SNMPv3 finally brings real security with user-based authentication and encryption.

It uses port 161 (UDP) for queries and port 162 (UDP) for traps.

From a pentesting angle, SNMP enumeration is worth doing for several reasons. It reveals which devices have SNMP enabled and whether they're leaking information they shouldn't be. It's a quick way to test for default or weak community strings like `public` or `private`, which can grant read (or sometimes write) access to device data. Once access is confirmed, it can pull system information such as device names, OS versions, and network interfaces, along with network configurations like routing tables and IP assignments, user and group data where available, and running services that could open up further attack paths.

#### SMB Relay Attack

An SMB relay attack is a man-in-the-middle technique where the attacker intercepts SMB authentication traffic and relays it to a different server to gain access, without ever needing to crack the captured credentials.

It plays out in four steps. First, interception: the attacker positions themselves between client and server, usually through ARP spoofing, DNS poisoning, or a rogue SMB server. Second, when the client authenticates against what it believes is a legitimate server, the attacker captures the NTLM hash sent during that exchange. Third, instead of trying to crack it, the attacker relays the hash directly to a server that trusts the source, impersonating the original user. Fourth, if the relay succeeds, the attacker gets into the target server's resources (files, databases, admin privileges) and can use that as a launchpad for lateral movement.

#### Dumping and Cracking NTLM Hashes

Windows stores its local user account passwords, hashed, in the SAM (Security Account Manager) database, while the Local Security Authority (LSA) handles authentication. On modern versions, the SAM database itself is also encrypted with a syskey for an extra layer of protection.

It's worth being precise about how these hashes actually get extracted, because two different sources tend to get conflated. The SAM database file can't be copied directly while Windows is running (the kernel keeps it locked), so pulling local account hashes usually means saving the SAM and SYSTEM registry hives offline (`reg save`, or tools like Impacket's `secretsdump.py`) and decrypting them using the syskey. That's a separate path from LSASS process memory, which holds the credentials of currently active sessions (interactive logons, cached domain credentials) and is what Mimikatz's `sekurlsa` module targets. Mimikatz's `lsadump::sam` module, by contrast, reads from the registry hives rather than LSASS memory. The two techniques target different data and require different access.

Windows has used two hash types historically. LM hashing, used by default up through Windows XP and Server 2003, is considered weak: it splits the password into two 7-character chunks and hashes them independently, which makes it trivial to crack. NTLM hashing replaced it from Windows Vista onward, using the MD4 algorithm. It's case-sensitive, supports symbols and Unicode, and doesn't split the password the way LM does, a meaningfully stronger design, though still crackable offline given enough compute.

Once administrative access is in hand, hashes can be pulled using Meterpreter's built-in `hashdump` command, which extracts SAM hashes via the registry hives, or Mimikatz, which can go further and pull plaintext credentials, hashes, and even Kerberos tickets straight out of LSASS memory.

From there, cracking happens offline. John the Ripper is the flexible, general-purpose option: dictionary attacks, rule-based mutations, brute force. Hashcat takes the GPU-accelerated route, handling large wordlists and more complex attack modes at speeds CPU-based tools can't really compete with.

#### Wrapping Up

This article covered the enumeration phase and two of the most common attack paths that follow it on Windows networks: what enumeration actually is, NetBIOS and its three services, SMB and its evolution from version 1.0 to 3.0+, SNMP and what a pentester looks for when enumerating it, the mechanics of an SMB relay attack, and how Windows stores and protects password hashes, along with the real difference between pulling them from the SAM registry hives versus LSASS memory, and how they get cracked afterward.

Enumeration is where recon turns into something actionable. A misconfigured SMB share, a weak SNMP community string, or a recoverable NTLM hash can be the difference between a foothold and a failed engagement.

---

#### Disclaimer

This material is intended for educational purposes and authorized security testing only.

Always obtain proper authorization before interacting with systems you do not own or explicitly have permission to test.

---
