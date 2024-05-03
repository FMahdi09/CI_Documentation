# Instructions

## VPC

1. Create a VPC with the following values

    - Name tag: CI-vpc
    - IPv4 CIDR: 10.0.0.0/16

## Internet Gateway

1. Create a internet gateway with the following values

    - Name tag: CI-igw

2. Attach the internet gateway to our VPC

    - Select our internet gateway > Actions > Attach to VPC > select our VPC

## Subnet

for simplicity only one public subnet

1. Create a public subnet with the following values

    - VPC ID: our VPC (CI-vpc)
    - Subnet name: CI-subnet-public
    - Availability zone: us-east-1a
    - IPv4 subnet CIRD block: 10.0.0.0/24

## Route Table

1. Create a route table with the following values

    - Name: CI-route-public
    - VPC: our VPC (CI-vpc)

2. Edit the routes so they look like the following

```
10.0.0.0/16 -> local
0.0.0.0/0 -> CI-igw
```

3. Associate our public route table with our public subnet

    - Select our public route table > Actions > Edit subnet associations > select our public subnet > save associations

##  Securtiy Groups

### DNS Server

1. Create a security group for our DNS servers with the following values
    - Securtiy group name: CI-security-dns
    - Description: IN: SSH, DNS; OUT: HTTPS; HTTP
    - VPC: our VPC (CI-vpc)

2. Give the security group the following rules:
    - IN: SSH port 22, Source: 0.0.0.0/0 (anywhere IPv4)
    - IN: TCP port 53, Source: 0.0.0.0/0 (anywhere IPv4)
    - IN: UDP port 53, Source: 0.0.0.0/0 (anywhere IPv4)
    - OUT: HTTPS port 443, Source: 0.0.0.0/0 (anywhere IPv4)
    - OUT: HTTP port 80, Source: 0.0.0.0/0 (anywhere IPv4)

## DNS Servers

1. Launch a EC2 instance for our primary dns with the following values
    - Name: CI-ec2-dns1
    - OS: Ubuntu 22.04 LTS, 64-bit (x86)
    - Instance Type: t2.micro (1 vCPU, 1 GiB Memory)
    - Key pair name: vockey (AWS lab default key)
    - Storage: 8 GiB

    Under Networksettings select the following
    - VPC: our VPC (CI-vpc)
    - Subnet: our private subnet (CI-subnet-public)
    - Auto-assing public IP: Enable
    - Select exisiting security group: CI-security-dns
    - Primary IP: 10.0.0.10

2. Launch a EC2 instance for our secondary dns with the following values
    - Name: CI-ec2-dns2
    - OS: Ubuntu 22.04 LTS, 64-bit (x86)
    - Instance Type: t2.micro (1 vCPU, 1 GiB Memory)
    - Key pair name: vockey (AWS lab default key)
    - Storage: 8 GiB

    Under Networksettings select the following
    - VPC: our VPC (CI-vpc)
    - Subnet: our private subnet (CI-subnet-public)
    - Auto-assing public IP: Enable
    - Select exisiting security group: CI-security-dns
    - Primary IP: 10.0.0.11

3. Activate the ssh-agent
    - start the ssh-agent
    ```
    eval $(ssh-agent -s) 
    ```
    - add your key to the ssh-agent
    ```
    ssh-add pathToYourKey
    ```

4. Connect to your dns servers with the following command
    ```
    ssh -A ubuntu@DnsIpAddressHere
    ```

    Ideally you have two shells open. One connected to each dns server.
    
5. Execute the following commands on both DNS servers
    - update the apt package cache
    ```
    sudo apt update
    ```
    - inistall BIND on each machine
    ```
    sudo apt install bind9 bind9utils bind9-doc
    ```
    - set BIND to IPv4 mode. Edit the "named" default settigs file.
    ```
    sudo vim /etc/default/named
    ```
    Add -4 at the end of the OPTIONS paramter
    ```
    . . .
    OPTIONS="-u bind -4"
    ```
    - Restart BIND to implement the changes
    ```
    sudo systemctl restart bind9
    ```

6. Configure the primary DNS server (only do these steps on the primary dns server)
    - open the named.conf.options file for editing
    ```
    sudo vim /etc/bind/named.conf.options
    ```
    - above the exisiting options block create a ACL block called trusted like so:
    ```
    acl "trusted" {
        10.0.0.10;          # dns1 
        10.0.0.11;          # dns2 
        10.0.0.12;          # git 
        10.0.0.13;          # runner 
    };

    options {

        . . .
    ```
    - now edit the options block. under the directory directive add the following configuration lines
    ```
            . . .

    };

    options {
            directory "/var/cache/bind";
            
            recursion yes;                  # enables recursive queries
            allow-recursion { trusted; };   # allows recursive queries from "trusted" clients
            listen-on { 10.0.0.10; };       # ns1 private IP address - listen on private network only
            allow-transfer { none; };       # disable zone transfers by default

            forwarders {
                    8.8.8.8;
                    8.8.4.4;
            };

            . . .
    };
    ```
    now save and close the named.conf.options file.

    - open the named.conf.local file for editing
    ```
    sudo vim /etc/bind/named.conf.local
    ```
    - Apart from comments the file will be empty. Add the forward zone with the following lines
    ```
    zone "ci.com" {
        type primary;
        file "/etc/bind/zones/db.ci.com";       # zone file path
        allow-transfer { 10.0.0.11; };              # ns2 private IP address - secondary
    };
    ```
    - Now add the reverse zone with the following lines
    ```
    zone "0.10.in-addr.arpa" {
        type primary;
        file "/etc/bind/zones/db.10.0";     # 10.0.0.0/16 subnet
        allow-transfer { 10.0.0.11; };      # ns2 private IP address - secondary
    };
    ```
    now save and close the named.conf.local file.

    - create the forward zone file (for DNS lookups) by first creating the directory where your zone files will reside
    ```
    sudo mkdir /etc/bind/zones
    ```
    - our forward zone file will be base on db.local. Copy it to the proper location
    ```
    sudo cp /etc/bind/db.local /etc/bind/zones/db.ci.com
    ```
    - open the forward zone file for editing
    ```
    sudo vim /etc/bind/zones/db.ci.com
    ```
    - initially it is going to look like this
    ```
    $TTL    604800
    @       IN      SOA     localhost. root.localhost. (
                                2           ; Serial
                            604800          ; Refresh
                            86400           ; Retry
                            2419200         ; Expire
                            604800 )        ; Negative Cache TTL
    ;   
    @       IN      NS      localhost.      ; delete this line
    @       IN      A       127.0.0.1       ; delete this line
    @       IN      AAAA    ::1             ; delete this line
    ```
    - edit the zone file so it looks like this
    ```
    ;
    ; BIND data file for local loopback interface
    ;
    $TTL    604800
    @       IN      SOA     dns1.ci.com. admin.ci.com. (
                                3         ; Serial
                            604800         ; Refresh
                            86400         ; Retry
                            2419200         ; Expire
                            604800 )       ; Negative Cache TTL
    ;
    ; name servers - NS records
        IN      NS      dns1.ci.com.
        IN      NS      dns2.ci.com.

    ; name servers - A records
    dns1.ci.com.          IN      A       10.0.0.10
    dns2.ci.com.          IN      A       10.0.0.11

    ; 10.0.0.0/16 - A records
    git.ci.com.           IN      A      10.0.0.12
    runner.ci.com.        IN      A      10.0.0.13
    ```
    now save and close the db.devops.com file.

    - create the reverse zone file for DNS reverse lookups. Our revers zone file will be base on db.127. Copy it to the proper location
    ```
    sudo cp /etc/bind/db.127 /etc/bind/zones/db.10.0
    ```
    - open the reverse zone file for editing
    ```
    sudo vim /etc/bind/zones/db.10.0
    ```
    - initially it is going to look like this
    ```
    $TTL    604800
    @       IN      SOA     localhost. root.localhost. (
                                1           ; Serial
                            604800          ; Refresh
                            86400           ; Retry
                            2419200         ; Expire
                            604800 )        ; Negative Cache TTL
    ;
    @       IN      NS      localhost.      ; delete this line
    1.0.0   IN      PTR     localhost.      ; delete this line
    ```
    - edit the zone file so it looks like this
    ```
    ;
    ; BIND reverse data file for local loopback interface
    ;
    $TTL    604800
    @       IN      SOA     ci.com. admin.ci.com. (
                                3         ; Serial
                            604800         ; Refresh
                            86400         ; Retry
                            2419200         ; Expire
                            604800 )       ; Negative Cache TTL
    ; name servers
        IN      NS      dns1.ci.com.
        IN      NS      dns2.ci.com.

    ; PTR Records
    10.0   IN      PTR     dns1.ci.com.    ; 10.0.0.10
    11.0   IN      PTR     dns2.ci.com.    ; 10.0.0.11
    12.0   IN      PTR     git.ci.com.    ; 10.0.0.12
    13.0   IN      PTR     runner.ci.com.    ; 10.0.0.13
    ```
    now save and close the reverse zone file.

    - now check the syntax of your editied files with the following command. If you get no error messages everything is in order.
    ```
    sudo named-checkconf
    ``` 

    - now check the correctness of your zone files with the following commands
    ```
    sudo named-checkzone ci.com /etc/bind/zones/db.ci.com
    ```
    ```
    sudo named-checkzone 0.10.in-addr.arpa /etc/bind/zones/db.10.0
    ```
    - if your zone files have no errors restart BIND
    ```
    sudo systemctl restart bind9
    ```

    - Then allow DNS connections to the server by altering the UFW firewall rules
    ```
    sudo ufw allow Bind9
    ```

7. Configure the secondary DNS server (only do these steps on the secondary dns server)
    - open the named.conf.options file for editing
    ```
    sudo vim /etc/bind/named.conf.options
    ```
    - above the exisiting options block add the ACL block with all trusted IP-addresses:
    ```
    acl "trusted" {
        10.0.0.10;          # dns1 
        10.0.0.11;          # dns2 
        10.0.0.12;          # git 
        10.0.0.13;          # runner 
    };

    options {

        . . .
    ```
    - now edit the options block. under the directory directive add the following configuration lines
    ```
            . . .

    };

    options {
            directory "/var/cache/bind";
            
            recursion yes;                  # enables recursive queries
            allow-recursion { trusted; };   # allows recursive queries from "trusted" clients
            listen-on { 10.0.0.11; };       # ns2 private IP address - listen on private network only
            allow-transfer { none; };       # disable zone transfers by default

            forwarders {
                    8.8.8.8;
                    8.8.4.4;
            };

            . . .
    };
    ```
    now save and close the named.conf.options file. Except for the listen-on IP-address this file should be identical to the same file on dns1.

    - open the named.conf.local file for editing
    ```
    sudo vim /etc/bind/named.conf.local
    ```
    - define the secondary zones that correspond to the primary zones on the DNS server. The added zones should look like this:
    ```
    zone "ci.com" {
        type secondary;
        file "db.ci.com";
        primaries { 10.0.0.10; };  # ns1 private IP
    };

    zone "0.10.in-addr.arpa" {
        type secondary;
        file "db.10.0";
        primaries { 10.0.0.10; };  # ns1 private IP
    };
    ```
    now save and close the named.conf.local file.

    - check the validity of your config files with the following command
    ```
    sudo named-checkconf
    ```
    - if the command returns no errors restart BIND
    ```
    sudo systemctl restart bind9
    ```
    - Then allow DNS connections to the server by altering the UFW firewall rules
    ```
    sudo ufw allow Bind9
    ```

## Forgejo Server

1. Launch a EC2 instance with the following values
    - Name: CI-ec2-git
    - OS: Ubuntu 22.04 LTS, 64-bit (x86)
    - Instance Type: t2.medium (1 vCPU, 1 GiB Memory)
    - Key pair name: vockey (AWS lab default key)
    - Storage: 8 GiB
    - Under Networksettings select the following

    Under Networksettings select the following
    - VPC: our VPC (CI-vpc)
    - Subnet: our public subnet (CI-subnet-public)
    - Auto-assing public IP: Enable
    - Select exisiting security group: CI-security-git
    - Primary IP: 10.0.0.12

2. Connect to your instance and setup DNS

3. Install the docker engine

4. Setup Caddy for HTTPS

    - Get a domain which points to the public ip of the forgejo server (I will refer to the domain as DOMAIN in the future)

    - create a caddy directory

    ```
    mkdir caddy
    ```

    - step into the caddy dir and create a "Caddyfile"

    ```
    vim Caddyfile
    ```

    - edit the Caddyfile so that it looks like the following:

    ```
    {
        email <YOUR EMAIL ADDRESS>
        admin off
    }

    <DOMAIN> {
        reverse_proxy 127.0.0.1:3000
    }
    ```

    close and save the Caddyfile

    - add a docker-compose.yml file

    ```
    vim docker-compose.yml
    ```

    - edit the docker compose file so that is loooks like the following:

    ```
    services:
    	caddy:
            image: "caddy:2.7.6-alpine"
            network_mode: "host"
            container_name: "caddy"
            logging:
                driver: "json-file"
                options:
                    max-size: "10m"
                    max-file: "10"
            volumes:
                - "./Caddyfile:/Caddyfile:ro"
                # This allows Caddy to cache the certificates.
                - "/data/caddy:/data:rw"
            command: "caddy run --config /Caddyfile --adapter caddyfile"
            restart: "unless-stopped"
    ```

    - start the caddy service

    ```
    sudo docker compose up -d
    ```

    - now the server should be accessible at "https://DOMAIN" (It should just show an empty page because nothing is listening for our request at port 3000)

5. Setup Forgejo

    - go back to the main directory and create a forgejo directory

    ```
    mkdir forgejo
    ```

    - step into the created directory and create the "app.ini" file

    ```
    vim app.ini
    ```

    - edit the file so it looks like the following

    ```
    APP_NAME = git
    RUN_USER = git
    RUN_MODE = prod
    WORK_PATH = /var/lib/forge

    [server]
    SSH_DOMAIN = DOMAIN
    HTTP_PORT = 3000
    ROOT_URL = https://DOMAIN
    DISABLE_SSH = true
    ; In rootless gitea container only internal ssh server is supported
    START_SSH_SERVER = true
    SSH_PORT = 2222
    SSH_LISTEN_PORT = 22
    BUILTIN_SSH_SERVER_USER = git

    [database]
    DB_TYPE = sqlite3
    HOST = localhost:3306
    NAME = forge
    USER = root
    PASSWD =

    [service]
    REQUIRE_SIGNIN_VIEW = false

    [actions]
    ENABLED = true
    DEFAULT_ACTIONS_URL = https://github.com
    ```

    - create script "setup.sh" which will setup the rootless work directory that Forgejo will use

    ```
    vim setup.sh
    ```

    - edit the file so it looks like the following

    ```
    set -e

    mkdir -p work
    mkdir -p work/data

    chown -R 1000:1000 work/data
    chmod 775 work/data
    chmod g+s work/data

    chown 1000:1000 app.ini
    chmod 775 app.ini
    chmod g+s app.ini
    ```

    - run the setup file

    ```
    bash setup.sh
    ```

    - add a docker-compose.yml file

    ```
    vim docker-compose.yml
    ```

    - edit the file so it looks like the following

    ```
    networks:
        forgejo:
            external: false

        services:
            gitea:
                image: 'codeberg.org/forgejo/forgejo:1.21-rootless'
                container_name: 'forgejo'
                environment:
                    USER_UID: '1000'
                    USER_GID: '1000'
                    FORGEJO_WORK_DIR: '/var/lib/forge'
                user: '1000:1000'
                networks:
                    - forgejo
                ports:
                    - '3000:3000'
                    - '2222:22'
                volumes:
                    - './app.ini:/etc/gitea/app.ini'
                    - './data:/data:rw'
                    - '/etc/timezone:/etc/timezone:ro'
                    - '/etc/localtime:/etc/localtime:ro'
                    # Depends on `FORGEJO_WORK_DIR`.
                    - './work:/var/lib/forge:rw'
                logging:
                    driver: "json-file"
                    options:
                        max-size: "10m"
                        max-file: "10"
                restart: 'unless-stopped'
    ```

    - now start forgejo

    ```
    sudo docker compose up -d
    ```

    - now go to the web GUI at https://DOMAIN and complete the initialization there

    Note that the first registered account will automatically become the admin account


## Docker Engine

Guide to install the Docker Engine on Ubuntu

1. Uninstall all old versions
```
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

2. Setup Docker's apt repository

```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

3. Install the lastes Version

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

4. Verify the installation

```
sudo docker run hello-world
```

## Setup DNS

- get the device associated with your private network
```
ip address show to 10.0.0.0/16
```
```
OUTPUT:
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    inet 10.0.0.10/24 metric 100 brd 10.0.0.255 scope global dynamic eth0
    valid_lft 2160sec preferred_lft 2160sec
```
Here our interface is eth0.

- create a new file called "00-private-nameservers.yaml" and open it
```
sudo vim /etc/netplan/00-private-nameservers.yaml
```
- inside the file add the following
```
network:
    version: 2
    ethernets:
        eth0:                                    # Private network interface
            nameservers:
                addresses: [10.0.0.10, 10.0.0.11]
                search: [ ci.com ]    # DNS zone
            dhcp4-overrides:
                use-dns: false
                use-domains: false
```
now save and close this file.

- open resolved.conf
```
sudo vim /etc/systemd/resolved.conf
```

- add options under [Resolve] for DNS and Domain
```
[Resolve]
DNS=8.8.8.8 8.8.4.4
Domains=~.
```

- execute the following command to validate the new configuration. If there are no errors press ENTER to accept the new configuration.
```
sudo netplan generate
```
now reboot your system.
- Now, check that the system’s DNS resolver to determine if your DNS configuration has been applied
```
sudo resolvectl status
```
A part of the output should look something like this:
```
Link 2 (eth0)
Current Scopes: DNS
        Protocols: +DefaultRoute +LLMNR -mDNS -DNSOverTLS
                DNSSEC=no/unsupported
Current DNS Server: 10.0.1.11
    DNS Servers: 10.0.1.11 10.0.1.12
    DNS Domain: devops.com ec2.internal
```

## Runner

1. Launch a EC2 instance with the following values
    - Name: CI-ec2-runner
    - OS: Ubuntu 22.04 LTS, 64-bit (x86)
    - Instance Type: t2.micro (1 vCPU, 1 GiB Memory)
    - Key pair name: vockey (AWS lab default key)
    - Storage: 8 GiB
    - Under Networksettings select the following

    Under Networksettings select the following
    - VPC: our VPC (CI-vpc)
    - Subnet: our public subnet (CI-subnet-public)
    - Auto-assing public IP: Enable
    - Select exisiting security group: CI-security-runner
    - Primary IP: 10.0.0.13

2. Setup DNS

3. Install the docker engine

4. Create a docker-compose file

```
vim docker-compose.yml
```

5. Edit it so it looks like the following

```
services:
  runner:
    image: 'code.forgejo.org/forgejo/runner:3.3.0'
    container_name: 'runner'
    volumes:
      - ./data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: 'unless-stopped'
    command: '/bin/sh -c "while : ; do sleep 1 ; done ;"
```

8. Start the container

```
sudo docker compose up -d
```

9. Enter into the container of the runner

```
sudo docker exec -it runner /bin/bash
```

10. Register the runner

```
forgejo-runner register
```

    - URL: https://DOMAIN
    - runner token: retrieve from https://DOMAIN/user/settings/actions/runners
    - name: any name
    - label: ubuntu-22.04:docker://ghcr.io/catthehacker/ubuntu:act-22.04

Now the runner should be visible at https://DOMAIN/user/settings/actions/runners as Offline

11. Make the runner go online

    - edit the command line in the docker compose file so it looks like the following

    ```
    command: 'forgejo-runner daemon"'
    ```

    - restart the runner docker container

Now the runner should be displayed as idle.


## SonarQube

1. Launch a EC2 instance with the following values
    - Name: CI-ec2-sonarqube
    - OS: Ubuntu 22.04 LTS, 64-bit (x86)
    - Instance Type: t2.medium (2 vCPU, 4 GiB Memory)
    - Key pair name: vockey (AWS lab default key)
    - Storage: 8 GiB

    Under Networksettings select the following
    - VPC: our VPC (CI-vpc)
    - Subnet: our public subnet (CI-subnet-public)
    - Auto-assing public IP: Enable
    - Select exisiting security group: CI-security-git
    - Primary IP: 10.0.0.14

3. Install the docker engine (see above)

4. Setup Caddy for HTTPS in a new directory caddy (see above)

5. Setup SonarQube

    - go back to the main directory and create a sonarqube directory

    ```
    mkdir sonarqube
    ```

    - add a docker-compose.yml file

    ```
    vim docker-compose.yml
    ```

    - edit the file so it looks like the following

    ```
    services:
      sonarqube:
        image: sonarqube:10.4.1-community
        depends_on:
          - db
        environment:
          SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
          SONAR_JDBC_USERNAME: sonar
          SONAR_JDBC_PASSWORD: sonar
        volumes:
          - sonarqube_data:/opt/sonarqube/data
          - sonarqube_extensions:/opt/sonarqube/extensions
          - sonarqube_logs:/opt/sonarqube/logs
        ports:
          - "9000:9000"
        restart: unless-stopped
    
      db:
        image: postgres:12
        environment:
          POSTGRES_USER: sonar
          POSTGRES_PASSWORD: sonar
        volumes:
          - postgresql:/var/lib/postgresql
          - postgresql_data:/var/lib/postgresql/data
        restart: unless-stopped
    
    volumes:
      sonarqube_data:
      sonarqube_extensions:
      sonarqube_logs:
      postgresql:
      postgresql_data:
    ```

    - now start sonarqube

    ```
    sudo docker compose up -d
    ```

    - now go to the web GUI of sonarqube at the configured host address and keep from there
  
6. Configure connection between SonarQube and Forgejo

    - In SonarQube GUI: reset the password, choose the option to create a project manually and pick GitHub Actions
    - Create necessary Tokens in Forgejo
    - Modify /workflows/build.yml file in Forgejo repository to include Sonarqube at linting stage:
   
    ```
    name: ci
    
    on:
      push:
        branches:
          - "main"
    
    jobs:
      lint:
        name: Static Code Analysis
        runs-on: ubuntu-22.04
        permissions: read-all
        steps:
          - name: Checkout Code
            uses: actions/checkout@v4
            with:
              fetch-depth: 0
          - name: SonarQube Scan
            uses: sonarsource/sonarqube-scan-action@v2.0.2
            env:
              SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
              SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
                
      build:
        needs: lint
        runs-on: ubuntu-22.04
        steps:
          -
            name: Checkout
            uses: actions/checkout@v4
          -
            name: Login to Docker Hub
            uses: docker/login-action@v3
            with:
              username: ${{ secrets.DOCKERHUB_USERNAME }}
              password: ${{ secrets.DOCKERHUB_TOKEN }}
          -
            name: Set up Docker Buildx
            uses: docker/setup-buildx-action@v3
          -
            name: Build and push
            uses: docker/build-push-action@v5
            with:
              context: .
              file: ./Dockerfile
              push: true
              tags: ${{ secrets.DOCKERHUB_USERNAME }}/ci_server:latest
    ```

    - outcome should be visible in SonarQube Gui
