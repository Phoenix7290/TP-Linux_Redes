# Exercício 11 — Socket operacional: bind/listen/accept/send/recv observados com ferramentas

---

## Tarefa 1 — Experimento com nc (listener + cliente)

### Terminal 1 — Servidor (listener):

```bash
❯ nc -lvp 55000
Listening on 0.0.0.0 55000
```

### Terminal 2 — Cliente (após servidor estar em escuta):

```bash
❯ nc localhost 55000
ciclo de socket: bind listen accept send recv
```

### Saída no servidor após conexão e envio:

```
Listening on 0.0.0.0 55000
Connection received on localhost 49103
ciclo de socket: bind listen accept send recv
```

---

## Tarefa 2 — Evidência de LISTEN (netstat) e processo dono (lsof)

### Estado LISTEN — netstat (antes da conexão do cliente):

```bash
❯ netstat -tnlp | grep 55000
tcp   0   0   0.0.0.0:55000   0.0.0.0:*   LISTEN   8214/nc
```

**Evidência de LISTEN:** Socket TCP em `0.0.0.0:55000`, estado `LISTEN`, pertencente ao PID 8214 (`nc`). O `0.0.0.0` indica que o bind foi feito em todas as interfaces IPv4.

### Processo dono — lsof:

```bash
❯ sudo lsof -i TCP:55000
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      8214 ryan    3u  IPv4  64017      0t0  TCP  *:55000 (LISTEN)
```

**Evidência do processo dono:** `lsof` confirma que o file descriptor 3 do processo `nc` (PID 8214, usuário `ryan`) é o socket TCP em `LISTEN` na porta 55000. O número de file descriptor (3) é esperado: 0=stdin, 1=stdout, 2=stderr, 3=primeiro socket aberto pelo processo.

---

## Tarefa 3 — Evidência do estado ESTABLISHED

### netstat após conexão do cliente:

```bash
❯ netstat -tnp | grep 55000
tcp   0   0   127.0.0.1:55000   127.0.0.1:49103   ESTABLISHED   8214/nc
tcp   0   0   127.0.0.1:49103   127.0.0.1:55000   ESTABLISHED   8231/nc
```

### lsof durante a sessão ativa:

```bash
❯ sudo lsof -i TCP:55000
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      8214 ryan    3u  IPv4  64017      0t0  TCP  *:55000 (LISTEN)
nc      8214 ryan    4u  IPv4  66193      0t0  TCP  localhost:55000->localhost:49103 (ESTABLISHED)
nc      8231 ryan    3u  IPv4  66194      0t0  TCP  localhost:49103->localhost:55000 (ESTABLISHED)
```

**Evidência de ESTABLISHED:** Dois sockets em `ESTABLISHED` aparecem simultaneamente — um para o lado servidor (`55000→49103`, PID 8214) e um para o lado cliente (`49103→55000`, PID 8231). O socket `LISTEN` original (fd=3, PID 8214) permanece aberto, pronto para aceitar novas conexões enquanto a sessão atual está ativa.

---

## Tarefa 4 — Mapeamento objetivo do ciclo de vida do socket

**bind:** O `nc -lvp 55000` chamou `bind()` internamente, associando o socket ao endereço `0.0.0.0:55000` — evidenciado pelo `netstat` mostrando `0.0.0.0:55000` como endereço local antes de qualquer conexão.

**listen:** Após o `bind()`, o `nc` chamou `listen()`, colocando o socket na fila de conexões pendentes — evidenciado pelo estado `LISTEN` no `netstat` e no `lsof` (`TCP *:55000 (LISTEN)`, fd=3, PID 8214).

**accept:** Quando o cliente se conectou, o kernel completou o three-way handshake e o `nc` chamou `accept()`, criando um novo socket dedicado à sessão — evidenciado pelo aparecimento do socket `ESTABLISHED` com fd=4 (PID 8214) ao lado do socket `LISTEN` original (fd=3), no `lsof`.

**send/recv:** A string `ciclo de socket: bind listen accept send recv` digitada no cliente foi enviada via `send()` e recebida pelo servidor via `recv()` — evidenciada pelo `netstat` mostrando `Recv-Q` e `Send-Q` zerados após a troca (dados entregues), e pela impressão da string no terminal do servidor.