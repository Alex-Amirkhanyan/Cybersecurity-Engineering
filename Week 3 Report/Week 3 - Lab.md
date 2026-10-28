
# Part 1 - Enumerate the system

The first Exercise is to enumerate the system. In order to do that, I launch the PowerShell and first run the command `Get-LocalUser`, which lists all the local user accounts that exist on the system: 

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 095909.png>)

Now to list all the local groups, I run the command `Get-LocalGroup`, which gives the following result:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 100036.png>)

Next, let's check which local accounts exist within the **Administrators** group. For that I run  `Get-LocalGroupMember -Group Administrators`, which gives the following result:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 100334 1.png>)

Now, to see the services and then the processes, I run two commands: `Get-Service | Select -First 10` and `Get-Process | Select -First 10`. I used `| Select -First 10` in order to keep the output short. The outputs are these:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 100559.png>)

Finally let's check which folders and files this machine shares. For that I run `Get-SmbShare`, which gives the following result:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 103054 1.png>)

And as we know these are the default shares.

# Part 2 - Become the attacker and find yourself

First, let's open PowerShell in administrator mode, which triggers the UAC, and then create an account with the command `net user backup_svh Passw0rd! /add`. I chose the name *back_svh* so it can blend in and not look suspicious at all:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 103839.png>)

Then I add the account to the **Administrators**, with `net localgroup Administrators backup_svh /add`:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 104141.png>)

Now to be sure, let's check that everything worked:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 111339 1.png>)

As we see, it worked, and the account that we created now belongs to the **Administrators** group.

Now for the hunting part, let's check the logs. There should be several relevant events: 4720 (a user account was created) and 4732 (a member was added to a security-enabled local group). The second one is the dangerous part, as the account got administrator privileges.

Event 4732 in Event Viewer does not show the account name, only the Security ID. We can find the name ourselves by matching Security IDs, so we check the 4720 and 4732 events with the same Security ID:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 115918 1.png>)

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 120150 1.png>)

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 120529 1.png>)

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-25 120938 1.png>)

As we can see, they match. Now let's undo the attack, in other words, clean up. For that we run these commands:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-26 094003.png>)

The first one removes the *backup_svh* account from the **Administrators** group, and the second one deletes the user account completely. Then, to be sure everything worked, correctly, I checked one last time:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-26 094337.png>)

# Part 3 - Logon types in real events

With Event Viewer already open, I clicked the Filter Current Log button and typed the event ID 4324, which as we know, is for successful logons. My first logon shows Type 2, which indicates a physical (interactive) logon to the system. There are also several Type 5 logons a few minutes into using the system, which means a background service started up using an assigned service account. In most cases it is either the antivirus or SQL services:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-26 094700.png>)

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-26 094711.png>)

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-26 094729.png>)

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-26 094739.png>)

# Part 4 - One safe registry edit

As we know, the registry is a hierarchical, tree-like database that stores configuration data and settings for the operating system, hardware installed programs, and user preferences. Most importantly, the registry has no undo option, so we have to be careful.

To open it, I hit the Windows icon, type `regedit`, and press Run as administrator. The UAC prompt appears and asks for permission to open the program, so all we need to do is hit Yes. After that, a new windows opens, which is the registry itself. In the address bar we write `HKEY_CURRENT_USER\Control Panel\Desktop`, as it affects only the current user and not the whole system:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-22 194516 1.png>)

Here we create a new string value with right click -> New -> String Value and name it LabTest. As we can see, the Data column is empty becuase the value contains no data yet. We double-click it, type "hello", as the value data, and save it. Now the Data column is no longer empty:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-27 134624.png>)

To delete this value, we right-click it and hit Delete. A warning windows appears:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-22 194847.png>)

and after hitting "Yes" the value is deleted.


# Part 5 - Own investigation script

To write the script, I opened Notepad. I started with the basic commands: printing a welcome message and showing the accounts in the Administrators group:

![PowerShell Get-LocalUser output](<Week3/Pasted image 20261027153256.png>)

Next, as my task required, I needed to print the events with ID 4625 (Failed logons). So I added this second command to my script, which filters the Security log and keeps only the events with ID 4625. The default output does not show the details I need, so I convert each event to XML and take out four field: the account name, the logon type, the IP address and the status code. I show the IP address only for logon type 10 (remote logon), and for the other types it prints N/A. The status code is a hex number, so i used a switch to translate the most common ones into a readable failure reason. At the end, the results are printed as a table:

![PowerShell Get-LocalUser output](<Week3/Pasted image 20261027153613.png>)

And finally I added a command that counts all the events with ID 4625, and prints the total number of failed logon attempts:

![PowerShell Get-LocalUser output](<Week3/Pasted image 20261027153716.png>)

Then I saved the files as hunt.ps1 on the Desktop, moved to the Desktop in PowerShell, and tried to run it. It threw an error saying that running scripts is disabled:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-22 205528.png>)

so to change that I ran `Set-ExecutionPolicy RemoteSigned`:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-22 205921.png>)

It printed a question to be answered, and nothing else after that which means the command execution was successful. Now let's run it again:

![PowerShell Get-LocalUser output](<Week3/Screenshot 2026-10-27 154232.png>)

And we got the results we needed.
