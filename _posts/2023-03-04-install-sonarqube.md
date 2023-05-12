---
layout: post
title: "How to Install Sonarqube in Ubuntu Linux"
date: 2023-03-04 22:00:00 -0500
categories: [sonarqube]
tags: [sonarqube]
#image:
#  path: /assets/img/headers/mirror-image-chess.jpg
---

SonarQube is an open-source platform developed by SonarSource for continuous inspection of code quality to perform automatic reviews with static analysis of code to detect bugs and code smells on 29 programming languages.

## Prerequsites 
- Virtual Machine running Ubuntu 22.04 or newer

### Update Package Repository and Upgrade Packages

``` sh
sudo apt update
sudo apt upgrade
```

## PostgreSQL
### Add PostgresSQL repository

``` sh
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
```

### Install PostgreSQL
``` sh
sudo apt update
sudo apt-get -y install postgresql postgresql-contrib
sudo systemctl enable postgresql
```

### Create Database for Sonarqube
``` sh
sudo passwd postgres
```

``` sh
su - postgres
```

``` sh
createuser sonar
```

``` sql
createuser sonar
psql 
ALTER USER sonar WITH ENCRYPTED password 'sonar';
CREATE DATABASE sonarqube OWNER sonar;
grant all privileges on DATABASE sonarqube to sonar;
\q
exit
```

## Adoptium Java 17
``` sh
sudo bash
```

### Add Adoptium repository
``` sh
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
```

### Install Java 17
``` sh
apt update
apt install temurin-17-jdk
update-alternatives --config java
/usr/bin/java --version
exit 
```

## Linux Kernel Tuning
### Increase Limits
``` sh
sudo vim /etc/security/limits.conf
```

Paste the below values at the bottom of the file
``` sh
sonarqube   -   nofile   65536
sonarqube   -   nproc    4096
```

### Increase Mapped Memory Regions
``` sh
sudo vim /etc/sysctl.conf
```

Paste the below values at the bottom of the file
``` sh
vm.max_map_count = 262144
```

### Reboot System
``` sh
sudo reboot
```

## Sonarqube
### Download and Extract
``` sh
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip
sudo apt install unzip
sudo unzip sonarqube-9.9.0.65466.zip -d /opt
sudo mv /opt/sonarqube-9.9.0.65466 /opt/sonarqube
```

### Create user and set permissions
``` sh
sudo groupadd sonar
sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar
sudo chown sonar:sonar /opt/sonarqube -R
```

### Update Sonarqube properties with DB credentials
``` sh
sudo vim /opt/sonarqube/conf/sonar.properties
```

Find and replace the below values, you might need to add the sonar.jdbc.url
``` sh
sonar.jdbc.username=sonar
sonar.jdbc.password=sonar
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
```

Create service for Sonarqube
``` sh
sudo vim /etc/systemd/system/sonar.service
```

Paste the below into the file
``` sh
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

User=sonar
Group=sonar
Restart=always

LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```

Start Sonarqube and Enable service
``` sh
sudo systemctl start sonar
sudo systemctl enable sonar
sudo systemctl status sonar
```

Watch log files and monitor for startup
``` sh
sudo tail -f /opt/sonarqube/logs/sonar.log
```

Access the Sonarqube UI
``` sh
http://<IP>:9000
```

# Optional Reverse Proxy and TLS Configuration
## Installing Nginx
``` sh
sudo apt install nginx
```

``` sh
vi /etc/nginx/sites-available/sonarqube.conf
```

Paste the contents below and be sure to update the domain name

``` sh
server {

    listen 80;
    server_name sonarqube.dev.dman.cloud;
    access_log /var/log/nginx/sonar.access.log;
    error_log /var/log/nginx/sonar.error.log;
    proxy_buffers 16 64k;
    proxy_buffer_size 128k;

    location / {
        proxy_pass http://127.0.0.1:9000;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto http;
    }
}
```

Next, activate the server block configuration 'sonarqube.conf' by creating a symlink of that file to the '/etc/nginx/sites-enabled' directory. Then, verify your Nginx configuration files.

``` sh
sudo ln -s /etc/nginx/sites-available/sonarqube.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## Installing Certbot
The first step to using Let’s Encrypt to obtain an SSL certificate is to install the Certbot software on your server.

``` sh
sudo apt install certbot python3-certbot-nginx
```

### Obtaining an SSL Certificate

Certbot provides a variety of ways to obtain SSL certificates through plugins. The Nginx plugin will take care of reconfiguring Nginx and reloading the config whenever necessary. To use this plugin, type the following:

``` sh
sudo certbot --nginx -d sonarqube.dev.dman.cloud
```

If that’s successful, certbot will ask how you’d like to configure your HTTPS settings.

Select your choice then hit ENTER. The configuration will be updated, and Nginx will reload to pick up the new settings. certbot will wrap up with a message telling you the process was successful and where your certificates are stored

Nginx should now be serving your domain name. You can test this by navigating to https://your_domain

That's it! You have now successfully installed Sonarque, if you found this tutotial helpful please consider subscribing to my [YouTube Channel](https://www.youtube.com/dineshmistry?sub_confirmation=1) for more tutorials like this. 
