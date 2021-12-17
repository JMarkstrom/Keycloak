# HOWTO deploy the SafeNet Agent for Keycloak in just 10 minutes
_These instructions provides detailed step-by-step guidance on setting up the SafeNet Keycloak Agent with Keycloak server for use with SafeNet Authentication Service (SAS) and/or SafeNet Trusted Access (STA). The guide is intended to be used to simplify configuration and reduce "time to market" when creating either a demo or test._

> :warning: **The instructions in this document must not be used towards setting up a production system!**

## Prerequisites
The following prerequisites must be met to successfully follow these instructions:

- Ubuntu 20.x
  - Networking (_duh!_)
  - DNS (FQDN resolvable)
  - FW rules
    - 22 (SSH)
    - 443 (SSL/TLS)
    - (80 -disable later) 


## Certificate generation with Letsencrypt
_On the target Keycloak Server host perform the following steps:_

1. SSH to the Keycloak Server host
2. Run the following command to install the Letsencrypt Certbot: `sudo snap install --classic certbot`
4. Next, run the following command to ensure that the certbot command can now be run: `sudo ln -s /snap/bin/certbot /usr/bin/certbot`
5. Now run Certbot to start the certificate generation: `sudo certbot certonly --standalone`
6. Follow the on-screen instructions to generate a SSL/TLS certificate and private key.

> 🥉 You now have a **fullchain.pem** and a **privkey.pem** file for use with SSL/TLS

## Install & Configure Docker

### Set up the repository
_Before installing Docker and Docker Compose the repository must be configured_

1. Update the apt package index and install packages to allow apt to use a repository over HTTPS: 

    `sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release`

2. Add Docker’s official GPG key: 
    
   `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg`

4. Use the following command to set up the stable repository:

    `echo \ "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \ $(lsb_release -cs) stable"   | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`

### Install Docker Engine
_With the repository done, now install and verify Docker Engine installation_

1. Update (should be present) the apt package index: `sudo apt-get update`
2. Then install the latest version of Docker Engine and Containerd: `sudo apt-get install docker-ce docker-ce-cli containerd.io`
3. Start Docker and (optionally) set it as a service and adding a user:

    `sudo systemctl start docker
    sudo useradd userName
    sudo usermod -a -G docker userName
    sudo systemctl enable docker.service
    sudo systemctl enable containerd.service`

4. Ensure that Docker is now running: `sudo docker version`

### Install Docker Compose
_As a final step to Docker installation/configuration we install Docker Compose_

1. Run this command to download the current stable release of Docker Compose: 

    `sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose`

2. Apply executable permissions to the binary: `sudo chmod +x /usr/local/bin/docker-compose`
3. Now test the installation (returns version): `docker-compose --version`

> 🥈 You now have a **working Docker setup**. Next we do the magics!

## Deploying the Docker Server

Run Docker with Keycloak: `sudo docker-compose -f keycloak.yml up`



Here (below as well as in the link) is an example **Dockerfile** [file](https://raw.githubusercontent.com/JMarkstrom/SafeNet-Keycloak-Agent/main/files/dockerfile):

    FROM quay.io/keycloak/keycloak:latest


Here (below as well as in the link) is an example **Docker Compose** file [file](https://raw.githubusercontent.com/JMarkstrom/SafeNet-Keycloak-Agent/main/files/keycloak.yml):

    version: '3.7'

    services:

    keycloak:
       image: quay.io/keycloak/keycloak:latest
      container_name: keycloak
      restart: always
      ports:
        - "80:8080" # Listen to HTTP on host port 80 and forward to keycloak on 8080
        - "443:8443" # Listen to HTTPS on host port 443 and forward to keycloak on 443
     
     volumes:
        - "./tmp/safenet/SafenetOtpRealm.json:/tmp/safenet/SafenetOtpRealm.json" # Mount the SafeNet agent config 
         - "./tmp/safenet/sas-login-ui/:/opt/jboss/keycloak/themes/sas-login-ui/" # Mount the SAS/STA theme
       
     environment:
        - "KEYCLOAK_IMPORT:/tmp/SafenetOtpRealm.json" # This is the SafeNet KeyCloak agent configuration from mounted volume
        - JAVA_OPTS_APPEND="-D keycloak.profile.feature.upload_script=enabled"
        - KEYCLOAK_USER=admin
        - "KEYCLOAK_PASSWORD=password" # Change the password buddy!
        - AUTH_PROFILE=basicAuth
        - "KEYCLOAK_HOSTNAME=fqdn" # Set the hostname / FQDN
        - "KEYCLOAK_DEFAULT_THEME=sas-login-ui" # Here we make the SAS/STA theme default




> 🥇 Awesome and **we are done**!



# Appendix

### Checking the log for errors
If things aren't working, check the log: `sudo docker logs keycloak -f`

### Starting, stopping and removing the container

###### Start
`sudo docker start keycloak`

###### Stop
`sudo docker stop keycloak`
 
###### Remove
`sudo docker rm keycloak`

