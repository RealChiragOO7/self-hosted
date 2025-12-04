# Headscale + Headplane
## Create api key to login into headplane -- run this command on headscale server
```
docker exec -it headscale headscale apikeys create -e 999d
```
    
## Install tailscale on rpi
```
curl -fsSL https://pkgs.tailscale.com/stable/debian/trixie.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null

curl -fsSL https://pkgs.tailscale.com/stable/debian/trixie.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list

sudo apt-get update

sudo apt-get install tailscale
```

## Connect to tailscale server on rpi
```
sudo tailscale up --login-server=https://headscale.your-domain.com
``` 


## Run these to enable ip forwarding
```
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf

echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf

sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

## Make rpi as exit node and allow lan access and setup advertise routes for rpi so as to allow remote access via these ip's only
```
sudo tailscale set --advertise-exit-node --exit-node-allow-lan-access=true --advertise-routes=192.168.0.0/24,100.64.0.0/24

## Now go into machines page > machine > machine settings in headplane site and select edit route settings and select use as exit node option

## Now go into machines page > machine > machine settings in headplane site and select edit route settings > allow the advertised routes.
```

    
## Some useful commands for debugging purposes
```
- Find tailscale ip4 address
tailscale ip -4

- Create a headscale user
docker exec -it headscale headscale users create <USERNAME>

- List existing headscale users
docker exec -it headscale headscale users list

- Login 
sudo tailscale up --login-server <YOUR_HEADSCALE_URL>
```