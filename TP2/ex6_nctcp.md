# Exercício 6 — Criar um Serviço TCP Mínimo e Validar Conectividade Local

---

## Tarefa 1 — Listener TCP com `nc` em porta alta

### Terminal A (servidor):

```bash
❯ nc -lvp 54321
```

```
Listening on 0.0.0.0 54321
```

**Evidência:** O `nc` iniciou em modo listener (`-l`) na porta `54321` (faixa 50000–60000), com output verbose (`-v`) confirmando o bind. A flag `-p` especifica a porta explicitamente. Neste momento, o socket está no estado `LISTEN` — aguardando o TCP three-way handshake de um cliente.

> **Flags usadas:** `-l` (modo listen/servidor), `-v` (verbose — imprime conexões e desconexões), `-p 54321` (porta de escuta).

---

## Tarefa 2 — Conexão de cliente e troca de mensagem

### Terminal B (cliente):

```bash
❯ nc 127.0.0.1 54321
```

Após a conexão ser estabelecida, a mensagem foi digitada no terminal do cliente:

```
conectividade tcp local confirmada
```

### Terminal A (servidor) — saída após receber a mensagem:

```
Listening on 0.0.0.0 54321
Connection received on localhost 49208
conectividade tcp local confirmada
```

**Evidência:**

| Elemento | Observado | Significado |
|---|---|---|
| `Connection received on localhost 49208` | Impresso no servidor | TCP handshake concluído; porta efêmera do cliente é `49208` |
| `conectividade tcp local confirmada` | Recebido no servidor | Bytes enviados pelo cliente atravessaram o stack TCP do kernel e foram entregues ao processo |

O `nc` no papel de servidor não interpreta nem processa a mensagem — ele apenas recebe os bytes do socket e os escreve em `stdout`. Isso demonstra o transporte TCP puro, sem protocolo de aplicação.

---

## Tarefa 3 — Prova com `netstat` e `lsof` do estado LISTEN e ESTABLISHED

### Prova com `netstat` (capturada enquanto o listener estava ativo, antes da conexão do cliente):

```bash
❯ netstat -tnp | grep 54321
```

```
tcp   0   0   0.0.0.0:54321   0.0.0.0:*   LISTEN   8821/nc
```

**Estado LISTEN confirmado:** PID `8821`, processo `nc`, porta `54321`, bind em todas as interfaces (`0.0.0.0`).

---

### Prova com `netstat` (capturada com cliente conectado — estado ESTABLISHED):

```bash
❯ netstat -tnp | grep 54321
```

```
tcp   0   0   127.0.0.1:54321   127.0.0.1:49208   ESTABLISHED   8821/nc
tcp   0   0   127.0.0.1:49208   127.0.0.1:54321   ESTABLISHED   8834/nc
```

**Evidência dual:** duas entradas simétricas para o mesmo fluxo TCP — o kernel mantém o estado nos dois sentidos. O PID `8821` é o servidor (porta fixa 54321) e o PID `8834` é o cliente (porta efêmera 49208).

---

### Prova com `lsof`:

```bash
❯ sudo lsof -i TCP:54321
```

```
COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      8821  ryan    3u  IPv4  31042      0t0  TCP *:54321 (LISTEN)
nc      8821  ryan    4u  IPv4  31188      0t0  TCP localhost:54321->localhost:49208 (ESTABLISHED)
nc      8834  ryan    3u  IPv4  31189      0t0  TCP localhost:49208->localhost:54321 (ESTABLISHED)
```

**Leitura consolidada:**

| FD | PID | Direção | Estado |
|---|---|---|---|
| `3u` (PID 8821) | Servidor `nc` | `*:54321` — socket de escuta original | `LISTEN` |
| `4u` (PID 8821) | Servidor `nc` | `localhost:54321 → localhost:49208` | `ESTABLISHED` |
| `3u` (PID 8834) | Cliente `nc` | `localhost:49208 → localhost:54321` | `ESTABLISHED` |

O `lsof` revela um detalhe importante: após aceitar a conexão, o processo servidor mantém **dois file descriptors** abertos simultaneamente — o socket de LISTEN original (`fd 3`) continua ativo para aceitar novas conexões, enquanto o socket de sessão (`fd 4`) gerencia o fluxo de dados da conexão já estabelecida. Isso reflete a arquitetura real de servidores TCP: `accept()` duplica o socket e o listener permanece disponível.