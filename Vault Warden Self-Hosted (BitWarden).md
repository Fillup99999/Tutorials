### Prerequisites: Configure a Wireguard (or other) VPN to this LAN otherwise you can only access on that 1 local WiFi/LAN
----------------
**SETUP DOCKER ENGINE:**
Follow from docker's website for the CLI version that includes docker compose (https://docs.docker.com/engine/install/). This should be straight forward if followed correctly. Just don't get confused and download docker desktop or something else.

Create a folder at ~/vaultwarden/
		`cd` into the folder

Open ports 80 & 443 on your server's SOFTWARE firewall (NOT THE ROUTER)
		If using ufw on linux:
			`sudo ufw allow 80`
			`sudo ufw allow 443`

**SETUP VAULTWARDEN:**
	Install:
		`sudo docker pull vaultwarden/server:latest`
		`sudo docker volume create vaultwarden_data`
	Run the container:
	`sudo docker run -d --name vaultwarden -p 80:80 -v /vw-data:/data/ vaultwarden/server:latest`
	Put the server's IP address into a browser and you should reach the page, but not be able to make an account as the connection is insecure (http not https) 
	Then stop it with: `sudo docker rm vaultwarden`
	
**SETUP CADDY:**
	Install xcaddy from binary on their git (https://github.com/caddyserver/xcaddy/releases) page. (Extract it and run it with: `sudo dpkg -i xcaddy-version.deb`")
		Install go from binary on their website (https://go.dev/dl/) and then also extract it: `tar -C /usr/local -xzf go1.22.1.linux-amd64.tar.gz`
		Add add it to your PATH: `export PATH=$PATH:/usr/local/go/bin`
		Verify: `go version`
		Compile caddy on the server: `xcaddy build --with github.com/caddyserver/dnsproviders/duckdns`
		IF YOU HAVE ISSUES WITH IT TAKING FOREVER ON A SMALL CPU (Raspberry Pi), increase the VRAM file (probably ask chatGPT for your specific system)
		Move the created binary (named "caddy") to the vaultwarden folder if not already located there: ~/vaultwarden/

**SETUP DUCKDNS:**
	Visit duckdns.org and create an account and subdomain.
	Should point to the LOCAL IP ADDRESS of your vaultwarden server computer.
	Example: vaultwarden.duckdns.org --> 192.168.0.13
	TAKE NOTE of the token code located on the dashboard.

**CONFIGURE FILES: Create them at (~/vaultwarden/)**
	Make sure the file names match as well as the contents, copy and past them then make the necessary changes (read through them I added comments to show):

**docker-compose.yml:**
~~~
version: '3'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      WEBSOCKET_ENABLED: "true"
    volumes:
       - /media/databank/vaultwarden:/data
  caddy:
    image: caddy:2
    container_name: caddy
    restart: always
    ports:
      - 80:80  # Needed for the ACME HTTP-01 challenge.
      - 443:443
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy-config:/config
      - ./caddy-data:/data
      - ./caddy:/usr/bin/caddy
    environment:
      - DOMAIN=https://vaultwarden.duckdns.org #Put your DuckDNS Domain here
      - EMAIL=#putemailhere(doesn't need to be real)
      - LOG_FILE=/data/access.log
      - DUCKDNS_TOKEN=#PasteDuckDNSTokenHere
~~~

**Caddyfile:**
~~~
{$DOMAIN}:443 {
	log {
		level INFO
		output file {$LOG_FILE} {
			roll_size 10MB
			roll_keep 10
		}
	}

	encode gzip

	tls {
		dns duckdns {env.DUCKDNS_TOKEN}
	}
	reverse_proxy /notifications/hub vaultwarden:3012
	reverse_proxy vaultwarden:80
	}
~~~

**caddy.env:**
~~~
DOMAIN=vaultwarden.duckdns.org #Paste your domain name here
DUCKDNS_TOKEN=#PasteTokenHere
LOG_FILE=/data/access.log
~~~

Now make sure again that all of these files are in this location with the previous contents (the caddy file is the binary from earlier, there may be additional folders like `caddy-config` and `caddy-data`):
~~~
yourname@raspberrypi:~/vaultwarden/ $
caddy    caddy.env    Caddyfile    docker-compose.yml
~~~
**Run Caddy:**
		`sudo caddy run --envfile caddy.env`
		wait for it to load for about 30 seconds then stop it: ctrl+c

**Run Docker Compose:** In the vaultwarden folder
	`sudo docker compose up`
If it is successful you should be able to access it from your browser at the domain you acquired from duckdns. If so, hit (ctrl+c).

**Now rerun it as a daemon:**
	`sudo docker compose up -d`
	TO SHUTDOWN:
	`sudo docker compose down`





If Docker Breaks (Illegal Instruction Error): Reinstall with an old version
`apt-cache madison docker-ce | awk '{ print $3 }'`

find what the old version was from /var/log/apt or just pick one
copy an old version (5:23.0.6-1~raspbian.11~bullseye in my case)

`sudo apt remove docker-ce docker-ce-cli && sudo apt install docker-ce=5:23.0.6-1~raspbian.11~bullseye docker-ce-cli=5:23.0.6-1~raspbian.11~bullseye containerd.io docker-buildx-plugin docker-compose-plugin`
or
`sudo apt remove docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y && sudo apt install docker-ce=5:24.0.7-1~raspbian.11~bullseye docker-ce-cli=5:24.0.7-1~raspbian.11~bullseye containerd.io=1.6.27-1 docker-buildx-plugin=0.11.2-1~raspbian.11~bullseye docker-compose-plugin=2.21.0-1~raspbian.11~bullseye -y`


`sudo systemctl start docker`

`sudo systemctl status docker`

Then set a "Hold" on future updates for those packages until you feel that the issue is fixed:
`sudo apt-mark hold docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`

`sudo apt-mark showhold`

Un-Hold with:
`sudo apt-mark unhold docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`

Submit a bug report to Docker

--------------------------------------------
RESOURCES:
Follow the github installation and set up for Vault Warden and Docker through the respective pages.

To set up DNS: https://github.com/dani-garcia/vaultwarden/wiki/Running-a-private-vaultwarden-instance-with-Let%27s-Encrypt-certs

https://github.com/dani-garcia/vaultwarden/wiki/Using-Docker-Compose
https://github.com/caddy-dns/duckdns

https://www.duckdns.org/

**Youtube Video gets you like 99% of the way: https://youtu.be/0Ri3GVDc4pM?si=r-EAolawLLs32EKU
~~~
MY DOCKER SETUP:
Client: Docker Engine - Community
 Version:           24.0.7
 API version:       1.43
 Go version:        go1.20.10
 Git commit:        afdd53b
 Built:             Thu Oct 26 09:08:26 2023
 OS/Arch:           linux/arm
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          24.0.7
  API version:      1.43 (minimum version 1.12)
  Go version:       go1.20.10
  Git commit:       311b9ff
  Built:            Thu Oct 26 09:08:26 2023
  OS/Arch:          linux/arm
  Experimental:     false
 containerd:
  Version:          1.6.27
  GitCommit:        a1496014c916f9e62104b33d1bb5bd03b0858e59
 runc:
  Version:          1.1.11
  GitCommit:        v1.1.11-0-g4bccb38
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
~~~

