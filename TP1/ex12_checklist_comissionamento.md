# Exercício 12 — Checklist de Comissionamento de Conectividade

> **Contexto:** Este checklist foi produzido para validar rapidamente se um host Linux (ou WSL2) está operacionalmente pronto para ser colocado em uso. Cada item deve ser executado em ordem, com evidência registrada. Tempo estimado de execução: 10 minutos.

---

## Checklist (10 itens)

| # | Objetivo | Comando(s) | Evidência Esperada | Critério |
|---|---|---|---|---|
| 1 | Endereço IPv4 atribuído e interface UP | `ip -4 addr show` | Pelo menos uma interface com `inet x.x.x.x/xx scope global` e estado `UP` | **Passou** se existir endereço global em interface UP; **Falhou** se todas as interfaces estiverem DOWN ou sem IP |
| 2 | Endereço IPv6 presente | `ip -6 addr show` | Pelo menos uma interface com `inet6 fe80::/64 scope link` | **Passou** se existir link-local em ao menos uma interface; **Falhou** se nenhuma interface tiver IPv6 |
| 3 | Rota default IPv4 configurada | `ip route \| grep default` | Linha `default via x.x.x.x dev <iface>` presente | **Passou** se existir rota default; **Falhou** se saída vazia |
| 4 | Rota default IPv6 (opcional em WSL) | `ip -6 route \| grep default` | Linha `default via ...` presente, ou justificativa documentada de ausência | **Passou** se existir; **Aceitável** em WSL2 sem IPv6 nativo desde que documentado |
| 5 | DNS funcional | `cat /etc/resolv.conf` e `getent hosts google.com` | `nameserver` configurado; `getent` retorna IP para google.com | **Passou** se ambos retornarem resultado; **Falhou** se `getent` retornar vazio |
| 6 | `/etc/hosts` íntegro | `cat /etc/hosts` | Contém `127.0.0.1 localhost` e hostname mapeado em `127.0.1.1` | **Passou** se entradas mínimas estiverem presentes; **Falhou** se arquivo vazio ou ausente |
| 7 | Hostname definido e consistente | `hostname` e `hostnamectl` | `hostname` e `Static hostname` do `hostnamectl` retornam o mesmo valor; `/etc/hostname` consistente | **Passou** se os três coincidirem; **Falhou** se houver divergência |
| 8 | Portas TCP em escuta verificadas | `ss -tulpen \| grep LISTEN` | Apenas portas esperadas visíveis; nenhuma porta sensível exposta em `0.0.0.0` sem justificativa | **Passou** se serviços de DB estiverem em `127.0.0.1`; **Falhou** se MySQL/Redis/Mongo estiverem expostos externamente |
| 9 | IP atribuído via DHCP ou estático documentado | `ip addr` + `cat /etc/resolv.conf` | IP, gateway e DNS identificados e registrados | **Passou** se os três parâmetros estiverem documentados; **Falhou** se gateway ou DNS estiverem ausentes |
| 10 | IP Forwarding em estado adequado ao papel do host | `sysctl net.ipv4.ip_forward` | `= 0` para servidores/estações de trabalho; `= 1` apenas se o host é roteador intencional | **Passou** se o valor for coerente com o papel; **Falhou** se forwarding estiver habilitado sem justificativa em host não-roteador |

---

## Evidências Reais (6 itens executados)

---

### Item 1 — Endereço IPv4

```
❯ ip -4 addr show
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 26.76.11.189/8 brd 26.255.255.255 scope global noprefixroute eth0
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 192.168.1.190/24 brd 192.168.1.255 scope global noprefixroute eth1
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2800 qdisc mq state UP group default qlen 1000
    inet 10.140.53.131/24 brd 10.140.53.255 scope global noprefixroute eth2
6: eth3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1280 qdisc mq state UP group default qlen 1000
    inet 100.66.217.127/32 brd 100.66.217.127 scope global noprefixroute eth3
```

**Resultado: ✅ Passou.** Quatro interfaces com endereço IPv4 global e estado UP. Interface principal: eth1 com `192.168.1.190/24`.

> **📸 Screenshot sugerido:** capturar `ip -4 addr show` com as linhas `inet` visíveis.

---

### Item 2 — Endereço IPv6

```
❯ ip -6 addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host
6: eth3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1280 state UP qlen 1000
    inet6 fd7a:115c:a1e0::3301:d981/128 scope global nodad noprefixroute
    inet6 fe80::af88:96bb:8610:3502/64 scope link nodad noprefixroute
```

**Resultado: ✅ Passou.** Link-local `fe80::` presente em eth3. Endereço global ULA da Tailscale também presente. Interfaces de LAN (eth0, eth1, eth2) sem IPv6 — limitação do WSL2, documentada.

---

### Item 3 — Rota Default IPv4

```
❯ ip route | grep default
default via 192.168.1.1 dev eth1 proto kernel metric 25
default via 26.0.0.1 dev eth0 proto kernel metric 9257
default via 25.255.255.254 dev eth2 proto kernel metric 10034
```

**Resultado: ✅ Passou.** Rota default primária via `192.168.1.1` por eth1 (metric 25). Duas rotas de failover adicionais presentes.

---

### Item 4 — Rota Default IPv6

```
❯ ip -6 route | grep default
(sem saída)
```

**Resultado: ⚠️ Aceitável com ressalva.** Não existe rota default IPv6 (`::/0`). Em WSL2 sem suporte IPv6 nativo do ISP, este comportamento é esperado e não representa falha operacional para uso geral. O host possui IPv6 funcional apenas dentro da rede Tailscale (`fd7a:115c:a1e0::/48`). **Documentado:** host não está pronto para conectividade IPv6 à internet pública.

---

### Item 5 — DNS Funcional

```
❯ cat /etc/resolv.conf
nameserver 10.255.255.254
search tail7f9064.ts.net

❯ getent hosts google.com
2800:3f0:4004:80d::200e google.com
```

**Resultado: ✅ Passou.** Nameserver configurado (`10.255.255.254` — DNS stub WSL). `getent hosts` resolveu google.com com sucesso, retornando endereço IPv6 (indicando preferência AAAA via Tailscale). Resolução de nomes operacional.

> **📸 Screenshot sugerido:** capturar os dois comandos em sequência — `cat /etc/resolv.conf` e `getent hosts google.com` — para evidenciar DNS funcional.

---

### Item 6 — `/etc/hosts` Íntegro

```
❯ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       WinCore.localdomain     WinCore
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

**Resultado: ✅ Passou.** Entradas mínimas presentes: `127.0.0.1 localhost` e hostname `WinCore` mapeado em `127.0.1.1`. Nenhuma entrada incorreta ou órfã detectada. Arquivo gerado automaticamente pelo WSL — íntegro.

---

### Item 7 — Hostname Consistente

```
❯ hostname
WinCore

❯ hostnamectl | grep "Static hostname"
 Static hostname: WinCore

❯ cat /etc/hostname
WinCore
```

**Resultado: ✅ Passou.** As três fontes de verdade (runtime, systemd, arquivo) retornam `WinCore` de forma consistente.

> **📸 Screenshot sugerido:** capturar os três comandos em sequência evidenciando o mesmo valor em todas as fontes.

---

### Item 8 — Portas TCP em Escuta

```
❯ ss -tulpen | grep LISTEN
tcp  LISTEN  127.0.0.1:33060   ... mysql.service
tcp  LISTEN  127.0.0.1:27017   ... mongod.service
tcp  LISTEN  127.0.0.1:3306    ... mysql.service
tcp  LISTEN  127.0.0.1:5432    ... postgresql
tcp  LISTEN  127.0.0.1:6379    ... redis-server.service
tcp  LISTEN  *:4000            ...
tcp  LISTEN  *:11434           ...
tcp  LISTEN  *:3000            ...
tcp  LISTEN  *:3002            ...
```

**Resultado: ✅ Passou (com observação).** Serviços de banco de dados (MySQL, MongoDB, PostgreSQL, Redis) corretamente restritos a `127.0.0.1`. Portas 3000, 3002, 4000 e 11434 expostas em todas as interfaces (`*`) — aceitável em ambiente de desenvolvimento, mas devem ser restritas antes de ir a produção. Observação registrada: **porta 11434 (Ollama) exposta externamente — avaliar necessidade.**

---

### Item 10 — IP Forwarding

```
❯ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
```

**Resultado: ⚠️ Atenção.** Forwarding está habilitado (`= 1`). Em WSL2, isso é configurado automaticamente pelo subsistema de rede do Windows e é esperado neste contexto. Em um servidor Linux convencional que não é roteador, este valor deveria ser `0`. **Documentado:** habilitado pelo WSL2 por design — não representa risco neste ambiente específico, mas deve ser verificado e justificado em qualquer host antes do comissionamento.

> **📸 Screenshot sugerido:** capturar `sysctl net.ipv4.ip_forward` retornando `1`, com nota sobre o contexto WSL.

---

## Decisões Técnicas

### 1. Sobre Rotas — Como agir em caso de falha de rota default

A rota default é o caminho de último recurso do kernel para qualquer destino sem rota mais específica. Sua ausência ou incorreção isola o host da internet e de redes externas, mesmo que o endereço IP e o enlace físico estejam funcionando. O diagnóstico começa sempre em `ip route | grep default`: se vazio, a rota foi perdida; se presente mas com gateway errado, o tráfego está sendo enviado ao lugar errado.

Em caso de falha, a ação imediata é restaurar a rota manualmente com o gateway correto (identificado no baseline documentado antes do comissionamento):

```bash
sudo ip route add default via <gateway> dev <interface>
```

Exemplo para este host: `sudo ip route add default via 192.168.1.1 dev eth1`

Esta ação é temporária — não sobrevive a reinicializações. Para corrigir de forma persistente, a rota deve ser configurada no gerenciador de rede em uso (Netplan em `/etc/netplan/*.yaml`, NetworkManager via `nmcli`, ou arquivo de configuração da distro). A regra operacional é: **sempre documentar o gateway correto de cada interface no momento do comissionamento**, antes de qualquer alteração — esse baseline é o que permite o rollback rápido em plantão.

Em ambientes com múltiplas rotas default (como este host com eth0, eth1 e eth2), verificar também as métricas: se a interface primária caiu e a métrica do failover não está sendo ativada, pode ser necessário ajustar `metric` manualmente ou verificar se o link detectou a queda.

---

### 2. Sobre Resolução — Quando usar `/etc/hosts` e quando não usar

`/etc/hosts` é uma ferramenta de intervenção local, não de configuração permanente. Deve ser usado quando se precisa sobrescrever ou fixar uma resolução de nome **em um único host específico**, de forma temporária e controlada — por exemplo, redirecionar `api.empresa.com` para um servidor de homologação durante testes, ou nomear serviços locais de desenvolvimento sem precisar de servidor DNS.

O critério prático é simples: se a mudança precisa ser vista por **mais de um host**, use DNS. Se precisa ser vista **apenas por este host** e por tempo limitado, `/etc/hosts` é aceitável — desde que a entrada seja removida após o uso e documentada com data e motivo.

Em produção, `/etc/hosts` como solução permanente cria dois problemas graves. O primeiro é de propagação: quando um IP muda, a atualização precisa ser feita manualmente em cada máquina que tem a entrada — em ambientes com dezenas de servidores, isso é inviável e garante inconsistências. O segundo é de rastreabilidade: entradas em `/etc/hosts` não têm TTL, não ficam visíveis em ferramentas de DNS e não são monitoradas — uma entrada errada pode ficar anos ativa sem que ninguém perceba, causando falhas intermitentes difíceis de diagnosticar.

A regra operacional é: **`/etc/hosts` para emergências e desenvolvimento local; DNS para qualquer coisa que precise durar ou se propagar**.