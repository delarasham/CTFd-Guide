# Creating a CTFd Instance - Beginner Guide
This is a quick beginner guide on how to spin up a CTFd instance for hosting a CTF. It assumes you know how to spin up a DigitalOcean droplet and ssh into it.


#### Delara Shamanian 


----------

- Create and ssh into a DigitalOcean droplet
- Create a user called ctfd by running the command:
```
useradd -m ctfd
```

 - Go the ctdf's home directory directory and clone the CTFd repo:
```
cd /home/ctfd/
sudo -u ctfd git clone https://github.com/CTFd/CTFd.git
```
 - Install gunicorn
```
sudo apt-get update
sudo apt-get install gunicorn
```

 - Check where gunicorn was installed by typing:  
`which gunicorn` --- save that address for later(`/usr/bin/gunicorn`)
 - Go to the CTFd directory : `cd /home/ctfd/CTFd` and run `pip install -r requirements.txt` --- you might have to install pip first
 - Run : `cd /etc/systemd/system` and make a file called `gunicorn.service`
 - Add the following to that file:
 
```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
Type=notify
# the specific user that our service will run as
User=ctfd
Group=ctfd
# another option for an even more restricted service is
# DynamicUser=yes
RuntimeDirectory=gunicorn
WorkingDirectory=/home/ctfd/CTFd
ExecStart=/usr/bin/gunicorn 'CTFd:create_app()'
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
```
 
 - Add another file to the same directory called `gunicorn.socket`
 - Add the following to the file:
```
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

User=www-data
# Optionally restrict the socket permissions even more.
# Mode=600

[Install]
WantedBy=sockets.target
```
 - Run the following commands:
```
systemctl enable --now gunicorn.socket
systemctl start gunicorn.service
systemctl status gunicorn.service
```
If everything worked correctly your CTFd instance should be running now.

- Install nginx:
```
	sudo apt install nginx
```
 - Run `cd /etc/nginx` and remove the `nginx.conf` and make a new one
 - Add the following to `nginx.config` 
```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 1024;
        # multi_accept on;
}

http {

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        server {

                listen 80;

                client_max_body_size 4G;

                location / {
                        proxy_pass http://unix:/run/gunicorn.sock;
                        proxy_redirect off;
                        proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                        proxy_set_header X-Forwarded-Host $server_name;
                }
        }

        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}
```
 - `cd sites-enabled` and remove `default`
 - run ` systemctl reload nginx` and this should spin up your CTFd instance.
 - Paste the ip address of your droplet into your browser and you should be able to see your CTFd.
 - Optionally you can spin up a MySQL and a Redis database on DigitalOcean and modify ` CTFd/config.ini ` accordingly.
