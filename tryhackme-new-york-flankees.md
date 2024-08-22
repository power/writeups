# TryHackMe - New York Flankees

## Enumeration

As usual, we start with our nmap scan which reveals a HTTP server on port 8080 and SSH on port 22.

```
# Nmap 7.94SVN scan initiated Wed Jul 24 09:46:42 2024 as: nmap -sCV -oN nmap/output 10.10.196.131
Nmap scan report for 10.10.196.131
Host is up (0.026s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ad:bc:7d:02:80:33:d9:35:8d:96:5b:bc:4f:94:ca:9f (RSA)
|   256 ba:e8:a1:e4:65:11:c4:26:75:ba:f0:10:58:db:78:78 (ECDSA)
|_  256 93:2c:0f:e9:73:fd:54:08:e9:b0:fc:51:57:c7:dd:eb (ED25519)
8080/tcp open  http    Octoshape P2P streaming web service
|_http-title: Hello world!
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Jul 24 09:46:50 2024 -- 1 IP address (1 host up) scanned in 8.64 seconds
```

We begin exploring the webpage and notice an "Admin Login" and a "Stefan Test" button at the top.

<figure><img src=".gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

The "Stefan Test" page contains two TODO's:

* Implement custom authentication
* Fix verbose error padding

<figure><img src=".gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

This doesn't appear to be too useful until we check the  source code which reveals some JavaScript and something that looks like it could be our in.

```javascript
    function stefanTest1002() {
        var xhr = new XMLHttpRequest();
        var url = "http://localhost/api/debug";
        // Submit the AES/CBC/PKCS payload to get an auth token
        // TODO: Finish logic to return token
        xhr.open("GET", url + "/39353661353931393932373334633638EA0DCC6E567F96414433DDF5DC29CDD5E418961C0504891F0DED96BA57BE8FCFF2642D7637186446142B2C95BCDEDCCB6D8D29BE4427F26D6C1B48471F810EF4", true);

        xhr.onreadystatechange = function () {
            if (xhr.readyState === 4 && xhr.status === 200) {
                console.log("Response: ", xhr.responseText);
            } else {
                console.error("Failed to send request.");
            }
        };
        xhr.send();
    }
```

This function gives us a URL and a location it'll be sent to, along with how it should be handled on response. We cannot run this through our developer console but find that we have some luck by visiting the URL as we gain a "Custom authentication success" message.

<figure><img src=".gitbook/assets/image (2) (1).png" alt=""><figcaption></figcaption></figure>

We try and alter the URL but quickly find our error message, "Decryption Error".

<figure><img src=".gitbook/assets/image (3) (1).png" alt=""><figcaption></figcaption></figure>

Where from here? Well, this stumped me for a bit until I tried to put some of the clues together, on the front page this fictional company is sponsored by "Oracle" and one of Stefan's things to do was fix the padding, so after many stabs in the dark we get a hit with this.

<figure><img src=".gitbook/assets/image (4) (1).png" alt=""><figcaption></figcaption></figure>

Well, this looks interesting. Depending on your interest in the area, this could be something of interest to follow but as I was focused on solving the box, I searched for a HackTricks article which helped me find what I needed.

{% embed url="https://book.hacktricks.xyz/crypto-and-stego/padding-oracle-priv" %}

HackTricks puts this attack into slightly more compressed terms, making it easier to follow. We can start by guessing the padding and figuring out the decrypted version of the string by starting at the end and finding null bytes, and then starting to build a padding and then developing a base for invalid characters, and then finding the values of correct characters. HackTricks recommends `padbuster` for this but the first tool I found was `padre` which did the trick for me.



## User

For `padre` to operate we'll pass the URL we're testing - `http://10.10.196.131:8080/api/debug/$"`

The encoding to apply to our data, which we'll specify for Hexadecimal - `-e lhex`

And then finally the data we found from the JavaScript function.

From this, we get a set of credentials!

<figure><img src=".gitbook/assets/image (5) (1).png" alt=""><figcaption></figcaption></figure>

We head back to the "Admin Login" page and log in, having access to submit commands now. We check whether we can see the output from commands but simply just get an "OK" if our command is valid, or no response if not.&#x20;

<figure><img src=".gitbook/assets/image (6) (1).png" alt=""><figcaption></figcaption></figure>

We try and launch a reverse shell through the browser but have no luck, so create a bash script that should launch us into a shell, which it does.

<figure><img src=".gitbook/assets/image (7) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (8) (1).png" alt=""><figcaption></figcaption></figure>



## Root

We stabilise our shell and then explore our environment, appearing to be in some kind of docker container, which we can confirm our suspicions from the question so check for docker breakouts with quite a common one.



We start by checking for other docker images for which we find a gradle docker image from a similar time. We then run the following which breaks us out!

```
docker run -it -v /:/host/ d5954e1d9fa4 chroot /host/ bash
```

<figure><img src=".gitbook/assets/image (9) (1).png" alt=""><figcaption></figcaption></figure>

We can see from the difference in bases of our directory that we've broken out and we have our final flag, marking this box as complete!
