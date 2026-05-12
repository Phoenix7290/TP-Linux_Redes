❯ ip route
default via 192.168.1.1 dev eth1 proto kernel metric 25
default via 26.0.0.1 dev eth0 proto kernel metric 9257
default via 25.255.255.254 dev eth2 proto kernel metric 10034
10.140.53.0/24 dev eth2 proto kernel scope link metric 291
25.255.255.254 dev eth2 proto kernel scope link metric 10034
26.0.0.0/8 dev eth0 proto kernel scope link metric 257
26.0.0.1 dev eth0 proto kernel scope link metric 9257
100.72.245.97 dev eth3 proto kernel scope link metric 5
100.85.29.79 dev eth3 proto kernel scope link metric 5
100.100.100.100 dev eth3 proto kernel scope link metric 5
100.110.248.52 dev eth3 proto kernel scope link metric 5
192.168.1.0/24 dev eth1 proto kernel scope link metric 281
192.168.1.1 dev eth1 proto kernel scope link metric 25

❯ sudo ip route add 8.8.8.8/32 via $(ip route | grep default | awk '{print $3}') dev eth0
Error: either "to" is duplicate, or "26.0.0.1" is a garbage.

❯ ip route | grep 8.8.8.8

❯

❯ sudo ip route del 8.8.8.8/32
RTNETLINK answers: No such process

❯ ip route | grep 8.8.8.8

❯