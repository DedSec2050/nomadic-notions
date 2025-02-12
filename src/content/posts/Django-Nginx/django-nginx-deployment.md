---
title: Deploying a Django Backend on an Azure VM with Gunicorn & Nginx
published: 2025-02-12
description: 'A step-by-step guide to deploying Django on an Azure VM using Gunicorn and Nginx'
image: "./django.webp"
tags: [Django, Azure, Deployment, Gunicorn, Nginx]
category: 'Deployment'
draft: false 
lang: ''
---

:::tip
Deploying a Django application with Nginx and Gunicorn on Azure can be challenging, but this comprehensive guide will walk you through the process step by step. We'll cover everything from setting up your server to securing it with SSL, ensuring your Django application runs efficiently in a production environment.
:::


## Setting Up the Server

### Update and Install Required Packages

```sh
sudo apt update && sudo apt upgrade -y
sudo apt install python3-pip python3-venv nginx -y
```

## Cloning the Django Project and Setting Up the Virtual Environment

### Navigate to Home Directory

```sh
cd /home/azureuser
```

### Clone the Project from GitHub

```sh
git clone https://github.com/your-username/your-django-repo.git django-brevo
cd django-brevo
```

### Create and Activate a Virtual Environment

```sh
python3 -m venv venv
source venv/bin/activate
```

### Install Dependencies

```sh
pip install --upgrade pip
pip install -r requirements.txt
```

## Configuring Django for Production

### Set Up Environment Variables (Optional)

Create a `.env` file:

```ini
SECRET_KEY=your-secret-key
DEBUG=False
ALLOWED_HOSTS=yourdomain.com
```

Modify `settings.py`:

```python
import os
from dotenv import load_dotenv
load_dotenv()

SECRET_KEY = os.getenv("SECRET_KEY")
DEBUG = os.getenv("DEBUG") == "True"
ALLOWED_HOSTS = os.getenv("ALLOWED_HOSTS").split(",")
```

### Apply Migrations

```sh
python manage.py migrate
```

## Setting Up Gunicorn

### Test Gunicorn Locally

```sh
gunicorn --workers 3 --bind 0.0.0.0:8000 backend.wsgi:application
```

### Create a Systemd Service for Gunicorn

```sh
sudo nano /etc/systemd/system/gunicorn.service
```

Paste the following configuration:

```ini
[Unit]
Description=Gunicorn Daemon for Django App
After=network.target

[Service]
User=azureuser
Group=www-data
WorkingDirectory=/home/azureuser/django-brevo
ExecStart=/home/azureuser/django-brevo/venv/bin/gunicorn --workers 3 --bind unix:/home/azureuser/django-brevo/gunicorn.sock backend.wsgi:application

[Install]
WantedBy=multi-user.target
```

### Start and Enable Gunicorn

```sh
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
```

### Check Gunicorn Status

```sh
sudo systemctl status gunicorn
```

## Setting Up Nginx as a Reverse Proxy

### Create Nginx Configuration File

```sh
sudo nano /etc/nginx/sites-available/django-brevo
```

Paste the following configuration:

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://unix:/home/azureuser/django-brevo/gunicorn.sock;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Enable the Configuration

```sh
sudo ln -s /etc/nginx/sites-available/django-brevo /etc/nginx/sites-enabled
```

### Test Nginx Configuration

```sh
sudo nginx -t
```

### Restart Nginx

```sh
sudo systemctl restart nginx
```

### Enable Nginx to Start on Boot

```sh
sudo systemctl enable nginx
```

## Fixing Permission Issues

### Fix Gunicorn Socket Permissions

```sh
sudo chown -R azureuser:www-data /home/azureuser/django-brevo
sudo chmod -R 750 /home/azureuser/django-brevo
sudo chown azureuser:www-data /home/azureuser/django-brevo/gunicorn.sock
sudo chmod 660 /home/azureuser/django-brevo/gunicorn.sock
```

### Restart Services

```sh
sudo systemctl restart gunicorn nginx
```

### Verify Everything is Running

```sh
sudo systemctl status gunicorn
sudo systemctl status nginx
```

## Allow Traffic and Secure the Server

### Open Firewall Ports

```sh
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

### Obtain SSL Certificate (Optional)

If using a domain, install Certbot for free SSL via Let's Encrypt:

```sh
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Enable auto-renewal:

```sh
sudo certbot renew --dry-run
```

## Monitoring and Debugging

### Check Logs for Issues

**Gunicorn logs:**

```sh
sudo journalctl -u gunicorn --no-pager -f
```

**Nginx error logs:**

```sh
sudo tail -f /var/log/nginx/error.log
