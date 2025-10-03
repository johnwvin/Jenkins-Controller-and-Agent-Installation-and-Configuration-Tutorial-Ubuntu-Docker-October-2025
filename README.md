# Jenkins controller and agent installation and configuration with Ubuntu and Docker (https or http)

## Jenkins installation with http-only Setup:


### Ready the Environment:

```
mkdir -p ~/jenkins
cd ~/jenkins
nano compose.yml
```

### Paste this in the .yml:

```
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts-jdk21
    container_name: jenkins-controller
    restart: always
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_data:/var/jenkins_home

volumes:
  jenkins_data:
```

### start container: 

```
sudo docker compose up -d

```

copy temp admin password from logs:

```
sudo docker logs jenkins-controller

```

### GUI config

navigate to gui using the server's ip address and default port of 8080 and use the temp key to gain access. Go through initial setup and when prompted for a URL just give the server's IP address.



## Jenkins with trusted HTTPS certificate using Cloudflare:

#### NOTE: You must update your DNS records (at least locally) for this to work: jenkins.[YOUR.DOMAIN]

### create these directories:

```
mkdir -p ~/jenkins
mkdir -p ~/jenkins/cert
mkdir -p ~/jenkins/p12-key
```

### install OpenSSL and Certbot with its Cloudflare plugin:

```
sudo apt install openssl
sudo apt install certbot python3-certbot-dns-cloudflare

```

### Get your API key from Cloudflare's website:

My Profile > API Tokens > Create Token > Edit zone DNS template > zone resource = your.domain > continue & create

### store api key on the jenkins server:

```
echo -e "dns_cloudflare_api_token = [REPLACE WITH YOUR API TOKEN !!!]" | sudo tee -a ~/jenkins/cert/cloudflare.ini

```

### Secure the token:

```
sudo chmod -R 600 ~/jenkins/cert

```

### Request the certificate:

```
sudo certbot certonly \
 --dns-cloudflare \
 --dns-cloudflare-credentials ~/jenkins/cert/cloudflare.ini \
 -d jenkins.[YOUR.DOMAIN !!!] \
 --agree-tos \
 -m [YOUR EMAIL ADDRESS !!!]

```

### Generate a PKCS #12 keystore to merge the letsencrypt certificate and key as a pair given by certbot (it will prompt you for a password, note it):

```
sudo openssl pkcs12 \
-inkey /etc/letsencrypt/live/jenkins.[YOUR.DOMAIN !!!]/privkey.pem \
-in /etc/letsencrypt/live/jenkins.[YOUR.DOMAIN !!!]/fullchain.pem \
-export -out ~/jenkins/p12-key/jenkins.p12
# replace placeholders

```
### Once the key is generated, give ownership to the Jenkins user:

```
sudo chown 1000:1000 ~/jenkins/p12-key/jenkins.p12

```

### create the Jenkins compose.yml file:

```

sudo nano ~/jenkins/compose.yml

```

Paste this into YAML file (replace placeholders w/ your info):

```
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts-jdk21
    container_name: jenkins-controller
    restart: always

    # Pass arguments to Jenkins to enable HTTPS(adds args to the entrypoint 
    # executable during execution)
    # 
    # this ">" is a folded block scalar that allows us to have these args on  
    # multiple lines(new lines = spaces in contrast with "|", a literal block
    # scalar, that causes YAML to treat a new line as a new line or "insert \n")
    # and the "command:" key is used here to pass args to the entrypoint (in
    # this case a shell script. If there were no entrypoint it would simply be the 
    # only thing executed by this file)

    command: >

      --httpPort=-1
      --httpsPort=443
      --httpsKeyStore=/etc/p12-key/jenkins.p12
      --httpsKeyStorePassword=[REPLACE_WITH_P12_PASSWORD !!!]

    ports:
      # Map host port 443 to container port 443 for HTTPS
      - "443:443"
      # 50000 is still needed for JNLP/Agents
      - "50000:50000"

    volumes:
      - jenkins_data:/var/jenkins_home
      - /home/[YOUR USERNAME !!!]/jenkins/p12-key/:/etc/p12-key:ro # ":ro" = read-only

volumes:
  jenkins_data:

```

### start server

```
cd ~/jenkins
sudo docker compose up -d
```

copy temp admin password from logs:

```
sudo docker logs jenkins-controller -f

```

### GUI configuration

Navigate to GUI using your URL you gave the IP in your DNS records and use the temp admin password to gain access. Go through the initial setup and when prompted for a URL give the same URL you used in your browser, making sure you prepend it with "https://".


## Create a Jenkins Agent (JNLP Method)


### Ready the Ubuntu Machine

1. Navigate to your other Ubuntu machine you will designate to serve as a Jenkins agent

2. Perform Update & Upgrade

3. install jdk21:

```
sudo apt install openjdk-21-jre -y

```

4. Create the agent directory in your home directory:

```
mkdir ~/agent

```

### Create the agent node in the Jenkins GUI

   ~ Navigate to Manage Jenkins > Nodes in the controller's GUI
   ~ Click create agent
   ~ Give a name and select permanent agent then continue
   ~ Select number of executers
   ~ Remote root directory needs to be as we configured: /home/[YOUR USERNAME !!!]/agent
   ~ label it as something such as agent-1
   ~ Save

### Connect the Agent
   ~ click on the agent in the nodes list
   
   ~ copy and paste the "Run from agent command line: (Unix)" command into your agent's Ubuntu CLI(make sure the agent can reach the controller's IP address or URL) into the agent dir(/home/[YOUR USERNAME !!! ]/agent)

### Make a systemd process to run the agent in the background and on reboot:

1. Create the service file:

```
sudo nano /etc/systemd/system/jenkins-agent.service
```

#### Paste this into the file and replace the placeholders with your info (your secret key can be found in the 'unix command' we got from the jenkins GUI):

```
[Unit]
Description=Jenkins Agent
After=network.target

[Service]
User=[YOUR USERNAME !!!]
WorkingDirectory=/home/[YOUR USERNAME !!!]/agent
ExecStart=/usr/bin/java -jar agent.jar -url [YOUR URL OR IP ADDRESS (SPECIFY HTTP:// OR HTTPS://) !!!] -secret [YOUR SECRET !!!] -name "[YOUR AGENT'S NAME !!!]" -webSocket -workDir "/home/[YOUR USERNAME !!!]/agent"
Restart=always

[Install]
WantedBy=multi-user.target

```

2. Start and enable the service:

```
sudo systemctl daemon-reload
sudo systemctl enable jenkins-agent.service
sudo systemctl start jenkins-agent.service

```

3. Check status:

```
sudo systemctl status jenkins-agent.service

```

4. Check connection status on node agent page in GUI (should say 'Agent is connected')



