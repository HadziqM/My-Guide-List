# WIREGUARD

Wireguard is VPN protocol that designed for simplicity, fast and secure at same time. Compared to other VPN Wireguard is a very light weight protocol and it has a very fast performance.

The reason i use Wireguard is just how powerfull is VPN for my everyday activity. i use it not only for personal but for productive work too. A Virtual Private Network (VPN) isn’t just for unblocking websites or streaming shows—it’s a powerful tool with many practical uses, especially for me as an developer and tech hobbyist.
Everyday Benefits of a VPN :

1. Privacy: A VPN hides your IP address, making it harder for websites, advertisers, or hackers to track your online activity.
2. Security: It encrypts your internet traffic, protecting sensitive data like passwords, financial information, or private emails, especially on public Wi-Fi.
3. Remote Access: IT experts often use VPNs to securely access company systems or private servers from anywhere.
4. Network Control: With a VPN, you can connect multiple devices to your own private network, even when working remotely.
5. Data Protection: A VPN prevents ISPs or malicious actors from snooping on your internet usage or stealing your data.


## SETUP WIREGUARD

before we start reminder i had only use it on my home device (Arch Linux), and VPS (Ubuntu Linux), other distro or OS shouldn't be too far since i will cover the meaning of each configuration.

### Installing Wireguard
On Ubuntu or debian based distro, only need this command :
```
sudo apt install wireguard
```
On arch linux you need two package :
```
sudo pacman -S wireguard wireguard-tools
```
you can check the installation by running :
```
wg --version
```

### Before configuring
Before configuring i need you to understand the concept of the VPN, and how Wireguard handle it.

- The basic of the VPN is almost same as your home router, every time you access an IP address or domain name, it will first pass trough your router and you will surf the IP you access using router public IP as your Identity
- you can create LAN using router as long as you follow router rule, that i will cover later, what i m trying to say is you can create LAN like connection like router with VPN but not limited by Wi-Fi signal, you can connect even from the other side of of globe.
- wireguard handle VPN by using profile, there two kind of configuration for simplicity sake, one is for server and one for client (we call it peer)
- you can have more than one profile in your machine, meaning you can run multiple VPN connection at same time. for example one for work and one for home.

### Configuring Wireguard Server

When starting wireguard its best to configure server first, i suggest to run server on machine that mostly on 24/7 with internet connection like VPS service.

#### Create Key pair
first you need to create private and public key for your server.
```
wg genkey | tee privatekey | wg pubkey > publickey
```
both private key `genkey` and public key `pubkey` command need to be on one pipeline otherwise it will be invalid/error, you can edit the command to save the key on other file. like any other file you can see the key using tool like `cat`

```
cat privatekey
```

> [!TIP]
> You might want to generate key pair for your peer too, to save time latter, for example if you only want 3 client, generate 4 key, one for server

#### Configuring Wireguard Profile
first create our first profile, for simplicity we name it `wg0`
```
sudo vim /etc/wireguard/wg0.conf
```
> [!NOTE]
> Vim will create file if not exist, you might want to create file first if using other tools.

```
[Interface]
Address = 10.0.0.1/24
PrivateKey = <Server-private-key>
ListenPort = 51820

[Peer]
PublicKey = <Peer-public-key>
AllowedIPs = 10.0.0.2/32
```

Its the basic server configuration, i will explain it bit by bit

- The interface field is the one used on your machine, Address there represent IP address that will be used by your server when connecting to this VPN profile, the /24 mean allowed IP by server, it mean it can handle/ manage IP form 10.0.0.1 to 10.0.0.255, so in term of subnet mask, its 255.255.255.0. For simpler explanation, it mean your VPN profile can handle 256 IP address. if you want larger range, you can use /16 to handle 65.535 IP address. ListenPort is self explanatory you can edit it if you want.
- The peer field is the list of address your server want to handle, the AllowedIPs is the IP address that server will give to the peer, you will want it to be at server managed IP (in this case 10.0.0.2 ~ 10.0.0.255) 10.0.0.1 is out of option since it used by server. it always advised to use /32 to limit the IP one peer has (/32 mean only one). This to make sure consistency of the IP mapping. you can have multiple `[peer]` in your configuration, if you want more than one client that can connect to this VPN profile.


> [!TIP]
> If you generate many key pair from step before you can use it to configure many peer right away, just make sure to remember which key pair correspond to IP configured since you will need it to configure the client

#### Finishing
you can start the server with :
```
sudo wg-quick up wg0
```
and to stop it with :
```
sudo wg-quick down wg0
```

If you want to automatically connect to wireguard profile when you start your machine, you can use systemd to start it
```
sudo systemctl enable wg-quick@wg0
```

Quick note that wg0 is the profile name, you need to change it to your profile name if it isnt wg0, you can repeat the step if you have multiple profile configured


#### Important Notes

Remember that VPN work like router, when you access any IP/domain for example `www.google.com` it will trough router first, same as VPN, the request will be trough VPN server first. The problem is the server will not forward your request by default, so you will be in state like Internet disconnect even though internet running fine and you connect to server VPN trough internet no problem.
To fix this you need to enable `IP forwarding` in your server

```
sudo vim /etc/sysctl.conf
```

Then add this line in the end of file
```
net.ipv4.ip_forward=1
```
Then enable the config with
```
sudo sysctl -p
```
you might want to restart your wireguard profile if you run the server before IP forwarding


### Configuring Wireguard Client (Peer)
configuring the client isnt that much different form server, so you will get easier time with the step

#### Generate Key
first you need to create private and public key for your client.
```
wg genkey | tee privatekey | wg pubkey > publickey
```
> [!TIP]
> You can skip this step if you already generate the key pair from the server, you can just use it here since the key isnt device tied at all

#### Configuring Wireguard Profile
same as server you need to create the profile, you can use different name as server but for simplicity just make the profile same name

```
sudo vim /etc/wireguard/wg0.conf
```

```
[Interface]
Address = 10.0.0.2/32
PrivateKey = <Your-private-key>

[Peer]
PublicKey = <Server-public-key>
Endpoint = <Server-IP-Adrress>:51820
AllowedIPs = 0.0.0.0/0
```

Its the basic client configuration.

- Interface, Address is the IP you want to use as client, make sure the server also configured to have the Address you choose, and use your public key in their peer config, (that why i advised to generate many key pair on server first then use it to configure server peer so you dont need go back into server and configure the peer each time you add new client). the config need to match the peer that configured in server.
- Peer, this used to connect to your server so you can only has one peer, the `EndPoint` need to match server IP address (usually internet since you want to connect to VPN via internet) for this case is your server public IP and the port i will use default, but change it if you use another port on server config.
AllowedIPs mean thea IP you can access using the VPS 0.0.0.0/0 mean no restriction, its mean you can connect to internet or any IP server has, although server need to be `IP forwarded` to make the effect

#### Finishing
you can start the vpn with :
```
sudo wg-quick up wg0
```
and to stop it with :
```
sudo wg-quick down wg0
```

If you want to automatically connect to wireguard profile when you start your machine, you can use systemd to start it
```
sudo systemctl enable wg-quick@wg0
```
#### Test
you can test if the VPN work by pingging the server VPN IP
```
ping 10.0.0.1
```
you can ping other peer too if you configured more than one client/peer.

you want to check if you can access internet to make sure `IP forwarding` worked on the server
```
ping 1.1.1.1
```


## Simple Use Case
with this you can use many benefit of VPN

- Unblock Geo located banned site. Yes, since all the connection is directed trough VPN server, ISP will only know you access your server but never know what you browse from it nor ban it.
- SSH / SFTP to your home computer, both SSH and SFTP need and IP adrress and you dont want to expose them to internet to do that, while its risky, its tricky too. with VPN each peer had own VPN ip, if both connected to same VPN, you can directly access SSH/SFTP using VPN ip associated to your home PC.
- Private Web/TCP server. for example you have Home Automation system, to monitor and turn on lamp if you away from home, you need server to control your home, while hosting the server works, you might doesnt want to expose your server into internet since it will be dangerous if hacker or public can control your home, you can host it locally trough IP VPN, for example your IoT device has VPN IP `10.0.0.2` then serve the server on `10.0.0.2:8080` or any port, the if you want to access it just connect to VPN and input `10.0.0.2:8080` in your browser, simple + secure.
- Quick internet serve, for example you want to create server that already has component in local environtment, for example you has game server in your Home PC and it has satcked configuration and database, and you want to invite friend to play with you, since you friend is tech savy, they dont know how to set up VPN like you did and prefer instant methode, you can migrate the game server and its dependencies like setting and database into server so your friend just need to connect to server IP, but it will be hassle to migrate them right?. You can just serve the game server into VPN IP, then in the server just proxy its public IP into your VPN IP, and then done, your game server can be accessed via server Public IP, you friend dan join without any VPN, and you dont need to migrate your game server into your VPS server, just chill.
