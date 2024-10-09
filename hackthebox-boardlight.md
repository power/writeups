# HackTheBox - Boardlight

## Enumeration

We start with our nmap scan which reveals two ports, a webserver and SSH service.

```
# Nmap 7.94SVN scan initiated Fri May 31 09:31:24 2024 as: nmap -sCV -oN nmap/output 10.10.11.11
Nmap scan report for 10.10.11.11
Host is up (0.032s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 06:2d:3b:85:10:59:ff:73:66:27:7f:0e:ae:03:ea:f4 (RSA)
|   256 59:03:dc:52:87:3a:35:99:34:44:74:33:78:31:35:fb (ECDSA)
|_  256 ab:13:38:e4:3e:e0:24:b4:69:38:a9:63:82:38:dd:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri May 31 09:31:34 2024 -- 1 IP address (1 host up) scanned in 10.23 seconds
```

We had to the website and scroll around, and find that this domain is associated with "board.htb" so add this to our /etc/hosts

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

Straight from this we do some subdomain enumeration and find the "crm" subdomain, so add this to our /etc/hosts and navigate to the index of that subdomain.

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

This reveals the "Dolibarr" service running on version 17.0.0. A quick Google search reveals that there is a PHP command injection vulnerability.

{% embed url="https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253" %}

We copy these files and follow the default usage and gain a shell on the machine!

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

## User

We then start enumerating the machine a find the website configuration file which contains information for the MySQL database associated with the website.

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

Taking the credentials "dolibarrowner:serverfun2$2023!!" we log into the MySQL service and find two password hashes... Copying these over to our machine we attempt to crack them with hashcat, which does not return any successful password hashes.

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

At a slight point of loss, we make a wild guess that these credentials may be reused and attempt to become the "larissa" user, which we know is on this machine. Sure enough, this is successful and we find our user flag!

<figure><img src=".gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

## Root

From this point we upload LinPEAS to the machine and wait for the results. Weridly enough we find some unknown SUID binaries that look to be of interest.

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

We look into whether any vulnerabilities exist related to this binary and sure enough they do! We download this onto our Kali machine then download it on the box, give ourselves executable permissions and run the exploit, which gives us root!

<figure><img src=".gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>
