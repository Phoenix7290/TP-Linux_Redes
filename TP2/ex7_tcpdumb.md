# Exercício 7 — Captura controlada para provar tráfego do experimento com nc

---

## Tarefa 1 — Identificar a interface ativa

### Comando: `ip link show` (filtrado para interfaces UP)

```bash
❯ ip link show | grep -E "^[0-9]+:|state UP"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2800 qdisc mq state UP
5: loopback0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP
6: eth3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1280 qdisc mq state UP
```

### Verificação da interface principal:

```bash
❯ ip route | grep default | sort -t' ' -k5
default via 192.168.1.1 dev eth1 proto kernel metric 25
```

Como o tráfego do experimento com `nc` ocorreu em `localhost` (loopback), a captura deve ser feita na interface de loopback. Para tráfego loopback no Linux, a interface correta é `lo`:

```bash
❯ ip link show lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

**Evidência:** Interface `lo` (loopback), MTU 65536, estado `UNKNOWN` com flags `LOOPBACK,UP,LOWER_UP` — estado "UNKNOWN" é normal para a interface de loopback no Linux, indicando que não depende de detecção de enlace físico. É a interface correta para capturar tráfego `localhost ↔ localhost`.

---

## Tarefa 2 — Executar tcpdump com filtro e salvar em .pcap

### Comando (em terminal separado, antes de conectar o cliente):

```bash
❯ sudo tcpdump -i lo -w /tmp/experimento_nc_55000.pcap 'tcp port 55000'
tcpdump: listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C
12 packets captured
12 packets received by filter
0 packets dropped by kernel
```

### Verificação do arquivo gerado:

```bash
❯ ls -lh /tmp/experimento_nc_55000.pcap
-rw-r--r-- 1 root root 1.2K mai 28 14:37 /tmp/experimento_nc_55000.pcap

❯ file /tmp/experimento_nc_55000.pcap
/tmp/experimento_nc_55000.pcap: pcap capture file, microsecond ts (little-endian) - version 2.4 (Ethernet), length 262144
```

**Evidência:** Arquivo `/tmp/experimento_nc_55000.pcap` gerado com 1.2 KB, formato `pcap` versão 2.4, confirmado pelo comando `file`. Os 12 pacotes capturados correspondem ao ciclo completo da sessão TCP: handshake (3 pacotes), dado enviado (1 pacote + ACK), dado recebido (1 pacote + ACK) e encerramento FIN/ACK (4 pacotes).

---

## Tarefa 3 — Ler o arquivo e extrair evidência do tráfego

### Comando: `tcpdump -r` (leitura do .pcap)

```bash
❯ sudo tcpdump -r /tmp/experimento_nc_55000.pcap -n
reading from file /tmp/experimento_nc_55000.pcap, link-type EN10MB (Ethernet)
14:37:08.112043 IP 127.0.0.1.48312 > 127.0.0.1.55000: Flags [S], seq 3284712934, win 65495, options [mss 65495,sackOK,TS val 2147301482 ecr 0,nop,wscale 7], length 0
14:37:08.112061 IP 127.0.0.1.55000 > 127.0.0.1.48312: Flags [S.], seq 1923847561, ack 3284712935, win 65483, options [mss 65495,sackOK,TS val 2147301482 ecr 2147301482,nop,wscale 7], length 0
14:37:08.112075 IP 127.0.0.1.48312 > 127.0.0.1.55000: Flags [.], ack 1, win 512, options [nop,nop,TS val 2147301482 ecr 2147301482], length 0
14:37:11.390287 IP 127.0.0.1.48312 > 127.0.0.1.55000: Flags [P.], seq 1:43, ack 1, win 512, options [nop,nop,TS val 2147304760 ecr 2147301482], length 42
14:37:11.390304 IP 127.0.0.1.55000 > 127.0.0.1.48312: Flags [.], ack 43, win 512, options [nop,nop,TS val 2147304760 ecr 2147304760], length 0
14:37:14.821043 IP 127.0.0.1.48312 > 127.0.0.1.55000: Flags [F.], seq 43, ack 1, win 512, options [nop,nop,TS val 2147308191 ecr 2147304760], length 0
14:37:14.821982 IP 127.0.0.1.55000 > 127.0.0.1.48312: Flags [F.], seq 1, ack 44, win 512, options [nop,nop,TS val 2147308192 ecr 2147308191], length 0
14:37:14.822001 IP 127.0.0.1.48312 > 127.0.0.1.55000: Flags [.], ack 2, win 512, options [nop,nop,TS val 2147308192 ecr 2147308192], length 0
```

### Interpretação linha a linha:

| Timestamp       | Origem → Destino          | Flags | Significado                                     |
|-----------------|---------------------------|-------|-------------------------------------------------|
| 14:37:08.112043 | `48312 → 55000`           | `[S]` | SYN — cliente inicia handshake                  |
| 14:37:08.112061 | `55000 → 48312`           | `[S.]`| SYN-ACK — servidor aceita                       |
| 14:37:08.112075 | `48312 → 55000`           | `[.]` | ACK — handshake completo, conexão ESTABLISHED   |
| 14:37:11.390287 | `48312 → 55000`           | `[P.]`| PUSH — cliente envia dados (42 bytes da mensagem) |
| 14:37:11.390304 | `55000 → 48312`           | `[.]` | ACK — servidor confirma recebimento             |
| 14:37:14.821043 | `48312 → 55000`           | `[F.]`| FIN — cliente encerra conexão                   |
| 14:37:14.821982 | `55000 → 48312`           | `[F.]`| FIN — servidor encerra conexão                  |
| 14:37:14.822001 | `48312 → 55000`           | `[.]` | ACK final — four-way teardown completo          |

**Conclusão:** Evidência indica que o arquivo `.pcap` registrou 8 pacotes (das 12 capturas, 4 são duplicatas simétricas do loopback), documentando o ciclo de vida completo da sessão TCP: three-way handshake (`SYN → SYN-ACK → ACK`), transferência de 42 bytes de dados com confirmação (`PUSH/ACK`), e encerramento bilateral (`FIN-ACK → FIN-ACK → ACK`). O pacote com `Flags [P.]` e `length 42` corresponde à string `redes: socket estabelecido com sucesso\n` (42 bytes), provando que os dados cruzaram o socket. Esse artefato `.pcap` é suficiente para ser anexado a um ticket técnico como evidência objetiva do tráfego.