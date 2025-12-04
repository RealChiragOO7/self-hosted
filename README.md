# self-hosted

## Ports being used for several services
- Bento-pdf - **http://\<ip-address\>:8000**
- Headscale - **http://\<ip-address\>:8010**
- Headplane - **http://\<ip-address\>:8020**
- Pihole - **http://\<ip-address\>:8030**
- Stirling-pdf - **http://\<ip-address\>:8040**
- Vaultwarden - **http://\<ip-address\>:8050**

## Some information for Headscale + Headplane(Headscale Web-UI)
- `config.yaml` present in headscale root folder belongs to headplane and **DO NOT CHANGE FILE NAME AND KEEP IT AS config.yaml**.
- `config.yaml` present in **headscale/container-config** folder belongs to headscale.
- If you get an error message that says `tls_get_more_records:packet length too long`, change headscale URL in headplane config from **https** to **http**.

## Some information for Pihole + Unbound.
There are 2 ways of making pihole and unbound communicate :-
1. `network_mode: service:pihole` - When using this in unbound, unbound is in same network namespace as Pihole and hence can be accessed at **'127.0.0.1#5335'**. Keep **interface: 127.0.0.1** in **pihole.conf**.
2. `networks:`<br>
    `- custom_network` - When using this in unbound, both pihole and unbound get their own docker network ip's hence to use unbound as upstream DNS, use **'unbound#5335'**. For this to work, change interface setting in **unbound-config/unbound.conf.d/pihole.conf** from **127.0.0.1** to **0.0.0.0**.
    It should look like **interface: 0.0.0.0**. This allows Unbound to listen on all container interfaces, making it reachable from Pi-hole on the same network.