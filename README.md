# Home reverse proxy setup

## Main Idea

Setup a reverse proxy to gain access to: 

- My synology NAS *(without quickconnect)*
  - DSM
  - Moments
  - Surveillance station
  - Web station
  - Others services... (TBD)

- Home Automation Web server Jeedom *(without using included OpenVPN)*
- My Router config Page (Asus RT-AC68)
- My ISP Router
- Others (maybe create interface to add/remove/edit proxy settings)

via `https://<my domain>.ddns.net/path_to_service`

## Config

- Dyndns already created (today on ISP router)
  - Move to RPI ? ddclient
- Reverse proxy will run on an unused Raspberry Pi

#TODO: check which version?

- Connection to anywhere has to be done through TLS encrypted https: 
  - target CryptCheck A or A+

- SSL/TLS certificates from Let's encrypt and renewed automatically

## Initial Config (Rpi - routers)

As i'm not at home for now, all config should be able to be done remotely

Initial steps to be able to work remotely:

- Setup a functional Raspberry Linux version (Raspbian **Lite**)
  - https://www.raspberrypi.org/downloads/ 
  
  - Create SD card using [BalenaEtcher](https://www.balena.io/etcher/)
  
  - create **ssh** file ('no extension') on the boot partition
  
  - Connect in ssh with default credentials: pi / raspberry 
  
    ```bash
    ssh <USERNAME>@<PI ip>
    ```
  
  - change password: 
  
    ```bash
    passwd
    ```
  
  - On your computer [generate](https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md) ssh key-pair for remote connection (easier) 
  
    ```bash
    ssh-keygen
    ```
  
  -  key is generated in ~/.ssh/ named id_rsa (private key) and id_rsa_pub (public key to be copied to RPi)
  
  -  Copy id to RPI
  
     ```bash
     ssh-copy-id <USERNAME>@<IP-ADDRESS>
     ```
  
  -  Backup private key (just in case)
  
     ```bash
     cp ~/.ssh/id_rsa ~/<keyname.pem>
     ```
  
  - Disable ssh authentication by password on Rpi
  
    -  Edit sshd_config
  
    ```bash
    sudo nano /etc/ssh/sshd_config
    ```
  
    - Uncomment and set to no, the line:
  
    ```bash
    # To disable tunneled clear text passwords, change to no here!
    PasswordAuthentication no
    ```
  
    -  Restart ssh service
  
    ```bash
     service ssh restart
    ```
  
- On ISP router:
  
  - Forward port 22 to internal router 222
  - Forward port 443 to internal router 8443  (web access to internal router config)
  - Forward Temporarly port range 9800-9850 ( just in case - no doable remotely )
  
- On internal router:
  
  - Bind RPI MAC address to <RPI IP> 
  - redirect 222 to <RPI IP>:22

- Check if remote access to Raspberry is working

- Check if remote access to my router is working ( just in case - if I need to fine tune settings)

## Install Nginx

Inspired by this [article](https://engineerworkshop.com/2019/01/16/setup-an-nginx-reverse-proxy-on-a-raspberry-pi-or-any-other-debian-os/)

- Update packages

  ```bash
  sudo apt-get update
  sudo apt-get upgrade
  ```

- Install Nginx

  ```bash
  sudo apt-get install nginx
  ```

- Check if Nginx is running, by going in web browser to <local RPI IP>

  <img src="Nginx_running.png" style="zoom:50%;" />
  
- check which version is running:

  ```bash
  sudo nginx -v
  ```

- check error log file

  ```bash
  cat /var/log/nginx/error.log
  ```

## Install certbot for SSL certificates

- Install [certbot](https://certbot.eff.org/lets-encrypt/debianbuster-nginx)

  ```bash
  sudo apt-get install certbot python-certbot-nginx
  ```

- Run certbot for Nginx for <yourdomain.com> **NOT DONE**

  ```bash
  sudo certbot --nginx -d <yourdomain.com>
  ```

- Test certificate renewal  **NOT DONE**

  ```bash
  sudo certbot renew --dry-run
  ```

## Configure Nginx to:

1. Use the Let's Encrypt HTTPS certificates provided by certbot.

2. Automatically redirect HTTP to HTTPS

3. Close connections for any subdomains we're not trying to proxy



- Edit default config  **NOT DONE**

```bash
sudo nano /etc/nginx/sites-enabled/default 
```

```nginx
# Default HTTP server -> redirect to HTTPS
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 301 https://$host$request_uri;
}

# Default HTTPS server (just disconnect)
server {
    listen [::]:443 ssl ipv6only=on; 
    listen 443 ssl; 

    ssl_certificate /etc/letsencrypt/live/www.yourdomain.com/fullchain.pem; # provided by Certbot
    ssl_certificate_key /etc/letsencrypt/live/www.yourdomain.com/privkey.pem; # provided by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # provided by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # provided by Certbot

    server_name _;
    return 444;
}
```



- Configure redirections:

  - Go to Nginx sites-available folder :

    ```bash
    cd /etc/nginx/sites-available/
    ```

  - Create config file for <yourdomain.com>:

    ```bash
    sudo nano <yourdomain.com>.conf
    ```

  - Edit Nginx config file like this: **(DSM not working - Internal router not working - ISP not working)**

```nginx
server {
        client_max_body_size 2m;
        listen 80;
        listen [::]:80;
        server_name <yourdomain.com>;
        location /website1 {
                index index.php;
                proxy_pass http://192.168.xx.xx:80/path/;
        }
        location /phpmyadmin {
                proxy_pass http://192.168.xx.xx:80/phpMyAdmin;
        }
        location /jeedom/ {
                root /var/www/hmtl;
                index index.php;
                proxy_pass http://192.168.xx.xx:80/;
        }
		location /dsm {
                index index.cgi;
                rewrite ^/dsm/(.*)$ /$1 break;
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_pass http://192.168.xx.xx:5000/;
        }
        location /router {
                proxy_pass "http://192.168.x.xx:xxxx/";
        }
}      
```

- Link config file

```bash
sudo ln -s /etc/nginx/sites-available/<yourdomain.com>.conf /etc/nginx/sites-enabled/<yourdomain.com>.conf
```

- Test config files syntax

```bash
sudo nginx -t
```

- Reload Nginx configuration files:

```bash
sudo nginx -s reload
```

## 

## Manage Dynamic DNS from RPi (instead of ISP router)