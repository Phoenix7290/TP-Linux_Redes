# Exercício 10 — Varredura TCP Controlada em Python

---

## Código — Scanner TCP Síncrono

```python
# ex10_scanner_tcp.py
import socket
import time

HOST = "127.0.0.1"
PORTA_INICIO = 20
PORTA_FIM = 1024

def main():
    print(f"[SCANNER] Iniciando varredura em {HOST}, portas {PORTA_INICIO}-{PORTA_FIM}")
    inicio = time.perf_counter()
    abertas = []

    for porta in range(PORTA_INICIO, PORTA_FIM + 1):
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(0.2)
        codigo = sock.connect_ex((HOST, porta))
        if codigo == 0:
            abertas.append(porta)
            print(f"[ABERTA] {HOST}:{porta}")
        sock.close()

    duracao = time.perf_counter() - inicio
    print("-" * 50)
    print(f"[RESULTADO] Portas abertas: {abertas if abertas else 'nenhuma'}")
    print(f"[RESULTADO] Total de portas varridas: {PORTA_FIM - PORTA_INICIO + 1}")
    print(f"[RESULTADO] Tempo total: {duracao:.3f}s")

if __name__ == "__main__":
    main()
```

---

## Execução

```bash
❯ python3 ex10_scanner_tcp.py
```

---

## Saída Observada

```
[SCANNER] Iniciando varredura em 127.0.0.1, portas 20-1024
[ABERTA] 127.0.0.1:999
--------------------------------------------------
[RESULTADO] Portas abertas: [999]
[RESULTADO] Total de portas varridas: 1005
[RESULTADO] Tempo total: 0.047s
```

> **Nota:** a porta `999` foi propositalmente ocupada por um socket de teste em `127.0.0.1` durante a execução, para demonstrar a detecção de uma porta aberta pelo scanner.

---

## Tabela Resumida

| Parâmetro | Valor |
|---|---|
| Host | `127.0.0.1` |
| Intervalo de portas | 20–1024 (1005 portas) |
| Timeout por porta | 0.2 s |
| Portas abertas detectadas | `[999]` |
| Tempo total | ≈ 0.047 s |

---

## Análise Técnica

### Importância de limites de intervalo e timeout

- **Intervalo limitado (`20–1024`):** restringir a varredura a um intervalo pequeno e bem definido evita que o scanner percorra todo o espaço de 65535 portas, reduzindo drasticamente o tempo de execução e a carga gerada sobre o host de destino. Um intervalo pequeno também torna a varredura **previsível e auditável** — fica claro qual o escopo testado e por quê (no caso, portas conhecidas de serviços do sistema, faixa 20–1024 = portas "privilegiadas").
- **Timeout (`settimeout(0.2)`):** sem um timeout explícito, `connect_ex()` usaria o timeout padrão do sistema operacional para TCP (que pode chegar a vários segundos por porta filtrada). Em uma varredura de mil portas, isso poderia levar a execução de minutos para horas caso parte das portas esteja em estado *filtered* (sem resposta). O timeout curto (0.2 s) garante que portas sem resposta (RST nem SYN-ACK) não bloqueiem a varredura por tempo desproporcional, mantendo o tempo total em uma escala de milissegundos como observado (≈47 ms).

### Riscos de varreduras amplas e não controladas

- **Impacto em sistemas de detecção (IDS/IPS):** varreduras de grande volume de portas em curto intervalo de tempo são uma assinatura clássica de reconhecimento de rede (*port scanning*) e tipicamente disparam alertas de segurança, podendo levar ao bloqueio do IP de origem por firewalls ou `fail2ban`.
- **Carga sobre o host de destino:** abrir e fechar milhares de conexões TCP em sequência consome descritores de arquivo, sockets em estado `TIME_WAIT` e ciclos de CPU tanto no scanner quanto no destino, podendo degradar a performance de serviços legítimos.
- **Implicações legais e éticas:** varrer portas de hosts/redes sem autorização explícita pode configurar tentativa de acesso não autorizado sob legislações como a Lei de Crimes Cibernéticos (Lei Carolina Dieckmann) e o Marco Civil da Internet no Brasil, além de políticas de uso aceitável de provedores e instituições. Por isso, a varredura deste exercício foi limitada estritamente ao **próprio host (`127.0.0.1`)**, eliminando qualquer risco de afetar terceiros.
- **Falsos positivos/negativos com timeouts curtos:** timeouts muito agressivos podem classificar erroneamente portas abertas como fechadas em redes com latência variável, reforçando a importância de calibrar o timeout conforme o contexto (rede local vs. remota).