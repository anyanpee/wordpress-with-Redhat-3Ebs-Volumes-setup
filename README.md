# WordPress with LVM Setup - RedHat EC2

## Overview
WordPress installation on RedHat with 3 attached EBS volumes (10GB each) configured using LVM (Logical Volume Manager).

## Architecture
- **Server**: RedHat EC2 Instance
- **Storage**: 3 x 10GB EBS Volumes
- **LVM**: Physical Volumes → Volume Group → Logical Volumes
- **Application**: WordPress + Apache + MySQL/MariaDB + PHP

## Step 1: Create EC2 Instance with EBS Volumes

### 1.1 Launch EC2 Instance
1. Go to AWS Console → EC2 → Launch Instance
2. Choose **Red Hat Enterprise Linux** AMI
3. Select instance type (t3.micro recommended)
4. Configure instance details
5. **Add Storage**: This is where you attach the 3 additional EBS volumes

![EC2 Instance Launch]
![](<web-server-Screenshot 2025-09-03 021147.png>)

### 1.2 Add EBS Volumes During Instance Creation
1. In the "Add Storage" section:
   - Keep the root volume (default 10GB)
   - Click "Add New Volume" 3 times
   - Set each additional volume to 10GB
   - Volume type: gp3 (recommended)
   - Total: 4 volumes (1 root + 3 additional)

![EBS Volumes Attachment]
![](<adding-volumes-Screenshot 2025-09-03 021000.png>)

### 1.3 Configure Security Group
1. Create new security group or use existing
2. Add rules:
   - SSH (Port 22): Source 0.0.0.0/0
   - HTTP (Port 80): Source 0.0.0.0/0

![Security Group Configuration]
![](<security-group-Screenshot 2025-09-04 034537.png>)

### 1.4 Launch and Connect
1. Review and launch instance
2. Select your key pair
3. Wait for instance to be running
4. Connect via SSH

## Step 2: Initial Volume Setup

### 2.1 Check Current Disk Status
```bash
df -h
```
```bash
lsblk
```
```bash
sudo fdisk -l
```

![Initial Volume Check]
![](<database -logical-volume-Screenshot 2025-09-03 131206.png>)

### 2.2 Identify New Volumes
The 3 new volumes should appear as:
- `/dev/xvdf` (10GB)
- `/dev/xvdg` (10GB) 
- `/dev/xvdh` (10GB)

![Volume Identification]
![](<four-volumes-running-Screenshot 2025-09-03 025649.png>)

## Step 3: Disk Partitioning with gdisk

### 3.1 Create Partitions on Each Disk
**Partition first disk:**
```bash
sudo gdisk /dev/xvdf
```
**In gdisk, enter these commands:**
- `n` (new partition)
- `1` (partition number) 
- `Enter` (default first sector)
- `Enter` (default last sector)
- `8e00` (Linux LVM partition type)
- `w` (write changes)
- `y` (confirm)

**Partition second disk:**
```bash
sudo gdisk /dev/xvdg
```
*Repeat same gdisk commands as above*

**Partition third disk:**
```bash
sudo gdisk /dev/xvdh
```
*Repeat same gdisk commands as above*

### 3.2 Verify Created Partitions
```bash
lsblk
```

![Disk Partitioning Complete]
![](<databasee -ddisk-partitioningg-Screenshot 2025-09-03 130547.png>)

## Step 4: LVM Setup

### 4.1 Install LVM2 Package
```bash
sudo yum update -y
```
```bash
sudo yum install lvm2 -y
```
```bash
lvm version
```

### 4.2 Create Physical Volumes
```bash
sudo pvcreate /dev/xvdf1
```
```bash
sudo pvcreate /dev/xvdg1
```
```bash
sudo pvcreate /dev/xvdh1
```
**Verify physical volumes:**
```bash
sudo pvs
```

![Physical Volumes Created]
![](<creating -physical-volumes-on-3partition-Screenshot 2025-09-03 030242.png>)

### 4.3 Create Volume Group
```bash
sudo vgcreate wordpress-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1
```
**Verify volume group:**
```bash
sudo vgs
```

![Volume Group Created]
![](<creating volume-group-Screenshot 2025-09-03 030453.png>)

### 4.4 Create Logical Volumes
```bash
sudo lvcreate -L 15G -n wordpress-lv wordpress-vg
```
```bash
sudo lvcreate -L 10G -n database-lv wordpress-vg
```
**Verify logical volumes:**
```bash
sudo lvs
```

![Logical Volumes Created]
![](<logical-volumes-creation-Screenshot 2025-09-03 030816.png>)

## Step 5: Filesystem Creation and Mounting

### 5.1 Create Filesystems
```bash
sudo mkfs.ext4 /dev/wordpress-vg/wordpress-lv
```
```bash
sudo mkfs.ext4 /dev/wordpress-vg/database-lv
```
### 5.2 Create Mount Points
```bash
sudo mkdir -p /var/www/wordpress
```
```bash
sudo mkdir -p /var/lib/mysql-data
```

### 5.3 Mount Logical Volumes
```bash
sudo mount /dev/wordpress-vg/wordpress-lv /var/www/wordpress
```
```bash
sudo mount /dev/wordpress-vg/database-lv /var/lib/mysql-data
```
**Verify mounts:**
```bash
df -h
```

### 5.4 Configure Permanent Mounts
```bash
echo '/dev/wordpress-vg/wordpress-lv /var/www/wordpress ext4 defaults 0 0' | sudo tee -a /etc/fstab
```
```bash
echo '/dev/wordpress-vg/database-lv /var/lib/mysql-data ext4 defaults 0 0' | sudo tee -a /etc/fstab
```
**Test fstab configuration:**
```bash
sudo mount -a
```

## Verification Commands

### Check All Storage Configuration
**View mounted filesystems:**
```bash
df -h
```
**View block devices:**
```bash
lsblk
```
**View LVM configuration:**
```bash
sudo pvs
```
```bash
sudo vgs
```
```bash
sudo lvs
```

## ✅ COMPLETED - LVM Configuration Status

### LVM Summary:
- **Physical Volumes**: 3 x 10GB (/dev/xvdb1, /dev/xvdc1, /dev/xvdd1)
- **Volume Group**: wordpress-vg (~30GB total)
- **Logical Volumes**: 
  - wordpress-lv: 15GB (for WordPress files)
  - database-lv: 10GB (for MySQL data)
- **Remaining Space**: ~5GB available for expansion

## Next Steps - WordPress Installation

### 5. Create Filesystems and Mount Points
```bash
# Create ext4 filesystems
sudo mkfs.ext4 /dev/wordpress-vg/wordpress-lv
sudo mkfs.ext4 /dev/wordpress-vg/database-lv

# Create mount directories
sudo mkdir -p /var/www/wordpress
sudo mkdir -p /var/lib/mysql-data

# Mount logical volumes
sudo mount /dev/wordpress-vg/wordpress-lv /var/www/wordpress
sudo mount /dev/wordpress-vg/database-lv /var/lib/mysql-data

# Add to fstab for permanent mounting
echo '/dev/wordpress-vg/wordpress-lv /var/www/wordpress ext4 defaults 0 0' | sudo tee -a /etc/fstab
echo '/dev/wordpress-vg/database-lv /var/lib/mysql-data ext4 defaults 0 0' | sudo tee -a /etc/fstab

# Verify mounts
df -h
```

### 6. Install LAMP Stack
**Update system:**
```bash
sudo yum update -y
```
**Install Apache:**
```bash
sudo yum install httpd -y
```
```bash
sudo systemctl start httpd
```
```bash
sudo systemctl enable httpd
```
**Install MySQL/MariaDB:**
```bash
sudo yum install mariadb-server -y
```
```bash
sudo systemctl start mariadb
```
```bash
sudo systemctl enable mariadb
```
**Install PHP:**
```bash
sudo yum install php php-mysqlnd php-fpm -y
```
```bash
sudo systemctl restart httpd
```

![LAMP Stack Installation]

### 7. Configure MySQL Data Directory
```bash
# Stop MariaDB
sudo systemctl stop mariadb

# Copy existing data to new location
sudo cp -R /var/lib/mysql/* /var/lib/mysql-data/

# Set ownership
sudo chown -R mysql:mysql /var/lib/mysql-data

# Update MySQL config
sudo sed -i 's|datadir=/var/lib/mysql|datadir=/var/lib/mysql-data|' /etc/my.cnf

# Start MariaDB
sudo systemctl start mariadb
```

### 8. Download and Configure WordPress
**Download WordPress:**
```bash
cd /tmp
```
```bash
wget https://wordpress.org/latest.tar.gz
```
```bash
tar xzf latest.tar.gz
```
**Move to web directory:**
```bash
sudo cp -R wordpress/* /var/www/wordpress/
```
```bash
sudo chown -R apache:apache /var/www/wordpress
```
```bash
sudo chmod -R 755 /var/www/wordpress
```

![WordPress Installation]
![](<apache-active-Screenshot 2025-09-03 141246.png>)

## Final Result

![WordPress Success Page]
![](<wordpress-frontpage-Screenshot 2025-09-04 030141.png>)

## Troubleshooting

### Common Issues
- **Partition not found**: Check if volumes are properly attached to EC2
- **LVM commands fail**: Ensure LVM2 package is installed
- **Mount fails**: Verify filesystem creation and mount point existence

### Useful Commands
```bash
# Check volume group free space
sudo vgdisplay wordpress-vg

# Extend logical volume if needed
sudo lvextend -L +5G /dev/wordpress-vg/wordpress-lv
sudo resize2fs /dev/wordpress-vg/wordpress-lv

# Remove LVM configuration (if needed to start over)
sudo umount /var/www/wordpress /var/lib/mysql-data
sudo lvremove /dev/wordpress-vg/wordpress-lv
sudo lvremove /dev/wordpress-vg/database-lv
sudo vgremove wordpress-vg
sudo pvremove /dev/xvdf1 /dev/xvdg1 /dev/xvdh1
```

## Author
Peter Anyankpele

## License
This project is for educational purposes.