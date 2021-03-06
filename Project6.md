## WEB SOLUTION WITH WORDPRESS
### Part 1: Configure storage subsystem for Web and Database servers based on Linux OS.
#### Steps
1. Launch an EC2 instance that will serve as "Web Server". Create 3 volumes in the same AZ as your Web Server EC2, each of 10 GiB.
2. Attach all three volumes one by one to your Web Server EC2 instance
3. Open up the Linux terminal to begin configuration of the instance. Use `lsblk` command to inspect what block devices are attached to the server. The 3 newly
created block devices are names **xvdf, xvdh, xvdg**
4. Use `df -h` command to see all mounts and free space on your server
5. Use gdisk utility to create a single partition on each of the 3 disks `sudo gdisk /dev/xvdf`
6. A prompt pops up, type **n** to create new partition, enter no of partition(1), hex code is **8e00**, **p** to view partition and **w** to save newly created partition.
7. Repeat this process for the other remaining block devices.
8. Type **lsblk** to view newly created partition.
9. Install **lvm2** package by typing: sudo yum install lvm2. Run `sudo lvmdiskscan` command to check for available partitions.
10. Create physical volume to be used by lvm by using the pvcreate command: 
```
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```
11. To check if the PV have been created type: `sudo pvs`
12. Next, Create the volume group and name it **webdata-vg**: `sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1`
13. View newly created volume group type: `sudo vgs`
14. Create 2 logical volumes using lvcreate utility. Name them: **apps-lv** for storing data for the Website and **logs-lv** for storing data for logs.
```
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```
15. Verify Logical Volume has been created successfully by running: `sudo lvs`
16. Next, format the logical volumes with ext4 filesystem: 
```
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
17. Next, create mount points for logical volumes. Create **/var/www/html** directory to store website files: `sudo mkdir -p /var/www/html` and mount **/var/www/html** on Mount /var/www/html on apps-lv logical volume : `sudo mount /dev/webdata-vg/apps-lv /var/www/html/`
18. Then create **/home/recovery/logs** to store backup of log data: `sudo mkdir -p /home/recovery/logs` 
19. Use **rsync** utility to backup all the files in the log directory **/var/log** into **/home/recovery/logs** (It is important to backup all data on the /var/log directory because all the data will be deleted during the mount process) Type the following command: `sudo rsync -av /var/log/. /home/recovery/logs/`
20. Mount /var/log on logs-lv logical volume: `sudo mount /dev/webdata-vg/logs-lv /var/log` 
21. Finally, restore deleted log files back into /var/log directory: `sudo rsync -av /home/recovery/logs/. /var/log`
22. Next, update **/etc/fstab** file so that the mount configuration will persist after restart of the server.
23. The UUID of the device will be used to update the /etc/fstab file to get the UUID type: `sudo blkid` and copy the both the apps-vg and logs-vg UUID (Excluding the double quotes)
24. Type sudo `vi /etc/fstab` to open editor and update using the UUID you copied.
25. Test the configuration and reload the daemon: 
```
sudo mount -a`
sudo systemctl daemon-reload
```
26. Verify your setup by running `df -h`

### Part 2 - Prepare the Database Server
27. Launch a second RedHat EC2 instance and name it **DB Server**
28. Repeat the same steps as for the Web Server, but instead of **apps-lv** create **db-lv** and mount it to **/db** directory instead of /var/www/html/.

### Part 3 -Install WordPress and connect it to a remote MySQL database server.
29. Update the repository: `sudo yum -y update`
30. Install wget, Apache and it???s dependencies: `sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json`
31. Start Apache
```
sudo systemctl enable httpd
sudo systemctl start httpd
```
32. install PHP and it???s depemdencies:
```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1
```
33. Restart Apache: `sudo systemctl restart httpd`
34. Download wordpress and copy wordpress to var/www/html
```
mkdir wordpress
cd   wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
cp -R wordpress /var/www/html/
35. Configure SELinux Policies:
```
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1

