# Database Server Setup - RedHat EC2 with LVM

## Overview
Separate database server for WordPress with LVM configuration on RedHat.

## Step 1: Launch Database Server Instance

### AWS Console Setup:
1. **Launch EC2 Instance**
   - AMI: Red Hat Enterprise Linux
   - Instance Type: t3.micro (or preferred)
   - Name: `database-server`
   - Key Pair: Same as web server
   - Security Group: Create new `database-sg`

2. **Attach 3 EBS Volumes**
   - Create 3 x 10GB gp3 volumes
   - Attach to database server as /dev/sdb, /dev/sdc, /dev/sdd

3. **Security Group Rules**
   ```
   SSH: Port 22, Source: 0.0.0.0/0
   MySQL: Port 3306, Source: WEB_SERVER_IP/32
   ```

## Step 2: SSH and Initial Setup

```bash
# SSH into database server
ssh -i oregon-keypair.pem ec2-user@DATABASE_SERVER_IP

# Check attached volumes
lsblk
sudo fdisk -l
```

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

## Step 4: LVM Configuration

```bash
# Install LVM2
sudo yum install lvm2 -y

# Create physical volumes
sudo pvcreate /dev/xvdb1 /dev/xvdc1 /dev/xvdd1
sudo pvs

# Create volume group
sudo vgcreate database-vg /dev/xvdb1 /dev/xvdc1 /dev/xvdd1
sudo vgs

# Create logical volumes
sudo lvcreate -L 15G -n db-lv database-vg
sudo lvcreate -L 10G -n logs-lv database-vg
sudo lvs

# Verify LVM structure
lsblk
```

## Step 5: Create Filesystems and Mount Points

```bash
# Format logical volumes
sudo mkfs.ext4 /dev/database-vg/db-lv
sudo mkfs.ext4 /dev/database-vg/logs-lv

# Create directories
sudo mkdir -p /db
sudo mkdir -p /home/recovery/logs

# Mount logical volumes
sudo mount /dev/database-vg/db-lv /db
sudo mount /dev/database-vg/logs-lv /home/recovery/logs

# Verify mounts
df -h
```

## Step 6: Configure Permanent Mounts

```bash
# Get UUIDs
sudo blkid /dev/database-vg/db-lv
sudo blkid /dev/database-vg/logs-lv

# Edit fstab with UUIDs
sudo vi /etc/fstab

# Add these lines (replace with actual UUIDs):
# UUID=your-db-lv-uuid /db ext4 defaults 0 0
# UUID=your-logs-lv-uuid /home/recovery/logs ext4 defaults 0 0

# Test fstab
sudo mount -a
```

## Step 7: Install and Configure MySQL

```bash
# Update system
sudo yum update -y

# Install MySQL/MariaDB
sudo yum install mariadb-server -y

# Start and enable MariaDB
sudo systemctl start mariadb
sudo systemctl enable mariadb

# Secure MySQL installation
sudo mysql_secure_installation
```

## Step 8: Configure MySQL for Remote Access

```bash
# Stop MariaDB
sudo systemctl stop mariadb

# Move MySQL data to custom directory
sudo mkdir -p /db/mysql
sudo chown mysql:mysql /db/mysql

# Copy existing data
sudo cp -R /var/lib/mysql/* /db/mysql/
sudo chown -R mysql:mysql /db/mysql

# Configure MySQL to use new data directory
sudo vi /etc/my.cnf

# Add under [mysqld] section:
# datadir=/db/mysql
# socket=/db/mysql/mysql.sock

# Update socket path
sudo vi /etc/my.cnf.d/client.cnf

# Add under [client] section:
# socket=/db/mysql/mysql.sock

# Start MariaDB
sudo systemctl start mariadb
```

## Step 9: Create WordPress Database and User

```bash
# Access MySQL
sudo mysql -u root -p

# Create database and user
CREATE DATABASE wordpress;
CREATE USER 'wpuser'@'%' IDENTIFIED BY 'wppassword123';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'%';
FLUSH PRIVILEGES;
EXIT;
```

## Step 10: Configure MySQL for Remote Connections

```bash
# Edit MySQL config
sudo vi /etc/my.cnf

# Add under [mysqld] section:
# bind-address = 0.0.0.0

# Restart MariaDB
sudo systemctl restart mariadb

# Test connection
sudo netstat -tlnp | grep 3306
```

## Verification Commands

```bash
# Check LVM status
sudo pvs && sudo vgs && sudo lvs

# Check mounts
df -h

# Check MySQL status
sudo systemctl status mariadb

# Test database connection
mysql -u wpuser -p -h localhost wordpress
```

## Expected LVM Structure

```
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
xvda                    202:0    0   10G  0 disk 
├─xvda1                 202:1    0    1M  0 part 
├─xvda2                 202:2    0  200M  0 part /boot/efi
└─xvda3                 202:3    0  9.8G  0 part /
xvdb                    202:16   0   10G  0 disk 
└─xvdb1                 202:17   0   10G  0 part 
  └─database--vg-db--lv 253:0    0   15G  0 lvm  /db
xvdc                    202:32   0   10G  0 disk 
└─xvdc1                 202:33   0   10G  0 part 
  ├─database--vg-db--lv 253:0    0   15G  0 lvm  /db
  └─database--vg-logs--lv 253:1  0   10G  0 lvm  /home/recovery/logs
xvdd                    202:48   0   10G  0 disk 
└─xvdd1                 202:49   0   10G  0 part 
  └─database--vg-logs--lv 253:1  0   10G  0 lvm  /home/recovery/logs
```

## Next Steps
1. Configure web server with WordPress
2. Connect WordPress to remote database
3. Test the complete setup

## Security Notes
- Restrict MySQL access to web server IP only
- Use strong passwords
- Enable firewall rules
- Regular backups of /db directory