# Exercício 3 — SSH em Operação: Configurar Serviço e Comprovar Acesso Remoto

---

## Status do Serviço SSH

### Comando: `systemctl status ssh`

```
❯ systemctl status ssh
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2026-05-28 11:47:03 UTC; 15min ago
       Docs: man:sshd(8)
             man:sshd_config(5)
   Main PID: 847 (sshd)
      Tasks: 1 (limit: 4614)
     Memory: 5.8M
        CPU: 42ms
     CGroup: /system.slice/ssh.service
             └─847 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

mai 28 11:47:03 pb-ryansubu-dev systemd[1]: Starting OpenBSD Secure Shell server...
mai 28 11:47:03 pb-ryansubu-dev sshd[847]: Server listening on 0.0.0.0 port 22.
mai 28 11:47:03 pb-ryansubu-dev sshd[847]: Server listening on :: port 22.
mai 28 11:47:03 pb-ryansubu-dev systemd[1]: Started OpenBSD Secure Shell server.
```

**Evidência:** O serviço `ssh.service` está `active (running)` desde o boot. O PID principal é `847` (processo `sshd` em modo listener). As linhas de log confirmam que o daemon está escutando em `0.0.0.0:22` (IPv4) e `:::22` (IPv6).

> Caso o serviço estivesse inativo, o comando para iniciar seria:
> ```bash
> sudo systemctl start ssh && sudo systemctl enable ssh
> ```

---

## Inspeção do `sshd_config`

### Comando: `grep -E '^(Port|PasswordAuthentication|PermitRootLogin)' /etc/ssh/sshd_config`

```
❯ grep -E '^(Port|PasswordAuthentication|PermitRootLogin)' /etc/ssh/sshd_config
Port 22
PasswordAuthentication yes
PermitRootLogin prohibit-password
```

**Valores efetivos registrados:**

| Diretiva | Valor | Implicação |
|---|---|---|
| `Port` | `22` | Porta padrão SSH — sem alteração de segurança por obscuridade |
| `PasswordAuthentication` | `yes` | Autenticação por senha habilitada — login com credencial funciona sem chave |
| `PermitRootLogin` | `prohibit-password` | Root pode logar, mas somente via chave pública — senha bloqueada para root |

**Observação de segurança:** `PasswordAuthentication yes` é aceitável em ambiente de laboratório, mas em produção recomenda-se `no` (forçando chave pública). `PermitRootLogin prohibit-password` é uma configuração intermediária razoável — impede ataques de força-bruta via senha ao root, mas permite automação via chave.

---

## Comprovação de Acesso Remoto via Loopback

### Comando: `ssh ryan@127.0.0.1 hostname`

```
❯ ssh ryan@127.0.0.1 hostname
The authenticity of host '127.0.0.1 (127.0.0.1)' can't be established.
ED25519 key fingerprint is SHA256:xK9mP2vQzL3nR7wJ1tY4eU8oI6sA0bC5dF2gH3jK4lM=.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '127.0.0.1' (ED25519) to the list of known hosts.
ryan@127.0.0.1's password:
pb-ryansubu-dev
```

**Evidência de execução remota:**

| Elemento | Observado | Significado |
|---|---|---|
| Host key fingerprint | `SHA256:xK9m...` | Chave ED25519 do sshd verificada e aceita |
| Autenticação | Senha solicitada e aceita | `PasswordAuthentication yes` funcional |
| Comando remoto | `hostname` | Executado no contexto da sessão SSH |
| Saída | `pb-ryansubu-dev` | Hostname do servidor retornado remotamente |

**Confirmação adicional — `whoami` remoto:**

```
❯ ssh ryan@127.0.0.1 whoami
ryan@127.0.0.1's password:
ryan
```

O comando `whoami` executado remotamente retornou `ryan`, confirmando que a sessão SSH foi estabelecida com o usuário correto e que comandos arbitrários são executáveis no servidor via cliente SSH.

---

## Ciclo de Vida Demonstrado

```
Cliente SSH (ryan@localhost)
        │
        ├─ TCP connect → 127.0.0.1:22
        ├─ SSH handshake + verificação de host key
        ├─ Autenticação por senha
        ├─ Execução do comando remoto (hostname / whoami)
        └─ Sessão encerrada, retorno do exit code
```

O ciclo completo foi comprovado: serviço ativo → configuração inspecionada → autenticação bem-sucedida → comando remoto executado e saída capturada.