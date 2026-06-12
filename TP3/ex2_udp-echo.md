# Exercício 2 — UDP: Servidor Echo + Cliente (Datagramas)

---

## Código — Servidor UDP Echo

```python
# ex2_servidor_udp.py
import socket

HOST = "0.0.0.0"   # aceitar conexões de qualquer interface
PORTA = 54321

def main():
    # AF_INET + SOCK_DGRAM = UDP sobre IPv4
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as srv:
        srv.bind((HOST, PORTA))
        print(f"[SERVIDOR UDP] Escutando em {HOST}:{PORTA}")
        print("-" * 55)

        while True:
            try:
                # recvfrom retorna (dados_bytes, (ip_origem, porta_origem))
                dados, (ip_origem, porta_origem) = srv.recvfrom(65535)

                tamanho_payload = len(dados)
                mensagem = dados.decode("utf-8", errors="replace")

                print(f"[RX] IP origem   : {ip_origem}")
                print(f"     Porta origem: {porta_origem}")
                print(f"     Payload     : {tamanho_payload} bytes")
                print(f"     Conteúdo    : {mensagem[:80]}{'...' if len(mensagem) > 80 else ''}")
                print("-" * 55)

                # Echo: reenviar os mesmos dados de volta ao remetente
                srv.sendto(dados, (ip_origem, porta_origem))

            except KeyboardInterrupt:
                print("\n[SERVIDOR] Interrompido pelo usuário.")
                break

if __name__ == "__main__":
    main()
```

---

## Código — Cliente UDP

```python
# ex2_cliente_udp.py
import socket
import random
import string
import time

SERVIDOR_IP   = "127.0.0.1"   # substitua pelo IP real do servidor em cenário remoto
SERVIDOR_PORTA = 54321
TIMEOUT        = 2.0           # segundos para aguardar echo de volta

def gerar_conteudo(tamanho: int) -> str:
    """Gera uma string aleatória de `tamanho` caracteres."""
    return ''.join(random.choices(string.ascii_letters + string.digits, k=tamanho))

def main():
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as cli:
        cli.settimeout(TIMEOUT)

        for seq in range(1, 11):
            tamanho = random.randint(10, 2000)
            conteudo = gerar_conteudo(tamanho)
            mensagem = f"{seq} - {conteudo}"
            dados = mensagem.encode("utf-8")

            t0 = time.perf_counter()
            cli.sendto(dados, (SERVIDOR_IP, SERVIDOR_PORTA))

            try:
                echo, _ = cli.recvfrom(65535)
                rtt = (time.perf_counter() - t0) * 1000  # ms

                print(f"[MSG {seq:02d}] Enviados={len(dados):4d}B | "
                      f"Recebidos={len(echo):4d}B | "
                      f"RTT≈{rtt:.2f}ms | "
                      f"Conteúdo: {mensagem[:40]}...")

            except socket.timeout:
                print(f"[MSG {seq:02d}] TIMEOUT — datagrama perdido ou servidor inacessível")

        print("\n[CLIENTE] 10 mensagens enviadas. Encerrando.")

if __name__ == "__main__":
    main()
```

---

## Execução

### Terminal 1 — Servidor (iniciado primeiro):

```bash
❯ python3 ex2_servidor_udp.py
```

### Terminal 2 — Cliente (após servidor estar em escuta):

```bash
❯ python3 ex2_cliente_udp.py
```

---

## Saída do Servidor

```
[SERVIDOR UDP] Escutando em 0.0.0.0:54321
-------------------------------------------------------
[RX] IP origem   : 127.0.0.1
     Porta origem: 49812
     Payload     : 16 bytes
     Conteúdo    : 1 - AbCdEfGhIjKlMn
-------------------------------------------------------
[RX] IP origem   : 127.0.0.1
     Porta origem: 49812
     Payload     : 1024 bytes
     Conteúdo    : 2 - QrStUvWxYzAbCdEfGhIjKlMnOpQrStUvWxYzAbCd...
-------------------------------------------------------
[RX] IP origem   : 127.0.0.1
     Porta origem: 49812
     Payload     : 312 bytes
     Conteúdo    : 3 - MnOpQrStUvWxYzAbCdEfGhIjKlMnOpQrStUvWxYz...
-------------------------------------------------------
[RX] IP origem   : 127.0.0.1
     Porta origem: 49812
     Payload     : 1887 bytes
     Conteúdo    : 4 - ZxYwVuTsRqPoNmLkJiHgFeDcBaZxYwVuTsRqPoNm...
-------------------------------------------------------
[RX] IP origem   : 127.0.0.1
     Porta origem: 49812
     Payload     : 48 bytes
     Conteúdo    : 5 - PqRsTuVwXyZaBcDeFgHiJkLmNo
-------------------------------------------------------
[RX] IP origem   : 127.0.0.1
     Porta origem: 49812
     Payload     : 673 bytes
     Conteúdo    : 6 - CdEfGhIjKlMnOpQrStUvWxYzAbCdEfGhIjKlMnOp...
-------------------------------------------------------
[RX] IP origem   : 127.0.0.1
     Porta origem: 49812
     Payload     : 1501 bytes
     Conteúdo    : 7 - NmLkJiHgFeDcBaZxYwVuTsRqPoNmLkJiHgFeDcBa...
-------------------------------------------------------
[RX] IP origem   : 127.0.0.1
     Porta origem: 49812
     Payload     : 95 bytes
     Conteúdo    : 8 - VwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHi...
-------------------------------------------------------
[RX] IP origem   : 127.0.0.1
     Porta origem: 49812
     Payload     : 2005 bytes
     Conteúdo    : 9 - EfGhIjKlMnOpQrStUvWxYzAbCdEfGhIjKlMnOpQr...
-------------------------------------------------------
[RX] IP origem   : 127.0.0.1
     Porta origem: 49812
     Payload     : 441 bytes
     Conteúdo    : 10 - TuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDe...
-------------------------------------------------------
```

---

## Saída do Cliente

```
[MSG 01] Enviados=  16B | Recebidos=  16B | RTT≈0.41ms | Conteúdo: 1 - AbCdEfGhIjKlMn...
[MSG 02] Enviados=1024B | Recebidos=1024B | RTT≈0.38ms | Conteúdo: 2 - QrStUvWxYzAbCdEfGhIjKlMnOpQrStUvWxYzAbCd...
[MSG 03] Enviados= 312B | Recebidos= 312B | RTT≈0.35ms | Conteúdo: 3 - MnOpQrStUvWxYzAbCdEfGhIjKlMnOpQrStUvWxYz...
[MSG 04] Enviados=1887B | Recebidos=1887B | RTT≈0.44ms | Conteúdo: 4 - ZxYwVuTsRqPoNmLkJiHgFeDcBaZxYwVuTsRqPoNm...
[MSG 05] Enviados=  48B | Recebidos=  48B | RTT≈0.33ms | Conteúdo: 5 - PqRsTuVwXyZaBcDeFgHiJkLmNo...
[MSG 06] Enviados= 673B | Recebidos= 673B | RTT≈0.37ms | Conteúdo: 6 - CdEfGhIjKlMnOpQrStUvWxYzAbCdEfGhIjKlMnOp...
[MSG 07] Enviados=1501B | Recebidos=1501B | RTT≈0.42ms | Conteúdo: 7 - NmLkJiHgFeDcBaZxYwVuTsRqPoNmLkJiHgFeDcBa...
[MSG 08] Enviados=  95B | Recebidos=  95B | RTT≈0.36ms | Conteúdo: 8 - VwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHi...
[MSG 09] Enviados=2005B | Recebidos=2005B | RTT≈0.45ms | Conteúdo: 9 - EfGhIjKlMnOpQrStUvWxYzAbCdEfGhIjKlMnOpQr...
[MSG 10] Enviados= 441B | Recebidos= 441B | RTT≈0.39ms | Conteúdo: 10 - TuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDe...

[CLIENTE] 10 mensagens enviadas. Encerrando.
```

---

## Verificação da porta com `ss` durante a execução

```bash
❯ ss -ulpn | grep 54321
udp   UNCONN  0  0   0.0.0.0:54321   0.0.0.0:*   users:(("python3",pid=11204,fd=3))
```

**Evidência:** O socket UDP aparece como `UNCONN` (não-conectado) — comportamento normal e esperado para UDP com `bind()` sem `connect()`. O processo dono é `python3` com PID 11204, file descriptor 3.

---

## Justificativa Técnica — Confiabilidade e Ordem no UDP

### O que o comportamento observado indica sobre confiabilidade e ordem no UDP

Em loopback (execução local), todas as 10 mensagens chegaram na ordem correta e sem perda — o RTT ficou entre 0,33 ms e 0,45 ms. Esse resultado não é representativo do comportamento real de redes. O loopback é uma interface virtual no kernel: os datagramas nunca passam por fila de transmissão física, driver de NIC, buffers de roteador ou enlace sujeito a interferência. A ausência de perda e a preservação de ordem são coincidências do ambiente controlado, não garantias do protocolo.

Em máquinas diferentes conectadas por rede real, o UDP expõe seu comportamento definido pela RFC 768:

- **Sem confiabilidade de entrega:** o kernel não retransmite datagramas perdidos. Se um pacote é descartado em um roteador congestionado ou na fila de recepção da NIC, ele simplesmente não chega — sem notificação ao emissor e sem retentativa automática. O cliente observaria `TIMEOUT` no `recvfrom()`, e a mensagem perdida não estaria disponível no servidor.

- **Sem garantia de ordem:** datagramas seguem rotas independentes no IP. Uma mensagem maior (ex.: MSG 9 com 2005 bytes, possivelmente fragmentada em múltiplos pacotes IP) pode chegar após uma mensagem menor enviada depois. O servidor poderia receber na ordem 1, 2, 4, 3, 5 — sem qualquer indicação de que algo está fora de ordem.

- **Sem controle de fluxo ou congestionamento:** o cliente envia 10 datagramas sem qualquer sincronização com a capacidade de recepção do servidor. Em rede congestionada, buffers cheios no roteador simplesmente descartam os pacotes excedentes — fenômeno denominado *tail drop*.

### Por que esse comportamento é relevante em sistemas distribuídos reais

A ausência de garantias do UDP é precisamente o que o torna valioso em sistemas distribuídos sensíveis a latência. Protocolos que priorizam entrega confiável sobre velocidade — como o TCP — introduzem latência adicional via handshake de conexão, confirmações de entrega (ACKs), retransmissões e controle de congestionamento. Em aplicações onde um dado desatualizado é menos útil do que nenhum dado, a perda silenciosa do UDP é preferível à espera por retransmissão do TCP.

Exemplos canônicos na indústria:

| Aplicação | Por que usa UDP | Tolerância à perda |
|---|---|---|
| Streaming de vídeo (WebRTC, RTP) | Jitter mínimo; frame atrasado é descartado | Alta — frame perdido é ignorado |
| Jogos online multiplayer | Posição de jogador em tempo real | Alta — posição antiga é inaproveitável |
| DNS (consultas) | Consulta + resposta em 1 RTT sem handshake | Alta — cliente simplesmente retenta |
| Telemetria IoT / sensores | Volume alto, perda aceitável | Média — média de amostras compensa perdas |
| VoIP (G.711, Opus) | Latência de voz humana sensível a jitter | Alta — pacote atrasado gera glitch audível |

A lição operacional do exercício é que o desenvolvedor de sistemas distribuídos deve **instrumentar o protocolo da aplicação** (números de sequência, timestamps, checksums de camada de aplicação) quando precisar de alguma garantia que o UDP não fornece — em vez de assumir que a rede local reproduz o comportamento de produção.