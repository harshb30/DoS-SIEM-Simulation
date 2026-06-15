# DoS Simulation & Splunk Threat Hunting

![Splunk](https://img.shields.io/badge/splunk-000000?style=for-the-badge&logo=splunk&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Kali](https://img.shields.io/badge/Kali-%23557C94.svg?style=for-the-badge&logo=kalilinux&logoColor=white)

## Objective
The objective of this project is to set up an isolated virtual environment to simulate an Application Layer (Layer 7) HTTP Flood attack then ingest the resulting web traffic into a SIEM (Splunk), and execute  mitigation techniques to defend the web server.

## Tools & Technologies

| Role | Tool / Technology |
|---|---|
| SIEM Setup | Splunk Enterprise (on Ubuntu Server 26.04) |
| User Endpoint (Victim VM) | Ubuntu Server 26.04 with Apache2 Server |
| Attacker Machine | Kali Linux |
| Attack Tools | Apache Bench |

---

## Architecture & Environment Setup

The home lab runs on VMware Workstation on a fully isolated network and consists of two virtual machines each having specific roles:

- **Ubuntu (SIEM + Apache2 + Target)** — Hosts the Splunk Enterprise instance and Apache2 server.
- **Kali (Attacker)** — Serves as the attack machine, the origin point of all attack activity during the simulation, including Nmap scans and Hydra brute force.

---

## Execution Phases

### Step 1: SIEM Configuration
Before launching the attack, Splunk was configured to ingest the following logs.

1. **Web Traffic Monitoring:** Added `/var/log/apache2/access.log` into Splunk.
2. **System/Firewall Monitoring:** Added `/var/log/syslog` into Splunk.

### Step 2: The Attack (Kali)
To simulate a Denial of Service without crashing the virtual environment, a controlled HTTP GET flood was launched against the Apache server using Apache Benchmark from the Kali VM.

```bash
ab -n 25000 -c 50 http://10.0.2.128/
```

### Step 3: Detection & Analysis (Splunk UI)

```bash
##spl query
source="/var/log/apache2/access.log" host="harshubuntu" sourcetype="apache2_access_logs" | timechart count span=1s
```
By setting the timechart to span=1s the speed of the automated attack became clear, making it easy to distinguish from normal human web traffic

### Step 4: Mitigation

After identifyting the malicious IP, two rules were added on the target VM, one for logging the requets and another one for dropping the connection.
```bash
#logging rule
sudo iptables -A INPUT -s 10.0.2.131 -p tcp --dport 80 -j LOG --log-prefix "DDoS_BLOCK: "
```
```bash
#block rule
sudo iptables -A INPUT -s 10.0.2.131 -p tcp --dport 80 -j DROP
```

### Step 5: Verification

The ab attack was launched for a second time from Kali. The connection timed out completely, proving the web server was shielded.
```bash
##spl query
source="/var/log/syslog" "DDoS_BLOCK:"
```
The attack attempted 25,000 requests again but only few firewall logs appeared in Splunk as the target VM (Ubunutu) successfully intercepted and dropped the initial TCP SYN packets during the handshake, which failed causing the HTTP requests to never actually be sent.
