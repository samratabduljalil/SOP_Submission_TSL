###  NGNIX Full Setup on Local Rocky Linux VMs (No External DNS)

---

## 📋 VM Layout

| Role | Hostname | IP |
|------|----------|----|
| Load Balancer | `lb` | `192.168.1.10` |
| Web Server 1 | `web1` | `192.168.1.11` |
| Web Server 2 | `web2` | `192.168.1.12` |

> ⚠️ Replace IPs with your actual VM IPs. Run `ip addr` on each VM to find them.



---

## STEP 1 — Find Your VM IPs (on each VM)

```bash
ip addr show | grep "inet "
```

---

## STEP 2 — Install Nginx on ALL 3 VMs

```bash
sudo dnf install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

Open firewall on all 3 VMs:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

Disable SELinux on all 3 VMs:

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

---

## STEP 3 — Configure web1 (run on web1 only)

### Create HTML page

```bash
sudo mkdir -p /var/www/html
sudo chown -R nginx:nginx /var/www/html

nano /var/www/html/index.html 
<!DOCTYPE html>
<html>
<body>
  <h1>Hello from web1 !!!</h1>
</body>
</html>



```

### Create Nginx config

```bash
nano /etc/nginx/conf.d/web1.conf 
server {
    listen 80;
    server_name web2 192.168.1.12 localhost;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

```

### Remove default config & reload

```bash
sudo rm -f /etc/nginx/conf.d/default.conf
sudo rm -f /usr/share/nginx/html/index.html
sudo nginx -t
sudo systemctl reload nginx
```

### Quick test from web1 itself

```bash
curl http://192.168.1.12
# Expected: Hello from web1 !!!
```

---

## STEP 4 — Configure web2 (run on web2 only)

### Create HTML page

```bash
sudo mkdir -p /var/www/html
sudo chown -R nginx:nginx /var/www/html
nano /var/www/html/index.html 
<!DOCTYPE html>
<html>
<body>
  <h1>Hello from web2 !!!</h1>
</body>
</html>

```

### Create Nginx config

```bash
nano /etc/nginx/conf.d/web2.conf 
server {
    listen 80;
    server_name web2 192.168.1.13 localhost;
    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

```

### Remove default config & reload

```bash
sudo rm -f /etc/nginx/conf.d/default.conf
sudo rm -f /usr/share/nginx/html/index.html
sudo nginx -t
sudo systemctl reload nginx
```

### Quick test from web2 itself

```bash
curl http://192.168.1.13
# Expected: Hello from web2 !!!
```

---

## STEP 5 — Configure Load Balancer (run on lb only)

### Create Load Balancer config

```bash
nano /etc/nginx/conf.d/lb.conf 
upstream backend_pool {
    server 192.168.1.11:80;   # web1
    server 192.168.1.12:80;   # web2
}

server {
    listen 80;
    server_name www.my.com my.com;

    location / {
        proxy_pass         http://backend_pool;
        proxy_http_version 1.1;

        proxy_set_header   Host             $host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
    }
}

```

### Remove default config & reload

```bash
sudo rm -f /etc/nginx/conf.d/default.conf
sudo nginx -t
sudo systemctl reload nginx
```

---

## STEP 6 — Edit /etc/hosts on ALL 3 VMs

This makes `www.my.com` resolve to your load balancer IP **without any DNS server**.

Run this on **lb**, **web1**, and **web2**:

```bash
echo "192.168.1.10  www.my.com my.com" | sudo tee -a /etc/hosts
```

Verify:

```bash
grep "my.com" /etc/hosts
```

---

## STEP 7 — Test: curl www.my.com

Run from **any of the 3 VMs**:

```bash
curl http://www.my.com
```

Run multiple times to see round-robin:

```bash
for i in {1..6}; do curl -s http://www.my.com; done
```

Expected output (alternates each request):

```
<h1>Hello from web1 !!!</h1>
<h1>Hello from web2 !!!</h1>
<h1>Hello from web1 !!!</h1>
<h1>Hello from web2 !!!</h1>
...
```

---


---

## Troubleshooting

### Check if Nginx is running

```bash
sudo systemctl status nginx
```

### Check config syntax

```bash
sudo nginx -t
```

### Check if port 80 is listening

```bash
sudo ss -tlnp | grep :80
```

### Test backends directly (from lb VM)

```bash
curl http://192.168.1.11   # Should show web1 page
curl http://192.168.1.12   # Should show web2 page
```

### Check firewall

```bash
sudo firewall-cmd --list-services   # Should include: http
```

### View live logs on lb

```bash
sudo tail -f /var/log/nginx/access.log
```

---

