# Exercício 4 — Routing Table com `ip route`

---

## Evidência: Tabela de Rotas Atual

```
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
```

> **📸 Screenshot sugerido:** capturar a saída completa de `ip route` no terminal.

---

## Identificação das Rotas

### Rota Default (Gateway)

Há **três rotas default** na tabela. O kernel seleciona a de menor métrica:

| Entrada | Gateway | Interface | Métrica | Status |
|---|---|---|---|---|
| `default via 192.168.1.1 dev eth1` | 192.168.1.1 | eth1 | **25** | ✅ Ativa (primária) |
| `default via 26.0.0.1 dev eth0` | 26.0.0.1 | eth0 | 9257 | Failover |
| `default via 25.255.255.254 dev eth2` | 25.255.255.254 | eth2 | 10034 | Failover |

**Rota default ativa:** `default via 192.168.1.1 dev eth1 proto kernel metric 25`

- **Gateway:** 192.168.1.1 (roteador doméstico)
- **Interface:** eth1 (LAN doméstica, 192.168.1.190/24)
- **Métrica:** 25 (menor valor = maior prioridade)
- **Proto:** `kernel` — rota instalada automaticamente pelo gerenciador de rede (WSL/NetworkManager)

Todo tráfego sem rota mais específica será encaminhado para 192.168.1.1 via eth1.

---

### Duas Rotas Conectadas (Diretamente Alcançáveis)

Rotas conectadas são criadas automaticamente quando um IP é atribuído a uma interface — representam redes diretamente acessíveis sem necessidade de gateway.

**Rota conectada 1:** `192.168.1.0/24 dev eth1 proto kernel scope link metric 281`
- **Prefixo:** 192.168.1.0/24 (sub-rede LAN doméstica)
- **Interface:** eth1
- **Significado:** Qualquer destino no range 192.168.1.0–192.168.1.255 é acessível diretamente via eth1, sem passar pelo gateway. Ex.: 192.168.1.210 (Raspberry Pi) é alcançado diretamente.

**Rota conectada 2:** `10.140.53.0/24 dev eth2 proto kernel scope link metric 291`
- **Prefixo:** 10.140.53.0/24 (rede WSL interna)
- **Interface:** eth2
- **Significado:** Hosts no range 10.140.53.0–10.140.53.255 são acessíveis diretamente via eth2. Esta rede é utilizada para comunicação entre o WSL2 e o host Windows.

---

## Simulação de Decisões de Roteamento

O kernel Linux seleciona rotas por **especificidade** (prefixo mais longo vence) e, em caso de empate, por **métrica** (menor valor vence).

---

### Destino 1 — Local: `192.168.1.210` (Raspberry Pi na LAN)

**Rota vencedora:** `192.168.1.0/24 dev eth1 proto kernel scope link metric 281`

**Justificativa:** O endereço 192.168.1.210 pertence à sub-rede 192.168.1.0/24 (prefixo /24). Esta rota conectada é **mais específica** que qualquer rota default (/0), portanto vence por especificidade. O pacote é entregue diretamente via eth1, sem passar pelo gateway — o kernel busca o MAC de 192.168.1.210 via ARP e encaminha o quadro diretamente.

---

### Destino 2 — Público: `8.8.8.8` (DNS do Google)

**Rota vencedora:** `default via 192.168.1.1 dev eth1 metric 25`

**Justificativa:** O endereço 8.8.8.8 não pertence a nenhuma sub-rede diretamente conectada (não cobre nenhuma das rotas /24 ou /32 da tabela). O kernel percorre todas as entradas e nenhuma é mais específica que /0. Entre as três rotas default disponíveis, a de **menor métrica** (25, via eth1) vence. O pacote é enviado ao gateway 192.168.1.1, que o encaminha para a internet.

---

### Destino 3 — Fora da LAN: `172.16.50.1` (rede corporativa hipotética)

**Rota vencedora:** `default via 192.168.1.1 dev eth1 metric 25`

**Justificativa:** O endereço 172.16.50.1 não coincide com nenhuma rota conectada ou específica na tabela. Não há rotas para o bloco 172.16.0.0/12. Portanto, o kernel cai no default, novamente escolhendo eth1 (metric 25) como caminho. **Atenção:** se esse destino fosse uma rede interna corporativa acessível via VPN ou outra interface, seria necessário adicionar uma rota específica (ex.: `ip route add 172.16.0.0/12 via <gateway-vpn> dev eth3`) para que o tráfego não fosse incorretamente enviado ao roteador doméstico.

---

## Seção: Riscos de Rota Default Incorreta

### Risco 1 — Perda total de conectividade externa

Se a rota default apontar para um gateway inexistente ou incorreto (ex.: gateway errado após troca de rede), todo tráfego destinado à internet será enviado para um IP que não responde. O host terá endereço IP válido e enlace ativo, mas nenhum pacote chegará ao destino — o diagnóstico é confuso porque `ip addr` e `ip link` mostram tudo normal. Somente `ip route` e um `ping` ao gateway revelariam o problema.

### Risco 2 — Blackhole silencioso (rota com gateway inalcançável)

Uma rota default com gateway em sub-rede diferente da interface (ex.: gateway 10.0.0.1 em uma interface com IP 192.168.1.x/24) cria um "blackhole" silencioso: os pacotes são enviados, mas o ARP não consegue resolver o MAC do gateway e eles são descartados sem mensagem de erro para a aplicação. Este cenário é especialmente difícil de diagnosticar porque a rota aparece correta em `ip route`, mas `arping <gateway>` não retorna resposta.

### Risco 3 — Roteamento assimétrico em ambientes multi-interface

Com múltiplas rotas default (como neste host, com eth0, eth1 e eth2), uma alteração incorreta de métricas pode fazer com que pacotes de entrada cheguem por eth1 e respostas saiam por eth0, passando por gateways diferentes. Isso causa falhas intermitentes em conexões TCP (que exigem os mesmos IPs de origem/destino em ambas as direções), frequentemente manifestas como "conexão cai após alguns segundos" ou "timeout em downloads". Ferramentas como `ip rule` e `ip route show table all` são necessárias para diagnosticar roteamento assimétrico.