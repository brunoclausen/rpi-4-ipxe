#!/bin/bash

# Update and upgrade the system
sudo apt update
sudo apt upgrade -y

# Install lighttpd
sudo apt install lighttpd -y

# Start and enable lighttpd
sudo systemctl start lighttpd
sudo systemctl enable lighttpd

# Create a simple HTML file for lighttpd
sudo bash -c 'cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html>
<head>
    <title>Welcome to lighttpd</title>
</head>
<body>
    <h1>Success! lighttpd is working!</h1>
</body>
</html>
EOF'

# Set permissions for lighttpd web directory
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html

# Install dnsmasq
sudo apt install dnsmasq -y

# Stop dnsmasq service to configure it
sudo systemctl stop dnsmasq

# Configure dnsmasq for DHCP and TFTP
sudo bash -c 'cat > /etc/dnsmasq.conf <<EOF
# DHCP settings
interface=eth0                # Use the correct network interface
dhcp-range=192.168.1.50,192.168.1.150,12h
dhcp-boot=undionly.kpxe

# TFTP settings
enable-tftp
tftp-root=/srv/tftp
EOF'

# Create TFTP directory and download iPXE files
sudo mkdir -p /srv/tftp
cd /srv/tftp
sudo wget https://boot.ipxe.org/undionly.kpxe

# Download a Linux kernel and initrd for network booting
sudo wget http://archive.ubuntu.com/ubuntu/dists/focal/main/installer-amd64/current/images/netboot/ubuntu-installer/amd64/linux -O vmlinuz
sudo wget http://archive.ubuntu.com/ubuntu/dists/focal/main/installer-amd64/current/images/netboot/ubuntu-installer/amd64/initrd.gz -O initrd.gz

# Create a simple iPXE script
sudo bash -c 'cat > /srv/tftp/boot.ipxe <<EOF
#!ipxe
dhcp
kernel tftp://$(hostname -I | awk '"'"'{print $1}'"'"')/vmlinuz
initrd tftp://$(hostname -I | awk '"'"'{print $1}'"'"')/initrd.gz
boot
EOF'

# Modify dnsmasq configuration to point to the iPXE script
sudo sed -i 's/dhcp-boot=undionly.kpxe/dhcp-boot=boot.ipxe/' /etc/dnsmasq.conf

# Start and enable dnsmasq
sudo systemctl start dnsmasq
sudo systemctl enable dnsmasq

# Restart lighttpd to apply any changes
sudo systemctl restart lighttpd

echo "lighttpd and iPXE proxy server installation and configuration complete."
echo "Visit http://$(hostname -I | awk '{print $1}') to see the web page."
