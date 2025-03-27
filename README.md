# Nginx-Wireguard-Proxy
This repository is an example on how to make a website accessible on the internet without creating port forwarding rules in your router.

## Foreword
Everything was tested on `Debian GNU/Linux 12 (bookworm) x86_64`.\

## Requiremants
For this example project you need the following things:
- A Linux Server within your network
- A Linux VPS
- A domain
- A webserver running on your internal Server

## Setup
First update both of the Servers (your internal Linux Server and your Linux VPS):
```bash
$ sudo apt update # Update the package list
$ sudo apt upgrade # Upgrade all packages
```

Now install the following packages on the VPS, these are needed for the later steps:
```bash
$ sudo apt install ufw wireguard nginx-full certbot python3-certbot-nginx
```

And on the internal Server run:
```bash
$ sudo apt install ufw wireguard
```

## Configure UFW (Firewall)
To make sure there is no unauthorized access to your servers i recommend configiuring a firewall. In this case ufw because it works and is pretty simple.

### On the VPS:
```bash
$ sudo ufw allow OpenSSH # Allow ssh on the VPS (Port 22)
$ sudo ufw allow 51820/udp # Allow the wireguard port with the udp protocol
$ sudo ufw allow "Nginx Full" # Allow "Nginx Full" our reverse proxy (Port 80 and 443)

$ sudo ufw enable # Enable the firewall, ensure you followed each step. Otherwise it may be so that you can't access your server over ssh.
$ sudo ufw status verbose # Show each configured rule and it's port
```

### On the internal Server:
```bash
$ sudo ufw allow OpenSSH # Allow ssh on the VPS (Port 22)
$ sudo ufw allow 51820/udp # Allow the wireguard port with the udp protocol
# Your also have to allow the port of your application here

$ sudo ufw enable # Enable the firewall, ensure you followed each step. Otherwise it may be so that you can't access your server over ssh.
$ sudo ufw status verbose # Show each configured rule and it's port
```

## Configure wireguard (P2P vpn connection)
Now we need to establisch a connection between the internal and external (VPS) server.

> [!CAUTION]
> Make sure you keep your keys private. Especially the private key. Otherwise the connection may be compromised. You can always generate new ones if you are unsure.

Run the below on both servers:
```bash
wg genkey > privatekey # Generate a private key 
wg pubkey < privatekey > publickey # Generate a public key, this gets later deployed to the oppisit server
```

After generating both the private key and the public key on both servers, we need to create some configs.\
First we open a file using `nano`. It should be installed by default.

### On the VPS:
```bash
$ sudo nano /etc/wireguard/wg0.conf
```

Now paste the file contens below and edit the marked places:
```toml
[Interface]
PrivateKey = <your-vps-private-key>
Address = 10.0.0.1/24

[Peer]
PublicKey = <your-PUBLIC-key-for-the-internal-server>
AllowedIPs = 10.0.0.2/32
```

To now exit and safe the file press: `Ctrl+x` => `y` => `Enter`.

### On the internal Server:
```bash
$ sudo nano /etc/wireguard/wg0.conf
```

Now paste the file contens below and edit the marked places:
```toml
[Interface]
PrivateKey = <your-internal-servers-private-key>
Address = 10.0.0.2/24

[Peer]
PublicKey = <your-PUBLIC-key-for-the-vps>
Endpoint = <the-public-ip-of-your-vps>
AllowedIPs = 10.0.0.1/32
PersistentKeepalive = 25 # Send keep alive packages so the connection doesn't break
```

To now exit and safe the file press: `Ctrl+x` => `y` => `Enter`.

The wireguard connections are not yet started. To do that run the following on both servers:
```bash
$ sudo systemctl enable --now wg-quick@wg0
$ sudo systemctl status wg-quick@wg0 # To check if the connection is enabled
```

The result of the status call should look something like this:
```bash
● wg-quick@wg0.service - WireGuard via wg-quick(8) for wg0
     Loaded: loaded (/lib/systemd/system/wg-quick@.service; enabled; preset: enabled)
     Active: active (exited) since Sun 2025-03-16 03:00:26 CET; 1 week 4 days ago
       Docs: man:wg-quick(8)
             man:wg(8)
             https://www.wireguard.com/
             https://www.wireguard.com/quickstart/
             https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8
             https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8
   Main PID: 756 (code=exited, status=0/SUCCESS)
        CPU: 12ms

Mar 16 03:00:26 pterodactyl systemd[1]: Starting wg-quick@wg0.service - WireGuard via wg-quick(8) for wg0...
Mar 16 03:00:26 pterodactyl wg-quick[756]: [#] ip link add wg0 type wireguard
Mar 16 03:00:26 pterodactyl wg-quick[756]: [#] wg setconf wg0 /dev/fd/63
Mar 16 03:00:26 pterodactyl wg-quick[756]: [#] ip -4 address add 10.0.0.3/24 dev wg0
Mar 16 03:00:26 pterodactyl wg-quick[756]: [#] ip link set mtu 1420 up dev wg0
Mar 16 03:00:26 pterodactyl systemd[1]: Finished wg-quick@wg0.service - WireGuard via wg-quick(8) for wg0.
```

When you ran the above commands successfully on both servers, you can see if the connections are healthy using:
```bash
$ sudo wg
# or
$ ping 10.0.0.2 # On the vps
# or
$ ping 10.0.0.1 # On the internal server
```

## Configure nginx (reverse proxy)
To configure nginx we need another config file. The following steps only apply to the vps.

First open the config file using nano:
```bash
$ sudo nano /etc/nginx/sites-available/myWebsite
```

Now add the following content:
```nginx
server {
    server_name <your-sub-domain>;

    location / {
        proxy_pass <http-or-https>://10.0.0.2:<the-port-your-app-is-running-on-internally>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Forwarded for=$remote_addr;
    }
}
```

To now exit and safe the file press: `Ctrl+x` => `y` => `Enter`.

To actually use the config u have to create a symlink using:
```bash
$ sudo ln -s /etc/nginx/sites-available/myWebsite /etc/nginx/sites-enabled/
# To disable a config you just need to remove the config from sites-enabled
$ sudo rm /etc/nginx/sites-enabled/myWebsite
```

After changing any nginx configs ALWAYS run the following to make sure the configuration is valid:
```bash
$ sudo nginx -t
```

Now we also need to setup ssl encryption using certbot. To do so, run the following:
```bash
certbot --nginx -d <your-sub-domain>
```

If you still have questions, i can recommend this simple [certbot tutorial](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04).

The last step is to restart the nginx service:
```bash
$ sudo systemctl restart nginx
```

Now you should have a fully working webserver hosted within your own network without setting up port forwarding.
