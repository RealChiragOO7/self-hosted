# Pi-hole and Unbound Configuration
##  Download the latest root.hints file

Unbound expects the root hints file at **/etc/unbound/root.hints** inside the container.
Since I am mounting **./unbound-config/root.hints** to **/etc/unbound/root.hints**, I need to download the file into **./unbound-config/root.hints** on the host.
```
wget https://www.internic.net/domain/named.root -qO- | sudo tee /home/rpi/services/pihole/unbound-config/root.hints
```

## Fix `so-rcvbuf` warning in Unbound (Optional)
The configuration in `pi-hole.conf` sets the **socket receive buffer size** for
incoming DNS queries to a higher-than-default value in order to handle high
query rates.

You may see this warning in unbound logs:

```bash
so-rcvbuf 4194304 was not granted. Got 425984. To fix: start with root permissions(linux) or sysctl bigger net.core.rmem_max(linux) or kern.ipc.maxsockbuf(bsd) values.
so-sndbuf 4194304 was not granted. Got 425984. To fix: start with root permissions(linux) or sysctl bigger net.core.wmem_max(linux) or kern.ipc.maxsockbuf(bsd) values.
```

To fix it, **run these commands son the host system**:

1. Check the current limit. This will show something like `net.core.rmem_max = 425984`:

    ```bash
    sudo sysctl net.core.rmem_max
    sudo sysctl net.core.wmem_max
    ```

2. Temporarily increase the limit to match Unbound's request:

    ```bash
    sudo sysctl -w net.core.rmem_max=4194304
    sudo sysctl -w net.core.wmem_max=4194304
    ```

3. Make it permanent. Edit `/etc/sysctl.d/99-unbound.conf` and add or edit the line using the command: `sudo nano /etc/sysctl.d/99-unbound.conf`

    ```bash
    net.core.rmem_max=4194304
    net.core.wmem_max=4194304
    ```

4. Save and apply:

    ```bash
    sudo systemctl restart systemd-sysctl  
    ```

## Setup dnsmasq.d
```
touch services/pihole/etc-dnsmasq.d/02-wildcard.conf
```

```
nano services/pihole/etc-dnsmasq.d/02-wildcard.conf

address=/030974.xyz/192.168.0.100
```

In the Tailscale admin console:

Add custom nameserver = RPi's Tailscale IP (100.x.x.x)
Enable split DNS, restrict to yourdomain.com

## Verify Unbound is working

To confirm Unbound is resolving queries correctly, run the following commands
**in the pihole container**:

Open a `bash` shell in the container:

```bash
docker exec -it pihole /bin/bash
```

```bash
dig pi-hole.net @unbound -p 5335
```

The first query may be quite slow, but subsequent queries should be fairly quick.

Test validation:

```bash
dig fail01.dnssec.works @unbound -p 5335
dig dnssec.works @unbound -p 5335
```

## Some modifications to Pihole dashboard
### Add temperature in Pihole dashboard

Run this command:
```
sudo vi /var/www/html/admin/scripts/lua/sidebar.lp
```

Find the following code: 

```
<div class="pull-left info">
    <p>Status</p>
    <span id="status"></span><br/>
    <span id="query_frequency"></span><br/>
    <span id="cpu"></span><br/>
    <span id="memory"></span>
</div>
```

Add following line for Farenheit below **\<p>Status\</p>**:
```
<span id="temperature">&nbsp;&nbsp;<i class="fa-solid fa-temperature-three-quarters text-green-light"></i>&nbsp;&nbsp; Temp: <%= string.format("%.1f°F", (tonumber(io.open("/sys/class/thermal/thermal_zone0/temp"):read("*a")) / 1000) * 9/5 + 32) %></span><br/>
```

Or add following line for Celsius below **\<p>Status\</p>**:
```
<span id="temperature">&nbsp;&nbsp;<i class="fa-solid fa-temperature-three-quarters text-green-light"></i>&nbsp;&nbsp; Temp: <%= string.format("%.1f°C", tonumber(io.open("/sys/class/thermal/thermal_zone0/temp"):read("*a")) / 1000) %></span><br/>
```

It should look like this
```
<div class="pull-left info">
    <p>Status</p>
    <span id="temperature">&nbsp;&nbsp;<i class="fa-solid fa-temperature-three-quarters text-green-light"></i>&nbsp;&nbsp; Temp: <%= string.format("%.1f°C", tonumber(io.open("/sys/class/thermal/thermal_zone0/temp"):read("*a")) / 1000) %></span><br/>
    <span id="status"></span><br/>
    <span id="query_frequency"></span><br/>
    <span id="cpu"></span><br/>
    <span id="memory"></span>
</div>
```