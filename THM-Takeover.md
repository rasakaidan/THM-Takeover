<p align="center">
    <strong>Difficulty:</strong> Easy<br>
    <strong>Room:</strong> <a href="https://tryhackme.com/room/takeover">TakeOver</a><br>
    <strong>Focus:</strong> Subdomain Enumeration & Virtual Host Discovery
</p>
## Initial Reconnaissance

### Certificate Warning
Upon visiting the website, we immediately notice a certificate warning, which we can click through to continue to the website.
### Nmap Scan
Let's start with a port scan to identify available services:

```bash
nmap -sV 10.10.193.63
```

**Results:**
```
Starting Nmap 7.80 ( https://nmap.org ) at 2025-09-17 17:03 BST
Nmap scan report for futurevera.thm (10.10.193.63)
Host is up (0.0068s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http     Apache httpd 2.4.41 ((Ubuntu))
443/tcp open  ssl/http Apache httpd 2.4.41 ((Ubuntu))
```

**Key Findings:**
We find Apache web server running on ports 80 (HTTP) and 443 (HTTPS)
There are no DNS services detected on the target
## Virtual Host & Subdomain Enumeration

Since no DNS services are running on the target machine, we need to enumerate subdomains and virtual hosts manually.
### Virtual Host Fuzzing

Virtual host fuzzing is a technique used to discover subdomains or virtual hosts that are configured on a web server but may not be publicly listed in DNS records. Web servers can host multiple websites on the same IP address by using the `Host` header in HTTP requests to determine which site to serve.
The process of fuzzing includes sending HTTP requests with different header `Host` values and filtering out different values to identify valid hosts from false positives, this can be done by looking at response sizes, status codes or content.
### ffuf

We'll use `ffuf` to enumerate potential subdomains:

```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt -H "Host: FUZZ.futurevera.thm" -u https://10.10.193.63
```

The initial scan returns many false positives with a consistent response size of 4605 bytes, indicating a default response for non-existent hosts.

**Filtered Command:**
```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt -H "Host: FUZZ.futurevera.thm" -u https://10.10.193.63 -fs 4605
```

The `-fs 4605` flag filters out responses with that specific size leaving us with two valid subdomains.

```
support    [Status: 200, Size: 1522, Words: 367, Lines: 34]
blog       [Status: 200, Size: 3838, Words: 1326, Lines: 81]
```

And we add the discovered subdomains to our `/etc/hosts` file:

```bash
echo "10.10.193.63 futurevera.thm support.futurevera.thm blog.futurevera.thm" >> /etc/hosts
```

## Certificate Analysis

It was at this point, after discovering the two subdomains, that progress slowed down. I couldn't find any information of note in the pages source code or other avenues. After trying different methods of information gathering such as a directory scan I realised that both of these pages had certificate warnings when first trying to access them, much like the one we saw on the main futurevera.thm page. From here I decided to dig a bit deeper into the self signed certs to see if anything was out of place. 

While the `blog` subdomain certificate appears normal, the `support` subdomain certificate contains interesting information in its Subject Alternative Name (SAN) field.
```
secrethelpdesk934752.support.futurevera.thm
```

and we update `/etc/hosts` again to include the newly discovered subdomain:

```bash
echo "10.10.193.63 secrethelpdesk934752.support.futurevera.thm" >> /etc/hosts
```

Which leads to an error page with the flag in the URL!!!

## Key Takeaways
My main takeaway from this CTF is that persistence is key to breakthroughs, sometimes traditional methods of enumeration can fail you and that valuable information can be hidden in error pages or certificates rather than main page content. 
Not my favourite CTF i've done, wish it was a bit more realistic but it was a nice refresher on ffuf. 
