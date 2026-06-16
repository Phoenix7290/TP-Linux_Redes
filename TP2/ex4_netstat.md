# Exercício 4 — Netstat como Inventário: Conexões, LISTEN e Dual-Stack (IPv4/IPv6)

---

## Tarefa 1 — Portas TCP em LISTEN

### Comando: `netstat -tlnp`

```
❯ netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:53            0.0.0.0:*               LISTEN      627/systemd-resolve
tcp        0      0 127.0.0.54:53           0.0.0.0:*               LISTEN      627/systemd-resolve
tcp        0      0 127.0.0.1:27017         0.0.0.0:*               LISTEN      843/mongod
tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN      901/redis-server
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      1102/mysqld
tcp        0      0 127.0.0.1:33060         0.0.0.0:*               LISTEN      1102/mysqld
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      1287/postgres
tcp        0      0 0.0.0.0:11434           0.0.0.0:*               LISTEN      1451/ollama
tcp        0      0 127.0.0.1:37177         0.0.0.0:*               LISTEN      5073/node
tcp        0      0 0.0.0.0:3000            0.0.0.0:*               LISTEN      5201/node
tcp        0      0 0.0.0.0:3002            0.0.0.0:*               LISTEN      5204/node
tcp        0      0 0.0.0.0:4000            0.0.0.0:*               LISTEN      5389/ruby
tcp6       0      0 :::6379                 :::*                    LISTEN      901/redis-server
```

**Evidência:** 13 entradas TCP em `LISTEN`. Serviços de banco de dados (`mongod`, `mysqld`, `postgres`, `redis`) estão vinculados exclusivamente ao loopback (`127.0.0.1`), inacessíveis externamente. `ollama` (porta 11434), `node` (3000 e 3002) e `ruby` (4000) estão vinculados a `0.0.0.0` — expostos em todas as interfaces de rede do host. `redis-server` aparece adicionalmente como `tcp6` em `:::6379`, indicando escuta dual-stack.

> **Flags usadas:** `-t` (TCP apenas), `-l` (somente LISTEN), `-n` (endereços numéricos, sem resolução DNS reversa), `-p` (mostra PID e nome do processo — requer `sudo` para processos de outros usuários).

---

## Tarefa 2 — Conexões ativas

### Comando: `netstat -tn`

```
❯ netstat -tn
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 172.26.48.12:52874      140.82.112.26:443       ESTABLISHED
tcp        0      0 172.26.48.12:58210      140.82.112.25:443       ESTABLISHED
tcp        0      0 172.26.48.12:41336      151.101.1.194:443       ESTABLISHED
tcp        0      0 172.26.48.12:49902      20.205.243.166:443      ESTABLISHED
tcp        0      0 127.0.0.1:37177         127.0.0.1:52301         ESTABLISHED
tcp        0      0 127.0.0.1:52301         127.0.0.1:37177         ESTABLISHED
tcp        0      0 127.0.0.1:3000          127.0.0.1:47812         ESTABLISHED
tcp        0      0 127.0.0.1:27017         127.0.0.1:48204         ESTABLISHED
```

**Evidência (trechos selecionados):**

| Local | Remoto | Estado | Interpretação |
|---|---|---|---|
| `172.26.48.12:52874` | `140.82.112.26:443` | `ESTABLISHED` | Conexão HTTPS ativa — GitHub (140.82.112.0/24) |
| `172.26.48.12:41336` | `151.101.1.194:443` | `ESTABLISHED` | Conexão HTTPS ativa — Fastly CDN |
| `127.0.0.1:37177` | `127.0.0.1:52301` | `ESTABLISHED` | Comunicação IPC loopback entre processos Node.js |
| `127.0.0.1:27017` | `127.0.0.1:48204` | `ESTABLISHED` | Cliente conectado ao MongoDB localmente |

O IP `172.26.48.12` é a interface `eth1` do WSL2 (LAN interna virtualizada pelo Hyper-V). Conexões saindo por essa interface atingem a internet via NAT do Windows.

---

## Tarefa 3 — Visão separada IPv4 e IPv6; SSH exposto em qual stack?

### IPv4 apenas — `netstat -tlnp -4`

```
❯ netstat -tlnp -4
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:53            0.0.0.0:*               LISTEN      627/systemd-resolve
tcp        0      0 127.0.0.54:53           0.0.0.0:*               LISTEN      627/systemd-resolve
tcp        0      0 127.0.0.1:27017         0.0.0.0:*               LISTEN      843/mongod
tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN      901/redis-server
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      1102/mysqld
tcp        0      0 127.0.0.1:33060         0.0.0.0:*               LISTEN      1102/mysqld
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      1287/postgres
tcp        0      0 0.0.0.0:11434           0.0.0.0:*               LISTEN      1451/ollama
tcp        0      0 127.0.0.1:37177         0.0.0.0:*               LISTEN      5073/node
tcp        0      0 0.0.0.0:3000            0.0.0.0:*               LISTEN      5201/node
tcp        0      0 0.0.0.0:3002            0.0.0.0:*               LISTEN      5204/node
tcp        0      0 0.0.0.0:4000            0.0.0.0:*               LISTEN      5389/ruby
```

### IPv6 apenas — `netstat -tlnp -6`

```
❯ netstat -tlnp -6
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp6       0      0 :::6379                 :::*                    LISTEN      901/redis-server
```

**Conclusão objetiva — SSH nos dois stacks:**

A porta 22 (SSH) **não aparece em nenhuma das duas visões** — nem IPv4 nem IPv6. O ambiente WSL2 não executa `sshd` por padrão; o acesso ao subsistema Linux é feito diretamente pelo terminal do Windows sem necessidade de um daemon SSH interno. O único serviço com presença dual-stack confirmada neste host é o `redis-server` (porta 6379): IPv4 via `127.0.0.1:6379` e IPv6 via `:::6379`. Todos os demais serviços escutam exclusivamente em IPv4.

| Stack | Porta 22 (SSH) presente? | Serviço dual-stack identificado |
|---|---|---|
| IPv4 | Não | `redis-server` (6379) |
| IPv6 | Não | `redis-server` (6379) |