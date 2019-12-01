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
  
    ```
    ssh <USERNAME>@<PI ip>
    ```
  
  - change password: 
  
    ```
    passwd
    ```
  
  - On your computer [generate](https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md) ssh key-pair for remote connection (easier) 
  
    ```
    ssh-keygen
    ```
  
  -  key is generated in ~/.ssh/ named id_rsa (private key) and id_rsa_pub (public key to be copied to RPi)
  
  -  Copy id to RPI
  
     ```
     ssh-copy-id <USERNAME>@<IP-ADDRESS>
     ```
  
  -  Backup private key (just in case)
  
     ```
     cp ~/.ssh/id_rsa ~/<keyname.pem>
     ```
  
  - Disable ssh authentication by password on Rpi
  
    -  Edit sshd_config
  
    ```
    sudo nano /etc/ssh/sshd_config
    ```
  
    - Uncomment and set to no, the line:
  
    ```
    # To disable tunneled clear text passwords, change to no here!
    PasswordAuthentication no
    ```
  
    -  Restart ssh service
  
    ```
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

  ```
  sudo apt-get update
  sudo apt-get upgrade
  ```

- Install Nginx

  ```
  sudo apt-get install nginx
  ```

- Check if Nginx is running, by going in web browser to <local RPI IP>

  <img src="Nginx_running.png" style="zoom:50%;" />

TO BE CONTINUED -----------------------------------------------------------

## Manage Dynamic DNS from RPi (instead of ISP router)





## Install certbot for SSL certificates

- Install certbot

  ```
  sudo apt-get install certbot python-certbot-nginx
  ```

- Run certbot for Nginx for <yourdomain.com>

  ```
  sudo certbot --nginx -d <yourdomain.com>
  ```

- Test certificate renewal

  ```
  sudo certbot renew --dry-run
  ```

## Configure Nginx to:

1. Use the Let's Encrypt HTTPS certificates provided by certbot.

2. Automatically redirect HTTP to HTTPS

3. Close connections for any subdomains we're not trying to proxy



- Edit default config

```
sudo nano /etc/nginx/sites-enabled/default 
```

 

```
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

  - Go to  Nginx sites-available folder :

    ```
    cd /etc/nginx/sites-available/
    ```

  - Create config file for <yourdomain.com>:

    ```
    sudo nano <yourdomain.com>.conf
    ```

  - Edit conf file like this:

example1:

```
server {
	listen 80;
	server_name example.com;
	location / {
	proxy_pass http://192.x.x.x;
	}
}
```

example 2

```
server {
    listen       443;

    ssl_certificate /etc/letsencrypt/live/www.yourdomain.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/www.yourdomain.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    server_name  orchid.yourdomain.com;
    location / {
        proxy_pass http://192.168.100.101;
    }
    location /service/streams/webrtc {

        proxy_read_timeout 31536000;
        proxy_pass http://192.168.100.101;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }
}
```



## 