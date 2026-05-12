❯ ip -6 addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
6: eth3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1280 state UP qlen 1000
    inet6 fd7a:115c:a1e0::3301:d981/128 scope global nodad noprefixroute
       valid_lft forever preferred_lft forever
    inet6 fe80::af88:96bb:8610:3502/64 scope link nodad noprefixroute
       valid_lft forever preferred_lft foreve

❯ ip -6 route show
fd7a:115c:a1e0::53 dev eth3 proto kernel metric 5 pref medium
fd7a:115c:a1e0::/48 via fd7a:115c:a1e0::53 dev eth3 proto kernel metric 5 pref medium