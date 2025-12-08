# Pi-hole and Unbound Configuration

## Communication Methods

### 1. Same Network Namespace (Using `network_mode: service:pihole`)

This method places the Unbound container directly into the network stack of the Pi-hole container.

- **Unbound Docker Configuration:** Use `network_mode: service:pihole`.
- **Unbound Access:** Since Unbound shares the same network interface as Pi-hole, it can be reached via the local loopback address.
- **Pi-hole Upstream DNS:** Set Unbound as the upstream DNS in Pi-hole using the address:
    ```
    127.0.0.1#5335
    ```
- **Unbound Configuration File (`pihole.conf`):** Ensure the `interface` setting in Unbound's configuration is set to listen only on the local loopback:
    ```
    interface: 127.0.0.1
    ```

### 2. Custom Docker Network

This method places both Pi-hole and Unbound on a shared, user-defined Docker network, allowing them to communicate via service name resolution.

- **Docker Configuration:** Use the `networks:` directive for both services (e.g., `- custom_network`).
- **Unbound Access:** Containers on the same Docker network can resolve each other by their service name.
- **Pi-hole Upstream DNS:** Set Unbound as the upstream DNS in Pi-hole using the Unbound service name:
    ```
    unbound#5335
    ```
    (Assuming your Unbound container service name is `unbound`).
- **Unbound Configuration File (`unbound-config/unbound.conf`):** Unbound must be configured to listen on **all** available container network interfaces to be reachable by other containers on the network:
    ```
    interface: 0.0.0.0
    ```
    *This change is crucial for communication in a custom network setup.*

##  Download the current root hints file

Unbound expects the root hints file at **/etc/unbound/root.hints** inside the container.
Since I am mounting **./unbound-config/root.hints** to **/etc/unbound/root.hints**, I need to download the file into **./unbound-config/root.hints** on the host.
```
wget https://www.internic.net/domain/named.root -qO- | sudo tee <path to unbound-config folder>/unbound-config/root.hints
```

## Fix `so-rcvbuf` warning in Unbound (Optional)
The configuration in `pi-hole.conf` sets the **socket receive buffer size** for
incoming DNS queries to a higher-than-default value in order to handle high
query rates.

You may see this warning in unbound logs:

```bash
so-rcvbuf 1048576 was not granted. Got 425984. To fix: start with root permissions(linux)
or sysctl bigger net.core.rmem_max(linux) 
or kern.ipc.maxsockbuf(bsd) values.
```

To fix it, **run these commands son the host system**:

1. Check the current limit. This will show something like `net.core.rmem_max = 425984`:

    ```bash
    sudo sysctl net.core.rmem_max
    ```

2. Temporarily increase the limit to match Unbound's request:

    ```bash
    sudo sysctl -w net.core.rmem_max=1048576
    ```

3. Make it permanent. Edit `/etc/sysctl.conf` and add or edit the line using the command: `sudo nano /etc/sysctl.conf`

    ```bash
    net.core.rmem_max=1048576
    ```

4. Save and apply:

    ```bash
    sudo sysctl -p
    ```

## Verify Unbound is working

To confirm Unbound is resolving queries correctly, run the following commands
**in the pihole container**:

Open a `bash` shell in the container:

```bash
docker exec -it pihole /bin/bash
```

### When using unbound in pihole network namespace (network_mode: service:pihole)
```bash
dig pi-hole.net @127.0.0.1 -p 5335
```

The first query may be quite slow, but subsequent queries should be fairly quick.

Test validation:

```bash
dig fail01.dnssec.works @127.0.0.1 -p 5335
dig dnssec.works @127.0.0.1 -p 5335
```

The first command should give a status report of SERVFAIL and no IP address. The
second should give NOERROR plus an IP address.

### When using unbound in its own network namespace:-
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
sudo nano /var/www/html/admin/scripts/lua/sidebar.lp
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