Install wireguard
```
rm -f /var/opkg-lists/*
opkg update
opkg install kmod-wireguard
opkg install wireguard-tools resolveip
opkg install luci-proto-wireguard
```
Reeboot router

WG tools

```
#Generate Server Private Key.
wg genkey > server_privatekey

#Generate Server Public Key from Private Key:
wg pubkey < server_privatekey > server_publickey

#Generate Client Private Key.
wg genkey > client_privatekey

#Generate Client Public Key from Private Key:
wg pubkey < client_privatekey > client_publickey

#or by one line
wg genkey | tee private.key | wg pubkey > public.key
```

Add WG interface
```
# ===  wg0
uci set network.wg0='interface'
uci set network.wg0.proto='wireguard'
uci add_list network.wg0.addresses='10.10.10.1/24'
uci set network.wg0.listen_port='51820'
uci set network.wg0.private_key='<SERVER_PRIVATE_KEY>'

# === peer (your client)
uci add network wireguard_wg0
uci set network.@wireguard_wg0[-1].description='client1'
uci set network.@wireguard_wg0[-1].public_key='<CLIENT_PUBLIC_KEY>'
uci add_list network.@wireguard_wg0[-1].allowed_ips='10.10.10.2/32'

uci commit network
/etc/init.d/network restart
```
Setup firewall

```

# === Firewall ===
uci add firewall zone
uci set firewall.@zone[-1].name='vpn'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='ACCEPT'
uci set firewall.@zone[-1].masq='1'
uci add_list firewall.@zone[-1].network='wg0'

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='vpn'
uci set firewall.@forwarding[-1].dest='lan'

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WireGuard-UDP'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].target='ACCEPT'

uci commit firewall
/etc/init.d/firewall restart
```
Setup WG client

Create conf file for client in `sudo vim /etc/wireguard/r36.conf` and enter
```
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.10.10.2/32
DNS = 10.10.10.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <IP_or_DDNS_of_router>:51820
AllowedIPs = 10.10.10.0/24, 192.168.36.0/24
PersistentKeepalive = 25
```
Switch on VPN connect `sudo wg-quick up r36` switch off `sudo wg-quick down r36`

Set up duckdns. Open `vim /etc/config/ddns` comment other config and add
```
config service 'duckdns_home'
        option enabled '1'
        option service_name 'duckdns.org'
        option domain 'mydns.duckdns.org'
        option username 'mydns'
        option password 'your_token'
        option use_https '0'
        option ip_source 'network'
        option ip_network 'wan'
        option interface 'wan'
        option check_interval '1'
        option check_unit 'minutes'
        option force_interval '12'
        option force_unit 'hours'
        option lookup_host 'mydns.duckdns.org'
```
Then restar 
```
/etc/init.d/ddns enable
/etc/init.d/ddns restart
```
and check `logread -e ddns` or `/usr/lib/ddns/dynamic_dns_updater.sh duckdns_home`
