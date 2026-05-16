# hostingbazis setup (ubuntu 24.04)

> Multi-project Django server. Each project gets its own virtualenv, gunicorn socket, and nginx block.

---

## User model

This setup uses **two unprivileged users** — never root directly.

| User | Purpose | Has sudo? |
|---|---|---|
| `adminuser` | System config: apt, nginx, systemd, certbot | **yes** |
| `django` | Owns all project files, runs gunicorn | **no** |

The app user (`django`) has no sudo. If a project is ever compromised, the attacker cannot touch system config, install packages, or affect other services. System-level tasks are always done from `adminuser`.

> Throughout this document, each code block is prefixed with the user who runs it:
> `[adminuser]` or `[django]`

### Switching between users

No need to log out or open a new SSH session. From any session, use `su -` to switch:

```bash
su - django      # switch to django (full login shell)
exit             # switch back to whoever you were before
```

The `-` flag is important — it gives you django's full environment and home directory,
as if you had SSH'd in as them directly.

---

## 1. Create users

> Run as **root** for this section only — immediately after first login.

```bash
# Admin user — your day-to-day login for system tasks
adduser adminuser
usermod -aG sudo adminuser

# App user — owns projects, no sudo
adduser django

# Allow nginx (www-data) to read files owned by django
usermod -aG django www-data
```

Log out of root. From here on, **never log in as root again** — use `adminuser` for system tasks and `django` for project work.

---

## 2. Update packages

```bash
# [adminuser]
sudo apt update && sudo apt upgrade -y
```

---

## 3. FTP server (vsftpd)

The `django` user's home directory (`/home/django`) is the FTP root. Files uploaded via FTP land there and are directly accessible to the app.

### Install and enable

```bash
# [adminuser]
sudo apt install vsftpd -y
sudo systemctl start vsftpd
sudo systemctl enable vsftpd
```

### Configure firewall

```bash
# [adminuser]
sudo ufw allow 20/tcp
sudo ufw allow 21/tcp
sudo ufw allow 40000:50000/tcp
sudo ufw allow ssh
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable
```

### Generate SSL/TLS certificate

```bash
# [adminuser]
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/vsftpd.key \
  -out /etc/ssl/certs/vsftpd.pem
```

### Configure vsftpd

```bash
# [adminuser]
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
# [adminuser]
sudo systemctl restart vsftpd
```

---

## 4. Install system packages and tools

```bash
# [adminuser] — system packages require sudo
sudo apt install git sqlite3 nginx -y
sudo apt install pipx -y
```

```bash
# [django] — uv is installed into the app user's own space, no sudo needed
pipx install uv
pipx ensurepath    # adds ~/.local/bin to PATH; re-login or source ~/.bashrc after this
```

---

## 5. Directory layout

```
/home/django/
    projects/
        mysite/          ← one directory per Django project
        blog/
        api/
```

```bash
# [django]
mkdir -p ~/projects
```

---

## 6. Per-project setup

Repeat this section for every new project. Replace `mysite` with your project name.
**All steps in this section run as `django`** — no sudo anywhere.

### Create project directory and virtualenv

```bash
# [django]
mkdir ~/projects/mysite
cd ~/projects/mysite
uv venv .venv --python 3.12
source .venv/bin/activate
python -m ensurepip
```

### Install Django and gunicorn

```bash
# [django]
python -m pip install django gunicorn
```

> Django 5.x is the current stable release. Do **not** pin `==6.0` — that version does not exist.
> Use `python -m pip` rather than bare `pip` — it guarantees you're using the pip that
> belongs to the active venv, avoiding PATH ambiguity.

### Start a new Django project

```bash
# [django]
django-admin startproject mysite .   # trailing dot keeps manage.py in the current directory
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
# [django]
python manage.py migrate
python manage.py collectstatic --noinput
python manage.py createsuperuser
```

### Set permissions so nginx can read project files

nginx runs as `www-data`, which was added to the `django` group in step 1.
Now set the right permissions so that group membership actually grants access:

```bash
# [django]
chmod 750 /home/django                              # www-data can traverse via group membership
chmod -R 755 ~/projects/mysite/staticfiles
chmod -R 755 ~/projects/mysite/media
```

---

## 7. Gunicorn — one socket per project

**Socket and service files go in `/etc/systemd/system/` — this requires `adminuser`.**
But note that the service itself runs the gunicorn process *as `django`*, so the app
never has elevated privileges at runtime.

### Socket file

```bash
# [adminuser]
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

### Service file

```bash
# [adminuser]
sudo nano /etc/systemd/system/gunicorn-mysite.service
```

```ini
[Unit]
Description=gunicorn daemon for mysite
Requires=gunicorn-mysite.socket
After=network.target

[Service]
User=django
Group=www-data
WorkingDirectory=/home/django/projects/mysite
ExecStart=/home/django/projects/mysite/.venv/bin/gunicorn \
          --workers 3 \
          --bind unix:/run/gunicorn-mysite.sock \
          mysite.wsgi:application

[Install]
WantedBy=multi-user.target
```

> **Workers:** A common rule of thumb is `2 × CPU cores + 1`.
>
> **WSGI entrypoint:** For Django it is always `<projectname>.wsgi:application` —
> the `projectname` is the inner directory containing `settings.py` and `wsgi.py`.
> (`app:app` is Flask syntax and will not work with Django.)

### Enable and start

```bash
# [adminuser]
sudo systemctl daemon-reload
sudo systemctl start gunicorn-mysite.socket
sudo systemctl enable gunicorn-mysite.socket
```

Verify the socket was created:

```bash
# [adminuser]
sudo systemctl status gunicorn-mysite.socket
ls /run/gunicorn-mysite.sock
```

---

## 8. nginx — one server block per project

### Site config

```bash
# [adminuser]
sudo nano /etc/nginx/sites-available/mysite
```

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Serve Django's collected static files directly — never proxy these to gunicorn
    location /static/ {
        alias /home/django/projects/mysite/staticfiles/;
    }

    # Serve user-uploaded media files directly
    location /media/ {
        alias /home/django/projects/mysite/media/;
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
# [adminuser]
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo nginx -t          # must print "syntax is ok" before restarting
sudo systemctl restart nginx
```

### Remove the default nginx site

nginx ships with a default site that will take priority over yours if left enabled:

```bash
# [adminuser]
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

---

## 9. HTTPS with Let's Encrypt (strongly recommended)

```bash
# [adminuser]
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot rewrites your nginx config to handle port 443 and redirect 80 → 443 automatically.
Renewal is handled by a systemd timer installed with certbot — no cron job needed.

---

## 10. Adding a second project

The pattern is identical. For a project called `blog`:

```bash
# [django] — project setup
mkdir ~/projects/blog && cd ~/projects/blog
uv venv .venv --python 3.12 && source .venv/bin/activate
python -m ensurepip
python -m pip install django gunicorn
django-admin startproject blog .
python manage.py migrate && python manage.py collectstatic --noinput
chmod -R 755 ~/projects/blog/staticfiles ~/projects/blog/media
```

```bash
# [adminuser] — system config
sudo nano /etc/systemd/system/gunicorn-blog.socket    # ListenStream=/run/gunicorn-blog.sock
sudo nano /etc/systemd/system/gunicorn-blog.service   # User=django, WSGI=blog.wsgi:application
sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn-blog.socket

sudo nano /etc/nginx/sites-available/blog             # server_name, proxy to gunicorn-blog.sock
sudo ln -s /etc/nginx/sites-available/blog /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 11. Useful commands

```bash
# [adminuser] — check gunicorn status
sudo systemctl status gunicorn-mysite

# [adminuser] — tail gunicorn logs
sudo journalctl -u gunicorn-mysite -f

# [adminuser] — tail nginx logs
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log

# [adminuser] — after changing nginx config
sudo nginx -t && sudo systemctl reload nginx
```

```bash
# [django] — after changing Django code, signal adminuser to restart gunicorn
# (django user cannot run systemctl — ask adminuser or set up a deploy script)
```

> **Deployment tip:** If restarting gunicorn after every code change feels cumbersome,
> grant the `django` user permission to restart *only* gunicorn services via `/etc/sudoers.d/`.
> See section 12 below.

---

## 12. Granting `django` permission to restart its own services

Without this, every code deploy requires you to SSH in as `adminuser` just to run
`systemctl restart`. Instead, create a narrow sudoers rule that allows `django` to
restart *only* gunicorn — nothing else.

### Create the rule file

```bash
# [adminuser]
sudo visudo -f /etc/sudoers.d/django-gunicorn
```

> Always use `visudo` — it validates the syntax before saving. A typo in a sudoers file
> can lock you out of sudo entirely.

Add this content:

```
# Allow the django user to restart gunicorn services only
django ALL=(root) NOPASSWD: /usr/bin/systemctl restart gunicorn-*.service
django ALL=(root) NOPASSWD: /usr/bin/systemctl restart gunicorn-*.socket
```

Save and exit. The wildcard `gunicorn-*` means the rule automatically covers every
project you add later without editing the file again.

### Verify it works

```bash
# [django]
sudo systemctl restart gunicorn-mysite.service
```

It should restart without prompting for a password. If you get a permission error,
double-check the file with:

```bash
# [adminuser]
sudo visudo -c -f /etc/sudoers.d/django-gunicorn
```

### Adding a new project later

No changes needed — the wildcard covers `gunicorn-blog.service`, `gunicorn-api.service`,
and so on automatically.

### What `django` still cannot do

This rule is intentionally narrow. The `django` user still cannot:

- `systemctl restart nginx` (only `adminuser` can)
- Run any other sudo command

That's the point — a compromised app process can bounce itself but cannot touch the
system around it.

---

## Summary: who does what

| Task | User |
|---|---|
| apt install, ufw, certbot | `adminuser` with sudo |
| Edit `/etc/nginx/`, `/etc/systemd/` | `adminuser` with sudo |
| Create virtualenvs, pip install | `django` |
| `manage.py` commands | `django` |
| Gunicorn process at runtime | `django` (set by `User=` in service file) |
| nginx process at runtime | `www-data` (reads django's files via group) |

## Summary: what each layer does

| Layer | Role |
|---|---|
| **vsftpd** | FTP access for uploading files |
| **Django** | Web framework; handles URLs, views, templates, DB |
| **gunicorn** | Runs Django as a WSGI process; one per project |
| **nginx** | Receives HTTP/S requests; serves static files itself; proxies dynamic requests to gunicorn |
| **systemd** | Keeps gunicorn alive and starts it on boot |
| **certbot** | Manages TLS certificates from Let's Encrypt |
