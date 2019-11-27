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



## Initial Config

As i'm not at home for now, all config should be able to be done remotely

Initial steps to be able to work remotely:

- Setup a functional Raspberry Linux version (Raspbian **Lite**)
  -  https://www.raspberrypi.org/downloads/ 
  - Create SD card using Etcher
  - **Maybe**: create ssh file ('no extension') on the boot partition
  - Connect in ssh with default credentials: pi / raspberry 
  - change password: `passwd`
  - [generate](https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md) ssh key-pair for remote connection (easier) 
- On ISP router:
  - Forward port 22 to internal router 222
  - Forward port 443 to internal router 443
  - Forward port 8443 to internal router 8443 (web access to internal router config)
  - Forward temporarly port range ( just in case - no doable remotely )
- On internal router:
  - Bind RPI MAC address to <RPI IP> 
  - redirect 222 to <RPI IP>:22
  - redirect 443 to <RPI IP>:443

- Check if remote access to Raspberry is working
- Check if remote access to my router is working ( just in case - if I need to fine tune settings)

## Config

- Dyndns already created
- Reverse proxy will run on an unused Raspberry Pi

#TODO: check which version?

- Connection to anywhere has to be done through TLS encrypted https: 
  - target CryptCheck A or A+

- SSL/TLS certificates from Let's encrypt and renewed automatically