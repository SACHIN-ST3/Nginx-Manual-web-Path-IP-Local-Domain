# LAB 2  Nginx setup: Server IP 192.168.x.x to  Local domain  — IP Address → and Local Domain Name using Windows hosts

## Goal
```text
Windows PC
    |
    |  myweb34.test
    ↓
Windows hosts file
    |
    | 54.227.183.187
    ↓
AWS EC2
    |
    ↓
Nginx
    |
    ↓
/home/Myweb/latestweb/index.html
```
This is a local DNS/hosts-file testing lab. You do not need to purchase a domain.
We will use:
* **Domain:** `myweb34.test`
* **Server IP:** `54.227.183.187`
* **Port:** `80`
* **Web server:** Nginx
* **OS client:** Windows

> **Note:** `.test` is a good choice because it is intended for testing.

---

## Step 1 — Verify your Nginx Server EC2 public IP
On your RHEL EC2 server:
```bash
curl ifconfig.me
```
You currently have:
`54.227.183.187`

You can also run:
```bash
hostname -I
```
*Be careful:* `hostname -I` will show the private IP, not the public IP.
For this lab, Windows needs to point to:
`54.227.183.187`

---

## Step 2 — Verify Nginx is running
On RHEL:
```bash
sudo systemctl status nginx
```
You want:
`Active: active (running)`

If it isn't running:
```bash
sudo systemctl start nginx
```
Enable it at boot:
```bash
sudo systemctl enable nginx
```

---

## Step 3 — Verify Nginx configuration
Run:
```bash
sudo nginx -t
```
Expected:
```text
syntax is ok
test is successful
```
Then:
```bash
sudo nginx -T | grep -A10 -B5 "server_name"
```
Your server block should contain: #myweb34.test
```nginx
server {
    listen 80;
    listen [::]:80;

    server_name myweb34.test;

    root /home/Myweb/latestweb;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
If you changed the configuration, reload:
```bash
sudo systemctl reload nginx
```

---

## Step 4 — Test the website directly using the EC2 IP
Before involving the Windows hosts file, verify that the website works using the IP.
From Windows PowerShell:
```powershell
curl.exe -I http://54.227.183.187
```
Expected:
```text
HTTP/1.1 200 OK
Server: nginx/1.26.3
```
You can also open `http://54.227.183.187` in Chrome.

*If this does not work, stop here. Fix the EC2/Nginx/AWS networking first.*

---

## Step 5 — Understand what /etc/hosts means
On Linux, the hosts file is: `/etc/hosts`
For example:
`54.227.183.187    myweb34.test`

But Windows does not use `/etc/hosts`.
Windows uses: `C:\Windows\System32\drivers\etc\hosts`

So:
* **Linux:** `/etc/hosts`
* **Windows:** `C:\Windows\System32\drivers\etc\hosts`

*This distinction is very important for interviews.*

---

## Step 6 — Open Windows hosts file correctly
Do not simply double-click the file. We need Administrator privileges.
1. Press: **Windows Key**
2. Search: **Notepad**
3. Right-click: **Notepad → Run as administrator**
4. Click: **Yes** when Windows asks for administrator permission.

---

## Step 7 — Open the Windows hosts file
Inside Administrator Notepad:
1. Go to: **File → Open**
2. Go to: `C:\Windows\System32\drivers\etc`
3. At the bottom-right, change: **Text Documents (*.txt)** to **All Files (*.*)**
4. Now you should see: `hosts`
5. Open: `hosts`

---

## Step 8 — Add the IP → domain mapping
At the bottom of the file add:
```text
54.227.183.187    myweb34.test
```
For example:
```text
# Loopback entries; do not change.
127.0.0.1       localhost
::1             localhost

54.227.183.187  myweb34.test
```
Meaning: Whenever this Windows computer asks for `myweb34.test`, send the request to `54.227.183.187`.

---

## Step 9 — Save the hosts file
Press: `Ctrl + S`

Because Notepad was opened as Administrator, it should save successfully.
**Important:** Make sure you do not create `hosts.txt`. The file must remain `hosts`.

---

## Step 10 — Verify the hosts file
Open PowerShell:
```powershell
Get-Content C:\Windows\System32\drivers\etc\hosts
```
At the bottom you should see:
`54.227.183.187  myweb34.test`

You can also check the file itself:
```powershell
Get-Item C:\Windows\System32\drivers\etc\hosts
```

---

## Step 11 — Flush Windows DNS cache
Run:
```powershell
ipconfig /flushdns
```
Expected:
```text
Windows IP Configuration
Successfully flushed the DNS Resolver Cache.
```
This makes the testing cleaner, although the hosts-file lookup itself is not dependent on your normal DNS server.

---

## Step 12 — Test hostname resolution
Run:
```powershell
ping myweb34.test
```
You should see something similar to:
```text
Pinging myweb34.test [54.227.183.187] with 32 bytes of data:
```
The ping itself might time out because ICMP can be blocked by AWS. That's okay. We are primarily checking whether `myweb34.test → 54.227.183.187` works.

---

## Step 13 — Test using PowerShell HTTP
This is more useful than ping.
Run:
```powershell
curl.exe -I http://myweb34.test
```
Expected:
```text
HTTP/1.1 200 OK
Server: nginx/1.26.3
```
Now we have proven: `myweb34.test → 54.227.183.187 → AWS EC2 → Port 80 → Nginx → Website`.

---

## Step 14 — Open the fake domain in Chrome
Open: `http://myweb34.test`

You should see your website.
Notice that you are no longer typing `http://54.227.183.187`. You are using `http://myweb34.test`, but underneath, Windows is resolving it to `54.227.183.187`.

---

## Step 15 — Test Nginx server_name
Your Nginx configuration contains:
`server_name myweb34.test;`

When the browser sends:
```http
GET / HTTP/1.1
Host: myweb34.test
```
Nginx sees `Host: myweb34.test` and selects the server block for that name, then serves `/home/Myweb/latestweb/index.html`.

**Complete Request Flow:**
```text
Chrome
  | http://myweb34.test
  ↓
Windows hosts file
  | 54.227.183.187
  ↓
Internet
  ↓
AWS EC2
  | TCP/80
  ↓
Nginx
  | server_name myweb34.test
  ↓
root /home/Myweb/latestweb
  ↓
index.html
```

---

## Step 16 — Verify from the EC2 itself
On RHEL, run:
```bash
curl -H "Host: myweb34.test" http://127.0.0.1/
```
This is an excellent Nginx test telling Nginx to pretend the request came for `myweb34.test`. You should receive your website HTML.

You can also run:
```bash
curl -I -H "Host: myweb34.test" http://127.0.0.1/
```
Expected: `HTTP/1.1 200 OK`

---

## Step 17 — Check the Nginx access log
On RHEL:
```bash
sudo tail -f /var/log/nginx/access.log
```
Then open `http://myweb34.test` in Chrome. You should see a request appear in the log.
Press `Ctrl + C` to stop watching the log.

---

## Step 18 — Troubleshooting checklist
If this doesn't work, check in this exact order:

**Problem 1 — Domain cannot be found**
Windows:
```powershell
Get-Content C:\Windows\System32\drivers\etc\hosts
```
Make sure `54.227.183.187 myweb34.test` exists.
Then run `ipconfig /flushdns` and `ping myweb34.test`.

**Problem 2 — ping says "could not find host"**
Check whether Windows sees the hosts entry:
```powershell
Get-Content C:\Windows\System32\drivers\etc\hosts | Select-String "myweb34"
```
If nothing appears, your hosts file is wrong or the entry wasn't saved.

**Problem 3 — Domain resolves but website doesn't open**
Test:
```powershell
curl.exe -I http://54.227.183.187
curl.exe -I http://myweb34.test
```
If IP works but domain doesn't, check Nginx:
```bash
sudo nginx -t
sudo nginx -T | grep -A10 "server_name myweb34.test"
```

**Problem 4 — Nginx isn't running**
```bash
sudo systemctl status nginx
sudo systemctl start nginx
```

**Problem 5 — AWS blocks port 80**
On RHEL:
```bash
sudo ss -lntp | grep ':80'
```
You want Nginx listening on port 80.
Check your AWS EC2 Security Group. Inbound should allow:
* **Type:** HTTP
* **Protocol:** TCP
* **Port:** 80
* **Source:** Your IP (or `0.0.0.0/0` temporarily)

---

## Step 19 — Important nslookup point
Do not use this as your primary test:
```powershell
nslookup myweb34.test
```
You may get: `*** UnKnown can't find myweb34.test: Non-existent domain`
That does not necessarily mean your hosts file is broken. `nslookup` primarily queries a DNS server. Your hosts file is a local hostname-resolution mechanism. Use `ping myweb34.test` or `curl.exe -I http://myweb34.test` instead.

---

## Step 20 — The difference between a real domain and this lab
This lab (`myweb34.test → Windows hosts → 54.227.183.187`) works only on computers where you manually add the hosts entry.

A real production setup would be:
`mycompany.com → DNS → 54.x.x.x → AWS → Nginx`

For DevOps learning, this lab teaches the relationship between IP addresses, DNS, hosts files, hostnames, domain names, Nginx server_name, and HTTP Host headers.

---

## Final Lab Verification
When everything is complete, these should work:

1. Windows:
   ```powershell
   Get-Content C:\Windows\System32\drivers\etc\hosts
   ```
   Shows: `54.227.183.187  myweb34.test`

2. Then:
   ```powershell
   ipconfig /flushdns
   ping myweb34.test
   ```
   Should resolve to: `54.227.183.187`

3. Then:
   ```powershell
   curl.exe -I http://myweb34.test
   ```
   Should return: `HTTP/1.1 200 OK`

4. And finally Chrome:
   Open `http://myweb34.test`

**Lab completed.**

> **DevOps note:** Your EC2 public IP can change after a stop/start unless you use an Elastic IP. For a permanent lab hostname, using an Elastic IP is the next logical step.
