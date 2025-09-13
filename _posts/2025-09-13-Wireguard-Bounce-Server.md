---
layout: post
---

--- 

## Table of contents
- [Table of contents](#table-of-contents)
- [Overview](#overview)
- [Wireguard Installation and Basic Server Configuration](#wireguard-installation-and-basic-server-configuration)
- [Public Server Configuration](#public-server-configuration)
- [Client Configuration](#client-configuration)
- [Private Server Configuration](#public-server-configuration)
- [Interface Monitoring](#interface-monitoring)

### [Overview](#overview)

Many homelab's all suffer from the problem of remote access: how do I securely access my services from outside of my network? You could port forward; however you must be aware of the security risks involved in doing so. You also have to worry about your public address changing. You also might not have administrator access to your router, preventing you from creating port forwarding rules altogether.

Another issue, commonly overlooked, is the fact you only get one address. In my case I would like to perform remote administration of my services, which requires <em>ssh</em> access to my servers. If I forward port 22 on my home router, I would only be able to redirect that to one server. 

Of course you could set up a rule to direct incoming traffic with a destination port of 9090 to an internal server port of 22, but your SSH config would have to take this into account. I use the same laptop for remote access as well as in-home administration, meaning my SSH config would depend on where I am physically located.  

To solve these problems I have setup two Wireguard servers. I have VPN1 and VPN2, where VPN1 is located inside of my protected network and behind NAT, while VPN2 is a public server hosted in AWS. I am using t2.micro EC2 instance, as this qualifies for AWS free tier. A rough topology is drawn below.

```
                ┌──────┐             
             wg0│      │wg0          
    NAT┌───────►│ VPN2 │◄──────┐     
     ──┼──      │      │       │     
       │wg0     └──────┘       │wg0  
    ┌──┴───┐                ┌──┴───┐ 
    │      │                │      │ 
    │ VPN1 │                │Client│ 
    │      │                │      │ 
    └──┬───┘                └──────┘ 
       │eth0                         
       │                             
       │ ┌────────────┐                      
       └►│10.40.0.0/24│              
         └────────────┘
         Private Network                      
```

VPN1 will initiate a connection to VPN2, establishing a session through NAT. The client(s) can then connect to VPN2 from anywhere, establishing a virtual encrypted tunnel from the client device all the way through VPN2 and VPN1, reaching the private network. From the perspective of the Client, VPN2 remains virtually invisible.  

---

### [Wireguard Installation and Basic Server Configuration](#wireguard-installation-and-basic-server-configuration)

The following steps should be run on both servers; server specific configuration will come later in this guide. 

The first thing to do on any server is update and upgrade. I will be running debian servers, so all installation and update commands will be based around the apt package manager. 

```sudo apt update && sudo apt upgrade -y```

The next step is to install Wireguard and iptables. Iptables is how the various routing rules will be configured and is sometimes not installed by default.

```sudo apt install wireguard iptables -y```

We also need to enable IPv4 forwarding on the interfaces. This will allow the server to take a packet with a destination address that is not on the local system and pass it off through another interface, turning our server into a router. Create a file in <em>/etc/sysctl.d/</em> called <em>99-wireguard.conf</em>.

```sudo touch /etc/sysctl.d/99-wireguard.conf```

Open the file with a text editor with sudo permissions and paste in the following line:

```net.ipv4.ip_forward = 1```

Save the file and reload system configurations with:

```sudo sysctl --system```

At the bottom of the output, you should see that <em>net.ipv4.ip_forward = 1</em> has been echoed back. On system boot, configuration options in <em>/etc/sysctl.d</em> will be loaded automatically. 

The last thing to set up on both servers is the Wireguard keys. Switch to the root user and navigate to the wireguard configuration folder to use the following command to create public and private key files:

```
    su -
    cd /etc/wireguard
    wg genkey | tee privatekey | wg pubkey > publickey
```

Make a note of the private and pubic keys for both servers, they will be used later on when setting up the wireguard configuration files.


---

### [Public Server Configuration](#public-server-configuration)

All wireguard interfaces will be on one virtual (private) network, for this example I am using 192.168.99.0/24. It is important that this network does not overlap with the local network you are currently on, or routing tables will be messed up and configuration will not work. Common private subnets to avoid would be <em>192.168.0.0/24</em> or <em>10.0.0.0/24</em>. 

The public server does not have to be hosted in AWS, however that is what I am most familiar with. Make sure whatever cloud provided you are using does not have restrictions around VPN traffic. I will only be routing traffic destined to my private network through this tunnel (split tunnel configuration), but if you are using the VPN tunnel as your primary gateway you might want to make sure you have enough bandwidth to support that. 

To start configuring the Wireguard interface, create a file in <em>/etc/wireguard</em>. 

```
    cd /etc/wireguard
    touch wg0.conf
```

Open the configuration file and copy in the following configuration:

```
[Interface]
Address = 192.168.99.2/24
ListenPort = 51820
PrivateKey = PRIVATE_KEY

PostUp = iptables -A FORWARD -i %i -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -o %i -j ACCEPT

# VPN1 Peer
[Peer]
PublicKey = VPN1_PUBLIC_KEY
AllowedIPs = 192.168.99.1/32, 10.40.0.0/24

#Client
[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 192.168.99.5/32
```

Going over this configuration line by line, we start with the Interface block. This defines how the local wireguard interface, <em>wg0</em> in this case, will be set up. Later on when we run the <em>wg-quick</em> command to enable the interface it will read from this section. I am setting up this server with the address 192.168.99.2, as it is my VPN2 server. VPN1 will have the .1 address. Add your own private key to the PrivateKey option. 

PostUp and PostDown are a bit harder to understand. After <em>wg-quick</em> creates the interfaces it will run the command specified in PostUp. When run, %i will be substituted for the current Wireguard interface, <em>wg0</em>. This rule allows traffic coming in on the <em>wg0</em> interface to also be routed back out of the <em>wg0</em> interface, allowing for the bounce functionality of our server. 

The last step in configuring the server is to enable the <em>wg-quick</em> service. This can be done with systemd:

```
sudo systemd enable wg-quick@wg0
sudo systemd start wg-quick@wg0
```

Connection status can be monitored using: 

```sudo wg```

---

### [Client Configuration](#client-configuration)

Client configuration depends on what wireguard VPN client you are using. I would like to remotely access my network from a Windows system, so I am going to use the official Wireguard Windows client. Most wireguard clients use the same wireguard conf file format for their configurations. The following configuration can be pasted into your client:

```
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 192.168.99.5/32
DNS = 10.40.0.13

[Peer]
PublicKey = SERVER_PUBLIC_KEY
AllowedIPs = 192.168.99.0/24, 10.40.0.0/24
Endpoint = vpn2_address:51820
```

Make sure the address set on your client matches the allowed IP set on your server config for the client peer. I have specified my private DNS server so I can access my local services by domain name instead of IP address, however this is optional. The system DNS server will be used if no DNS is specified.

---

### [Private Server Configuration](#public-server-configuration)

The private server configuration is similar to the public one with a few key differences. To start configuring the Wireguard interface, create a file in <em>/etc/wireguard</em>. 

```
    cd /etc/wireguard
    touch wg0.conf
```

Open the configuration file and copy in the following configuration:

```
[Interface]
Address = 192.168.99.1/24
PrivateKey = VPN1_PRIVATE_KEY

PostUp = iptables -t nat -A POSTROUTING -s 192.168.99.0/24 -d 10.40.0.0/24 -j MASQUERADE
PostUp = iptables -A FORWARD -i %i -o ens19 -s 192.168.99.0/24 -d 10.40.0.0/24 -j ACCEPT
PostUp = iptables -A FORWARD -i ens19 -o %i -s 10.40.0.0/24 -d 192.168.99.0/24 -j ACCEPT

PostDown = iptables -t nat -D POSTROUTING -s 192.168.99.0/24 -d 10.40.0.0/24 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -o ens19 -s 192.168.99.0/24 -d 10.40.0.0/24 -j ACCEPT
PostDown = iptables -D FORWARD -i ens19 -o %i -s 10.40.0.0/24 -d 192.168.99.0/24 -j ACCEPT

[Peer]
PublicKey = VPN2_PUBLIC_KEY
AllowedIPs = 192.168.99.0/24
Endpoint = vpn2_address:51820
PersistentKeepalive = 25
```

Make sure to substitute the correct public and private keys into your configuration. Notice in the peer configuration the line <em>PersistentKeepalive</em> was added. When wireguard is not actively sending encrypted VPN packets, it will not by default send any keepalive messages. Without any packets being sent, the NAT gateway will terminate the connection and prevent packets originating for VPN1 to reach VPN2. This configuration option tells wireguard to send a "keepalive" or "hello" packet every 25 seconds, which then tells the NAT gateway that the session should not be terminated. 

There are a lot of PostUp and PostDown lines in this configuration. The PostDown options delete the iptable rules created in PostUp. They are almost exactly the same except with <em>-D</em> to delete the rule instead of <em>-A</em>.

The first PostUp rule creates a NAT entry. Devices in our private subnet are not aware of the 192.168.99.0/24 network and will try to send packets with a destination address in that subnet to their default gateway. What we can do is translate all packets originating from a remote client, (source address 192.168.99.5), to the local interface address of VPN1. 

The next two rules are similar to the ones created on our server configuration, allowing traffic incoming  on one interface to be sent out through the other, however instead of allowing wireguard traffic to enter and exit through the same interface, we allow the traffic to enter and exit on different interfaces.

Save the file and startup wireguard:

```
sudo systemd enable wg-quick@wg0
sudo systemd start wg-quick@wg0
```

---

### [Interface Monitoring](#interface-monitoring)

After the servers and client have all been settup, we can monitor wireguard information using the <em>wg</em> command:

```
sudo wg
```

If the servers have established a connection we can see an output like this:

```
netadmin@vpn1:~$ sudo wg
interface: wg0
  public key: VPN1_PUBLIC_KEY
  private key: (hidden)
  listening port: 56425

peer: VPN2_PUBLIC_KEY
  endpoint: x.x.x.x:51820
  allowed ips: 192.168.99.0/24
  latest handshake: 12 seconds ago
  transfer: 4.47 GiB received, 132.80 MiB sent
  persistent keepalive: every 25 seconds
```
