# Apache & Local DNS Setup on Rocky Linux
> After completing this guide, typing `my.com` in your browser will open your local website.

---

## Prerequisites

```bash
# Switch to root
sudo su -

# Check your Rocky Linux version
cat /etc/rocky-release
```

---

## Step 1 — Install Apache (httpd)

```bash
# Install Apache
dnf install httpd -y

# Start and enable Apache on boot
systemctl start httpd
systemctl enable httpd

# Verify Apache is running
systemctl status httpd
```

---

## Step 2 — Configure Firewall

```bash
# Allow HTTP and HTTPS traffic
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload

# Confirm rules are active
firewall-cmd --list-all
```

---

## Step 3 — Create Your Website Directory & Files

```bash
# Create the web root directory for my.com
mkdir -p /var/www/my.com/html

# Set correct ownership
chown -R apache:apache /var/www/my.com/html

# Set correct permissions
chmod -R 755 /var/www/my.com

# Create a simple index page
cat > /var/www/my.com/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Welcome to my.com</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 80px; background: #f0f4f8; }
        h1   { color: #2c3e50; }
        p    { color: #555; }
    </style>
</head>
<body>
    <h1>🎉 Welcome to my.com!</h1>
    <p>Your local Apache server is working correctly.</p>
</body>
</html>
EOF
```

---

## Step 4 — Create Apache Virtual Host Config

```bash
# Create the virtual host config file
cat > /etc/httpd/conf.d/my.com.conf << 'EOF'
<VirtualHost *:80>
    ServerName   my.com
    ServerAlias  www.my.com
    DocumentRoot /var/www/my.com/html

    <Directory /var/www/my.com/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog  /var/log/httpd/my.com_error.log
    CustomLog /var/log/httpd/my.com_access.log combined
</VirtualHost>
EOF

# Test Apache config for syntax errors
apachectl configtest

# Reload Apache to apply changes
systemctl reload httpd
```

---

## Step 5 — Configure Local DNS (hosts file)

This makes `my.com` resolve to your local machine without a real DNS server.

```bash
# Check your local IP address (use the one shown, e.g., 192.168.x.x or 127.0.0.1)
hostname -I

# Add my.com to /etc/hosts (using 127.0.0.1 for same machine)
echo "127.0.0.1   my.com www.my.com" >> /etc/hosts

# Verify the entry was added
grep "my.com" /etc/hosts
```

> **Note:** If you want OTHER machines on your network to reach this site,
> replace `127.0.0.1` with your server's actual LAN IP (e.g., `192.168.1.100`).

---

## Step 6 — SELinux Configuration

Rocky Linux has SELinux enabled by default. Allow Apache to serve your web files:

```bash
# Apply the correct SELinux context to your web directory
restorecon -Rv /var/www/my.com/html

# If you moved files from another location, set context manually
chcon -Rt httpd_sys_content_t /var/www/my.com/html

# (Optional) Check current SELinux status
getenforce
```

---

## Step 7 — Test Your Setup

```bash
# Test with curl from the same machine
curl http://my.com

# Or open in your browser
# → http://my.com
```

You should see the HTML page you created in Step 3.

---

## Useful Commands (Quick Reference)

| Task | Command |
|------|---------|
| Restart Apache | `systemctl restart httpd` |
| Reload config (no downtime) | `systemctl reload httpd` |
| Check Apache status | `systemctl status httpd` |
| Test config syntax | `apachectl configtest` |
| View error logs | `tail -f /var/log/httpd/my.com_error.log` |
| View access logs | `tail -f /var/log/httpd/my.com_access.log` |
| List virtual hosts | `apachectl -S` |

---

## Troubleshooting

```bash
# Apache not starting? Check for port conflicts
ss -tlnp | grep :80

# SELinux blocking access?
ausearch -c 'httpd' --raw | audit2allow -M mypol
semodule -i mypol.pp

# DNS not resolving? Verify hosts file
cat /etc/hosts | grep my.com

# Firewall blocking?
firewall-cmd --list-services
```

---

## Summary of What Was Configured

```
Browser: http://my.com
        ↓
/etc/hosts  →  resolves my.com to 127.0.0.1
        ↓
Apache (httpd)  →  reads /etc/httpd/conf.d/my.com.conf
        ↓
Serves files from  →  /var/www/my.com/html/index.html
```
