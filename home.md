# ARP poisoning and DNS spoofing and phishing attack
## Introduction
In this demo, an ARP poisoning attack is performed to intercept network traffic between a victim and the gateway. The attacker then uses DNS spoofing to redirect the victim's web traffic to a malicious website, which is designed to look like a legitimate site. Finally, a phishing attack is carried out to steal the victim's credentials.
## Configuration

* **Threat model**:
    * The attacker is a malicious actor who has access to the same network as the victim. They can perform ARP poisoning to intercept traffic and DNS spoofing to redirect the victim's web traffic.
* **Tools**:
    * **Bettercap**: A simple command line tool to perform ARP poisoning and DNS spoofing attacks.
    * **Evilnginx2**: An attacker-in-the-middle proxy tailored for phishing attacks. It can be used to create a fake website that looks identical to the legitimate one, allowing the attacker to steal credentials.
    * **Nginx proxy manager**: A reverse proxy that allows evilnginx2 to receive certificates even when behind a NAT
    * **Nextcloud**: A self-hosted file sharing and collaboration platform that will be used as the legitimate website for the phishing attack.
    * **Docker**: A platform that allows the attacker to run the necessary tools in isolated containers, making it easier to manage dependencies and configurations.


* **Machines**:
    * **Kali Linux**: (IP:192.168.11.96) that will run Evilnginx2 and Nginx proxy manager in Docker containers.
    * **Windows 11**: (IP: 192.168.11.218) it will be the victim that will be targeted by the ARP poisoning and DNS spoofing attacks.
    * **Raspberry Pi**: (IP:192.168.11.61) that will be used to perform the ARP poisoning attack using Bettercap.
    * **Ubuntu Server**: will serve the nextcloud instance that the attacker will impersonate for the phishing attack.

## Domain configuration
Because this is self-contained demo, I have used a domain I already own ```leobloomfield.eu```, and managed by cloudflare. \
The records I have added are:
```kali.leobloomfield.eu A 192.168.11.96```
```fake.leobloomfield.eu CNAME kali.leobloomfield.eu```
I also have records that point to the nextcloud instance, but they are not relevant for this demo.
Then I also created zone dns tokens for the SSL certificate generation.

