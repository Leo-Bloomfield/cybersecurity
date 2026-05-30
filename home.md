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


* **Machines**: _conferma i vari ip_
    * **Kali Linux**: (IP:192.168.11.96) that will run Evilnginx2 and Nginx proxy manager in Docker containers.
    * **Windows 11**: (IP: 192.168.11.118) it will be the victim that will be targeted by the ARP poisoning and DNS spoofing attacks.
    * **Raspberry Pi**: (IP:192.168.11.61) that will be used to perform the ARP poisoning attack using Bettercap.
    * **Ubuntu Server**: will serve the nextcloud instance that the attacker will impersonate for the phishing attack.

## Domain configuration

Because this is self-contained demo, I have used a domain I already own ```leobloomfield.eu```, and managed by cloudflare. \
The records I have added are:
```kali.leobloomfield.eu A 192.168.11.96```
```fake.leobloomfield.eu CNAME kali.leobloomfield.eu```
I also have records that point to the nextcloud instance, but they are not relevant for this demo.\
Then I also created zone dns tokens for the SSL certificate generation. It will later be used by nginx proxy manager to generate the certificate for the fake domain.

## ARP poisoning attack

The ARP poisoning attack is performed using Bettercap on the Raspberry Pi. I tried running bettercap on the Kali Linux machine, but it was not able to perform the attack successfully, probably because virtualization, while on the Raspberry Pi it worked without any issues. In practice the evil proxy may not be on the same machine as the one performing the ARP poisoning attack, so this setup is still valid for the demonstration.\
The command used to perform the ARP poisoning attack is:
```bash
sudo bettercap -iface eth0
 ```
and the subsequent commands to perform the attack are:
_cambia address per tailscale_
```bash
net.recon off
net.probe off
set arp.spoof.targets 192.168.11.118
set arp.spoof.fullduplex true
arp.spoof on
set dns.spoof.targets 192.168.11.118
set dns.spoof.domains nc.leobloomfield.eu
set dns.spoof.address 100.64.1.216
dns.spoof on
```

## Evilnginx2

Evilnginx2 is used to create a fake website that looks identical to the legitimate one, allowing the attacker to steal credentials. The fake website is hosted on the Kali Linux machine, and it is accessible through the domain ```fake.leobloomfield.eu```.\
The configuration of Evilnginx2 is done through a configuration file, called phishlet, which specifies the domains to be spoofed and the credentials to be stolen. The configuration file is as follows:
_aggiungi phishlet_
```yaml```
_Aggiungi i vari comandi per la configurazione di evilnginx2_
Evilnginx2 is inside a docker container, and it is configured to use the Nginx proxy manager as a reverse proxy to receive the SSL certificate for the fake domain. The Nginx proxy manager is also running in a docker container on the Kali Linux machine.
This is because otherwise I would be unable to get certificates for the fake domain, because I'm behind a NAT and I don't have a public IP address.\
Nginx proxy manager can use DNS challenge to get the certificate, usig the zone dns token I created on cloudflare.
I also put a redirection host that when a host access the ip of kali machine but with the `nc.leobloomfield.eu` domain, it redirects to the `fake.leobloomfield.eu` domain. I used there a self-signed certificate.



