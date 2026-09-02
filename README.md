# Wireshark FTP Traffic Analysis

This is my first Wireshark-based network traffic analysis project. In this lab, I analysed an FTP PCAP file to investigate FTP activity and identify potential indicators of data exfiltration.

The investigation focuses on identifying:

* FTP authentication attempts
* Usernames and passwords
* File uploads using `STOR`
* File downloads using `RETR`
* Specific file types such as CSV files
* Large FTP payloads
* Internal-to-external FTP communication
* TCP streams containing transferred data

## First Step: Identify FTP Traffic

The first filter used to analyse the FTP PCAP file is:

```text
ftp
```

This filter displays FTP-related packets and provides an initial view of the FTP communication.

To display both FTP control traffic and FTP data traffic, use:

```text
ftp || ftp-data
```

### Why Start With This Filter?

Starting with `ftp` helps establish the basic FTP communication between the client and server. From there, we can investigate authentication, file operations, and potential data exfiltration activity.

The investigation can then proceed with additional Wireshark filters such as:

```text
ftp.request.command == "USER" || ftp.request.command == "PASS"
```

```text
ftp contains "STOR"
```

```text
ftp contains "RETR"
```

```text
ftp contains "csv"
```

```text
ftp && frame.len > 90
```

The final goal is to determine whether the observed FTP activity represents legitimate file transfer or potential data exfiltration.
