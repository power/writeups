# TryHackMe - W1seGuy

This is a slightly different box then what we're used to but we start by analysing the Python code behind this challenge. We find that of interest is the `setup` & `start` functions.&#x20;

````python
```
def setup(server, key):
    flag = 'THM{thisisafakeflag}' 
    xored = ""

    for i in range(0,len(flag)):
        xored += chr(ord(flag[i]) ^ ord(key[i%len(key)]))

    hex_encoded = xored.encode().hex()
    return hex_encoded
```
````

We see earlier in the code that the flag is defined by the actual flag on the server and the function then XOR's each character with the key.

````python
```
def start(server):
    res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))
    key = str(res)
    hex_encoded = setup(server, key)
    send_message(server, "This XOR encoded text has flag 1: " + hex_encoded + "\n")
    
    send_message(server,"What is the encryption key? ")
    key_answer = server.recv(4096).decode().strip()
```
````

From here we can see that our key is defined by the original value of the "res" variable. The res variable is a random set of letters and numbers that we know is 5 characters. We can tell from the `setup` function that our flag starts with the string "THM{" and must end with "}" meaning that we can bruteforce and find the key and the text inside of it. How can we do this?



Well, we know that on every connection to the server it'll gives us a new key so we must keep it open whilst we run our script, but we can try every combination of strings and letters through the use of some iteration until we eventually find a combination that meets the requirements that we know. There's multiple Python libraries that contain "xor" functionality but we'll utilise the "pwn" library.

```python
#!/usr/bin/env python3

from pwn import xor
import string

encrypted = bytes.fromhex("162b7828237302593d27071b4112273657563830030d4760322e2f4c3b0630174c6326301b7a212e")
start = xor(b"THM{", encrypted[:4])
for i in string.ascii_letters + string.digits:
    key = start + i.encode()
    print(f"{key} : {xor(encrypted, key)}")
```

We start with our imports of the ASCII letters and digits from the "string" library and then importing "xor" from the "pwn" library.



We start by defining a variable with the decoded bytes of the hexadecimal XOR'd flag. We then know what the decoded version will start with so define a variable that stores the first 4 letters of the "encrypted" variable as the XOR'd equivelant to "THM{". Since we know the first 4 letters of the key, we're only testing for the 5th and final letter so can then concatenate each string or digit and then print the output.



We run the script and we're looking for whichever version ends with a "}" and know that'll be our key, and first flag.

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

We find our flag and then pass the relevant information to the script, and get our 2nd flag.

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>
