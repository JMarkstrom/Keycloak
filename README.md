# Deploying Keycloak with SSL in just 10 minutes!
_These instructions provides detailed step-by-step instructions on setting up and running a **Keycloak Server** -complete with host-facing **SSL** AND without the need for a reverse proxy like Nginx (!)._

_In addition detailed step-by-step instructions are also provided for integrating the **SafeNet Keycloak Agent** with the Keycloak Server, including both the module and the SafeNet Authentication Service login theme. The goal of these instructions is to simplify configuration and reduce "time to market" when creating either a demo or test._

> :warning: **The instructions in this document must not be used towards setting up a production system!**

### Table Of Contents
  * [Prerequisites](https://github.com/JMarkstrom/Keycloak#prerequisites)
  * [Certificate generation with Lets Encrypt](https://github.com/JMarkstrom/Keycloak#certificate-generation-with-letsencrypt)
  * [Install & Configure Docker](https://github.com/JMarkstrom/Keycloak#install--configure-docker)
  * [Keycloak pre-configuration](https://github.com/JMarkstrom/Keycloak#keycloak-server-pre-configuration)
  * [Deploying the Keycloak Server](https://github.com/JMarkstrom/Keycloak#deploying-the-keycloak-server)
  * [Testing the solution](https://github.com/JMarkstrom/Keycloak#testing-the-solution)
  * [Appendix](https://github.com/JMarkstrom/Keycloak#appendix)


## Prerequisites
The following prerequisites must be met to successfully follow these instructions:

- Ubuntu 20.04 (_Example_ Azure Product ID: `0001-com-ubuntu-server-focal`)
  - DNS (FQDN resolvable)
  - FW/networking rules:
    - 22 (SSH)
    - 443 (SSL/TLS)
    - (80) 


## Certificate generation with Lets Encrypt
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

## Keycloak server pre-configuration
At this point certificates have been generated and Docker (all) have been installed and verified. Next, the certificate files and any additional configuration like the SafeNet theme or your own theme as well as any plugin/module needs to be moved or kept at path(s) where they can be consumed when running the solution.

1. Copy the certificate `.pem` files to a location where they can be read by Docker/Docker-Compose (observe example below)
2. Set and _test_ for permissions for the above file to ensure they can be accessed: `sudo chmod 655 ./keycloak/certs/*`

### Example files and file structure
Here is an example file structure (as used in the provided **Docker Compose** file [file](https://raw.githubusercontent.com/JMarkstrom/SafeNet-Keycloak-Agent/main/files/keycloak.yml) shared with this repository) showing location of certificate (`fullchain.pem`), key (`privkey.pem`), a custom module (`SafeNetOtpRealm.json`) and a custom theme (`sas-login-ui`)

> :warning: **The Lets Encrypt are not to be renamed or converted to** `.crt`**!**

```
├── keycloak.yml
├── dockerfile
├── certs
│   └── fullchain.pem
│   └── privkey.pem
├── safenet
    ├── SafeNetOtpRealm.json
    └── sas-login-ui
        └── login
            └── [...]
```

Here is an example **Docker Compose** file:

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

Lastly, here is a sample **Dockerfile**:

    FROM quay.io/keycloak/keycloak:latest


## Deploying the Keycloak Server
At this point, assuming all the files are in the requisite directories _and_ adequate permissions are set you are now ready to standup the service!

Run Docker with Keycloak: `sudo docker-compose -f keycloak.yml up`



# Testing the solution
To verify everything is working, open a browser and navigate to your keycloak host using `https://` e.g. `https://keycloak.swedemo.tk`

- The Keycloak main page displays without certificate errors ✔
- Clicking on **Administration Console** Keycloak login displays with your set theme (optional) ✔

![image](https://user-images.githubusercontent.com/57787248/146521524-4f3862c1-96ea-4e27-b21c-aaa4bbc01bcd.png)

![image](https://user-images.githubusercontent.com/57787248/146521326-f0153113-da16-4539-90a8-62ac3d70cf91.png)


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

