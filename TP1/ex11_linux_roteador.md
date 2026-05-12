❯ sysctl net.ipv4.ip_forward
cat /proc/sys/net/ipv4/ip_forward
net.ipv4.ip_forward = 1
1


Explicação técnica

O que ip_forward=1 permite?
Permite que o kernel Linux encaminhe pacotes IP entre interfaces (forwarding).
O que ainda faltaria para ser um roteador real?
Regras de roteamento (ip route)
NAT/Masquerading (iptables ou nftables)
Firewall (regras de aceitação/rejeição)
Possivelmente BGP ou roteamento dinâmico