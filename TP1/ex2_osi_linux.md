# Exercício 2 — OSI Aplicado: Tabela Operacional OSI → Linux

---

## Tabela OSI → Linux

| Comando | Camada OSI | Justificativa (resumo) |
|---|---|---|
| `ip link` | Camada 2 — Enlace (Data Link) | Exibe interfaces, MACs e estado físico |
| `arp -n` | Camada 2 — Enlace (Data Link) | Tabela de resolução IP→MAC (protocolo ARP) |
| `ip addr` | Camada 3 — Rede (Network) | Endereços IPv4/IPv6 atribuídos às interfaces |
| `ping` | Camada 3 — Rede (Network) | Testa alcançabilidade IP via ICMP |
| `ss -tuln` | Camada 4 — Transporte (Transport) | Lista sockets TCP/UDP em escuta |
| `netstat -tuln` | Camada 4 — Transporte (Transport) | Lista conexões e portas ativas (legado) |
| `who` | Camada 5 — Sessão (Session) | Exibe sessões de usuário ativas |
| `openssl s_client -connect google.com:443` | Camada 6 — Apresentação (Presentation) | Negocia e inspeciona sessão TLS/SSL |
| `wget` | Camada 7 — Aplicação (Application) | Realiza requisição HTTP/HTTPS completa |

---

## Evidências e Justificativas Detalhadas

---

### `ip link` — Camada 2 (Enlace)

```
❯ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:50:2a:35:a7:24 brd ff:ff:ff:ff:ff:ff
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether bc:fc:e7:ca:84:05 brd ff:ff:ff:ff:ff:ff
6: eth3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1280 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 00:15:5d:ff:76:1c brd ff:ff:ff:ff:ff:ff
7: eth4: <BROADCAST,MULTICAST> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
    link/ether 58:02:05:51:06:9e brd ff:ff:ff:ff:ff:ff
```

**Justificativa:** `ip link` opera diretamente na Camada 2 (Enlace/Data Link) do modelo OSI. A saída exibe o endereço MAC de cada interface (`link/ether`), que é o identificador de camada 2 — sem ele, quadros Ethernet não podem ser endereçados. As flags `UP` e `LOWER_UP` indicam, respectivamente, que a interface está habilitada administrativamente e que o enlace físico (cabo ou sinal wireless) está ativo. O campo MTU (Maximum Transmission Unit) define o tamanho máximo do quadro na camada 2 — valores diferentes do padrão 1500 (como eth3 com 1280) indicam encapsulamento adicional, como ocorre em túneis VPN. A ausência de endereços IP neste comando confirma que ele é estritamente de camada 2.

> **📸 Screenshot sugerido:** capturar `ip link` com saída colorida do terminal mostrando os estados UP/DOWN.

---

### `arp -n` — Camada 2 (Enlace)

```
❯ arp -n
Address                  HWtype  HWaddress           Flags Mask  Iface
169.254.73.152           ether   00:11:22:33:44:55   CM          loopback0
169.254.73.152           ether   00:11:22:33:44:55   CM          eth1
169.254.73.152           ether   00:11:22:33:44:55   CM          eth3
169.254.73.152           ether   00:11:22:33:44:55   CM          eth0
169.254.73.152           ether   00:11:22:33:44:55   CM          eth2
192.168.1.1              ether   d8:44:89:71:b4:0c   C           eth1
```

**Justificativa:** O protocolo ARP (Address Resolution Protocol) é definido no RFC 826 e opera exclusivamente na Camada 2 (Enlace). Sua função é mapear endereços IP (Camada 3) para endereços MAC (Camada 2), permitindo que quadros Ethernet sejam entregues ao destinatário correto na rede local. A saída de `arp -n` exibe a tabela ARP do kernel, onde cada linha associa um IP a um MAC (`HWaddress`) e à interface (`Iface`) correspondente. O flag `C` indica entrada aprendida dinamicamente; `CM` indica entrada permanente (comum em interfaces WSL). Sem esta resolução, mesmo que IP e rota estejam corretos, os quadros não poderiam ser encaminhados no segmento local.

---

### `ip addr` — Camada 3 (Rede)

```
❯ ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 26.76.11.189/8 brd 26.255.255.255 scope global noprefixroute eth0
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.1.190/24 brd 192.168.1.255 scope global noprefixroute eth1
6: eth3: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 100.66.217.127/32 scope global noprefixroute eth3
    inet6 fd7a:115c:a1e0::3301:d981/128 scope global nodad noprefixroute
    inet6 fe80::af88:96bb:8610:3502/64 scope link nodad noprefixroute
```

**Justificativa:** `ip addr` expõe os endereços IPv4 e IPv6 atribuídos a cada interface, que são identificadores de Camada 3 (Rede) no modelo OSI. Ao contrário de `ip link`, este comando revela informações de roteamento: o prefixo (`/24`, `/8`, `/32`) define a sub-rede à qual o host pertence, determinando quais destinos são locais e quais exigem roteamento. O campo `scope global` indica endereço roteável globalmente, enquanto `scope link` (como o fe80::/64) é restrito ao segmento local. A presença de endereços IPv4 e IPv6 em eth3 indica dual-stack via Tailscale, confirmando capacidade de comunicação em ambos os protocolos de camada 3.

---

### `ping` — Camada 3 (Rede)

```
❯ ping 192.168.1.210
PING 192.168.1.210 (192.168.1.210) 56(84) bytes of data.
64 bytes from 192.168.1.210: icmp_seq=1 ttl=64 time=1.64 ms
64 bytes from 192.168.1.210: icmp_seq=2 ttl=64 time=0.846 ms
20 packets transmitted, 20 received, 0% packet loss
rtt min/avg/max/mdev = 0.781/0.893/1.644/0.179 ms
```

**Justificativa:** O `ping` utiliza o protocolo ICMP (Internet Control Message Protocol), definido no RFC 792, que opera na Camada 3 (Rede) do modelo OSI. Envia pacotes ICMP Echo Request e aguarda ICMP Echo Reply, medindo latência e verificando alcançabilidade IP entre dois hosts. O campo `ttl=64` (Time To Live) é um atributo do cabeçalho IP, confirmando operação em camada 3 — cada roteador decrementa o TTL em 1 ao encaminhar o pacote. O `ping` não abre portas TCP/UDP (camada 4) nem estabelece sessão de aplicação (camada 7), validando isoladamente a conectividade de rede. A latência de ~0,9 ms para 192.168.1.210 (Raspberry Pi local) confirma alcançabilidade dentro da mesma LAN (eth1).

> **📸 Screenshot sugerido:** capturar saída do `ping` mostrando os campos `ttl`, `time` e a linha de estatísticas.

---

### `ss -tuln` — Camada 4 (Transporte)

```
❯ ss -tuln
Proto  State   Local Address:Port  Peer Address:Port
tcp    LISTEN  127.0.0.1:33060     0.0.0.0:*
tcp    LISTEN  127.0.0.1:27017     0.0.0.0:*
tcp    LISTEN  127.0.0.1:3306      0.0.0.0:*
tcp    LISTEN  127.0.0.1:5432      0.0.0.0:*
tcp    LISTEN  127.0.0.1:6379      0.0.0.0:*
tcp    LISTEN  *:4000              *:*
tcp    LISTEN  *:11434             *:*
udp    UNCONN  0.0.0.0:5353        0.0.0.0:*
```

**Justificativa:** `ss` (socket statistics) inspeciona sockets diretamente no kernel Linux, operando na Camada 4 (Transporte). As colunas `Proto` (TCP ou UDP), `Local Address:Port` e `State` (LISTEN, ESTABLISHED, etc.) são conceitos exclusivos da camada de transporte — portas são o mecanismo de multiplexação que permite múltiplos serviços coexistirem no mesmo endereço IP. O estado `LISTEN` indica que o processo aguarda conexões entrantes naquela porta. A distinção entre `127.0.0.1:port` (só loopback) e `*:port` (todas as interfaces) é decisão de bind da camada 4 que impacta diretamente a superfície de ataque do sistema. UDP em `UNCONN` indica socket sem conexão estabelecida, típico de protocolos como DNS e mDNS.

---

### `netstat -tuln` — Camada 4 (Transporte)

```
❯ netstat -tuln
Proto Recv-Q Send-Q Local Address    Foreign Address  State
tcp        0      0 127.0.0.1:33060  0.0.0.0:*        LISTEN
tcp        0      0 127.0.0.1:3306   0.0.0.0:*        LISTEN
tcp6       0      0 :::4000          :::*              LISTEN
tcp6       0      0 :::3000          :::*              LISTEN
udp        0      0 0.0.0.0:5353     0.0.0.0:*
```

**Justificativa:** `netstat -tuln` é a ferramenta legada equivalente ao `ss`, também operando na Camada 4 (Transporte). As flags `-t` (TCP), `-u` (UDP), `-l` (listening) e `-n` (numérico, sem resolução de nomes) exibem os mesmos sockets de transporte. O prefixo `tcp6` e endereços `:::port` indicam sockets IPv6 dual-stack, aceitando conexões tanto IPv4 quanto IPv6. A exibição de `Recv-Q` e `Send-Q` mostra pacotes enfileirados no buffer de recepção/envio do socket, permitindo identificar congestionamento ou backlog em serviços sobrecarregados. Apesar de funcionalidade similar ao `ss`, `netstat` é considerado obsoleto em distribuições modernas, sendo `ss` a ferramenta recomendada.

---

### `who` — Camada 5 (Sessão)

```
❯ who
root     pts/3        2026-05-11 22:15
ryansubu pts/1        2026-05-11 22:15
```

**Justificativa:** O comando `who` exibe sessões de login ativas no sistema, mapeando para a Camada 5 (Sessão) do modelo OSI. A camada de sessão é responsável por estabelecer, gerenciar e encerrar sessões de comunicação entre processos. Cada linha da saída representa uma sessão ativa: o campo `pts/N` identifica um pseudo-terminal (sessão SSH ou terminal local), e o timestamp indica quando a sessão foi iniciada. Em contexto de rede, cada entrada `pts/` pode corresponder a uma sessão SSH remota, evidenciando que a Camada 5 gerencia o ciclo de vida dessas conexões — autenticação, manutenção e encerramento. O usuário `root` em `pts/3` e `ryansubu` em `pts/1` indicam duas sessões paralelas independentes.

---

### `openssl s_client -connect google.com:443` — Camada 6 (Apresentação)

```
❯ openssl s_client -connect google.com:443
CONNECTED(00000003)
depth=2 C = US, O = Google Trust Services LLC, CN = GTS Root R1
depth=1 C = US, O = Google Trust Services, CN = WR2
depth=0 CN = *.google.com
Verification: OK
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 256 bit
SSL handshake has read 6636 bytes and written 392 bytes
Verify return code: 0 (ok)
```

**Justificativa:** `openssl s_client` opera na Camada 6 (Apresentação) do modelo OSI. A camada de apresentação é responsável por tradução, criptografia e compressão de dados — e TLS (Transport Layer Security) é o protocolo que implementa essas funções no contexto de redes modernas. A saída revela: a cadeia de certificados (`depth=0,1,2`) que valida a identidade do servidor via PKI, o protocolo negociado (`TLSv1.3`), a cifra escolhida (`TLS_AES_256_GCM_SHA384`) que define o algoritmo de criptografia simétrica dos dados em trânsito, e o resultado de verificação (`Verify return code: 0 = ok`), confirmando que o certificado é válido e confiável. Sem esta camada, os dados da aplicação trafegam em texto claro.

> **📸 Screenshot sugerido:** capturar a saída do `openssl s_client` mostrando a cadeia de certificados e o cipher suite negociado.

---

### `wget` — Camada 7 (Aplicação)

```
❯ curl -I https://www.google.com
HTTP/2 200
content-type: text/html; charset=ISO-8859-1
server: gws
date: Tue, 12 May 2026 01:35:04 GMT
```

*(Nota: wget e curl operam na mesma camada; curl -I foi usado como alternativa equivalente para evidência da camada de aplicação)*

**Justificativa:** `wget` (e `curl`) operam na Camada 7 (Aplicação) do modelo OSI, utilizando o protocolo HTTP/HTTPS para transferência de dados. Esta é a camada mais próxima do usuário final, onde protocolos definem a semântica das requisições — método (`GET`, `HEAD`), headers (`content-type`, `accept-ch`), códigos de status (`200 OK`, `404 Not Found`) e versões de protocolo (`HTTP/2`). O response header `content-type: text/html` exemplifica a função de apresentação de dados da camada de aplicação. A execução de `wget` pressupõe que todas as camadas inferiores estejam funcionando: enlace ativo, IP roteável, TCP conectado, TLS negociado e DNS resolvido — tornando-o o teste mais completo de conectividade ponta-a-ponta.