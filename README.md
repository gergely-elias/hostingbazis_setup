# hostingbazis setup (ubuntu 24.04)

## create non-root user

`adduser <username>`

`usermod -aG sudo <username>`

## update packages

`sudo apt update && sudo apt upgrade -y`

## FTP server setup

### install vsftpd

`sudo apt install vsftpd -y`

### start, enable, and check status

`sudo systemctl start vsftpd`

`sudo systemctl enable vsftpd`

`sudo systemctl status vsftpd`

### configure firewall

`sudo ufw allow 20/tcp`

`sudo ufw allow 21/tcp`

`sudo ufw allow 40000:50000/tcp`

`sudo ufw allow ssh`

`sudo ufw enable`

### create FTP user group

`sudo groupadd ftpusers`

### add user to user group

`sudo usermod -aG ftpusers <username>`

### create dedicated FTP directory

`sudo mkdir -p /home/ftpusers/<username>`

`sudo chown -R <username>:ftpusers /home/ftpusers/<username>`

`sudo chmod -R 750 /home/ftpusers/<username>`

### generate SSL/TLS certificates

`sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/ssl-cert-snakeoil.key -out /etc/ssl/certs/ssl-cert-snakeoil.pem`

### configure vsftpd

`sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak`

`sudo nano /etc/vsftpd.conf`

```
# Allow local users to log in
local_enable=YES

# Allow uploads and other write commands
write_enable=YES

# Restrict users to their home directories (Chroot)
chroot_local_user=YES
# Allow writable chroot users (Ubuntu 24.04 specific for security)
allow_writeable_chroot=YES

# This option specifies the location of the RSA certificate to use for SSL
# encrypted connections.
rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ssl_enable=YES
force_local_data_ssl=YES
force_local_logins_ssl=YES

# Passive mode settings (important for firewalls)
pasv_min_port=40000
pasv_max_port=50000

# Optional: Configure for a specific user's directory
user_sub_token=$USER
local_root=/home/ftpusers/$USER
```

### restart vsftpd

`sudo systemctl restart vsftpd`
