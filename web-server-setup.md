# Web Server Setup - RedHat EC2 with LVM Storage

## Overview
Web server for WordPress with LVM storage configuration on RedHat Enterprise Linux.

## Step 1: Launch Web Server Instance

### AWS Console Setup:
1. **Launch EC2 Instance**
   - AMI: Red Hat Enterprise Linux
   - Instance Type: t3.micro (or preferred)
   - Name: `web-server`
   - Key Pair: Create or use existing
   - Security Group: Create new `web-sg`

![Web Server Instance Launch](screenshots/web-server-launch.png)

2. **Attach 3 EBS Volumes**
   - Create 3 x 10GB gp3 volumes
   - Attach to web server as /dev/sdb, /dev/sdc, /dev/sdd

![EBS Volumes Attachment](screenshots/ebs-volumes-attach.png)

3. **Security Group Rules**
   ```
   SSH: Port 22, Source: 0.0.0.0/0
   HTTP: Port 80, Source: 0.0.0.0/0
   ```

![Security Group Configuration](screenshots/security-group-rules.png)

## Step 2: SSH and Initial Setup

```bash
# SSH into web server
ssh -i your-keypair.pem ec2-user@WEB_SERVER_IP

# Check attached volumes
lsblk
sudo fdisk -l
```

![Initial Volume Check](screenshots/initial-volume-check.png)

## Step 3: Partition Disks with fdisk

```bash
# Partition first disk
sudo fdisk /dev/xvdb
# Commands: n → p → 1 → Enter → Enter → t → 8e → w

# Partition second disk  
sudo fdisk /dev/xvdc
# Repeat same commands

# Partition third disk
sudo fdisk /dev/xvdd
# Repeat same commands

# Verify partitions
lsblk
```

![Disk Partitioning](screenshots/disk-partitioning.png)

## Step 4: LVM Configuration

```bash
# Install LVM2
sudo yum install lvm2 -y

# Create physical volumes
sudo pvcreate /dev/xvdb1 /dev/xvdc1 /dev/xvdd1
sudo pvs

# Create volume group
sudo vgcreate webdata-vg /dev/xvdb1 /dev/xvdc1 /dev/xvdd1
sudo vgs

# Create logical volumes
sudo lvcreate -L 14G -n apps-lv webdata-vg
sudo lvcreate -L 14G -n logs-lv webdata-vg
sudo lvs

# Verify LVM structure
lsblk
```

![LVM Configuration](screenshots/lvm-configuration.png)

## Step 5: Create Filesystems and Mount Points

```bash
# Format logical volumes
sudo mkfs.ext4 /dev/webdata-vg/apps-lv
sudo mkfs.ext4 /dev/webdata-vg/logs-lv

# Create directories
sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs

# Mount logical volumes
sudo mount /dev/webdata-vg/apps-lv /var/www/html
sudo mount /dev/webdata-vg/logs-lv /home/recovery/logs

# Verify mounts
df -h
```

![Filesystem Creation and Mounting](screenshots/filesystem-mounting.png)

## Step 6: Configure Permanent Mounts

```bash
# Get UUIDs
sudo blkid /dev/webdata-vg/apps-lv
sudo blkid /dev/webdata-vg/logs-lv

# Edit fstab with UUIDs
sudo vi /etc/fstab

# Add these lines (replace with actual UUIDs):
# UUID=your-apps-lv-uuid /var/www/html ext4 defaults 0 0
# UUID=your-logs-lv-uuid /home/recovery/logs ext4 defaults 0 0

# Test fstab
sudo mount -a
```

![Permanent Mount Configuration](screenshots/fstab-configuration.png)

## Step 7: Install Web Server Stack

```bash
# Update system
sudo yum update -y

# Install Apache, PHP, and MySQL client
sudo yum install httpd php php-mysqlnd mysql -y

# Start and enable Apache
sudo systemctl start httpd
sudo systemctl enable httpd

# Start and enable PHP-FPM
sudo systemctl start php-fpm
sudo systemctl enable php-fpm

# Verify services
sudo systemctl status httpd
sudo systemctl status php-fpm
```

![Web Stack Installation](screenshots/web-stack-install.png)

## Step 8: Install WordPress

```bash
# Download WordPress
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz

# Copy WordPress files
sudo cp -R wordpress/* /var/www/html/

# Set permissions
sudo chown -R apache:apache /var/www/html/
sudo chmod -R 755 /var/www/html/

# Create wp-config.php
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
```

![WordPress Installation](screenshots/wordpress-install.png)

## Step 9: Configure SELinux for Network Connections

```bash
# Check SELinux status
getenforce

# Allow Apache to make network connections
sudo setsebool -P httpd_can_network_connect on

# Verify setting
getsebool httpd_can_network_connect

# Restart services
sudo systemctl restart httpd
sudo systemctl restart php-fpm
```

![SELinux Configuration](screenshots/selinux-config.png)

## Step 10: Configure WordPress Database Connection

```bash
# Edit wp-config.php
sudo vi /var/www/html/wp-config.php

# Update database settings:
# define('DB_NAME', 'wordpress');
# define('DB_USER', 'wpuser');
# define('DB_PASSWORD', 'your_password');
# define('DB_HOST', 'DATABASE_SERVER_IP');
```

![WordPress Configuration](screenshots/wp-config-setup.png)

## Step 11: Test Installation

```bash
# Test Apache
sudo ss -tlnp | grep :80

# Test PHP
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
curl http://localhost/info.php

# Access WordPress
# Open browser: http://YOUR_WEB_SERVER_PUBLIC_IP
```

![Installation Testing](screenshots/installation-test.png)

## Final LVM Structure

```
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
xvda                      202:0    0   10G  0 disk 
├─xvda1                   202:1    0    1M  0 part 
├─xvda2                   202:2    0  200M  0 part /boot/efi
└─xvda3                   202:3    0  9.8G  0 part /
xvdb                      202:16   0   10G  0 disk 
└─xvdb1                   202:17   0   10G  0 part 
  ├─webdata--vg-apps--lv  253:0    0   14G  0 lvm  /var/www/html
  └─webdata--vg-logs--lv  253:1    0   14G  0 lvm  /home/recovery/logs
xvdc                      202:32   0   10G  0 disk 
└─xvdc1                   202:33   0   10G  0 part 
  ├─webdata--vg-apps--lv  253:0    0   14G  0 lvm  /var/www/html
  └─webdata--vg-logs--lv  253:1    0   14G  0 lvm  /home/recovery/logs
xvdd                      202:48   0   10G  0 disk 
└─xvdd1                   202:49   0   10G  0 part 
  └─webdata--vg-logs--lv  253:1    0   14G  0 lvm  /home/recovery/logs
```

![Final LVM Structure](screenshots/final-lvm-structure.png)

## WordPress Installation Success

![WordPress Installation Page](screenshots/wordpress-success.png)

## Troubleshooting

### Common Issues:
1. **Database Connection Error**: Check SELinux settings
2. **Permission Denied**: Verify file ownership and permissions
3. **Service Not Starting**: Check logs with `journalctl -u httpd`

### Verification Commands:
```bash
# Check LVM status
sudo pvs && sudo vgs && sudo lvs

# Check mounts
df -h

# Check services
sudo systemctl status httpd php-fpm

# Test connectivity
curl http://localhost
```

## Security Notes
- Keep system updated
- Use strong passwords
- Configure firewall rules
- Regular backups of /var/www/html
- Monitor logs in /home/recovery/logs

## Author
Peter Anyankpele

## License
This project is for educational purposes.