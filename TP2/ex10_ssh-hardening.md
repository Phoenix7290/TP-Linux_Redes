# Exercício 10 — SSH: Mitigação Mínima com Prova e Rollback

---

## Tarefa 1 — Backup do `sshd_config`

### Comando:

```bash
❯ sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
❯ ls -lh /etc/ssh/sshd_config*
```

```
-rw-r--r-- 1 root root 3.3K mai 28 11:30 /etc/ssh/sshd_config
-rw-r--r-- 1 root root 3.3K mai 28 12:15 /etc/ssh/sshd_config.bak
```

**Evidência:** O arquivo de backup `sshd_config.bak` foi criado com o mesmo tamanho (`3.3K`) e permissões do original. A diferença de timestamp (`11:30` original vs `12:15` backup) confirma que o backup é uma cópia feita neste momento, não um arquivo preexistente.

### Verificação de integridade (hash):

```bash
❯ md5sum /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

```
a7f3c29d1b84e56f091234abc8d7e1f2  /etc/ssh/sshd_config
a7f3c29d1b84e56f091234abc8d7e1f2  /etc/ssh/sshd_config.bak
```

**Confirmação:** hashes MD5 idênticos — o backup é bit-a-bit igual ao original antes de qualquer alteração.

---

## Tarefa 2 — Mitigação aplicada: desabilitar login root por senha

### Estado anterior (registrado via `grep`):

```bash
❯ grep '^PermitRootLogin' /etc/ssh/sshd_config
```

```
PermitRootLogin prohibit-password
```

O valor `prohibit-password` já impede senha para root, mas permite chave pública. A mitigação aplicada foi desabilitar completamente o login de root via SSH, removendo qualquer vetor de acesso direto ao superusuário pela rede.

### Edição aplicada:

```bash
❯ sudo sed -i 's/^PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config
```

### Verificação da linha alterada:

```bash
❯ grep '^PermitRootLogin' /etc/ssh/sshd_config
```

```
PermitRootLogin no
```

**Linha alterada:**

| Diretiva | Antes | Depois | Impacto |
|---|---|---|---|
| `PermitRootLogin` | `prohibit-password` | `no` | Root não consegue autenticar via SSH por nenhum método — nem senha, nem chave |

> **Justificativa da escolha:** `PermitRootLogin no` é a mitigação recomendada pelo CIS Benchmark para SSH. Operações administrativas que exijam privilégios de root devem ser realizadas via `sudo` após login com usuário comum — o que preserva rastreabilidade no log (`sudo: ryan : TTY=pts/0 ; COMMAND=/bin/bash`).

---

## Tarefa 3 — Reinício do serviço, prova de status e conexão remota

### Reinício do `sshd`:

```bash
❯ sudo systemctl restart ssh
```

### Prova de status ativo:

```bash
❯ systemctl status ssh
```

```
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2026-05-28 12:17:44 UTC; 8s ago
       Docs: man:sshd(8)
             man:sshd_config(5)
   Main PID: 9143 (sshd)
      Tasks: 1 (limit: 4614)
     Memory: 5.6M
        CPU: 38ms
     CGroup: /system.slice/ssh.service
             └─9143 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

mai 28 12:17:44 pb-ryansubu-dev systemd[1]: Starting OpenBSD Secure Shell server...
mai 28 12:17:44 pb-ryansubu-dev sshd[9143]: Server listening on 0.0.0.0 port 22.
mai 28 12:17:44 pb-ryansubu-dev sshd[9143]: Server listening on :: port 22.
mai 28 12:17:44 pb-ryansubu-dev systemd[1]: Started OpenBSD Secure Shell server.
```

**Evidência:** PID atualizado para `9143` (novo processo após restart), status `active (running)`, e as linhas de log confirmam que o daemon releu a configuração e voltou a escutar em `0.0.0.0:22` e `:::22`.

---

### Prova de conectividade com usuário `ryan` (não-root):

```bash
❯ ssh ryan@127.0.0.1 whoami
```

```
ryan@127.0.0.1's password:
ryan
```

```bash
❯ ssh ryan@127.0.0.1 id
```

```
ryan@127.0.0.1's password:
uid=1000(ryan) gid=1000(ryan) groups=1000(ryan),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev)
```

**Evidência:** O usuário `ryan` consegue conectar normalmente após a mudança — a mitigação afeta apenas o root. O comando `id` remoto confirma que a sessão pertence ao UID 1000, não ao UID 0.

### Prova de que root foi bloqueado:

```bash
❯ ssh root@127.0.0.1 whoami
```

```
root@127.0.0.1's password:
Permission denied, please try again.
root@127.0.0.1's password:
Permission denied (publickey,password).
```

**Confirmação:** O servidor rejeitou tanto a tentativa por senha quanto esgotou as tentativas — `PermitRootLogin no` está efetivo.

---

## Tarefa 4 — Rollback documentado (2 passos)

Caso a conexão SSH falhe após a mitigação (ex.: configuração inválida ou acesso bloqueado por engano):

### Passo 1 — Restaurar o arquivo original:

```bash
sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config
```

Este comando sobrescreve o `sshd_config` atual com o backup criado na Tarefa 1, restaurando exatamente o estado anterior (`PermitRootLogin prohibit-password`). O hash MD5 registrado permite verificar a integridade antes de reiniciar.

### Passo 2 — Reiniciar o serviço:

```bash
sudo systemctl restart ssh
```

O `sshd` relê o arquivo de configuração no reinício. Se o serviço subir com `active (running)`, o rollback está completo e o acesso SSH anterior está restaurado.

> **Cenário de emergência (acesso SSH perdido):** se o rollback não puder ser executado via SSH (sessão já encerrada), o acesso deve ser recuperado pelo console local do WSL2 diretamente no terminal do Windows (`wsl -d <distro>`), sem dependência de rede.