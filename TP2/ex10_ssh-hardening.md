# Exercício 9 — Vulnerabilidade típica: serviço em claro e risco operacional

---

## Tarefa 1 — Verificar portas de serviços legados/inseguros

### Comando: `nmap -sT -p 21,23,69,79,110,143 localhost`

```bash
❯ nmap -sT -p 21,23,69,79,110,143 localhost
Starting Nmap 7.94 ( https://nmap.org ) at 2025-05-28 15:02 -03
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000071s latency).

PORT    STATE  SERVICE
21/tcp  closed ftp
23/tcp  closed telnet
69/tcp  closed tftp
79/tcp  closed finger
110/tcp closed pop3
143/tcp closed imap

Nmap done: 1 IP address (1 host up) scanned in 0.06s
```

### Confirmação com netstat:

```bash
❯ netstat -tnlp | grep -E ':21|:23|:69|:79|:110|:143'
(sem saída)
```

**Evidência:** Nenhuma das portas associadas a serviços legados inseguros está em estado `LISTEN` no host. O `nmap` retorna `closed` para todas — o que significa que o kernel respondeu com `RST` (reset TCP), confirmando que não há processo algum em escuta nessas portas. A ausência de saída no `netstat` corrobora: nenhum socket ativo nessas portas.

---

## Tarefa 2 — "Não exposto": evidência e motivos

### Evidência consolidada — ausência de serviços inseguros:

```bash
❯ ss -tnlp | grep -E ':21 |:23 |:110 |:143 '
(sem saída)

❯ systemctl is-active telnetd ftp vsftpd 2>/dev/null
inactive
inactive
inactive
```

**Motivo 1 — Eliminação de vetor de captura de credenciais:** Telnet, FTP e POP3 transmitem usuário, senha e dados em texto plano na rede. Qualquer host na mesma LAN com acesso à interface de rede pode capturar essas credenciais com `tcpdump` sem qualquer autenticação adicional — o que transforma um serviço "aparentemente funcional" em um vetor trivial de comprometimento de conta.

**Motivo 2 — Redução de superfície de ataque e de carga de auditoria:** Cada serviço em escuta representa um processo adicional que precisa ser mantido, monitorado e auditado. Serviços legados raramente recebem patches de segurança ativos, e a presença deles obriga a manter versões desatualizadas ou aceitar risco de CVE conhecidos. Não expor é mais simples e mais seguro do que expor e tentar mitigar.

---

## Tarefa 3 — Mitigações mínimas para servidor corporativo

1. **Desinstalar ou desabilitar pacotes de serviços legados via gerenciador de pacotes e systemd:** executar `sudo apt purge telnetd vsftpd inetutils-ftpd` para remover os binários, e `sudo systemctl disable --now telnetd.socket` para garantir que não reiniciem após update ou reboot. A remoção do pacote é preferível à simples desativação do serviço, pois elimina o binário como superfície de ataque mesmo que o daemon seja ativado acidentalmente.

2. **Aplicar política de firewall (iptables/nftables/ufw) com deny-by-default para portas de entrada:** configurar o firewall com política padrão `DROP` em INPUT e permitir explicitamente apenas as portas necessárias (ex.: 22/TCP para SSH, 443/TCP para HTTPS). Qualquer tentativa de conexão a portas não autorizadas — incluindo 21, 23 e 110 — seria silenciosamente descartada pelo kernel antes de chegar a qualquer processo, adicionando uma camada de defesa independente do estado dos serviços.