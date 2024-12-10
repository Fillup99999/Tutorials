Install wireguard from wireguard.com using the preferred method.

On the ***HOST server*** (where the vpn routes to) create a directory in /etc named "wireguard" 
```
sudo mkdir /etc/wireguard
```

Then generate the keys in that directory:
```
umask 077

wg genkey | sudo tee /etc/wireguard/privatekey

sudo chmod 600 /etc/wireguard/privatekey

sudo cat /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey

```

Create the config file:
```
sudo nano /etc/wireguard/wg0.conf
```


Configure the wg0.conf file: (Read the #comments carefully)
```
[Interface]
#New unused private subnet (use this one if you don't know)
Address = 192.168.250.1/24

#Port Forwarding Rules:
PreUp = ip rule add table main suppress_prefixlength 0

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

PreDown = ip rule del table main suppress_prefixlength 0

PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

#This UDP port will be exposed in your Firewall
ListenPort = 51820

#Make this up but it must start with 0x and only contain numbers and leters from 0-9 & a-f
FwMark = 0xadf2 

#This has the filepath to the privatekey
PrivateKey = $(cat /etc/wireguard/privatekey)

[Peer]
#Public key of Client 1
PublicKey = lskdjfshdfojdflksdf
#IP 1 inside the same subnet
AllowedIPs = 192.168.250.10/32
#Android Phone

[Peer]
#Public key of Client 2
PublicKey = iweiojdnjkfodfj438923edfjk
#IP 2 inside that same subnet
AllowedIPs = 192.168.250.11/32
#Windows PC

```

On the ***CLIENT devices*** download and install wireguard the same way generating a key or using the GUI on the phone/tablets. The configuration file is simpler:

/etc/wireguard/wg0.conf

```
[Interface]
Address = 192.168.250.10/24
DNS = 1.1.1.1
#This should be the path to the privatekey file
PrivateKey = $(cat /etc/wireguard/privatekey)

[Peer]
#PublicKey of the HOST server.
PublicKey = sljdflksdjflskjdflksjd
Endpoint = <IP of the HOST server>
#Example: Endpoint = 8.8.8.8
AllowedIPs = 0.0.0.0/0
```

Run this command on the HOST server to start:
```
#To start:
sudo wg-quick up wg0

#This will check the status:
sudo wg show

#This will shutdown the hosting VPN:
sudo wg-quick down wg0
```

Open port 51820 on your firewall and point it to the local IP address of the HOST server, make sure the server is hardened beforehand. It may be useful to locally run nmap against it to check for weakpoints.

Then run the same command on the client device or turn on the VPN if GUI. Go to ifconfig.me or ipchicken.com and see if the IP is the same Public IP (WAN) as the HOST network.
