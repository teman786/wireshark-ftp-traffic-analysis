# ICMP Traffic Analysis in Wireshark

## Introduction

**ICMP (Internet Control Message Protocol)** is a Network Layer (Layer 3) protocol mainly used for network diagnostics, error reporting, and control messages.

One of the most common uses of ICMP is the `ping` command, which uses ICMP Echo Request and Echo Reply messages to check whether a host is reachable.

Although ICMP is not designed for transferring application data, attackers can abuse its payload to create covert communication channels and exfiltrate data from compromised systems.

---

## How ICMP Works

A typical ICMP ping communication looks like this:

```text
Internal Host
     |
     | ICMP Echo Request (Type 8)
     ↓
Destination Host
     |
     | ICMP Echo Reply (Type 0)
     ↓
Internal Host
```

The Echo Request is sent by the source host, and the destination responds with an Echo Reply.

---

## Important ICMP Types

ICMP messages contain a **Type** and **Code** field that identify the purpose of the message.

| Type | Description             |
| ---: | ----------------------- |
|    0 | Echo Reply              |
|    3 | Destination Unreachable |
|    5 | Redirect                |
|    8 | Echo Request            |
|   11 | Time Exceeded           |
|   12 | Parameter Problem       |
|   13 | Timestamp Request       |
|   14 | Timestamp Reply         |

For traffic analysis, **Type 8 (Echo Request)** and **Type 0 (Echo Reply)** are particularly important.

---

# ICMP Data Exfiltration

**Data exfiltration** is the unauthorized transfer of data from a compromised system or network to an external destination.

In ICMP-based exfiltration, an attacker can place encoded, encrypted, or otherwise obfuscated data inside the ICMP payload.

The basic concept is:

```text
Sensitive Data
      ↓
Encode / Encrypt
      ↓
Split into chunks
      ↓
ICMP Payload
      ↓
External Host
```

For example, a large file can be divided into multiple chunks and transmitted through a series of ICMP packets. A remote system controlled by the attacker can collect and reconstruct the data.

---

## How Attackers Abuse ICMP

### 1. Echo Request/Reply Tunneling

Attackers can place data inside ICMP Echo Request or Echo Reply packets.

```text
Internal Host
      |
      | ICMP Echo Request + Data
      ↓
External Server
```

Multiple packets can be used to transfer different chunks of the stolen data.

---

### 2. Data Encoding

Attackers may encode data before placing it inside the ICMP payload.

Common examples include:

* Base64
* Hexadecimal encoding
* Other custom encoding schemes

Encoded data may appear as random-looking characters during packet inspection.

Encoding alone does not prove malicious activity, so additional context is required.

---

### 3. Fragmentation

Large data can be divided into smaller pieces:

```text
File
 ↓
Chunk 1 → ICMP Packet
Chunk 2 → ICMP Packet
Chunk 3 → ICMP Packet
Chunk 4 → ICMP Packet
```

The receiving system can then reconstruct the original data.

---

### 4. Encryption and Obfuscation

Attackers may encrypt or obfuscate the data before transmitting it.

This makes the payload difficult to identify using simple string-based detection.

---

# Indicators of ICMP Exfiltration

A SOC analyst should look for several indicators rather than relying on a single packet.

### 1. High ICMP Volume

A single internal host repeatedly communicating with the same external IP can be suspicious.

```text
192.168.1.10 → External IP
192.168.1.10 → External IP
192.168.1.10 → External IP
192.168.1.10 → External IP
```

If there is no legitimate monitoring or troubleshooting reason, the activity should be investigated.

---

### 2. Unusually Large Payloads

Normal ping traffic generally uses relatively small payloads.

Unusually large ICMP packets may indicate that additional data is being carried inside the payload.

---

### 3. High-Entropy or Encoded Data

ICMP payloads containing random-looking data or patterns resembling Base64 or hexadecimal encoding may require further investigation.

---

### 4. Regular Communication Intervals

Repeated ICMP packets at consistent intervals can indicate automated communication or beaconing behavior.

Example:

```text
10:00:00 → ICMP
10:00:10 → ICMP
10:00:20 → ICMP
10:00:30 → ICMP
```

Regular timing combined with unusual payloads is more suspicious than occasional ping traffic.

---

### 5. Unusual ICMP Types or Codes

Most normal diagnostic traffic uses common ICMP types such as Echo Request and Echo Reply.

Unusual ICMP types or codes may require additional investigation, especially when combined with unexpected external communication.

---

# Wireshark Analysis

Open the provided PCAP file in Wireshark and begin by filtering all ICMP traffic.

## 1. Filter All ICMP Traffic

Use:

```text
icmp
```

This filter isolates ICMP packets from the rest of the network traffic.

Review:

* Source IP
* Destination IP
* ICMP Type
* ICMP Code
* Packet length
* Payload

The objective is to identify unusually frequent or unusually large ICMP communication.

---

## 2. Filter ICMP Echo Requests

Use:

```text
icmp.type == 8
```

This filter displays only **ICMP Echo Request** packets.

Echo Requests are important because attackers may abuse them to carry data inside their payload.

Investigate whether:

* One internal host sends many requests
* The destination is an unusual external IP
* Payloads are unusually large
* Payload contents appear encoded or random
* Packets are sent at regular intervals

---

## 3. Identify Large ICMP Packets

A useful investigation filter is:

```text
icmp && frame.len > 100
```

This highlights ICMP frames larger than 100 bytes.

Large packets should then be inspected to determine whether the additional data is legitimate or potentially related to exfiltration.

> The `100` byte threshold is an investigation example, not a universal definition of malicious ICMP traffic.

---

## 4. Inspect the ICMP Payload

Select an interesting packet and expand:

```text
Internet Control Message Protocol
```

Inspect the **Data/Payload** section.

Look for:

* Readable information
* Large amounts of data
* Base64-like strings
* Hexadecimal data
* Random or high-entropy content
* Repeated payload structures

---

# Normal ICMP vs Potential ICMP Exfiltration

| Normal ICMP              | Potential Exfiltration               |
| ------------------------ | ------------------------------------ |
| Occasional ping          | Continuous or high-volume traffic    |
| Small payload            | Unusually large payload              |
| Known destination        | Unknown/unusual external destination |
| Diagnostic purpose       | No clear legitimate purpose          |
| Predictable traffic      | Encoded/random-looking payload       |
| Occasional communication | Regular periodic communication       |

---

# SOC Investigation Workflow

A practical investigation can follow this sequence:

```text
ICMP Traffic
     ↓
Identify Source IP
     ↓
Identify Destination IP
     ↓
Check ICMP Type/Code
     ↓
Check Packet Size
     ↓
Inspect Payload
     ↓
Check Encoding/Obfuscation
     ↓
Analyze Traffic Frequency
     ↓
Check Internal → External Communication
     ↓
Determine Legitimate or Suspicious Activity
```

---

# Conclusion

ICMP is a legitimate network protocol used for diagnostics and error reporting, but attackers can abuse its payload for covert communication and data exfiltration.

A single ICMP packet is not enough to classify activity as malicious. Stronger evidence comes from correlating multiple indicators such as:

**Internal host + unusual external destination + high ICMP volume + large/encoded payload + regular timing**

By using Wireshark filters and inspecting packet payloads, SOC analysts can identify suspicious ICMP communication and investigate potential data exfiltration.
