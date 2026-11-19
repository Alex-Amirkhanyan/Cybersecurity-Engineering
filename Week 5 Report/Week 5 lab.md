
We first check if the UFW is installed on our Ubuntu server:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105192313.png>)



Then, the first thing we do is set the golden rule, which is setting default deny for incoming but allow anything outgoing.

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105192439.png>)

And as we see, we got confirmation from both commands.

Now we allow what we need, in this case it is the SSH port:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105192552.png>)

Now we turn on the firewall:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105192641.png>)

And I got the confirmation message as well.

Now I check if it really blocks. First I check if a connection through SSH can be made: 

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105192936.png>)

And yes it can be made, now it's time to check another port:

![PowerShell Get-LocalUser output](<Week5/Screenshot 2026-11-05 193406.png>)

As we see, I got the message **timed out**, which confirms that our uncomplicated firewall did its job correctly.
### Now I cut the network into two separate zones that cannot see each other

First we power off our machines, so we can make the changes.

The next step is to assign machines to segments.

ubuntu-lab:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105195110.png>)

Kali:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105195148.png>)

The change I made here is in the name. ubuntu-lab got Adapter 1 with the name labnet, and kali got Adapter 1 with the name labnet2. 

Two different names mean two separate, isolated networks.

Now I boot the machines back up and give each an address in its own subnet:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105195513.png>)

![PowerShell Get-LocalUser output](<Week5/Screenshot 2026-11-05 195600.png>)

As it can be seen, the third numbers differ: different 3rd number, hence a different subnet.

Now I check if they truly are isolated, for that I will try to ping kali from ubuntu-lab:

![PowerShell Get-LocalUser output](<Week5/Screenshot 2026-11-05 200059.png>)

"Network is unreachable" is what we needed to get as a response: this proved that the networks are isolated, and there is no path between them.

### Putting a gateway between segments

As mentioned previously the two systems are isolated, because they are cut into segments. Now I will make ubuntu-srv (Ubuntu server) the router/gateway between them. For the first step:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105200939.png>)

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105200952.png>)

Now the server has one card in each segment. After applying the settings I start the ubuntu-srv, and run the following command in order to find the two interfaces:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105201156.png>)

Now I give each an address, as a gateway of its segment:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105201346.png>)

This way I assigned gateway IP addresses to my ubuntu-srv network interfaces so it can act as a router between them and ufw can enforce the rules. Now I need to point my both systems: kali and ubuntu-lab, to gateway. For that:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105202203.png>)

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105202702.png>)

The `.0` address is the network address, which identifies the subnet. 

Because by default Linux will not pass traffic between its cards, I enable it on ubuntu-srv:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105202849.png>)

And I got a response, which means the command ran successfully.

To test if everything works, I will ping kali from ubuntu-lab:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105203534.png>)

And it was successful.

### Controlling the crossing with ufw

First I will get a real service to test with:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105203728.png>)

For that my first step is starting and enabling the SSH service on kali, as it is turned off there by default.

The default traffic flow is disabled. The server does not let anything flow through itself, besides the pings. To watch the traffic cross and then close it myself, I first open the gateway:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105204124.png>)

And I got the confirmation.

Now to see the traffic flow first:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105204618.png>)

but it timed out. A valuable lesson here: ufw lets ping through always, it is checked before the rules and never test a firewall with ping, but test real protocol and port.

Now, I will allow a specific crossing. For example, ubuntu-lab can reach kali ONLY over SSH, nothing else. On the gateway:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105205007.png>)

Now I will test it from ubuntu-lab, trying to connect to kali:

![PowerShell Get-LocalUser output](<Week5/Pasted image 20261105205149.png>)

Three commands: 2 using SSH port, one HTTP port. And the result was as expected: the SSH connection worked, and the HTTP port did not. 
