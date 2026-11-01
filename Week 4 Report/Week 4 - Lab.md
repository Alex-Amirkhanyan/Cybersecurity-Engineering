
First we need to install Wireshark on Ubuntu-lab, for that we switch the Network type to NAT in VirtualBox and boot into it, after which we write `sudo apt install wireshark` in the terminal. Now while Wireshark is being downloaded, we check some DNS with `nslookup`:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101175345.png>)


Now some more information with the `dig` command:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101175841.png>)

And as the download finished, let's move on to the next task. But before that we change the ubuntu-lab's Network type back to Internal Network. As later I am going to catch a password flying across the network in plain text, ubuntu-srv needs an old, unencrypted login service, called Telnet. Let's set it up first. First step is to change the Network type from Internal Network to NAT, and then boot it. Now, the following steps:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101184003.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101184256.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101184318.png>)

As ubuntu ships telnet disabled, next step is to enable it:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101184456.png>)

And we check, if something listens to telnet's port:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101184759.png>)

Now, as the task requires, I create a new user with commands `sudo useradd -m -s /bin/bash labuser` and add password with the command `sudo passwd labuser`, which is `Cleartext123`. Now it is time to change the Network Type back to Internal Network. After which I reboot and restore the lab's IP on my own:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101185611.png>)

Now I test it from the ubuntu-lab:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101190112.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101190126.png>)

As we see, everything worked out fine. Now let's capture some traffic:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101190937.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101190955.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101192431.png>)

And we check the pings. For that we stop the recording by pressing the red square inside the Wireshark and filter out the `icmp`:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101192532.png>)

Let's try out some more filters.

`arp`:
![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101193037.png>)

`ip.addr == 192.168.56.11`:
![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101193249.png>)

`tcp.port == 22`:
![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101193401.png>)

Now a combination of filters, `ip.addr == 192.168.56.11 and tcp.port == 22`
![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101193553.png>)

For the next part I have to catch the password in plain text. For that, I will record myself logging in as `labuser` through telnet using Wireshark:
![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101194230.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101194408.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101194424.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101194548.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101194630.png>)

And as we see, both the login and the password are visible. To compare with SSH, I relaunch the Wireshark's capture and connect through SSH:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101194830.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101194856.png>)


### Real attack Investigation

And as we see, everything is encrypted. Now it is time to investigate a real attack. I got the file `lab4_investigation` for that, which is provided with this directory for your own, if you will want it. I drag and dropped it into my ubuntu, and to start the investigation, we just type the following:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101203026.png>)

Which opens the files inside Wireshark:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101203036.png>)

It is provided that "*This capture is NOT clean. It is a busy network: someone streaming video, someone on email, someone playing a game, someone browsing the web — hundreds of normal packets. Hidden inside all that noise is one attacker doing three things. Your job is to separate the signal from the noise. This is exactly what a real analyst faces: attacks do not arrive on an empty network.*"

#### For the investigation I must answer several questions.
##### Question 1: Which IP is doing the fast sweep? How is that different from the normal ARP you also see? What is it trying to find out?

The IP address which does the fast sweep is `192.168.1.50`. It is different from normal ARPs in the way that it asks for every IP address, trying to find out the IP addresses with their corresponding MACs, and that is what the attacker was trying to find out.

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101204529.png>)

##### Question 2: Filter the connection attempts. Who is scanning whom? Which ports answered? Filter the open ports, the ones that replied with SYN-ACK. How is this different from Alice watching YouTube livestream or Carol's game traffic?

To filter the connection attempts, we filter `tcp.flags.syn == 1 and tcp.flags.ack == 0`:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101205135.png>)

The scanner is, as we know, the 192.168.1.50, and is scanning for open ports, mostly 192.168.1.7 and 192.168.1.10.

The following ports answered (The SYN-ACK connection):

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101232008.png>)


And it is mainly 443 and 23 ports.

Attacker's SYN'a answers, the SYN-ACKs, is different from YouTube's livestream or game's traffic, in the way that the latter 2's connection are long lasting, but the attacker's one is one-off connection probes across different ports.

##### Question 3: Filter arp again. Somewhere in the normal ARP replies is a lie: an ARP reply claiming that 192.168.1.1 (the gateway) is at the attacker's MAC address. The real gateway also replies honestly — you must spot the fake by comparing the MAC. What is the attacker's MAC? Find the ARP reply where 192.168.1.1 is claimed to be at THAT MAC. What attack is this, and what does it let the attacker do?

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101215630.png>)

As we see, normally the IP addresses lead to the MAC `44:d9:e7:1a:2b:3c`:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101215734.png>)

But here, we can see, that the attacker sneaked in his MAC address:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101215834.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101220059.png>)

the `dc:a6:32:04:9f:e1`. What led me to this conclusion is that the source name `RaspberryPiT_04:9f...` occurs every time when the attacker made a request, but  `Ubiquiti_1a:2b:3c` occurred rarely. This is what the man in the middle does, in this case it is ARP spoofing. It redirects whatever is needed to the path the attacker chooses.
##### Question 4: The stolen password. Filter telnet . What username and password did the attacker capture? Why was it readable, when the other users' traffic was not?

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101222304.png>)

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101222351.png>)

As we see the attacker captured `username:admin and password: Sup3rSecret!`. It was readable, because telnet was used when logging in, and it, as we know, encrypts nothing, and transfers data in plain text.

##### Question 5: What did they do next?

Their next step was to run the following command: `cat /etc/shadow`,

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101222724.png>)

which displayed the list of local user accounts and their authentication data. And now the attacker has data about all the accounts and their password hashes. 

And now to finish it all, we just disable the telnet service in ubuntu server:

![PowerShell Get-LocalUser output](<Week4/Pasted image 20261101223330.png>)
