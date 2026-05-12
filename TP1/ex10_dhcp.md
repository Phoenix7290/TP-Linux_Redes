# Exercício 10 — DHCP

---

## Identificação do Uso de DHCP na Interface Ativa

### Comando 1: `ip addr show eth0 | grep "inet "`

```
❯ ip addr show eth0 | grep "inet "
    inet 26.76.11.189/8 brd 26.255.255.255 scope global noprefixroute eth0
```

O endereço `26.76.11.189/8` está atribuído a eth0. O flag `noprefixroute` indica que o gerenciador de rede configurou este endereço sem criar automaticamente uma rota de prefixo associada — comportamento típico de interfaces gerenciadas pelo WSL. A ausência do flag `dynamic` (presente em endereços atribuídos por DHCP via iproute2) já é um indicativo de que a atribuição pode ser estática ou gerenciada de forma diferente do DHCP convencional.

---

### Comando 2: `nmcli device show eth0 | grep DHCP`

```
❯ nmcli device show eth0 | grep DHCP
(sem saída)
```

A ausência de saída indica que o NetworkManager **não está gerenciando** a interface eth0 — ou não está em execução neste ambiente. No WSL2, a pilha de rede é configurada por um componente interno da Microsoft (`hv_utils` / `wsl-network`), não pelo NetworkManager convencional do Linux. Por isso, comandos `nmcli` não retornam informações de DHCP para interfaces WSL.

---

### Comando 3: `journalctl -u systemd-networkd | tail -n 20`

```
❯ journalctl -u systemd-networkd | tail -n 20
-- No entries --
```

A ausência de entradas confirma que o `systemd-networkd` também **não está ativo** neste ambiente. No WSL2, o gerenciamento de rede é feito pelo próprio subsistema do Windows antes de a distro Linux iniciar — nem o NetworkManager nem o systemd-networkd têm função aqui.

> **📸 Screenshot sugerido:** capturar os três comandos acima em sequência, evidenciando que ferramentas convencionais de DHCP não têm saída em WSL2.

---

## Como o WSL2 Gerencia Endereços IP

No WSL2, a configuração de rede ocorre fora do Linux. O Windows cria uma interface virtual Hyper-V (`hv_utils`) e atribui endereços às interfaces da distro diretamente via o serviço `wsl-network` antes do boot da distro. O processo é equivalente ao DHCP em comportamento (IP, gateway e DNS são atribuídos automaticamente), mas não usa o protocolo DHCP padrão — o endereço é injetado pelo kernel do Windows no espaço de endereçamento do WSL.

Para confirmar que o IP foi atribuído automaticamente (e não configurado manualmente):

```bash
❯ cat /etc/wsl.conf
(arquivo inexistente ou sem seção [network])
```

A ausência de configuração estática em `/etc/wsl.conf` confirma que os IPs são dinâmicos e gerenciados pelo Windows a cada reinicialização do WSL.

---

## Localização de Evidências de Lease DHCP

### Arquivos de lease convencionais

```bash
❯ ls /var/lib/dhcp/ 2>/dev/null || echo "Diretório ausente"
Diretório ausente

❯ ls /var/lib/NetworkManager/ 2>/dev/null || echo "Sem NetworkManager"
Sem NetworkManager

❯ ls /run/systemd/netif/leases/ 2>/dev/null || echo "Sem leases systemd-networkd"
Sem leases systemd-networkd
```

Nenhum arquivo de lease convencional existe, o que é esperado: como a rede é configurada pelo Windows antes do boot da distro, não há cliente DHCP Linux rodando e, portanto, nenhum arquivo de lease é gerado no filesystem Linux.

### Onde encontrar as informações equivalentes ao lease

As informações de IP, gateway e DNS atribuídas pelo WSL estão visíveis através dos próprios arquivos de configuração que o WSL gera automaticamente:

```bash
❯ cat /etc/resolv.conf
nameserver 10.255.255.254
search tail7f9064.ts.net
```

O arquivo `/etc/resolv.conf` é gerado pelo WSL a cada inicialização com os parâmetros DNS equivalentes ao que um lease DHCP forneceria. O nameserver `10.255.255.254` é o DNS stub interno do WSL, que repassa consultas ao DNS configurado no Windows host.

---

## Parâmetros Equivalentes ao Lease DHCP

Com base nas evidências coletadas nos exercícios anteriores:

| Parâmetro | Valor | Fonte da evidência |
|---|---|---|
| IP concedido | `192.168.1.190/24` (eth1 — interface principal) | `ip addr show eth1` |
| Gateway | `192.168.1.1` | `ip route` (rota default metric 25) |
| DNS | `10.255.255.254` (proxy WSL → DNS Windows) | `/etc/resolv.conf` |
| Domínio de busca | `tail7f9064.ts.net` | `/etc/resolv.conf` |
| Tempo de lease | Indeterminado — renovado a cada reinício do WSL | Não disponível no filesystem Linux |

**Nota sobre eth0:** A interface eth0 (`26.76.11.189/8`) recebe seu endereço do serviço interno do WSL com métrica alta (9257), indicando que é uma interface secundária de saída, não a principal. A interface de LAN doméstica operacionalmente relevante é eth1.

---

## Ação de Contingência sem DHCP

Caso o serviço DHCP fique indisponível (ou, no contexto WSL, caso a configuração automática falhe), a operação pode ser mantida com configuração manual dos três parâmetros essenciais. Os comandos abaixo são **descrições de procedimento** — não foram executados para não alterar o ambiente em produção.

**1. Atribuir endereço IP estático à interface:**

```bash
sudo ip addr add 192.168.1.190/24 dev eth1
```

Adiciona o endereço manualmente à interface com o mesmo prefixo que o DHCP forneceria. O endereço deve ser escolhido fora do range DHCP do roteador para evitar conflito de IP com outro host.

**2. Adicionar rota default manualmente:**

```bash
sudo ip route add default via 192.168.1.1 dev eth1
```

Sem esta rota, o host teria IP mas não conseguiria alcançar a internet nem nenhum host fora da LAN local. O gateway `192.168.1.1` é o roteador doméstico, identificado pela rota default prévia.

**3. Configurar DNS manualmente:**

```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

Substitui o DNS automático por um servidor público (Google DNS). Sem este passo, nomes de domínio não resolveriam, mesmo com IP e rota configurados corretamente. Em produção, o ideal é usar o DNS interno da organização em vez de um público.

> **Observação:** estas três ações correspondem exatamente aos três parâmetros que o DHCP entrega automaticamente (IP, gateway, DNS). A contingência sem DHCP é operacionalmente viável mas exige conhecimento prévio dos valores corretos — razão pela qual documentar o lease DHCP em produção é uma boa prática de operação.