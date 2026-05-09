# hostingbazis setup (ubuntu 24.04)

> Multi-project Django server. Each project gets its own virtualenv, gunicorn socket, and nginx block.

---

## 1. Create non-root user

```bash
adduser <username>
usermod -aG sudo <username>
```

## 2. Update packages

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 3. FTP server (vsftpd)

### Install and enable

```bash
sudo apt install vsftpd -y
sudo systemctl start vsftpd
sudo systemctl enable vsftpd
```

### Configure firewall

```bash
sudo ufw allow 20/tcp
sudo ufw allow 21/tcp
sudo ufw allow 40000:50000/tcp
sudo ufw allow ssh
sudo ufw enable
```

### Generate SSL/TLS certificate

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/vsftpd.key \
  -out /etc/ssl/certs/vsftpd.pem
```

### Configure vsftpd

```bash
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak
sudo nano /etc/vsftpd.conf
```

```ini
local_enable=YES
write_enable=YES

# Chroot users to their home directory
chroot_local_user=YES
allow_writeable_chroot=YES

# SSL
rsa_cert_file=/etc/ssl/certs/vsftpd.pem
rsa_private_key_file=/etc/ssl/private/vsftpd.key
ssl_enable=YES
force_local_data_ssl=YES
force_local_logins_ssl=YES

# Passive mode (required for most FTP clients behind NAT)
pasv_min_port=40000
pasv_max_port=50000

# Chroot each user to their own home
user_sub_token=$USER
local_root=/home/$USER
```

```bash
sudo systemctl restart vsftpd
```

> **Note:** With this setup your FTP root is simply `/home/<username>`. No separate `ftpusers` group
> is needed — the user owns their own files, and nginx/gunicorn run as `www-data` with group
> read access granted below. Keep it simple.

---

## 4. Install core tools

```bash
sudo apt install git sqlite3 -y

# pipx manages isolated CLI tools; uv is a fast pip/venv replacement
sudo apt install pipx -y
pipx install uv
pipx ensurepath          # adds ~/.local/bin to PATH; re-login or source ~/.bashrc after this
```

---

## 5. Directory layout

Adopt a consistent layout before creating any projects:

```
/home/<username>/
    projects/
        mysite/          ← one directory per Django project
        blog/
        api/
```

```bash
mkdir -p ~/projects
```

---

## 6. Per-project setup

Repeat this section for every new Django project. Replace `mysite` with your project name.

### Create project directory and virtualenv

```bash
mkdir ~/projects/mysite
cd ~/projects/mysite
uv venv .venv --python 3.12
source .venv/bin/activate
python -m ensurepip
```

### Install Django and gunicorn

```bash
pip install django gunicorn
```

> Django 5.x is the current stable release. Do **not** pin `==6.0` — that version does not exist.

### Start a new Django project

```bash
django-admin startproject mysite .   # the trailing dot puts manage.py in the current directory
```

### Configure settings for production

In `mysite/settings.py`:

```python
# Allow your domain and server IP
ALLOWED_HOSTS = ['yourdomain.com', 'www.yourdomain.com', '<server_IP>']

# Static files — nginx will serve these directly
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'   # collectstatic writes here

# Media files (user uploads)
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

# SQLite is fine for small/medium projects
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

### Run migrations and collect static files

```bash
python manage.py migrate
python manage.py collectstatic --noinput
python manage.py createsuperuser
```

### Set file permissions so nginx can read static files

```bash
chmod 710 /home/<username>                        # www-data needs execute to traverse
chmod -R 755 ~/projects/mysite/staticfiles
chmod -R 755 ~/projects/mysite/media
sudo usermod -aG <username> www-data              # add www-data to your group
```

---

## 7. Gunicorn — one socket per project

Each Django project gets its own socket and service. This keeps projects fully isolated.

### Socket file: `/etc/systemd/system/gunicorn-mysite.socket`

```bash
sudo nano /etc/systemd/system/gunicorn-mysite.socket
```

```ini
[Unit]
Description=gunicorn socket for mysite

[Socket]
ListenStream=/run/gunicorn-mysite.sock

[Install]
WantedBy=sockets.target
```

### Service file: `/etc/systemd/system/gunicorn-mysite.service`

```bash
sudo nano /etc/systemd/system/gunicorn-mysite.service
```

```ini
[Unit]
Description=gunicorn daemon for mysite
Requires=gunicorn-mysite.socket
After=network.target

[Service]
User=<username>
Group=www-data
WorkingDirectory=/home/<username>/projects/mysite
ExecStart=/home/<username>/projects/mysite/.venv/bin/gunicorn \
          --workers 3 \
          --bind unix:/run/gunicorn-mysite.sock \
          mysite.wsgi:application

[Install]
WantedBy=multi-user.target
```

> **Workers:** A common rule of thumb is `2 × CPU cores + 1`.
>
> **WSGI entrypoint:** For Django it is always `<projectname>.wsgi:application` —
> the `projectname` here is the inner directory that contains `settings.py`, `wsgi.py`, etc.
> (`app:app` is Flask syntax and will not work with Django.)

### Enable and start

```bash
sudo systemctl daemon-reload
sudo systemctl start gunicorn-mysite.socket
sudo systemctl enable gunicorn-mysite.socket
```

Verify the socket was created:

```bash
sudo systemctl status gunicorn-mysite.socket
ls /run/gunicorn-mysite.sock
```

---

## 8. nginx — one server block per project

### Install nginx

```bash
sudo apt install nginx -y
```

### Site config: `/etc/nginx/sites-available/mysite`

```bash
sudo nano /etc/nginx/sites-available/mysite
```

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Serve Django's collected static files directly — do NOT proxy these to gunicorn
    location /static/ {
        alias /home/<username>/projects/mysite/staticfiles/;
    }

    # Serve user-uploaded media files directly
    location /media/ {
        alias /home/<username>/projects/mysite/media/;
    }

    # Everything else goes to gunicorn
    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn-mysite.sock;
    }
}
```

### Enable the site

```bash
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo nginx -t          # must print "syntax is ok" before restarting
sudo systemctl restart nginx
```

---

## 9. HTTPS with Let's Encrypt (strongly recommended)

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot will rewrite your nginx config to handle 443 and redirect 80 → 443 automatically.
Renewal is handled by a systemd timer installed with certbot — no cron job needed.

---

## 10. Adding a second project

The pattern is the same. For a project called `blog`:

1. Create `~/projects/blog/`, set up venv, install django + gunicorn, run `startproject blog .`
2. Create `/etc/systemd/system/gunicorn-blog.socket` and `gunicorn-blog.service` (socket path: `/run/gunicorn-blog.sock`, WSGI: `blog.wsgi:application`)
3. Create `/etc/nginx/sites-available/blog` pointing to a different `server_name` and the `blog.sock`
4. `systemctl enable --now gunicorn-blog.socket && nginx -t && systemctl restart nginx`

---

## 11. Useful commands

```bash
# Check gunicorn is running after a request hits nginx
sudo systemctl status gunicorn-mysite

# Tail gunicorn logs
sudo journalctl -u gunicorn-mysite -f

# Tail nginx logs
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log

# After changing Django code, restart gunicorn (no nginx restart needed)
sudo systemctl restart gunicorn-mysite

# After changing nginx config
sudo nginx -t && sudo systemctl reload nginx
```

---

## Summary: what each layer does

| Layer | Role |
|---|---|
| **vsftpd** | FTP access for uploading files |
| **Django** | Web framework; handles URLs, views, templates, DB |
| **gunicorn** | Runs Django as a WSGI process; one per project |
| **nginx** | Receives HTTP/S requests; serves static files itself; proxies dynamic requests to gunicorn |
| **systemd** | Keeps gunicorn alive and starts it on boot |
| **certbot** | Manages TLS certificates from Let's Encrypt |
