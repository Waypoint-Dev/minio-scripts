# Waypoint MinIO Install Guide

### QUICK SERVICE COMMANDS
- sudo systemctl restart minio.service
- sudo systemctl start minio.service
- sudo systemctl stop minio.service
- sudo systemctl status minio.service

### MINIO MINIMUM RESOURCES
 - 4vCPUs
 - 16GB RAM
 - 60GB HDD
 
## Installation Order
1. Install and patch Ubuntu 20.04 LTS
    - Get Ubuntu 20.04 from Waypoint FTP or Ubuntu site direct
    - Install in VM or baremetal host
    - Patch to latest releases
        - **sudo apt-get update**
        - **sudo apt-get ugprade**
2. Download all of our required files or configs
    - **cd ~**
    - **8sudo ( cd /etc/systemd/system/; curl -O https://raw.githubusercontent.com/Waypoint-Dev/minio-scripts/InitialPush/minio )**