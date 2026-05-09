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

## install git

`sudo apt install git -y`

## install SQLite

`sudo apt install sqlite3 -y`

## install pip

`sudo apt install pipx -y`

## install uv

`pipx install uv`

## updating PATH

`pipx ensurepath`

`pipx completions` (for instructions)

`eval "$(register-python-argcomplete pipx)"`

## create virtual environment

`uv venv <envname> --python 3.12`

## activate environment

`source <envname>/bin/activate`

## install packaged version of pip

`python -m ensurepip`

## install Django

`python -m pip install Django==6.0`

## install nginx

`sudo apt install nginx -y`

## install gunicorn

`python -m pip install gunicorn`

## configure gunicorn with systemd

### create gunicorn socket file

`sudo nano /etc/systemd/system/gunicorn.socket`

```
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

### create gunicorn service file

`sudo nano /etc/systemd/system/gunicorn.service`

```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=<username>
Group=www-data
WorkingDirectory=/home/<username>/my_app
ExecStart=/home/<username>/my_app/venv/bin/gunicorn \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          app:app  # Use 'project_name.wsgi' for Django

[Install]
WantedBy=multi-user.target
```

(workers: 2 x cores + 1 for optimal performance)

## configure nginx as reverse proxy

`sudo nano /etc/nginx/sites-available/my_app`

```
server {
    listen 80;
    server_name <domain_or_IP>;

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

## enable and start services

`sudo systemctl start gunicorn.socket`

`sudo systemctl enable gunicorn.socket`

`sudo ln -s /etc/nginx/sites-available/my_app /etc/nginx/sites-enabled`

`sudo nginx -t`

`sudo systemctl restart nginx`
