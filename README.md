# Waypoint MinIO Install Guide

### QUICK SERVICE COMMANDS
- sudo systemctl restart minio.service
- sudo systemctl start minio.service
- sudo systemctl stop minio.service
- sudo systemctl status minio.service

### MINIO MINIMUM RESOURCES
- 4vCPUs
- 16GB RAM
- 60GB disk for OS
- XXGB disk for MinIO DAta
    - If this drive causes issues with boot make sure the SCSI drive ID is higher then 3
    - This would ideally be a combination of multiple drives to create a proper erasure coding stack. But for testing and/or POC a location with multiple data segments will work. 
    
### VEEAM MINIMUM REQUIREMENTS
- Needs to be v10.x and higher
- Needs to be licensed for Enterprise of higher

## SERVER LAYOUT
- ```~/``` = home directory where we will stage alot of files and keep backups for safe keeping
- ```/usr/local/etc/miniossl``` = SSL directory for certificate and private key
- ```/usr/local/bin/``` = core directory for the MinIO executables
- ```/minio_data/``` = MinIO data directory parent, this will ideally contain multiple 'disk' folders for mapping to physical or logical drives.
 
## Installation Order
1. Install and patch Ubuntu 20.04 LTS
    - Get Ubuntu 20.04 from Waypoint FTP or Ubuntu site direct
    - Install in VM or baremetal host
    - Patch to latest releases
    ```
    sudo apt-get update
    sudo apt-get ugprade
   ```
2. Install any required utilities 
    ```
   # install xfsprogs to 'format' the disk later
   sudo apt-get install xfsprogs
   ```
3. Download all of our required files or configs
    ```
    cd ~
   ( cd ~/; curl -O https://raw.githubusercontent.com/Waypoint-Dev/minio-scripts/minio )
   ( cd ~/; curl -O https://raw.githubusercontent.com/Waypoint-Dev/minio-scripts/minio.service )
   ( cd ~/; curl -O https://raw.githubusercontent.com/Waypoint-Dev/minio-scripts/openssl.conf )
   ( cd ~/; curl -O https://raw.githubusercontent.com/Waypoint-Dev/minio-scripts/00-installer-config.yaml )
   wget https://dl.minio.io/server/minio/release/linux-amd64/minio
   wget https://dl.minio.io/client/mc/release/linux-amd64/mc
   ```
4. Make all our required directories
    ```
   # our SSL directory 
   sudo mkdir /usr/local/etc/miniossl
   # our MinIO data directory
   sudo mkdir /minio_data
   ```
5. Copy our MinIO binaries and make executable
    ```
   # move the minio binaries 
   sudo cp ./minio /usr/local/bin/minio
   # make it executable 
   sudo chmod +x /usr/local/bin/minio
   ```
6. Setup a static IP address for this system 
    ```
   # edit the NIC config file and follow the comments, press CTRL+X, type Y to save and exit
   nano ~/00-installer-config.yaml
   # now move and overwrite the default config file
   sudo cp ~/00-installer-config.yaml /etc/netplan/00-installer-config.yaml
   # apply the changes, this will probably disconnect you if you are runnnig over SSH
   sudo netplan apply
   ```
7. Setup the additional drive as a target for MinIO data
    ```
   # setup 2nd drive as the target storage, find it first, most then likely it will be /dev/sdb, match it by size
   sudo fdisk -l
   #this will be change to the /dev/sd* of what you found
   sudo fdisk /dev/sdb
   # type "n" for new partition and hit enter
   # type "p" for primary partition 
   # type "1" for partition number one 
   # type "w" to write the partition table to the drive
   # now this command should show a new partition like /dev/sdb1 
   sudo ls /dev/sdb*
   # format the disk
   sudo mkfs.xfs /dev/sdb1
   # now configure the auto mount of the disk 
   sudo nano /etc/fstab
   # this will add to the bottome of the file to enable auto mount 
   sudo echo '/dev/sdb1 /minio_data xfs defaults 0 0' >> /etc/fstab
   ```
8. Create the additional sub directories for the MinIO erasure system, this is REQUIRED for object lock to work properly. Ideally you would have multiple physical drives and mapping each to a folder but for testing or POC we can do it this way. Minimum of 4 required, optmimally would be 8 or 16 for larger data sets. 
    ```
   # our MinIO sub data directories, this is REQUIRED for object lock capability
   sudo mkdir /minio_data/disk1
   sudo mkdir /minio_data/disk2
   sudo mkdir /minio_data/disk3
   sudo mkdir /minio_data/disk4
   ```
9. SSL certificate self-signed, if the client would prefer to use a commercial signed certificate you just need to drop the private.key and public.crt into the /usr/local/etc/miniossl directory
    ```
   cd ~
   # edit the ssl conf file and follow the comments, press CTRL+X, type Y to save and exit
   sudo nano ~/openssl.conf
   # generate the self signed SSL that the system will use, this uses the above openssl.conf file 
   openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout private.key -out public.crt -config openssl.conf
   # copy the SSL cert over to the central folder
   sudo cp /home/wpadmin/private.key /usr/local/etc/minio/private.key
   sudo cp /home/wpadmin/public.crt /usr/local/etc/minio/public.crt
   # also copy to the system cert store for client use later
   sudo cp /home/wpadmin/public.crt /etc/ssl/certs/minio.crt
   ```
10. Install the minio.service and restart daemon
    ```
    # install the service
    sudo cp ~/minio.service /etc/systemd/system/minio.service
    # refresh the daemon
    sudo systemctl daemon-reload
    # now restart the minio.service 
    sudo systemctl restart minio.service
    # now to make sure it shows up, looking for Active: active (running)
    sudo systemctl status minio.service 
    ```
11. Setup a local DNS entry for the host, ex:minio.waypointsolutions.com -> internal LAN IP
12. Log into the MinIO browser from a LAN computer by visiting the URL setup, ex: https://minio.waypointsolutions.com
    - You will use the access key and secret key set in the minio file earlier
    - Nothing else needs to be done besides making sure things load properly
13. Create a bucket for Veeam
    ```
    # make the client binaries executable
    cd ~
    chmod +x mc
    # connect to the MinIO instance we just created, use the DNS record name, access key and secret key fromt earlier!!
    ./mc config host add minio https://minio.waypointsolutions.com Access_key Secret_key --insecure
    # create a veeam backup bucket with lock enabled 
    ./mc mb --with-lock minio/veeambackup
    ```