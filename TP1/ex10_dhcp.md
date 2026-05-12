ip addr show eth0 | grep "inet "
nmcli device show eth0 | grep DHCP

❯ ip addr show eth0 | grep "inet "
    inet 26.76.11.189/8 brd 26.255.255.255 scope global noprefixroute eth0

nmcli device show eth0 | grep DHCP

❯ journalctl -u systemd-networkd | tail -n 20
-- No entries --
                %

❯ ls /var/lib/NetworkManager/ 2>/dev/null || echo "Sem NetworkManager"

sudo ip addr add 172.20.0.100/20 dev eth0
sudo ip route add default via 172.20.0.1
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

❯ sudo ip addr add 172.20.0.100/20 dev eth0
sudo ip route add default via 172.20.0.1
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
Error: ipv4: Address already assigned.
nameserver 8.8.8.8