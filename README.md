# Deploying Keycloak with SSL in just 10 minutes!
_These instructions provides detailed step-by-step instructions on setting up and running a **Keycloak Server** -complete with host-facing **SSL** AND without the need for a reverse proxy like nginx (!)._

_In addition detailed step-by-step instructions are also provided for integrating the **SafeNet Keycloak Agent** with the Keycloak Server, including both the module and the SafeNet Trusted Access login theme. The goal of these instructions is to simplify configuration and reduce "time to market" when creating either a demo or test._

> :warning: **The instructions in this document must not be used towards setting up a production system!**

## Prerequisites
The following prerequisites must be met to successfully follow these instructions:

- Ubuntu 20.04 (_Example_ Azure Product ID: `0001-com-ubuntu-server-focal`)
  - DNS (FQDN resolvable)
  - FW/networking rules:
    - 22 (SSH)
    - 443 (SSL/TLS)
    - (80) 


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
        - 80:8080
        - 443:8443
     
      volumes:
         - ./safenet/SafeNetOtpRealm.json:/tmp/safenet/SafeNetOtpRealm.json
        - ./safenet/sas-login-ui/:/opt/jboss/keycloak/themes/sas-login-ui/
        - ./certs/fullchain.pem:/etc/x509/https/tls.crt"
        - ./certs/privkey.pem:/etc/x509/https/tls.key
       
      environment:
        - KEYCLOAK_IMPORT:/tmp/SafeNetOtpRealm.json
        - JAVA_OPTS_APPEND="-D keycloak.profile.feature.upload_script=enabled"
        - KEYCLOAK_USER=admin
        - KEYCLOAK_PASSWORD=password
        - AUTH_PROFILE=basicAuth
        - "KEYCLOAK_HOSTNAME=fqdn"
        - "KEYCLOAK_DEFAULT_THEME=sas-login-ui"




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

