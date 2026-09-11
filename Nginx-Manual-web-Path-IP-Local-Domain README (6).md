# Nginx Static Website  on manual web path like /home/mywebsite/... Hosting on RHEL 10

## Project Objective

```text
Client / Browser
      |
      | HTTP :80
      v
   Nginx
      |
      | document root
      v
/home/Myweb/latestweb/
      |
      └── index.html
```

You learned an important real-world concept here: Nginx does not require the website to live in `/usr/share/nginx/html`. You can use a custom document root such as `/home/Myweb/latestweb`.

---

## 1. Environment

Example environment:
* **OS:** RHEL 10
* **Web:** Nginx
* **Protocol:** HTTP
* **Port:** 80
* **Website:** Static HTML
* **Document Root:** `/home/Myweb/latestweb`

Check your system:
```bash
cat /etc/redhat-release
```
Check Nginx:
```bash
nginx -v
```
Check service:
```bash
systemctl status nginx
```

---

## 2. Install Nginx

If Nginx is not installed:
```bash
sudo dnf install nginx -y
```
Enable it at boot:
```bash
sudo systemctl enable nginx
```
Start it:
```bash
sudo systemctl start nginx
```
Check:
```bash
sudo systemctl status nginx
```
You want: `Active: active (running)`

---

## 3. Understand the RHEL Nginx configuration structure

On your RHEL machine:
```text
/etc/nginx/
├── nginx.conf
├── conf.d/
├── default.d/
├── mime.types
└── ...
```
The important files are:
* `/etc/nginx/nginx.conf`: Main Nginx configuration.
* `/etc/nginx/conf.d/*.conf`: Additional configuration files.
* `/etc/nginx/default.d/*.conf`: Additional configuration included by the default server.

Unlike Ubuntu/Debian, you generally won't have:
* `/etc/nginx/sites-available/`
* `/etc/nginx/sites-enabled/`
*(Those are commonly used on Debian/Ubuntu.)*

---

## 4. Create your custom website directory

Create:
```bash
sudo mkdir -p /home/Myweb/latestweb
```
Check:
```bash
ls -ld /home/Myweb/latestweb
```

---

## 5. Create the website

Create:
```bash
sudo vi /home/Myweb/latestweb/index.html
```
For example:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Nginx Website</title>
</head>
<body>
    <h1>Welcome to My Website</h1>
    <p>This website is being served by Nginx.</p>
    <p>Document Root: /home/Myweb/latestweb</p>
</body>
</html>
```
Save it. Verify:
```bash
cat /home/Myweb/latestweb/index.html
```

---

## 6. Give Nginx permission to access the website

Because the website is under `/home`, Nginx needs permission to traverse the directories.
Run:
```bash
sudo chmod 755 /home
sudo chmod 755 /home/Myweb
sudo chmod 755 /home/Myweb/latestweb
sudo chmod 644 /home/Myweb/latestweb/index.html
```
Verify:
```bash
ls -ld /home
ls -ld /home/Myweb
ls -ld /home/Myweb/latestweb
ls -l /home/Myweb/latestweb/index.html
```
Expected permissions should look approximately like `drwxr-xr-x` for directories and `-rw-r--r--` for `index.html`.

---

## 7. Configure Nginx

On your final setup, `/etc/nginx/nginx.conf` contains the main server configuration.
The important part is:
```nginx
http {
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    types_hash_max_size 4096;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen 80;
        listen [::]:80;

        server_name _;

        root /home/Myweb/latestweb;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }

        include /etc/nginx/default.d/*.conf;
    }
}
```
The complete top-level structure must also contain:
```nginx
events {
    worker_connections 1024;
}
```
The two major blocks you must never accidentally delete are `events {}` and `http {}`.

---

## 8. Understand server_name _

This:
```nginx
server_name _;
```
is commonly used as a catch-all/default-style server name. It is useful for testing because you can access the server using `http://SERVER-IP`. You do not need `/etc/hosts` just to test it.
For example:
```bash
curl http://localhost/
```
Later, if you want `http://myweb.local`, then `/etc/hosts` can become relevant.

---

## 9. Why /etc/hosts is not required right now

`/etc/hosts` performs hostname resolution (e.g., `myweb.local → 192.168.1.50`). It does NOT tell Nginx where your website files are.

Nginx configuration (`root /home/Myweb/latestweb;`) tells Nginx: `Website files → /home/Myweb/latestweb`.

Remember:
* `/etc/hosts` → hostname → IP
* `nginx.conf` → HTTP request → website files

---

## 10. Check the configuration

Never immediately reload Nginx after editing its configuration.
First:
```bash
sudo nginx -t
```
Successful:
```text
syntax is ok
test is successful
```
Only then:
```bash
sudo systemctl reload nginx
```
This is a very important production habit: `EDIT` → `nginx -t` → if successful → `reload`.

---

## 11. Test locally

Test:
```bash
curl -i http://localhost/
```
Or:
```bash
curl -i http://localhost/index.html
```
Successful response:
```http
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
```
Check only the status:
```bash
curl -I http://localhost/
```

---

## 12. Check whether Nginx is listening on port 80

Run:
```bash
sudo ss -lntp | grep ':80'
```
Expected: `LISTEN ... 0.0.0.0:80 ... nginx`
This tells you that Nginx is listening.

---

## 13. Check the Nginx process

```bash
ps aux | grep nginx
```
You should see the master process and worker processes. Or run `systemctl status nginx`.

---

## 14. Check Nginx configuration actually loaded

This is one of the most useful commands:
```bash
sudo nginx -T
```
Unlike `nginx -t` which mainly validates the configuration, `nginx -T` dumps the effective configuration that Nginx has loaded.
Search for your document root:
```bash
sudo nginx -T | grep -A10 -B5 "root /home/Myweb/latestweb"
```

---

## 15. Check which configuration files are being included

Your configuration contains `include /etc/nginx/conf.d/*.conf;`.
Therefore check:
```bash
sudo ls -la /etc/nginx/conf.d/
sudo grep -RniE "server_name|listen|root" /etc/nginx/conf.d/
```
This is extremely useful when debugging conflicting server blocks.

---

## 16. The problem you actually encountered: conflicting server names

You previously had: `conflicting server name "_" on 0.0.0.0:80, ignored`
This happened because you had more than one configuration similar to `server_name _;` listening on port 80. Nginx then has competing server blocks.
Find them with:
```bash
sudo nginx -T 2>&1 | grep -A15 -B5 "server_name  _"
sudo grep -Rni "server_name" /etc/nginx/
```
For this simple project, keep one HTTP server block.

---

## 17. If you create Mywebsite.conf

You can alternatively keep your main `nginx.conf` relatively clean and put your website server block into `/etc/nginx/conf.d/Mywebsite.conf`:
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    root /home/Myweb/latestweb;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
If you use this approach, don't also keep another competing default server block in `nginx.conf`. You need one clear owner for port 80.

---

## 18. Important mistake you encountered: missing }

You got: `unexpected end of file, expecting "}"`
This means Nginx reached the end of the configuration while a block was still open.
Use:
```bash
sudo nginx -t
```
to detect this. If you want line numbers:
```bash
sudo nl -ba /etc/nginx/nginx.conf
```

---

## 19. Another error you encountered: no events section

You got: `no "events" section in configuration`
That happened because the events block was accidentally removed. A valid Nginx configuration needs:
```nginx
events {
    worker_connections 1024;
}
```

---

## 20. RHEL + SELinux

This is particularly important when hosting from `/home`.
Check:
```bash
getenforce
```
If `Enforcing`, Nginx may be prevented from reading files under your custom directory.
Install SELinux utilities if necessary:
```bash
sudo dnf install policycoreutils-python-utils -y
```
Assign the web-content SELinux context:
```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/home/Myweb/latestweb(/.*)?"
```
Apply it:
```bash
sudo restorecon -Rv /home/Myweb/latestweb
```
Check:
```bash
ls -Zd /home/Myweb/latestweb
ls -Z /home/Myweb/latestweb/index.html
```

---

## 21. If you get 403 Forbidden

If `curl -i http://localhost/` returns `HTTP/1.1 403 Forbidden`, check directory permissions:
```bash
ls -ld /home /home/Myweb /home/Myweb/latestweb
```
Then check `getenforce`. If SELinux is enforcing:
```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/home/Myweb/latestweb(/.*)?"
sudo restorecon -Rv /home/Myweb/latestweb
sudo systemctl reload nginx
```

---

## 22. If you get 404 Not Found

Check that the file exists:
```bash
ls -l /home/Myweb/latestweb/index.html
cat /home/Myweb/latestweb/index.html
```
Check the active root:
```bash
sudo nginx -T | grep -A10 -B5 "root /home/Myweb/latestweb"
```
Check logs:
```bash
sudo tail -f /var/log/nginx/error.log
```

---

## 23. If you get the old Nginx welcome page

This usually means Nginx is serving a different document root.
Check:
```bash
sudo nginx -T | grep -n "root "
```
If you see `root /usr/share/nginx/html;` and your intention is `/home/Myweb/latestweb`, you have another server configuration active. Find it:
```bash
sudo grep -Rni "root /usr/share/nginx/html" /etc/nginx/
```

---

## 24. If Nginx won't reload

Always run:
```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo journalctl -xeu nginx
sudo tail -50 /var/log/nginx/error.log
```
Do not repeatedly run `systemctl reload nginx` without checking the configuration first.

---

## 25. If Nginx isn't running

Check: `sudo systemctl status nginx`
Try: `sudo systemctl start nginx`
If it fails: `sudo nginx -t` and `sudo journalctl -xeu nginx`.

---

## 26. If port 80 is already occupied

Check: `sudo ss -lntp | grep ':80'`
You might see another service such as Apache/httpd.
Check: `sudo systemctl status httpd`
If you don't need Apache:
```bash
sudo systemctl stop httpd
sudo systemctl disable httpd
sudo systemctl start nginx
```

---

## 27. AWS EC2 troubleshooting

If this website is running on EC2 and `curl http://localhost/` works, but your browser cannot access `http://EC2-PUBLIC-IP`, then Nginx itself may be fine. Check AWS Security Group.
You need an inbound rule similar to:
* **Type:** HTTP
* **Protocol:** TCP
* **Port:** 80
* **Source:** your IP / `0.0.0.0/0` (temporary lab)

Also check `sudo ss -lntp | grep ':80'`. You want Nginx listening on `0.0.0.0:80`, not only `127.0.0.1:80`.

---

## 28. Complete rebuild procedure

If you want to reproduce this project from scratch later, use this sequence.

1. **Install:** `sudo dnf install nginx -y`
2. **Enable:** `sudo systemctl enable --now nginx`
3. **Create directory:** `sudo mkdir -p /home/Myweb/latestweb`
4. **Create HTML:** `sudo vi /home/Myweb/latestweb/index.html`
5. **Permissions:**
   ```bash
   sudo chmod 755 /home
   sudo chmod 755 /home/Myweb
   sudo chmod 755 /home/Myweb/latestweb
   sudo chmod 644 /home/Myweb/latestweb/index.html
   ```
6. **Configure Nginx:** `sudo vi /etc/nginx/nginx.conf`
   Ensure you have `events { worker_connections 1024; }` and your HTTP server listening on port 80 pointing to `root /home/Myweb/latestweb;`.
7. **Verify & Reload:**
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   ```
8. **SELinux (if Enforcing):**
   ```bash
   sudo dnf install policycoreutils-python-utils -y
   sudo semanage fcontext -a -t httpd_sys_content_t "/home/Myweb/latestweb(/.*)?"
   sudo restorecon -Rv /home/Myweb/latestweb
   ```
9. **Test:** `curl -i http://localhost/`

---

## 29. Final project architecture

Your final project can be represented as:

```text
                    Internet / Browser
                           |
                           | HTTP :80
                           |
                           v
                    +-------------+
                    |    Nginx    |
                    |  RHEL 10    |
                    +-------------+
                           |
                           | root
                           v
                /home/Myweb/latestweb/
                           |
                           +── index.html
                           +── css/
                           +── js/
                           +── images/
```
Nginx simply maps the requested URL to files under the configured root.

---

## 30. Commands you should memorize for interviews

* `systemctl status nginx` / `start` / `stop` / `restart` / `reload`
* `nginx -t`
* `nginx -T`
* `nginx -v`
* `ss -lntp | grep ':80'`
* `curl -I http://localhost/`
* `curl -i http://localhost/`
* `tail -f /var/log/nginx/error.log`
* `tail -f /var/log/nginx/access.log`
* `getenforce`
* `restorecon -Rv /home/Myweb/latestweb`
* `journalctl -xeu nginx`
* `grep -Rni "server_name" /etc/nginx/`
* `nginx -T | grep "root "`

---

## 31. The troubleshooting flow to memorize

When a website isn't working, don't randomly change things. Follow this order:

1. **Is Nginx running?** → `systemctl status nginx`
2. **Is configuration valid?** → `nginx -t`
3. **Is Nginx listening on port 80?** → `ss -lntp | grep ':80'`
4. **Can the server itself access the website?** → `curl -I http://localhost/`
5. **Is the correct document root active?** → `nginx -T | grep "root "`
6. **Does index.html actually exist?** → `ls -l /home/Myweb/latestweb/`
7. **Are Linux permissions correct?** → `ls -ld /home /home/Myweb /home/Myweb/latestweb`
8. **Is SELinux blocking access?** → `getenforce`
9. **What does Nginx's error log say?** → `tail -50 /var/log/nginx/error.log`
10. **If localhost works but external access doesn't:** → Check AWS Security Group / firewall / networking.

> **Key Lesson:** Troubleshoot from inside outward rather than changing multiple layers at once (application → Nginx → Linux permissions/SELinux → port/firewall → AWS networking).
