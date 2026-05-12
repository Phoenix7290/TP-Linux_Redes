# Exercício 12 - Checklist de Comissionamento de Conectividade

## Checklist (10 itens)

| # | Objetivo                          | Comando                              | Evidência Esperada                  | Status |
|---|-----------------------------------|--------------------------------------|-------------------------------------|--------|
| 1 | IPv4 Address                      | `ip -4 addr show eth0`              | IP no range 172.x ou 192.168.x     | Passou |
| 2 | IPv6 Address                      | `ip -6 addr`                        | Presença de fe80::                  | Passou |
| 3 | Rota Default IPv4                 | `ip route | grep default`            | Existe rota default                 | Passou |
| 4 | Rota Default IPv6                 | `ip -6 route | grep default`         | Pode não existir (WSL)              | Falhou |
| 5 | DNS Resolution                    | `cat /etc/resolv.conf`              | Nameservers configurados            | Passou |
| 6 | /etc/hosts                        | `cat /etc/hosts`                    | localhost + hostname                | Passou |
| 7 | Hostname                          | `hostname`                          | Nome definido corretamente          | Passou |
| 8 | Portas TCP em escuta              | `ss -tulpen | grep LISTEN`           | Apenas portas esperadas             | Passou |
| 9 | DHCP / Configuração de IP         | `ip addr` + journalctl              | IP atribuído corretamente           | Passou |
|10 | IP Forwarding                     | `sysctl net.ipv4.ip_forward`        | Deve estar = 0 em produção          | Passou |

## Evidências Reais
*(Cole aqui trechos importantes dos comandos acima)*

## Decisões Técnicas

**1. Sobre Rotas**  
Em caso de falha de rota default, a ação recomendada é adicionar uma rota manual com `ip route add default via <gateway>` ou corrigir via NetworkManager/netplan.

**2. Sobre Resolução de Nomes**  
Usar `/etc/hosts` apenas para nomes internos de desenvolvimento ou serviços locais. Em produção, deve-se priorizar DNS para facilitar manutenção e consistência entre múltiplos hosts.