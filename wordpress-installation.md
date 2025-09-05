# WordPress Installation and Configuration

## Overview
Install WordPress on web server and configure it to connect to remote database server.

## Step 1: Install LAMP Stack on Web Server

```bash
# SSH into web server
ssh -i oregon-keypair.pem ec2-user@WEB_SERVER_IP

# Update system
sudo yum update -y

# Install Apache, PHP and extensions
sudo yum install httpd php php-mysqlnd php-fpm php-json php-curl php-mbstring php-xml php-zip -y

# Start and enable Apache
sudo systemctl start httpd
sudo systemctl enable httpd

# Start and enable PHP-FPM
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
```

## Step 2: Download and Install WordPress

```bash
# Download WordPress
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar xzf latest.tar.gz

# Copy WordPress files to web directory
sudo cp -R wordpress/* /var/www/html/
sudo rm /var/www/html/test.txt

# Set ownership and permissions
sudo chown -R apache:apache /var/www/html/
sudo chmod -R 755 /var/www/html/
```

## Step 3: Configure WordPress Database Connection

```bash
# Create WordPress config file
cd /var/www/html
sudo cp wp-config-sample.php wp-config.php

# Edit WordPress configuration
sudo vi wp-config.php
```

### Update these lines in wp-config.php:
```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'wpuser');
define('DB_PASSWORD', 'wppassword123');
define('DB_HOST', 'DATABASE_SERVER_PRIVATE_IP');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');
```

## Step 4: Configure Security Groups

### Web Server Security Group:
```
SSH: Port 22, Source: 0.0.0.0/0
HTTP: Port 80, Source: 0.0.0.0/0
HTTPS: Port 443, Source: 0.0.0.0/0
```

### Database Server Security Group:
```
SSH: Port 22, Source: 0.0.0.0/0
MySQL: Port 3306, Source: WEB_SERVER_PRIVATE_IP/32
```

## Step 5: Test Database Connection

```bash
# Install MySQL client on web server
sudo yum install mysql -y

# Test connection to database server
mysql -u wpuser -p -h DATABASE_SERVER_PRIVATE_IP wordpress

# If successful, you should see MySQL prompt
# Type 'exit' to quit
```

## Step 6: Configure Apache Virtual Host

```bash
# Create WordPress virtual host
sudo vi /etc/httpd/conf.d/wordpress.conf
```

### Add this configuration:
```apache
<VirtualHost *:80>
    ServerName your-domain.com
    DocumentRoot /var/www/html
    
    <Directory /var/www/html>
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog /var/log/httpd/wordpress_error.log
    CustomLog /var/log/httpd/wordpress_access.log combined
</VirtualHost>
```

```bash
# Test Apache configuration
sudo httpd -t

# Restart Apache
sudo systemctl restart httpd
```

## Step 7: Complete WordPress Setup

1. **Open browser and navigate to**: `http://WEB_SERVER_PUBLIC_IP`
2. **WordPress Installation Wizard will appear**
3. **Fill in the details**:
   - Site Title: Your WordPress Site
   - Username: admin
   - Password: Strong password
   - Email: your-email@domain.com
4. **Click "Install WordPress"**

## Step 8: Verify Installation

```bash
# Check Apache status
sudo systemctl status httpd

# Check PHP-FPM status
sudo systemctl status php-fpm

# Check WordPress files
ls -la /var/www/html/

# Check Apache logs
sudo tail -f /var/log/httpd/wordpress_access.log
sudo tail -f /var/log/httpd/wordpress_error.log
```

## Step 9: Test Database Connection from WordPress

```bash
# On database server, check MySQL connections
sudo mysql -u root -p

# Show active connections
SHOW PROCESSLIST;

# Check WordPress database
USE wordpress;
SHOW TABLES;
EXIT;
```

## Troubleshooting

### Common Issues:

1. **Connection Refused**:
   ```bash
   # Check if MySQL is running on database server
   sudo systemctl status mariadb
   
   # Check if port 3306 is listening
   sudo netstat -tlnp | grep 3306
   ```

2. **Permission Denied**:
   ```bash
   # Fix WordPress file permissions
   sudo chown -R apache:apache /var/www/html/
   sudo chmod -R 755 /var/www/html/
   ```

3. **Database Connection Error**:
   ```bash
   # Verify MySQL user and permissions
   mysql -u root -p
   SELECT User, Host FROM mysql.user WHERE User='wpuser';
   SHOW GRANTS FOR 'wpuser'@'%';
   ```

4. **SELinux Issues** (if enabled):
   ```bash
   # Allow Apache to connect to network
   sudo setsebool -P httpd_can_network_connect 1
   
   # Check SELinux status
   getenforce
   ```

## Security Hardening

```bash
# Remove default Apache page
sudo rm -f /etc/httpd/conf.d/welcome.conf

# Hide Apache version
echo "ServerTokens Prod" | sudo tee -a /etc/httpd/conf/httpd.conf
echo "ServerSignature Off" | sudo tee -a /etc/httpd/conf/httpd.conf

# Restart Apache
sudo systemctl restart httpd

# Set up firewall (optional)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

## Final Architecture

```
┌─────────────────┐    HTTP/HTTPS    ┌─────────────────┐
│   Internet      │◄─────────────────┤   Web Server    │
│   Users         │                  │   (WordPress)   │
└─────────────────┘                  │   - Apache      │
                                     │   - PHP         │
                                     │   - LVM Storage │
                                     └─────────┬───────┘
                                               │
                                               │ MySQL
                                               │ Port 3306
                                               │
                                     ┌─────────▼───────┐
                                     │  Database       │
                                     │  Server         │
                                     │  - MariaDB      │
                                     │  - LVM Storage  │
                                     │  - /db mount    │
                                     └─────────────────┘
```

## Backup Strategy

```bash
# Database backup (run on database server)
mysqldump -u root -p wordpress > /home/recovery/logs/wordpress_backup_$(date +%Y%m%d).sql

# WordPress files backup (run on web server)
sudo tar -czf /home/recovery/logs/wordpress_files_$(date +%Y%m%d).tar.gz /var/www/html/
```

## Performance Optimization

```bash
# Install PHP OPcache
sudo yum install php-opcache -y

# Restart services
sudo systemctl restart php-fpm httpd
```

This completes the WordPress installation with remote database configuration!