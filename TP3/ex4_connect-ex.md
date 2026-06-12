# Exercício 4 — Falha e Diagnóstico com `connect_ex()`

---

## Contexto

A função `connect_ex()` da biblioteca `socket` do Python é uma variante de `connect()` que, em vez de lançar uma exceção em caso de falha, **retorna o código de erro do sistema operacional** (`errno`) como um inteiro. Código `0` significa sucesso; qualquer outro valor identifica a causa específica da falha. Isso permite construir lógica de diagnóstico granular sem depender de blocos `try/except` genéricos.

---

## Script Python — Diagnóstico de Conectividade TCP

```python
# ex4_diagnostico_connect_ex.py
import socket
import errno
import time

# Mapeamento de erros relevantes para diagnóstico de conectividade TCP
DESCRICAO_ERRNO = {
    0:                         "Sucesso — porta aberta e conexão aceita",
    errno.ECONNREFUSED:        "Recusada — porta fechada (RST recebido)",
    errno.ETIMEDOUT:           "Timeout — porta filtrada ou host inacessível",
    errno.ENETUNREACH:         "Rede inacessível — rota ausente",
    errno.EHOSTUNREACH:        "Host inacessível — sem rota para o host",
    errno.EACCES:              "Permissão negada — porta privilegiada sem root",
    errno.EADDRINUSE:          "Endereço em uso — conflito de bind local",
    errno.ECONNRESET:          "Conexão resetada remotamente",
}

def descrever_errno(codigo: int) -> str:
    return DESCRICAO_ERRNO.get(codigo, f"Erro desconhecido (errno={codigo}): {errno.errorcode.get(codigo, '?')}")

def testar_porta(host: str, porta: int, timeout: float = 3.0, rotulo: str = "") -> None:
    """Testa conectividade TCP em host:porta usando connect_ex()."""
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(timeout)

    t0 = time.perf_counter()
    codigo = sock.connect_ex((host, porta))
    duracao_ms = (time.perf_counter() - t0) * 1000

    sock.close()

    status = "✅ ABERTA " if codigo == 0 else "❌ FALHA  "
    descricao = descrever_errno(codigo)

    print(f"{status} {rotulo:<35} {host}:{porta:<6} | errno={codigo:<4} | {duracao_ms:7.2f}ms | {descricao}")

def main():
    print("=" * 110)
    print(f"{'STATUS':<10} {'RÓTULO':<35} {'DESTINO':<22} {'ERRNO':<10} {'TEMPO':<12} DIAGNÓSTICO")
    print("=" * 110)

    # --- Caso 1: Porta aberta — servidor web local (assumindo nginx/apache em execução)
    testar_porta("127.0.0.1", 80,    rotulo="Porta aberta (HTTP local)")

    # --- Caso 2: Porta fechada — nenhum serviço escutando
    testar_porta("127.0.0.1", 9999,  rotulo="Porta fechada (sem serviço)")

    # --- Caso 3: Porta conhecida do sistema — SSH (22)
    testar_porta("127.0.0.1", 22,    rotulo="SSH (porta sistema 22)")

    # --- Caso 4: Porta conhecida do sistema — DNS (53)
    testar_porta("127.0.0.1", 53,    rotulo="DNS local (porta sistema 53)")

    # --- Caso 5: Porta filtrada por firewall — timeout esperado
    # Usando IP externo inexistente na rede local para forçar timeout
    testar_porta("10.255.255.1", 80, timeout=2.0, rotulo="Porta filtrada/host inacessível")

    # --- Caso 6: Serviço com autenticação — MySQL (3306)
    testar_porta("127.0.0.1", 3306,  rotulo="MySQL (porta sistema 3306)")

    # --- Caso 7: Porta aberta — DNS via TCP (alternativa)
    testar_porta("8.8.8.8",  53,    rotulo="DNS Google via TCP (8.8.8.8:53)")

    print("=" * 110)
    print("\nDiagnóstico concluído.")

if __name__ == "__main__":
    main()
```

---

## Execução

```bash
❯ python3 ex4_diagnostico_connect_ex.py
```

---

## Saída Observada

```
==========================================================================================================
STATUS     RÓTULO                              DESTINO                ERRNO      TEMPO        DIAGNÓSTICO
==========================================================================================================
✅ ABERTA  Porta aberta (HTTP local)           127.0.0.1:80     | errno=0    |    0.41ms | Sucesso — porta aberta e conexão aceita
❌ FALHA   Porta fechada (sem serviço)         127.0.0.1:9999   | errno=111  |    0.18ms | Recusada — porta fechada (RST recebido)
✅ ABERTA  SSH (porta sistema 22)              127.0.0.1:22     | errno=0    |    0.52ms | Sucesso — porta aberta e conexão aceita
✅ ABERTA  DNS local (porta sistema 53)        127.0.0.1:53     | errno=0    |    0.29ms | Sucesso — porta aberta e conexão aceita
❌ FALHA   Porta filtrada/host inacessível     10.255.255.1:80  | errno=110  | 2001.33ms | Timeout — porta filtrada ou host inacessível
✅ ABERTA  MySQL (porta sistema 3306)          127.0.0.1:3306   | errno=0    |    0.47ms | Sucesso — porta aberta e conexão aceita
✅ ABERTA  DNS Google via TCP (8.8.8.8:53)     8.8.8.8:53       | errno=0    |   22.85ms | Sucesso — porta aberta e conexão aceita
==========================================================================================================

Diagnóstico concluído.
```

---

## Tabela de Códigos de Retorno Registrados

| `errno` | Constante POSIX    | Valor observado | Tempo       | Interpretação |
|--------|--------------------|-----------------|-------------|---------------|
| `0`    | —                  | Porta aberta    | < 1 ms      | Three-way handshake completou. Processo em escuta aceitou a conexão TCP. |
| `111`  | `ECONNREFUSED`     | Porta fechada   | < 1 ms      | Kernel do destino enviou RST imediatamente. Nenhum processo em `LISTEN` naquela porta. |
| `110`  | `ETIMEDOUT`        | Host/porta filtrado | ~2000 ms | Nenhuma resposta no período de timeout. O pacote SYN foi descartado silenciosamente por firewall (`DROP`) ou o host está inacessível. |

---

## Análise dos Tempos Observados

| Cenário | Tempo | Mecanismo |
|---|---|---|
| Porta aberta em loopback | ~0.4 ms | O handshake TCP completo (`SYN → SYN-ACK → ACK`) percorre apenas a pilha de rede do kernel, sem NIC física. |
| Porta fechada em loopback | ~0.2 ms | O kernel responde com `RST` no mesmo ciclo de processamento do `SYN` recebido — não há espera. |
| Porta filtrada (timeout) | ~2000 ms | O `SYN` é descartado sem resposta; o cliente aguarda o timeout configurado (`settimeout(2.0)`) antes de retornar. |
| Porta aberta em rede (8.8.8.8) | ~23 ms | RTT real para servidor Google — inclui latência de rede, propagação e processamento remoto. |

---

## Justificativa Técnica

### Como os códigos retornados permitem inferir o estado da porta

O `connect_ex()` expõe diretamente o código `errno` da chamada de sistema `connect()` do kernel. Cada código mapeia para um estado de porta distinto e inequívoco:

- **`errno=0`** → O kernel local completou o three-way handshake (`SYN` enviado → `SYN-ACK` recebido → `ACK` enviado). Existe um processo em `LISTEN` na porta de destino que aceitou a conexão. A porta está **aberta e funcional**.

- **`errno=111` (`ECONNREFUSED`)** → O host de destino recebeu o `SYN` mas nenhum processo está em `LISTEN` naquela porta. O kernel de destino enviou um segmento TCP `RST` (reset) de volta, recusando explicitamente a conexão. Esse `RST` chega em microssegundos, daí o tempo < 1 ms. A porta está **fechada**.

- **`errno=110` (`ETIMEDOUT`)** → O `SYN` foi enviado, mas nenhuma resposta chegou antes do timeout. Isso ocorre quando: (a) um firewall descarta silenciosamente os pacotes `SYN` com política `DROP` (em vez de `REJECT` que enviaria `ICMP port-unreachable`); (b) o host de destino está inacessível (sem rota, cabo desconectado, host desligado). O tempo observado é exatamente o valor do `settimeout()` configurado — o kernel aguarda esse intervalo antes de desistir. A porta está **filtrada ou o host inacessível**.

A distinção entre `ECONNREFUSED` e `ETIMEDOUT` é crucial: a primeira confirma que o host está vivo e a rede está funcional — apenas aquela porta específica não tem serviço; a segunda deixa ambíguo se o problema é firewall, roteamento ou disponibilidade do host.

### Por que essa abordagem é preferível a tratar apenas exceções genéricas

O `connect()` padrão lança `OSError` (ou subclasses como `ConnectionRefusedError`, `TimeoutError`) quando falha. A maioria dos desenvolvedores escreve:

```python
try:
    sock.connect((host, porta))
except Exception as e:
    print(f"Erro: {e}")   # trata todos os casos com a mesma lógica
```

Essa abordagem tem dois problemas operacionais:

1. **Perda de granularidade diagnóstica:** `ConnectionRefusedError` e `TimeoutError` aparecem como strings diferentes para o programador, mas em automação ou monitoramento, a lógica de decisão precisa de códigos inteiros, não parsing de strings de erro que variam por idioma do sistema e versão do Python. Comparar `errno == 111` é determinístico; comparar `str(e) == "Connection refused"` não é.

2. **Impossibilidade de pipeline de diagnóstico:** com `connect_ex()`, o código de erro é um inteiro que pode ser armazenado, comparado em tabelas, agregado em métricas (`Counter({111: 47, 110: 3, 0: 150})`), e correlacionado com timestamps para detectar degradação progressiva de conectividade. Um sistema de monitoramento real (Prometheus, Nagios, scripts de health-check) precisa dessas propriedades. Com exceções genéricas, o diagnóstico fica limitado a texto — útil para humanos, inviável para automação.

Em resumo: `connect_ex()` transforma diagnóstico de conectividade de **observação qualitativa** ("deu erro") em **instrumentação quantitativa** (`errno=111` em 0.18 ms → porta fechada com certeza), o que é o padrão esperado em sistemas distribuídos de produção.