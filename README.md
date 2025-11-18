# Wireguard Project
*Calvin Wasilevich*

## Setup DigitalOcean Droplet

The first step in setting up a remote VPN with Wireguard is starting a remote server to connect to. This is done with [DigitalOcean](https://www.digitalocean.com/), a cloud infrastructure provider.

After setting up the account, navigate to "Create a Droplet". "Droplet" is the term used by DigitalOcean to describe their virtual machines. Spin up a droplet with the following specifications:
- The closest region to you
- Ubuntu 24.04 (LTS) x64 
- Basic shared CPU
- Regular CPU, $6/mo option is sufficient

Then, configure an ssh key pair to allow a secure ssh connection into the vm.
- Use the command `ssh-keygen` to generate a public/private key pair
   - Follow all instructions given by this command
- Upload the public key to the VM before starting it

SSH into the VM by using `ssh -i <path/to/private/key> root@<VM ipv4 address>` 

## Install Docker

- Update packages with `apt update` and `apt upgrade`
- Follow the instructions on [docker's website](https://docs.docker.com/engine/install/ubuntu/) to install docker engine
    - Uninstall all unofficial docker installs with 
    ```bash
    sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
    ```

    - Setup docker's apt repository
    ```bash
    # Add Docker's official GPG key:
    sudo apt update
    sudo apt install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
    Types: deb
    URIs: https://download.docker.com/linux/ubuntu
    Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
    Components: stable
    Signed-By: /etc/apt/keyrings/docker.asc
    EOF

    sudo apt update
    ```

    - Install the latest version of docker with
    ```bash
    sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

    - Verify docker install with
    ```bash
    sudo docker run hello-world
    ```

    ![Working docker run](./screenshots/docker%20verify.png)

## Create Wireguard Image with Docker

- Create the following directory structure with `mkdir -p wireguard/config` \
![file structure](./screenshots/file%20structure.png)
- Make the file `wireguard/compose.yml` and paste the following code:
```yaml
version: '3.8'
services:
  wireguard:
    container_name: wireguard
    image: linuxserver/wireguard
    environment:
      - PUID=1000
      - PGID=1000
      - TZ= # set this line with the time zone
      - SERVERURL= # set this line with the server url 
      - SERVERPORT=51820
      - PEERS=2 # set this line with the number of peers
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.0.0.0
    ports:
      - 51820:51820/udp
    volumes:
      - type: bind
        source: ./config/
        target: /config/
      - type: bind
        source: /lib/modules
        target: /lib/modules
    restart: always
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
```
- Modify the commented areas:
    - `TZ` is the timezone; consult [this page](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) for the proper format
    - `SERVERURL` is the ip address of the cloud VM
    - `PEERS` is the original number of clients to make configuration files for. For this project, we only need two devices, so two is set for peers.
- Run `sudo docker compose up -d` while in the `wireguard` directory to start the image
   - The `-d` indicates to run in detached mode
   ![docker compose up](./screenshots/docker%20compose%20up.png)


## Connecting to Wireguard

### Windows Laptop

- Download and install the proper wireguard client from the [wireguard website](https://www.wireguard.com/install/)
- Get the peer configuration file from wireguard
    - In the `wireguard` directory in the VM, locate the `config/peer1` directory
    - In this directory, copy the `peer1.conf` file (highlighted in green) to the windows machine

    ![configuration file](./screenshots/config%20file.png)

    - Use the `scp` command to use ssh to tranfer the file
       ```sh
       scp -i <path/to/private/key> root@<VM ipv4 address>:</path/to/config/file> .
       ```
- From the Wireguard client, add a tunnel from configuration file with the newly copied file
- Click activate


Before Connection: ![before connection](./screenshots/disconnect%20ip.png)
After Connection: ![after connection](./screenshots/connect%20ip.png)

### Android Device

- Download and install the wireguard client app from the Google Play Store
- Get the peer configuration file from wireguard
    - In the `wireguard` directory in the VM, locate the `config/peer2` directory
    - In the this directory, copy the `peer2.png` file
        - This is a QR code for easy tunnel setup
    - Use the `scp` command as in the windows setup
- From the Wireguard app, select the plus and then select the option to scan from QR code
- Scan the QR code downloaded from the server
- Activate the VPN connection

Before Connection: \
<img src="./screenshots/mobile%20disconnected.jpg" width=250 />

After Connection: \
<img src="./screenshots/mobile%20connected.jpg" width=250 />