# Wireguard Site-to-Site VPN

## Server Configuration

The server file created during the setup will have the basics you need to get connected from a WG Client to the WG Server.

In order to enable traffic to be passed from the client network to the private subnet of the server, you will need to add the following option.

* PostUp IPTables rules: This will enable traffic flow and masquerade for the traffic coming from the client private network. Make sure you replace `INTERNAL_IP_INTERFACE` with the interface ID of your private network on your VPS, eg `eth1`.
* PostDown IPTables rules: Undoes the IPTables rules one the tunnel is disconnected. Make sure you replace `INTERNAL_IP_INTERFACE` with the interface ID of your private network on your VPS, eg `eth1`.
* Allowed IPs: Think of this as an ACL or allow rule for what IPs are allowed to pass traffic. The first IP is the IP of your WG Client. Add another range for the entire subnet of your CLIENT network. In the example below, my subnet it 192.168.100.0/23, which includes IPs `192.168.100.1 - 192.168.101.254`.

*Before:*

```
[Interface]
Address = 10.9.0.1/24
ListenPort = 31030
PrivateKey = PRIVATE_KEY
SaveConfig = false

# client1
[Peer]
PublicKey = PUBLIC_KEY
AllowedIPs = 10.9.0.2/32
```

*After:*

```
[Interface]
Address = 10.9.0.1/24
ListenPort = 31030
PrivateKey = PRIVATE_KEY
SaveConfig = false

PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o INTERNAL_IP_INTERFACE -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o INTERNAL_IP_INTERFACE -j MASQUERADE

# client1
[Peer]
PublicKey = PUBLIC_KEY
AllowedIPs = 10.9.0.2/32, 192.168.100.0/23
```

## Client Configuration

In order to properly route traffic for the SERVER subnet and back, you will need to add a couple of items on the client side.

You will need to update the following on your client side.

* PostUP: Config to the client .conf file to add IPtables rules to allow traffic back from the SERVER private subnet.
* PostDown: Config to remove the IPtables rules after connection shutdown 
* Routing rules to access the SERVER private network via the wireguard server

#### Edit the client .conf file

*Before*

```
[Interface]
PrivateKey = PRIVATE_KEY
Address = 10.9.0.2/24
DNS = 1.1.1.1, 1.0.0.1

[Peer]
PublicKey = PUBLIC_KEY
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = WIREGUARD_SERVER_PUBLICIP:LISTENING_PORT
PersistentKeepalive = 25
```

*After:*

```
[Interface]
PrivateKey = PRIVATE_KEY
Address = 10.9.0.2/24
DNS = 1.1.1.1, 1.0.0.1

PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth1 -j MASQUERADE

[Peer]
PublicKey = PUBLIC_KEY
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = WIREGUARD_SERVER_PUBLICIP:LISTENING_PORT
PersistentKeepalive = 25
```

#### Update the routing

If you want all machines on your CLIENT private network to be able to access the SERVER private subnet using the WG Client machine as a gateway, you will need to add routing information.

Below are examples and will need to be adjusted for your specific networks and config

* **Wireguard Windows client machine (self):** route add 10.132.0.0/16 MASK 255.255.0.0 10.9.0.2
* **Wireguard Windows client machine (another host):** route add 10.132.0.0/16 MASK 255.255.0.0 PRIVATE_IP_OF_WG_CLIENT
* **Wireguard Linux client machine (self):** route add 10.132.0.0/16 via 10.9.0.2 dev wg0
* **Wireguard Linux client machine (another host):** route add -net 10.132.0.0 netmask 255.255.0.0 gw PRIVATE_IP_OF_WG_CLIENT
* **Entire Network:** Add a route to the entire SERVER private network on your router. Pointing 10.132.0.0/16 to the IP of the Wireguard CLIENT on your network.

Please ask me any question about the setup or post any corrections under "Issues".

Twitter: https://twitter.com/mjtechguy
