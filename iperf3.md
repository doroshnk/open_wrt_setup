Install iperf
```
rm -f /var/opkg-lists/*
opkg update

opkg install iperf3
```

Run server as demaon
```
iperf3 -s -D
```
Set up for autostart

Create `/etc/init.d/iperf3` with
```
#!/bin/sh /etc/rc.common
# init script for iperf3 server
START=99
STOP=10
USE_PROCD=1

start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/iperf3 -s
    procd_set_param respawn
    procd_close_instance
}
```
And then start server
```
chmod +x /etc/init.d/iperf3
/etc/init.d/iperf3 enable
/etc/init.d/iperf3 start

```
