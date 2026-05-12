# Exercício 3 — IPv4 vs IPv6: Inventário e Prontidão Operacional

---

## Comandos Executados

### `ip -4 addr`

```
❯ ip -4 addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 10.255.255.254/32 brd 10.255.255.254 scope global lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 26.76.11.189/8 brd 26.255.255.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 192.168.1.190/24 brd 192.168.1.255 scope global noprefixroute eth1
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2800 qdisc mq state UP group default qlen 1000
    inet 10.140.53.131/24 brd 10.140.53.255 scope global noprefixroute eth2
       valid_lft forever preferred_lft forever
6: eth3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1280 qdisc mq state UP group default qlen 1000
    inet 100.66.217.127/32 brd 100.66.217.127 scope global noprefixroute eth3
       valid_lft forever preferred_lft forever
```

> **📸 Screenshot sugerido:** capturar a saída de `ip -4 addr` mostrando todas as interfaces com seus endereços e prefixos.

---

### `ip -6 addr`

```
❯ ip -6 addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
6: eth3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1280 state UP qlen 1000
    inet6 fd7a:115c:a1e0::3301:d981/128 scope global nodad noprefixroute
       valid_lft forever preferred_lft forever
    inet6 fe80::af88:96bb:8610:3502/64 scope link nodad noprefixroute
       valid_lft forever preferred_lft forever
```

> **📸 Screenshot sugerido:** capturar a saída de `ip -6 addr` destacando as linhas `inet6` de eth3 com seus escopos (global e link).

---

### `ip -4 route`

```
❯ ip -4 route
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
```

---

### `ip -6 route`

```
❯ ip -6 route
fd7a:115c:a1e0::53 dev eth3 proto kernel metric 5 pref medium
fd7a:115c:a1e0::/48 via fd7a:115c:a1e0::53 dev eth3 proto kernel metric 5 pref medium
```

---

## Análise com Base nas Evidências

### 1. Qual é o IPv4 principal e seu prefixo?

O IPv4 principal é **192.168.1.190/24** na interface **eth1**.

Essa conclusão é suportada por dois critérios:
- eth1 possui a **rota default com menor métrica** (`metric 25`), sendo o gateway preferencial para tráfego externo.
- O endereço 192.168.1.190/24 pertence à LAN doméstica (RFC 1918), que é a rede de acesso à internet neste ambiente.

Os demais endereços IPv4 (eth0: 26.76.11.189/8, eth2: 10.140.53.131/24) são interfaces WSL auxiliares; eth3 (100.66.217.127/32) pertence à rede Tailscale (VPN).

---

### 2. Existe IPv6 fe80::/10? Em qual interface?

**Sim.** O endereço `fe80::af88:96bb:8610:3502/64` com `scope link` está presente na interface **eth3**.

O prefixo `fe80::/10` define o espaço de endereçamento link-local IPv6 (RFC 4291). Endereços nesse bloco são autoconfigurados pelo kernel via SLAAC (Stateless Address Autoconfiguration) e são visíveis apenas no segmento de rede local — não são roteáveis além do link. Note que apenas eth3 possui endereço fe80::, o que significa que somente esta interface (Tailscale) executou a configuração IPv6 de camada 2.

---

### 3. Existe rota default IPv4? E IPv6?

**IPv4:** Sim, há **três rotas default IPv4**:

| Gateway | Interface | Métrica |
|---|---|---|
| 192.168.1.1 | eth1 | 25 (ativa — menor métrica) |
| 26.0.0.1 | eth0 | 9257 (failover) |
| 25.255.255.254 | eth2 | 10034 (failover) |

O kernel utiliza a de menor métrica; as demais ficam como rotas de contingência.

**IPv6:** **Não há rota default IPv6.** As únicas rotas IPv6 presentes são:
- `fd7a:115c:a1e0::53 dev eth3` — rota para o DNS da Tailscale
- `fd7a:115c:a1e0::/48 via fd7a:115c:a1e0::53 dev eth3` — rota para a rede interna Tailscale

Não existe linha `default` ou `::/0` na tabela IPv6, o que significa que o host **não pode rotear tráfego IPv6 para destinos fora da rede Tailscale**.

---

## Conclusão: Prontidão IPv6

**O host não está operacionalmente pronto para IPv6 de uso geral.** Duas evidências objetivas:

1. **Ausência de rota default IPv6 (`ip -6 route` sem `::/0`):** Sem esta rota, pacotes destinados a qualquer endereço IPv6 fora da sub-rede Tailscale (fd7a:115c:a1e0::/48) serão descartados pelo kernel. Por exemplo, um `ping6 2606:4700:4700::1111` (Cloudflare DNS) falharia com "Network unreachable".

2. **IPv6 global restrito à interface Tailscale (eth3):** O único endereço global (`fd7a:115c:a1e0::3301:d981/128`) pertence ao espaço de endereçamento privado da VPN Tailscale (ULA — Unique Local Address, prefixo `fd`). As interfaces principais (eth0, eth1, eth2) não possuem endereço IPv6 algum. Em um ambiente com conectividade IPv6 nativa, o ISP ou roteador atribuiria um endereço GUA (Global Unicast Address, prefixo `2xxx` ou `3xxx`) via DHCPv6 ou SLAAC, o que não ocorre aqui.

> **Resumo:** O host possui conectividade IPv6 funcional **apenas dentro da rede Tailscale**. Para uso geral da internet via IPv6, seria necessário: (a) suporte IPv6 nativo do ISP/roteador repassado ao WSL, ou (b) configuração de um túnel IPv6 (ex.: 6to4, Teredo, ou Hurricane Electric).