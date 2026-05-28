# Exercício 8 — Varredura controlada no host e validação cruzada com evidências locais

---

## Tarefa 1 — Executar nmap no próprio host

### Comando: `nmap -sT localhost`

```bash
❯ nmap -sT localhost
Starting Nmap 7.94 ( https://nmap.org ) at 2025-05-28 14:41 -03
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000084s latency).
Not shown: 988 closed tcp ports (conn-refused)
PORT      STATE SERVICE
3000/tcp  open  ppp
3002/tcp  open  exlm-agent
3306/tcp  open  mysql
4000/tcp  open  remoteanything
5432/tcp  open  postgresql
6379/tcp  open  redis
11434/tcp open  unknown
27017/tcp open  mongod
33060/tcp open  mysqlx
37177/tcp open  unknown
55000/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 0.15s
```

**Trecho de evidência (saída filtrada):**

```
PORT      STATE SERVICE
3306/tcp  open  mysql
5432/tcp  open  postgresql
6379/tcp  open  redis
27017/tcp open  mongod
55000/tcp open  unknown
```

**Conclusão:** Evidência indica que o `nmap` identificou 11 portas TCP abertas no host. A varredura `-sT` utiliza o connect scan — tenta completar o three-way handshake em cada porta — e reporta `open` apenas onde recebe `SYN-ACK`. Isso representa a visão externa do host: o que um cliente legítimo (ou adversário) enxergaria ao sondar o sistema. O latency `0.000084s` confirma que a varredura ocorreu em loopback, sem tráfego de rede real.

---

## Tarefa 2 — Identificar a porta do listener nc na saída do nmap

### Evidência direta — porta 55000 aparece como aberta:

```bash
❯ nmap -sT -p 55000 localhost
Starting Nmap 7.94 ( https://nmap.org ) at 2025-05-28 14:41 -03
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000061s latency).

PORT      STATE SERVICE
55000/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 0.04s
```

**Conclusão:** Evidência indica que a porta 55000, iniciada com `nc -lvp 55000` no exercício anterior, foi detectada pelo `nmap` como `open`. O campo `SERVICE` retorna `unknown` porque o `nc` não implementa nenhum protocolo de aplicação reconhecível — ao completar o handshake, nenhum banner é enviado, então o `nmap` não consegue identificar o serviço pela resposta. Isso é comportamento esperado de um listener TCP mínimo.

---

## Tarefa 3 — Validação cruzada: provar o processo com lsof e netstat

### Cruzamento com lsof:

```bash
❯ sudo lsof -i TCP:55000
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      7831 ryan    3u  IPv4  61204      0t0  TCP  *:55000 (LISTEN)
```

| Campo    | Valor            |
|----------|------------------|
| PID      | 7831             |
| Processo | `nc`             |
| Usuário  | `ryan`           |
| Bind     | `*:55000`        |
| Estado   | `LISTEN`         |

### Cruzamento com netstat:

```bash
❯ netstat -tnlp | grep 55000
tcp   0   0   0.0.0.0:55000   0.0.0.0:*   LISTEN   7831/nc
```

### Cruzamento com ss:

```bash
❯ ss -tnlp | grep 55000
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
LISTEN  0       1       0.0.0.0:55000       0.0.0.0:*          users:(("nc",pid=7831,fd=3))
```

**Conclusão:** Evidência indica consistência total entre as três ferramentas de diagnóstico:

| Ferramenta | O que prova                                                        |
|------------|--------------------------------------------------------------------|
| `nmap`     | Visão externa: porta 55000 está `open` e aceita conexões          |
| `lsof`     | Visão interna: PID 7831 (`nc`, usuário `ryan`) detém o socket     |
| `netstat`  | Visão interna: socket em `LISTEN` em `0.0.0.0:55000`, PID 7831   |
| `ss`       | Visão interna: confirma PID 7831 com file descriptor 3            |

Essa consistência é a evidência de domínio operacional exigida pelo exercício: o `nmap` vê a porta aberta de fora, e as ferramentas locais provam qual processo a mantém, com qual usuário e em qual file descriptor. Qualquer divergência entre essas camadas — por exemplo, `nmap` reporta porta aberta mas `lsof` não encontra processo — seria evidência de filtro de pacotes, socket em estado `TIME_WAIT`, ou até processo com capabilities elevadas ocultando o socket.