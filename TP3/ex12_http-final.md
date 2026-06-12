# Exercício 12 — Integração Final: Serviço HTTP Consolidado

---

## Código — Servidor HTTP Mínimo

```python
# ex12_servidor_http.py
import socket
from datetime import datetime

HOST = "0.0.0.0"
PORTA = 50120

def construir_resposta(status: str, corpo: str) -> bytes:
    corpo_bytes = corpo.encode("utf-8")
    headers = (
        f"HTTP/1.1 {status}\r\n"
        f"Content-Type: text/plain; charset=UTF-8\r\n"
        f"Content-Length: {len(corpo_bytes)}\r\n"
        f"Connection: close\r\n"
        f"\r\n"
    )
    return headers.encode("utf-8") + corpo_bytes

def parse_request_line(raw: bytes):
    try:
        primeira_linha = raw.split(b"\r\n")[0].decode("utf-8")
        partes = primeira_linha.split(" ")
        return partes[0], partes[1]
    except Exception:
        return "", ""

def tratar_conexao(conn, addr):
    ip, porta = addr
    with conn:
        raw = conn.recv(4096)
        if not raw:
            return

        metodo, caminho = parse_request_line(raw)
        agora = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        print(f"[{agora}] {ip}:{porta} -> {metodo} {caminho}")

        if caminho == "/":
            resposta = construir_resposta("200 OK", "OK")
        elif caminho == "/admin":
            resposta = construir_resposta("403 Forbidden", "Forbidden")
        else:
            resposta = construir_resposta("404 Not Found", "Not Found")

        conn.sendall(resposta)
        print(f"[{agora}] Resposta enviada: {resposta.split(b'\\r\\n')[0].decode()}")

def main():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
        srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        srv.bind((HOST, PORTA))
        srv.listen(5)
        print(f"[SERVIDOR HTTP] Escutando em {HOST}:{PORTA}")

        while True:
            conn, addr = srv.accept()
            tratar_conexao(conn, addr)

if __name__ == "__main__":
    main()
```

---

## Execução

```bash
❯ python3 ex12_servidor_http.py &

❯ curl -s -o /dev/null -w "GET / -> %{http_code}\n" http://127.0.0.1:50120/
❯ curl -s -o /dev/null -w "GET /admin -> %{http_code}\n" http://127.0.0.1:50120/admin
❯ curl -s -o /dev/null -w "GET /qualquer -> %{http_code}\n" http://127.0.0.1:50120/qualquer
```

---

## Saída Observada — `curl`

```
GET / -> 200
GET /admin -> 403
GET /qualquer -> 404
```

## Saída Observada — Servidor

```
[SERVIDOR HTTP] Escutando em 0.0.0.0:50120
[2026-06-12 01:04:25] 127.0.0.1:36196 -> GET /
[2026-06-12 01:04:25] Resposta enviada: HTTP/1.1 200 OK
[2026-06-12 01:04:25] 127.0.0.1:36212 -> GET /admin
[2026-06-12 01:04:25] Resposta enviada: HTTP/1.1 403 Forbidden
[2026-06-12 01:04:25] 127.0.0.1:36224 -> GET /qualquer
[2026-06-12 01:04:25] Resposta enviada: HTTP/1.1 404 Not Found
```

---

## Tabela Resumida de Rotas

| Rota | Status retornado | Corpo |
|---|---|---|
| `/` | `200 OK` | `OK` |
| `/admin` | `403 Forbidden` | `Forbidden` |
| qualquer outra (`/qualquer`, `/teste`, etc.) | `404 Not Found` | `Not Found` |

---

## Análise Técnica — O Servidor Poderia Tratar Qualquer Rota Possível?

**Não, não na forma atual.** A implementação cobre apenas três casos: a rota raiz `/`, a rota `/admin` e um *fallback* genérico para "qualquer outra rota" — mas esse *fallback* é tratado de forma **uniforme**, sem distinção real entre as infinitas rotas possíveis que um cliente pode enviar. Pontos relevantes:

- **Cobertura sintática vs. semântica:** o servidor *sintaticamente* trata qualquer string de caminho (`caminho`), pois o `else` captura tudo que não seja `/` ou `/admin` e retorna `404`. Nesse sentido restrito, sim, "qualquer rota" recebe uma resposta válida. Porém, *semanticamente*, o servidor não conhece nem implementa lógica para rotas além das duas explicitamente mapeadas — não há roteamento dinâmico, parâmetros de path (`/usuarios/<id>`), query strings, ou métodos HTTP além de `GET` sendo tratados com lógica própria.
- **Ausência de tratamento de métodos HTTP:** o `parse_request_line` extrai o método (`GET`, `POST`, `PUT`, etc.), mas o servidor não verifica esse valor — uma requisição `POST /` recebe a mesma resposta `200 OK` que uma `GET /`, o que viola a semântica HTTP (idealmente retornaria `405 Method Not Allowed` para métodos não suportados em uma rota).
- **Requisições malformadas:** o `parse_request_line` tem um bloco `try/except` que retorna `("", "")` em caso de falha de parsing, e nesse caso o caminho vazio cairia no `else` (`404`). Isso evita que o servidor quebre com entradas malformadas, mas uma requisição verdadeiramente corrompida ou um payload não-HTTP enviado à porta TCP poderia ainda assim causar comportamento inesperado, já que o `recv(4096)` tem tamanho fixo e requisições maiores (ex.: com headers grandes ou corpo de `POST`) seriam truncadas.
- **Concorrência:** o `main()` processa uma conexão por vez (`accept()` seguido de `tratar_conexao()` de forma síncrona/bloqueante). Um cliente lento ou uma rota que demandasse processamento custoso bloquearia o atendimento de todos os outros clientes — o servidor não escala para tratar múltiplas rotas/requisições simultâneas sem threads, processos ou um event loop assíncrono.

**Conclusão:** o servidor trata *qualquer rota* apenas no sentido de que nenhuma entrada faz o processo travar ou lançar exceção não tratada — todo caminho recebe uma resposta HTTP válida (200, 403 ou 404). No entanto, ele **não implementa roteamento genérico**: não há mapeamento dinâmico de rotas, tratamento por método HTTP, suporte a parâmetros, nem concorrência. Para um "serviço HTTP consolidado" de verdade, seria necessário um roteador de fato (tabela de rotas com padrões/regex, despacho por método, e um modelo de concorrência como `threading` ou `asyncio`), o que está fora do escopo deste servidor mínimo baseado em socket puro.