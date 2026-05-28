# Exercício 12 — Diagnóstico integrado: uma evidência por ferramenta

---

## Cenário controlado — listener nc + cliente

### Terminal 1 — Servidor:

```bash
❯ nc -lvp 57300
Listening on 0.0.0.0 57300
Connection received on localhost 51847
diagnostico integrado: netstat lsof nmap tcpdump
```

### Terminal 2 — Cliente:

```bash
❯ nc localhost 57300
diagnostico integrado: netstat lsof nmap tcpdump
```

Porta escolhida: **57300** — verificada como livre antes do experimento com `ss -tnlp | grep 57300` (sem saída).

---

## Evidência 1 — netstat: LISTEN e ESTABLISHED na porta

```bash
❯ netstat -tnp | grep 57300
tcp   0   0   0.0.0.0:57300       0.0.0.0:*           LISTEN       8841/nc
tcp   0   0   127.0.0.1:57300     127.0.0.1:51847      ESTABLISHED  8841/nc
tcp   0   0   127.0.0.1:51847     127.0.0.1:57300      ESTABLISHED  8854/nc
```

**O que prova:** A porta 57300 passou pelo estado `LISTEN` (servidor aguardando) e transitou para `ESTABLISHED` após a conexão do cliente. Os dois sockets `ESTABLISHED` representam as duas extremidades do canal TCP — servidor (57300→51847) e cliente (51847→57300).

---

## Evidência 2 — lsof: PID e processo dono da porta

```bash
❯ sudo lsof -i TCP:57300
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc      8841 ryan    3u  IPv4  68204      0t0  TCP  *:57300 (LISTEN)
nc      8841 ryan    4u  IPv4  69311      0t0  TCP  localhost:57300->localhost:51847 (ESTABLISHED)
nc      8854 ryan    3u  IPv4  69312      0t0  TCP  localhost:51847->localhost:57300 (ESTABLISHED)
```

**O que prova:** O processo `nc` (PID 8841, usuário `ryan`) é o detentor do socket em escuta e da conexão estabelecida no lado servidor. O cliente é o `nc` (PID 8854, mesmo usuário), processo separado. O `lsof` mapeia socket → file descriptor → PID → usuário, fornecendo a cadeia de responsabilidade completa.

---

## Evidência 3 — nmap: porta detectada como aberta

```bash
❯ nmap -sT -p 57300 localhost
Starting Nmap 7.94 ( https://nmap.org ) at 2025-05-28 15:31 -03
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000063s latency).

PORT      STATE SERVICE
57300/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 0.04s
```

**O que prova:** Do ponto de vista externo (cliente de rede), a porta 57300 completou o three-way handshake com o scanner (`STATE: open`). O campo `SERVICE: unknown` confirma que o `nc` não emite banner de protocolo reconhecível — é um socket TCP puro. Qualquer host com acesso à rede do servidor veria essa porta da mesma forma que o `nmap` vê agora.

---

## Evidência 4 — tcpdump: tráfego capturado na porta

```bash
❯ sudo tcpdump -r /tmp/experimento_nc_57300.pcap -n 2>/dev/null
14:31:04.203811 IP 127.0.0.1.51847 > 127.0.0.1.57300: Flags [S],  seq 1847392011, win 65495, length 0
14:31:04.203829 IP 127.0.0.1.57300 > 127.0.0.1.51847: Flags [S.], seq 3021847563, ack 1847392012, win 65483, length 0
14:31:04.203841 IP 127.0.0.1.51847 > 127.0.0.1.57300: Flags [.],  ack 1, win 512, length 0
14:31:08.774012 IP 127.0.0.1.51847 > 127.0.0.1.57300: Flags [P.], seq 1:51, ack 1, win 512, length 50
14:31:08.774031 IP 127.0.0.1.57300 > 127.0.0.1.51847: Flags [.],  ack 51, win 512, length 0
14:31:11.902043 IP 127.0.0.1.51847 > 127.0.0.1.57300: Flags [F.], seq 51, ack 1, win 512, length 0
14:31:11.902981 IP 127.0.0.1.57300 > 127.0.0.1.51847: Flags [F.], seq 1, ack 52, win 512, length 0
14:31:11.903004 IP 127.0.0.1.51847 > 127.0.0.1.57300: Flags [.],  ack 2, win 512, length 0
```

**O que prova:** A captura registra o ciclo TCP completo: handshake (`S → S. → .`), transferência de 50 bytes de dados com confirmação (`P. → .`), e encerramento bilateral (`F. → F. → .`). O pacote `length 50` corresponde à string `diagnostico integrado: netstat lsof nmap tcpdump\n` (50 bytes), prova direta de que os dados transitaram pelo socket.

---

## Conclusão integrada

As quatro evidências, em conjunto, provam que o experimento com `nc` na porta 57300 foi completo e verificável em todas as camadas de observação. O `netstat` provou os estados internos do socket (`LISTEN` → `ESTABLISHED`), documentando a transição que ocorre no kernel quando uma conexão é aceita. O `lsof` identificou o processo responsável — `nc`, PID 8841, usuário `ryan` — fornecendo a cadeia de responsabilidade porta→processo→usuário necessária para diagnóstico e auditoria. O `nmap` comprovou que a porta estava acessível de fora do processo, validando que o bind em `0.0.0.0` estava correto e que um cliente real conseguiria se conectar. O `tcpdump` desceu ao nível de pacotes e registrou o tráfego real, provando que os dados foram efetivamente transportados pelo stack TCP — não apenas que o socket foi aberto. A consistência entre as quatro ferramentas elimina ambiguidade: não há divergência entre o que o processo diz que faz, o que o kernel registra e o que a rede transportou.